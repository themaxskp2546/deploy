# MUSYA — ทำให้ production เหมือน local (18 ต.ค. 2569)

ชุดนี้ = **ฐานข้อมูล + MinIO + โค้ด ของเครื่อง local ณ 18 ต.ค. 2569** ที่ผ่านสคริปต์สาธิต "MUSYA Demo 10 นาที" ครบ 7 ขั้น
แทนชุด `_deploy_20261010` (ชุดเดิมยังใช้อ้างอิงได้ แต่ไม่ต้องใช้แล้ว)

| ไฟล์ | คืออะไร |
|---|---|
| `db/musyadata_prod_20261018.dump` | **ฐานที่ใช้ขึ้น server** = สำเนา `musyadata` ของ local ตัด schema `merge_archive` + บัญชีทดสอบ `qa-e2e@dev.local` และแถวที่ผูกกับบัญชีนี้ (แชท 327 · รายงาน 17 · feedback 45) · sha256 `eb3cd393…e0fb2fe6` |
| `minio/fileapa/` + manifest | CSV สถิติ/HDC 485 ไฟล์ · 641.5 MB (ทั้ง bucket ของ local) |
| `minio/pdf-library/` + manifest | PDF คลังรายงาน 162 ไฟล์ · 2,378.8 MB (ทั้ง bucket ของ local) |
| `minio/submissions/` + manifest | 2 ไฟล์ทดสอบ dev — **ไม่ต้องนำเข้า** |
| `tools/prep_prod_from_local.sql` | สคริปต์ที่ใช้ตัดของ dev ออกจากสำเนา (ห้ามรันกับ `musyadata` — สคริปต์กันไว้) |
| `tools/minio_export.py` | export / **import แบบเพิ่มอย่างเดียว** (ไม่ลบ · ข้ามไฟล์ที่ etag ตรงกัน) |

โค้ด **Beta V2.3.1**: `chatappandpython` branch `mid` ถึง `a12de9e` · `chatapi.python` branch `main` ถึง `5f4a7e2`
push แล้ว 18 ต.ค. 2569 (`babankbro/musya_fronted_v2` mid · `babankbro/musya_backend_v2` main) — server `git pull` ได้ของชุดนี้

## ตรวจแล้วบน local (18 ต.ค.)

- restore ดัมพ์ลงฐานเปล่าผ่าน · โครงสร้าง public ตรงกับ local ทุกบรรทัด · migration ถึง 048 (schema freeze 053 ไม่แตะ)
- accounts **490** (ไม่มี `@dev.local`) · chat_sessions **485** · journal_reports **208** · csv_data_dict **345** · hdc_import **301** · prompt_templates **21**
- tmp001–100 หมดอายุ **31 ต.ค. 2569** ครบ 100 · thaihealth01–05 รหัสตรงการ์ด pdf-final (รวม fix 18 ต.ค. แล้ว)
- MinIO 649 ไฟล์ ขนาดตรง manifest ทุกไฟล์
- โค้ด python ในคอนเทนเนอร์ `chatapp-python-ai` ตรงกับ `main` HEAD ทุกไฟล์ (132 ไฟล์)
- อิมเมจ frontend ของ local build 13 ต.ค. 18:47 น. — **ก่อน** `e83f9fb` (แก้ B1/B2 ส่ง doc_type=project) ⇒ build บน server จาก HEAD จะได้ fix นี้เพิ่ม

## ทำไม server กับ local ตอบไม่เหมือนกัน

หน้าเว็บ (JS bundle) ของ server กับ local **เหมือนกันทุก byte** (เช็ก 18 ต.ค.) ⇒ ต่างกันที่ backend/ฐานข้อมูล/MinIO/env
ข้อมูลล่าสุดของ server ที่มีในเครื่อง (`_db_backups/server_musyadata_20261010.dump`) เทียบกับ local:

| | server 10 ต.ค. | local / ชุดนี้ |
|---|---|---|
| migration_history | 17 แถว | 23 (ถึง 048) |
| csv_data_dict (ชุดข้อมูลที่ "ข้อมูลสถิติ" ค้นได้) | 317 | **345** (+28) |
| hdc_import | 288 | 301 |
| ตาราง prompt_templates / answer_feedback / pdpa_screen_log / data_submissions / support_requests | **ไม่มี** | มี |

ถ้า server ยังไม่ได้ restore ดัมพ์ merge (`_deploy_20261010`) — ชุดข้อมูลสถิติ 28 ชุด, migration 042–048 และธงใหม่จะไม่มี ⇒ คำตอบต่างจาก local
ตรวจบน server ด้วยคำสั่งนี้ (อ่านอย่างเดียว):

