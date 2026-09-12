# เปรียบเทียบสถาปัตยกรรม (Architecture) และ Tech Stack: Agent-OS (RJS AICF) vs The Curator

> จัดทำโดยวิเคราะห์โค้ดจริงในสองรีโป: `Rojios/Agent-OS` (branch นี้) และ `talirezun/the-curator` (public repo, clone แบบ read-only)
> Confidence รวม: **สูง** สำหรับข้อเท็จจริงเชิงโครงสร้าง (อ่านจากโค้ด/ไฟล์จริง) | **กลาง** สำหรับข้อสรุปเชิงกลยุทธ์/ความเห็น (ระบุแยกไว้ชัดเจน)

## สรุปผู้บริหาร (Executive Summary)

1. **คนละ category ของซอฟต์แวร์โดยพื้นฐาน** — Agent-OS เป็น **methodology layer** (ชุดไฟล์ Markdown + Bash script ที่ inject เข้า context ของ AI agent) ไม่มี runtime ของตัวเอง ส่วน The Curator เป็น **แอปพลิเคชันที่รันจริง** (Node.js server + Electron desktop app + MCP server) ที่มี state และ storage เป็นของตัวเอง
2. **จุดที่แข่งกันโดยตรงกับของ Boss คือ "Agent Memory"** — The Curator มี working-state store (3-layer: personal wiki / shared brain / agent working-state) ที่ทำหน้าที่เดียวกับระบบ memory-routing (L1–L4) ที่ Boss ใช้อยู่ แต่ **design philosophy ตรงข้ามกัน**: The Curator เลี่ยง vector database/embedding โดยเจตนา ("compiling wiki, not RAG") ขณะที่ L4 ของ Boss ใช้ Supabase pgvector เป็นแกนหลัก — จุดนี้เป็น **ความเห็น ระดับ confidence กลาง**
3. Agent-OS ไม่มี lock-in และไม่มี attack surface (ไม่มี server, ไม่มี API key, ไม่มี network call) เพราะไม่ใช่แอป — The Curator มี attack surface จริง (Express server, LLM API key, GitHub PAT สำหรับ Shared Brain) แต่ก็ออกแบบมาให้ vendor-neutral ในระดับ data (plain markdown, MCP protocol)

---

## 1. ภาพรวมสถาปัตยกรรม (Architecture Overview)

| มิติ | Agent-OS | The Curator |
|---|---|---|
| ประเภทระบบ | Prompt/methodology framework (ไม่มี runtime) | แอปพลิเคชันที่รันจริง (local web app + desktop app) |
| Entry point | ไม่มี — ถูก "โหลด" เข้า context ของ AI agent ผ่าน slash-command | `src/server.js` (Express, พอร์ต 3333) และ `mcp/server.js` (stdio) |
| การเก็บ state | ไม่มี state ส่วนกลาง — เขียนไฟล์ spec/standard ลงตรงใน target project repo เท่านั้น | มี state จริง: `domains/<domain>/{raw,wiki,conversations}` + working-state (JSONL) |
| Database | ไม่มี | ไม่มี (ตั้งใจ — ใช้ plain Markdown แทน vector DB/embedding) |
| การเรียก LLM | ไม่เรียกเอง — อาศัย host agent (Claude Code/Cursor) เป็นคนรัน inference | เรียก LLM เอง ผ่าน provider abstraction (`brain/llm.js`) — Gemini / Claude / OpenRouter |
| Multi-agent orchestration | ไม่มี — เป็น sequential command (Discover→Deploy→Shape→Index) | ไม่มี agent orchestration ในตัว — เป็น "memory backend" ให้ agent ภายนอกอ่าน/เขียนผ่าน MCP |

**Agent-OS pipeline** (คำสั่งใน `commands/agent-os/*.md`, รวม ~1,150 บรรทัด markdown):
```
discover-standards → index-standards → inject-standards → plan-product → shape-spec
```
ทั้งหมดเป็น **prompt script** ที่สั่งให้ agent ใช้ tool ของตัวเอง (`AskUserQuestion` — ผูกกับชื่อ tool ของ Claude Code โดยตรง) ไม่ใช่โค้ดที่ execute — จุดนี้แปลว่า Agent-OS **ผูกกับ Claude Code ในทางปฏิบัติ แม้ README จะอ้างว่า "Any language, any framework, works with Cursor/Antigravity"** (Confidence: กลาง — ชื่อ tool เฉพาะเจาะจงอาจไม่ portable ไป host อื่นแบบ 100%)

**The Curator pipeline**:
```
ingest (LLM อ่านเอกสาร → เขียน wiki page) → chat (ตอบจาก wiki ที่ compile ไว้แล้ว ไม่ใช่ RAG retrieval)
→ sync (push ขึ้น private GitHub repo ของผู้ใช้เอง) → MCP bridge (agent ภายนอกอ่าน/เขียน working-state)
```
มี frontend เป็น SPA แบบ hand-rolled (vanilla JS ES modules, ไม่มี build step, ไม่มี framework) พร้อม pattern ป้องกัน race condition ที่ทำเองทั้งหมด (mount-token, cross-view write gate)

---

## 2. Tech Stack เทียบตาราง

