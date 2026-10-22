# HANDOFF — csi-ppt-master (ย้ายจาก cloud session ไปเครื่อง local)

อัปเดต 2026-10-22 · branch `claude/csi-ppt-master` · repo `rojios/agent-os`

## สถานะ

| ช่วง | สถานะ |
|---|---|
| 0 — ตรวจของที่มี | ✅ เสร็จ |
| 1 — ทดสอบ PPT Master กับภาษาไทย | ✅ ผ่าน (ดู `tests/phase1/REPORT-phase1.md`) |
| 2 — สร้างทักษะ | ⏳ รอ Boss อนุมัติ 2 ข้อด้านล่าง |
| 3 — ทดสอบทั้งท่อ | ยังไม่เริ่ม |

## การตัดสินใจที่ Boss อนุมัติแล้ว

1. แยกเป็น 4 ทักษะ: proposal-master-lean / **csi-ppt-master** (ตัวประสาน) / ppt-master / proposal-master
2. csi-ppt-master ถือโครง 13 หน้าเอง (`references/page-section-map.md`) ไม่เรียกตรรกะ Step 2-4 ของ lean
3. เก็บทักษะใน repo นี้ที่ `rjs-skills/csi-ppt-master/`
4. ใช้ ppt-master โหมด Quick คู่กับสไตล์ `solution-proposal`
5. ห้ามแก้ proposal-master, proposal-master-lean และโค้ดของ PPT Master

## รออนุมัติจาก Boss ก่อนเริ่มช่วงที่ 2

1. ฟอนต์หลักใน brand.yaml: Sarabun หรือ Leelawadee UI (Leelawadee UI มีติดเครื่อง Windows)
2. เพิ่มกติกาเว้นระยะเหนือหัวสไลด์ไทย (baseline ห่างอย่างน้อย 1.5 เท่าของขนาดหัวสไลด์) ใช่ไหม
3. ใช้ไฟล์ `Digital_AI_Transformation_Cus.pptx` (OneDrive ของ Boss) เป็นแหล่งข้อมูลแบรนด์ CSI ใช่ไหม
   ถ้าใช่ ให้ดึงเฉพาะสี ฟอนต์ โลโก้ และโครงหน้า ห้ามใช้เนื้อหาลูกค้า

## สิ่งที่ต้องติดตั้งบนเครื่อง local (Mac)

```bash
# 1. PPT Master (ห้ามแก้โค้ด)
git clone https://github.com/hugohe3/ppt-master.git ~/tools/ppt-master
cp -r ~/tools/ppt-master/skills/ppt-master ~/.claude/skills/ppt-master
python3 -m venv ~/tools/ppt-master/.venv
~/tools/ppt-master/.venv/bin/pip install -r ~/tools/ppt-master/requirements.txt

# 2. LibreOffice (แปลง pptx เป็น png) + poppler (pdftoppm)
brew install --cask libreoffice && brew install poppler

# 3. ฟอนต์ไทยสำหรับทดสอบ: Sarabun, Noto Sans Thai, IBM Plex Sans Thai (จาก Google Fonts)
```

ตรวจว่าทักษะ proposal-master และ proposal-master-lean อยู่ใน `~/.claude/skills/` ของเครื่อง local แล้ว
ถ้าไม่มี ให้วางไว้ที่ `~/.claude/skills/<ชื่อ>/SKILL.md`
ค่า checksum เดิมจาก cloud (sha256):
- proposal-master: `f11be8ef5d72a6cbe2518ef61fc9ab76dc4dbeeabe58e0818f03454784864e33`
- proposal-master-lean: `28311cdc890ed7a9e627d4a8f8984c2a303fceef8aa48fed819f33d30b05c1fc`

## ข้อสังเกตจากช่วงที่ 1 ที่ต้องใช้ต่อ

- ภาพ PNG จาก LibreOffice ทำให้ข้อความเลื่อน (`spAutoFit` + `wrap="none"`) ส่วนไฟล์จริงถูกต้อง
  ให้เปลี่ยน `spAutoFit` เป็น `noAutofit` เฉพาะในสำเนาที่ใช้ดูภาพ
- PPT Master ใส่ฟอนต์ไทยในช่อง `cs` ซึ่งใช้ฟอนต์เดียวกับช่อง latin ใน stack ส่วนช่อง `ea` = Microsoft YaHei ไม่กระทบภาษาไทย
- กล่องข้อความไม่ตัดบรรทัดเอง ตัวเลือก `--reflow-text` ยังไม่ได้ทดสอบ
- ส่งออกด้วย `svg_to_pptx.py <project> --quick-generate --with-notes --native-charts-and-tables`
- ตรวจไฟล์ด้วย `python3 scripts/verify_pptx.py <deck.pptx> --expect-font <ฟอนต์>`