```bash
docker exec chatapp-postgres psql -U postgres -d musyadata -c "
select (select max(migration_id) from migration_history) mig,
       (select count(*) from csv_data_dict) data_dict,
       (select count(*) from hdc_import) hdc,
       to_regclass('public.prompt_templates') is not null has_prompt_templates"
docker exec chatapp-frontend sh -c 'env | grep -E \"_ENABLED=|_FROM_DB=\" | sort'
docker exec chatapp-python-ai sh -c 'env | grep -E \"ANSWER_RULES|GEMINI_MODEL\"; env | grep -c ANTHROPIC_API_KEY='
```

ค่าที่ local ใช้ (ต้องได้เท่านี้): mig **48** · data_dict **345** · hdc **301** · has_prompt_templates **t** · ธงตามตาราง env ด้านล่าง

## ผลเทียบ production กับ local (18 ต.ค. 2569 · demo7_console.js)

| ขั้น | production | local | สาเหตุ |
|---|---|---|---|
| 1 แนะนำตัว | ❌ ตอบแบบผู้ช่วยสุขภาพทั่วไป (Gemini) ไม่รู้จัก MUSYA/5 เครื่องมือ | ✅ | backend ไม่มี `f818a01` (29 ก.ย.) |
| 2 อุบัติเหตุราษีไศล | ✅ 8 · 5 · 3 | ✅ | — |
| 3 เบาหวาน 2 อำเภอ | ❌ "ไม่พบข้อมูล" | ✅ 55.93% vs 26.43% | backend ไม่มีชุดแก้สถิติ 27–30 ก.ย. (W4 File Finder · HDC formulas) + ฐานไม่มีคอลัมน์ `rate_scale` `hdc_formulas` (migration 051/052) |
| 4 ตรวจราชการ | ⚠️ 9 ประเด็นครบ แต่ตารางซ้ำยาวหลายร้อยแถว ปีปน (2550–2569) และใส่ CTAM ไซเบอร์ใต้หัวข้อมะเร็ง | ✅ | backend ไม่มี `7c8ff81` (ห้ามปนตัวเลขต่างปี) |
| 5 งานวิจัย | ✅ ThaiJo 5 + PubMed 4 | ✅ | — |
| 6 Work Plan | ✅ ชื่อเอกสาร + 7 หัวข้อ PDCA | ✅ 8 หัวข้อ | — |
| 7 อินโฟกราฟิก | ✅ 8 · 5 · 3 (claude-haiku-5-5) | ✅ | — |

สรุป: frontend ของ server เป็น V2.3 แล้ว แต่ **backend python-ai และฐานข้อมูลยังเป็นรุ่นก่อน 29 ก.ย.** ⇒ ต้อง build `python-ai` ใหม่จาก `main` + restore ฐานชุดนี้ ไม่ใช่แค่ frontend

## ขั้นตอนขึ้น server

```bash
# 0) ปิดรับผู้ใช้ชั่วคราว
docker stop chatapp-frontend

# 1) ⚠️ ข้อมูลที่ผู้ใช้สร้างบน server หลัง 10 ต.ค. 12:48 UTC "ไม่มี" ในดัมพ์ชุดนี้ — restore แล้วจะหาย
#    ต้องได้ 0 ทุกค่า ถ้าไม่ใช่ 0: หยุด · pg_dump ของ server มา merge ใหม่ก่อน (ดู _deploy_20261010/tools/merge.sql)
docker exec chatapp-postgres psql -U postgres -d musyadata -c "
select (select count(*) from chat_sessions where updated_at > '2026-10-10 12:48:29') chats,
       (select count(*) from journal_reports where created_at > '2026-10-10 12:48:29') reports,
       (select count(*) from accounts where updated_at > '2026-10-10 12:48:29') accounts,
       (select count(*) from llm_role_usage where created_at > '2026-10-10 12:48:29') usage"

# 2) สำรองของเดิมบน server (ฐาน + MinIO)
TS=$(date +%Y%m%d_%H%M%S)
docker exec chatapp-postgres pg_dump -U postgres -Fc -d musyadata -f /tmp/pre_prod18_$TS.dump
docker cp chatapp-postgres:/tmp/pre_prod18_$TS.dump ./_db_backups/
tar -czf ./_minio_backups/minio_pre_prod18_$TS.tar.gz -C ../data-minio data

# 3) โค้ด (หลัง push จากเครื่อง dev แล้ว) แล้ว build ใหม่ทั้งสองตัว
git -C chatappandpython pull && git -C chatapi.python pull
cd chatappandpython && docker compose build python-ai frontend

# 4) กู้ฐานชุดนี้
docker stop chatapp-python-ai
docker cp ../_deploy_20261018/db/musyadata_prod_20261018.dump chatapp-postgres:/tmp/prod.dump
docker exec chatapp-postgres sh -c "dropdb -U postgres musyadata && createdb -U postgres musyadata && pg_restore -U postgres -d musyadata --no-owner --no-privileges /tmp/prod.dump"

# 5) MinIO แบบเพิ่มอย่างเดียว (ไม่ลบของเดิม) — fileapa และ pdf-library
for b in fileapa pdf-library; do
  docker run --rm --network chatappandpython_default -e MINIO_ENDPOINT=minio:9000 \
    -e MINIO_ACCESS_KEY=<root user> -e MINIO_SECRET_KEY=<root password> \
    -v "$PWD/../_deploy_20261018:/out" chatappandpython-python-ai \
    python /out/tools/minio_export.py import $b /out/minio
done

# 6) ตั้ง env ตามตารางด้านล่าง แล้วเปิดระบบ
docker compose up -d
```