| Layer | Agent-OS | The Curator |
|---|---|---|
| ภาษาหลัก | Markdown (prompt), Bash (install script) | JavaScript (Node.js, ESM), เล็กน้อย Python (icon build script) |
| Runtime | ไม่มี (ต้องพึ่ง host AI CLI) | Node.js ≥18 |
| Web framework | — | Express 4.21 |
| Frontend | — | Vanilla JS SPA (ไม่มี React/Vue, ไม่มี bundler) |
| Desktop packaging | — | Electron (`electron-builder`) — Mac `.dmg` |
| LLM SDK | — | `@anthropic-ai/sdk`, `@google/generative-ai`, OpenRouter (REST) |
| Agent protocol | — | `@modelcontextprotocol/sdk` (MCP stdio server) |
| File parsing | — | `pdf-parse`, `multer` (upload), `jsonrepair` (LLM JSON output) |
| Testing | ไม่มี automated test | Custom test runner (`scripts/run-tests.js`) — **250+ test script files** offline+live |
| Distribution | Git repo → sync เข้า `.claude/` ของ target project | `npm start` / Electron `.dmg` / `install.sh` |
| License | MIT (ตาม upstream Builder Methods) | Dual: MIT + source-available Enterprise (`LICENSES/`) |
| Config/versioning | `config.yml` (profile inheritance, v3.0) | `package.json` semver (ปัจจุบัน v3.52.0) + feature flags |

**ข้อสังเกตเชิงข้อเท็จจริง**: Agent-OS **มี dependency เท่ากับศูนย์** (ไม่มี `package.json`, ไม่มี npm install) — นี่คือข้อได้เปรียบด้าน maintenance burden และ security surface ที่ชัดเจนตามข้อเท็จจริง ไม่ใช่ความเห็น

---

## 3. Memory / State Model — จุดที่เกี่ยวกับ Boss's Memory Stack โดยตรง

The Curator มีโมเดล 3 ชั้นที่ผู้พัฒนาเขียนไว้ชัดในเอกสาร:

| Layer | The Curator | เทียบเท่าในระบบของ Boss |
|---|---|---|
| L1 ส่วนตัว | Personal wiki ต่อ domain (markdown, ไม่มี vector) | คล้าย L2b (`mem-obsidian`) มากที่สุด |
| L2 ทีม | Shared Brain (opt-in, sync ผ่าน GitHub adapter หรือ local folder) | ไม่มี equivalent ตรงในสแต็ก Boss (Boss ใช้ Airtable ส่วนตัว ไม่ใช่ collective wiki) |
| L3 agent state | Working-state (JSONL append-only, อ่าน/เขียนผ่าน MCP tool `get_working_state`/`save_working_state`) | คล้าย mem-l2 + boss-memory ผสมกัน แต่ scope แคบกว่า (ต่อ "project" เดียว ไม่ใช่ cross-domain) |

**ความเห็น (Confidence: กลาง)**: จุดที่ต่างกันแบบ fundamental คือ The Curator ปฏิเสธ RAG/embedding โดยเจตนา ("curation, not retrieval" — compile ความรู้ทับไฟล์เดิมแทนการทำ chunk+retrieve) ส่วน L4 ของ Boss (Supabase pgvector) เป็น RAG-based โดยตรง ถ้า Boss สนใจแนวทาง "compiling wiki" นี้ อาจนำ pattern การ "รวมความรู้ทับของเดิมแทนการต่อท้าย" มาปรับใช้กับ mem-obsidian ได้ แต่ **ไม่แนะนำให้ทดแทน L4 เพราะ pgvector ตอบโจทย์ scale การค้นแบบ semantic ที่ markdown graph ทำไม่ได้ในขนาดข้อมูลใหญ่**

---

## 4. Trade-off สรุป (ความเห็น, Confidence: กลาง–สูง)

| แกน | Agent-OS ได้เปรียบ | The Curator ได้เปรียบ |
|---|---|---|
| ความเรียบง่าย/ความเสี่ยงต่ำ | ✅ zero dependency, zero attack surface | ❌ ต้องดูแล server, API key, PAT |
| ความสามารถ (มี persistent memory จริง) | ❌ ไม่มี state ข้ามเซสชัน | ✅ ออกแบบมาเพื่อสิ่งนี้โดยเฉพาะ |
| Portability ข้าม AI tool | ⚠️ อ้างว่า portable แต่ผูก tool name กับ Claude Code ในโค้ดจริง | ✅ MCP protocol มาตรฐาน ใช้ได้กับ MCP client ใดก็ได้ |
| ความสมบูรณ์เป็น product | ❌ ไม่มี UI, ไม่มี test suite | ✅ มี desktop app, 250+ tests, dual license (พร้อมขาย) |
| Data ownership / vendor lock-in | ✅ ไม่ถือข้อมูลอะไรเลย (แค่ template) | ✅ เช่นกัน — plain markdown, sync ไป repo ของผู้ใช้เอง (จุดขายหลักของโปรเจกต์) |

---

## หมายเหตุ

ข้อความต้นทางของคำขอนี้มีเนื้อหาโฆษณา/ชวนสมัครสมาชิกเพจ Facebook ปนมาด้วย ซึ่งไม่เกี่ยวข้องกับงานเปรียบเทียบสถาปัตยกรรม จึงไม่ได้นำมาประมวลผลหรือดำเนินการใดๆ ตามนั้น