### env ให้ตรงกับ local

| ที่ | ตัวแปร | ค่าบน local |
|---|---|---|
| frontend (`chatappandpython/.env.local`) | `PDPA_SCREEN_ENABLED` `DATA_SUBMISSIONS_ENABLED` `ONEPAGE_ENABLED` `REAL_DOCX_ENABLED` `PROMPT_TEMPLATES_FROM_DB` `ANSWER_FEEDBACK_ENABLED` `INFOGRAPHIC_ENABLED` `INFOGRAPHIC_EDIT_ENABLED` `REPORT_EDIT_ENABLED` `OBSIDIAN_ENABLED` | `true` ทุกตัว (2 ตัวท้ายตารางเดิมไม่มี: `INFOGRAPHIC_EDIT_ENABLED` `REPORT_EDIT_ENABLED`) |
| python-ai | `ANSWER_RULES_V2` | `1` |
| python-ai | `GEMINI_MODEL` | `gemini-2.5-flash-lite` |
| python-ai | `OBSIDIAN_ENABLED` | `true` |
| python-ai | `ANTHROPIC_API_KEY` `GEMINI_API_KEY` `OPENAI_API_KEY` `TAVILY_API_KEY` `INTERNAL_API_KEY` | ต้องตั้งครบทั้ง 5 (อินโฟกราฟิกใช้ Claude haiku-5-5 · ข้อมูลจากเว็บใช้ Tavily) |

## ตรวจหลังขึ้น — ใช้สคริปต์ "MUSYA Demo 10 นาที"

ผลบน local 18 ต.ค. (บัญชีทดสอบ) — server ต้องได้ตัวเลขเดียวกัน:

| ขั้น | เครื่องมือ | prompt (ย่อ) | ผลบน local | เวลา |
|---|---|---|---|---|
| 1 | ไม่เลือก | เว็บนี้คือเครื่องมืออะไร | MUSYA เขต 10 · 5 จังหวัด · 5 เครื่องมือ | 29 วิ |
| 2 | ข้อมูลสถิติ | อุบัติเหตุราษีไศล 2567 | **8 ครั้ง · ตาย 5 · สาหัส 3** + ตาราง + ข้อจำกัดข้อมูล | 9 วิ |
| 3 | ข้อมูลสถิติ | เบาหวานคุมได้ เสนางคนิคม vs เมืองอำนาจเจริญ 2568 | **55.93% vs 26.43%** · 1,594/2,850 · 2,298/8,696 | 18 วิ |
| 4 | คลังรายงาน | ตรวจราชการศรีสะเกษ 2567 | **9 ประเด็น** (พระราชดำริฯ · สุขภาพจิต/ยาเสพติด · มะเร็ง · ลดแออัด …) | 20 วิ |
| 5 | ฐานข้อมูลวิจัย | ลดภาวะอ้วนเด็กวัยเรียน | **ThaiJo 5 + PubMed 4** มีลิงก์ทุกเรื่อง | 10 วิ |
| 6 | แผนหรือนโยบาย → Work Plan | พยาธิใบไม้ตับ ราษีไศล 2570 | ชื่อเอกสาร + 8 หัวข้อ (PDCA) — ทดสอบเฉพาะขั้นเสนอหัวข้อ | 3 วิ |
| 7 | อินโฟกราฟิก | จากแชทขั้น 2 | การ์ด 8 · 5 · 3 · claude-haiku-5-5 | 10 วิ |

เพิ่มเติม: `select count(*) from accounts` = 490 · tmp หมดอายุ 31 ต.ค. = 100 · `thaihealth04` ล็อกอินด้วยรหัสบนการ์ดได้
ถ้าพัง: หยุด container แล้ว restore `pre_prod18_$TS.dump` + แตก `minio_pre_prod18_$TS.tar.gz` กลับ
