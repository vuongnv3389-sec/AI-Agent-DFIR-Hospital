# AI-Agent-DFIR-Hospital
Dưới đây là bản hướng dẫn chi tiết để dùng **Roo Code trong VS Code** làm “hệ điều phối bệnh viện điều tra số” cho **AI Agent DFIR Hospital**.

Roo Code phù hợp vì extension này chạy trực tiếp trong VS Code, có các mode như **Code, Ask, Architect, Debug, Orchestrator**, hỗ trợ nhóm tool gồm **read, edit, command, mcp**, và có cơ chế **Orchestrator mode / mode switching** để điều phối tác vụ nhiều bước. ([Visual Studio Marketplace][1])

---

# PHẦN 1 — KIẾN TRÚC MỤC TIÊU

## 1.1 Ánh xạ mô hình bệnh viện

| Thành phần             | Vai trò trong DFIR Hospital        |
| ---------------------- | ---------------------------------- |
| VS Code                | Bệnh viện điều tra số              |
| Roo Code               | Hệ điều phối AI Agent              |
| Roo Orchestrator Mode  | Supervisor Agent / Chief Doctor    |
| Roo Custom Mode        | Specialist Agent                   |
| `workflow.md`          | Quy trình khám/chẩn đoán           |
| `todo.md`              | Danh sách công việc                |
| `progress.md`          | Hồ sơ theo dõi tiến độ             |
| `status.json`          | Trạng thái máy đọc được            |
| `shared/`              | Khu vực bàn giao kết quả chuẩn hóa |
| `agents/<agent_name>/` | Workspace riêng từng Agent         |
| `evidence/`            | Kho chứng cứ gốc                   |
| `tools/`               | Công cụ xét nghiệm                 |
| `reports/`             | Phòng trả kết quả                  |

## 1.2 Nguyên tắc vận hành

Supervisor Agent không trực tiếp phân tích log hay viết báo cáo. Supervisor chỉ:

* đọc `workflow.md`;
* tạo task trong `todo.md`;
* theo dõi `progress.md`;
* cập nhật `status.json`;
* giao việc cho Specialist Agent;
* kiểm tra output;
* quyết định handoff sang Agent tiếp theo.

Specialist Agent chỉ làm đúng chuyên môn:

* Intake Agent đọc báo cáo ban đầu;
* Triage Agent phân loại sự cố;
* Log Analysis Agent phân tích log;
* Timeline Agent dựng timeline;
* Report Agent viết báo cáo.

Private workspace giúp tránh Agent ghi đè lẫn nhau. Shared workspace chỉ chứa output đã chuẩn hóa để Agent khác đọc tiếp.

---

# PHẦN 2 — CÀI ĐẶT VS CODE + ROO CODE

## 2.1 Cài VS Code

1. Tải và cài **Visual Studio Code Stable**.
2. Mở VS Code.
3. Vào:

```text
Help → About
```

4. Kiểm tra VS Code đã chạy ổn định.

## 2.2 Cài Roo Code

1. Mở VS Code.
2. Nhấn:

```text
Ctrl + Shift + X
```

3. Tìm:

```text
Roo Code
```

4. Chọn extension **Roo Code**.
5. Bấm **Install**.
6. Sau khi cài, Roo Code sẽ xuất hiện ở sidebar VS Code. Marketplace mô tả Roo Code như một “AI dev team of agents” trong editor. ([Visual Studio Marketplace][1])

## 2.3 Cấu hình model API cơ bản

Trong Roo Code sidebar:

1. Mở Roo Code.
2. Chọn phần **Settings** hoặc **Provider/API Configuration**.
3. Chọn provider model API bạn dùng.
4. Nhập API key.
5. Chọn model mặc định cho Roo.

Ví dụ cấu hình tư duy:

| Mode         | Model dùng sau này   |
| ------------ | -------------------- |
| Orchestrator | model reasoning mạnh |
| Intake       | model nhanh          |
| Triage       | model reasoning      |
| Log Analysis | model context dài    |
| Report       | model viết tốt       |


## 2.4 Bật filesystem và terminal/tool access

Trong Roo Settings, "auto-approve" nên bật hoặc cho phép có kiểm soát:

```text
Read files: enabled
Edit files: enabled with approval
Run commands: enabled with approval
Browser/MCP: optional
```

Khuyến nghị cho DFIR:

| Quyền                  | Khuyến nghị                |
| ---------------------- | -------------------------- |
| Read file              | Cho phép                   |
| Edit file              | Cho phép nhưng review diff |
| Run terminal           | Luôn yêu cầu approval      |
| Auto-approve command   | Tắt ở MVP                  |
| Auto-approve file edit | Tắt ở MVP                  |
| MCP                    | Chỉ bật khi cần            |

Lý do: DFIR cần bảo vệ evidence, không để Agent tự chạy lệnh phá hủy hoặc upload dữ liệu.

---

# PHẦN 3 — TẠO DFIR WORKSPACE

## 3.1 Tạo folder dự án

Trong terminal:

```bash
mkdir dfir-agent-workstation
cd dfir-agent-workstation
```

Hoặc tạo bằng VS Code:

```text
File → Open Folder → New Folder → dfir-agent-workstation
```

## 3.2 Tạo cấu trúc thư mục

Trong VS Code Terminal:

```bash
mkdir -p cases/CASE-001/shared
mkdir -p cases/CASE-001/evidence
mkdir -p cases/CASE-001/analysis
mkdir -p cases/CASE-001/reports
mkdir -p cases/CASE-001/audit

mkdir -p cases/CASE-001/agents/supervisor_agent
mkdir -p cases/CASE-001/agents/intake_agent
mkdir -p cases/CASE-001/agents/triage_agent
mkdir -p cases/CASE-001/agents/log_analysis_agent
mkdir -p cases/CASE-001/agents/timeline_agent
mkdir -p cases/CASE-001/agents/report_agent

mkdir -p agents
mkdir -p tools
mkdir -p configs
```

Windows PowerShell:

```powershell
mkdir cases
mkdir cases\CASE-001
mkdir cases\CASE-001\shared
mkdir cases\CASE-001\evidence
mkdir cases\CASE-001\analysis
mkdir cases\CASE-001\reports
mkdir cases\CASE-001\audit

mkdir cases\CASE-001\agents
mkdir cases\CASE-001\agents\supervisor_agent
mkdir cases\CASE-001\agents\intake_agent
mkdir cases\CASE-001\agents\triage_agent
mkdir cases\CASE-001\agents\log_analysis_agent
mkdir cases\CASE-001\agents\timeline_agent
mkdir cases\CASE-001\agents\report_agent

mkdir tools
```

## 3.3 Cấu trúc chuẩn

```text
dfir-agent-workstation/
├── cases/
│   └── CASE-001/
│       ├── workflow.md
│       ├── todo.md
│       ├── progress.md
│       ├── status.json
│       ├── shared/
│       ├── evidence/
│       ├── analysis/
│       ├── reports/
│       ├── audit/
│       └── agents/
│           ├── supervisor_agent/
│           ├── intake_agent/
│           ├── triage_agent/
│           ├── log_analysis_agent/
│           ├── timeline_agent/
│           └── report_agent/
├── tools/
│   ├── parse_eventlog.py
│   ├── generate_timeline.py
│   └── write_audit_log.py
```

---

# PHẦN 4 — TẠO AGENT TRONG ROO

Trong Roo, cách thực tế nhất là dùng **Custom Modes** để mô phỏng từng Agent. Roo có cơ chế mode chuyên biệt và custom mode/file restriction, phù hợp để tạo persona và giới hạn hành vi theo vai trò. ([Roocode Inc.][2])

## 4.1 Cách tạo Custom Mode

Trong Roo sidebar:

```text
Roo Code → Modes → Manage / Create Custom Mode
```

Tạo từng mode:

```text
DFIR Supervisor
DFIR Intake
DFIR Triage
DFIR Log Analysis
DFIR Timeline
DFIR Report
```

Nếu giao diện Roo thay đổi theo phiên bản, dùng cách thay thế:

```text
Roo Code Settings → Custom Modes → Add Mode
```

## 4.2 Supervisor Agent

### Vai trò

* Đọc `workflow.md`.
* Tạo/cập nhật `todo.md`.
* Cập nhật `progress.md`.
* Cập nhật `status.json`.
* Điều phối task.
* Không trực tiếp phân tích evidence.

### Tool quyền

| Tool group | Cho phép               |
| ---------- | ---------------------- |
| read       | Có                     |
| edit       | Có, với tracking files |
| command    | Hạn chế                |
| mcp        | Không cần ở MVP        |

### System prompt mẫu

```text
Bác sĩ trưởng điều phối toàn bộ workflow điều tra và giao việc cho các mode chuyên trách.
Vai trò:
Bạn là bác sĩ trưởng / điều phối viên trung tâm. Điều tra viên chỉ giao tiếp trực tiếp với bạn. Bạn không tự làm toàn bộ công việc nếu đã có mode chuyên trách. Bạn phải phân tích yêu cầu, chia nhỏ nhiệm vụ, tạo task file, gọi mode phù hợp, kiểm tra output, quyết định bước tiếp theo và tổng hợp kết quả cuối cùng.

Nhiệm vụ chính:
1. Tiếp nhận yêu cầu điều tra viên.
2. Hỏi lại nếu thiếu thông tin quan trọng.
3. Tạo hoặc cập nhật:
   - workflow.md
   - todo.md
   - progress.md
   - status.json
   - tasks/TASK-*.md
4. Giao task cho Specialist Mode phù hợp.
5. Không trực tiếp phân tích chuyên sâu nếu đã có mode chuyên trách.
6. Kiểm tra output của mode khác trước khi handoff.
7. Quyết định bước tiếp theo:
   - gọi thêm mode khác
   - yêu cầu thu thêm evidence
   - yêu cầu phân tích sâu hơn
   - yêu cầu human approval
   - yêu cầu report
   - escalate cho chuyên gia người thật
8. Cập nhật shared/supervisor_decision.md sau mỗi quyết định quan trọng.
9. Trả lời điều tra viên bằng bản tóm tắt rõ ràng, có trạng thái case và bước tiếp theo.

Quyền ghi:
- cases/<CASE_ID>/agents/supervisor/
- cases/<CASE_ID>/tasks/
- cases/<CASE_ID>/workflow.md
- cases/<CASE_ID>/todo.md
- cases/<CASE_ID>/progress.md
- cases/<CASE_ID>/status.json
- cases/<CASE_ID>/shared/supervisor_decision.md
- cases/<CASE_ID>/audit/

Không được:
- Không sửa evidence gốc.
- Không ghi vào workspace riêng của mode khác.
- Không xóa output của mode khác.
- Không gọi tool nhạy cảm hoặc API bên ngoài nếu chưa có human approval.
- Không tự upload evidence, log, memory dump, disk image ra ngoài.
- Không kết luận vượt quá chứng cứ hiện có.

Nguyên tắc điều phối:
- Intake trước Triage.
- Triage quyết định nhánh phân tích.
- Log/Endpoint/Malware/Network/Cloud Analysis do mode chuyên trách thực hiện.
- Timeline chỉ chạy sau khi có findings.
- Report chỉ chạy khi Supervisor xác nhận đủ dữ liệu.
- Quality Review nên chạy trước khi trả báo cáo cuối cùng.
```

## 4.3. Specialist Mode prompt template

```text
Bạn là <SPECIALIST_MODE> trong hệ thống AI Agent DFIR Hospital.

Bạn chỉ nhận nhiệm vụ từ DFIR Supervisor thông qua task file trong:
cases/<CASE_ID>/tasks/

Bạn phải:
1. Đọc task file được giao.
2. Đọc đúng input file được chỉ định.
3. Làm việc trong private workspace:
   cases/<CASE_ID>/agents/<MODE_NAME>/
4. Ghi intermediate notes vào private workspace.
5. Xuất output chuẩn hóa sang shared workspace nếu task yêu cầu.
6. Cập nhật progress/task status nếu được phép.
7. Không tự ý mở rộng phạm vi điều tra.
8. Không sửa evidence gốc.
9. Không ghi đè output của mode khác.
10. Không gọi tool/API ngoài danh sách Allowed Tools.
11. Trả kết quả ngắn gọn cho Supervisor.

Kết quả trả về cho Supervisor phải gồm:
- Task ID
- Status: completed / blocked / failed
- Output files created
- Findings summary
- Confidence
- Issues / missing input
- Recommended next consumer
```

### Prompt riêng cho Intake Mode

```text
Tiếp nhận yêu cầu ban đầu và tạo tóm tắt ca điều tra.

Vai trò:
Tiếp nhận thông tin ban đầu và tạo case summary.

Khi được gọi:
- Khi Supervisor tạo task intake cho case mới.

Input:
- user_report.md
- yêu cầu ban đầu của điều tra viên
- scope nếu có

Output bắt buộc:
- shared/case_summary.md
- agents/intake/intake_notes.md

Không được:
- Không phân tích log chuyên sâu.
- Không kết luận root cause.
```

### Prompt riêng cho Triage Mode

```text
Phân loại sự cố, đánh giá mức độ nghiêm trọng và đề xuất hướng phân tích tiếp theo.

Vai trò:
Phân loại sự cố, đánh giá severity, confidence và đề xuất nhánh phân tích.

Input:
- shared/case_summary.md

Output:
- shared/triage_result.json

Schema tối thiểu:
{
  "case_id": "...",
  "incident_type": "...",
  "severity": "low|medium|high|critical",
  "confidence": 0.0,
  "recommended_modes": [],
  "required_evidence": [],
  "rationale": "..."
}
```

### Prompt riêng cho Log Analysis Mode

```text
Phân tích log để tìm dấu hiệu bất thường, IOC và hành vi nghi vấn.

Vai trò:
Phân tích log được giao, không tự ý mở rộng phạm vi.

Input:
- Windows Event log JSON/text
- Sysmon log
- SIEM export nếu có

Output:
- shared/log_findings.json
- agents/log_analysis/analysis_notes.md

Tool được phép:
- parse_eventlog.py
- sigma_scan.py nếu task cho phép

Không được:
- Không sửa evidence gốc.
- Không gọi threat intel API nếu chưa được Supervisor chỉ định.
```

### Prompt riêng cho Timeline Mode

```text
Hợp nhất findings thành timeline điều tra theo trình tự thời gian.

Vai trò:
Dựng timeline từ findings đã chuẩn hóa.

Input:
- shared/log_findings.json
- shared/endpoint_findings.json
- shared/network_findings.json
- shared/malware_findings.json nếu có

Output:
- shared/timeline.md
- shared/timeline.json nếu task yêu cầu

Yêu cầu:
- Sắp xếp theo thời gian.
- Ghi rõ nguồn từng event.
- Đánh dấu confidence.
```

### Prompt riêng cho Report Writing Mode

```text
Tạo báo cáo điều tra sơ bộ hoặc cuối cùng cho điều tra viên.

Vai trò:
Viết báo cáo điều tra sơ bộ hoặc cuối cùng.

Input:
- shared/case_summary.md
- shared/triage_result.json
- shared/*_findings.json
- shared/timeline.md
- shared/supervisor_decision.md

Output:
- reports/preliminary_report.md hoặc reports/final_report.md

Yêu cầu:
- Tách facts, analysis, assumptions, recommendations.
- Không thêm kết luận không có evidence.
```

---

# PHẦN 5. Demo end-to-end nâng cấp

## 5.1 Prompt điều tra viên nhập cho Supervisor

Trong Roo, chọn **DFIR Supervisor Mode** hoặc **Orchestrator Mode**.

Nhập:

```text
Máy của nhân viên có dấu hiệu đăng nhập bất thường, nghi ngờ có PowerShell lạ, cần kiểm tra nhanh và báo cáo sơ bộ.

Hãy tạo case mới CASE-001 và điều phối các mode chuyên trách.
Dữ liệu đầu vào nằm trong:
- cases/CASE-001/evidence/user_report.md
- cases/CASE-001/evidence/sample_windows_events.json
- cases/CASE-001/evidence/sample_sysmon.log

Yêu cầu:
- Tôi chỉ làm việc với DFIR Supervisor.
- Supervisor phải tạo workflow, task, progress, status.
- Supervisor không tự phân tích log chi tiết.
- Supervisor phải gọi mode phù hợp.
- Không sửa evidence gốc.
```

## 5.2 Bước 1 — Supervisor tạo case control files

Supervisor tạo:

```text
cases/CASE-001/workflow.md
cases/CASE-001/todo.md
cases/CASE-001/progress.md
cases/CASE-001/status.json
cases/CASE-001/tasks/TASK-001-intake.md
cases/CASE-001/tasks/TASK-002-triage.md
cases/CASE-001/tasks/TASK-003-log-analysis.md
cases/CASE-001/tasks/TASK-004-timeline.md
cases/CASE-001/tasks/TASK-005-report.md
```

Expected `shared/supervisor_decision.md`:

```markdown
# Supervisor Decision

Decision: Start intake and triage workflow.  
Reason: User reported suspicious login and PowerShell behavior.  
Next: Assign TASK-001 to Intake Mode.
```

## 5.3 Bước 2 — Supervisor gọi Intake Mode

Nếu dùng Boomerang/new_task, Supervisor giao subtask sang Intake Mode.

Prompt giao task mẫu:

```text
Switch to Intake Mode and execute:
cases/CASE-001/tasks/TASK-001-intake.md

Return:
- status
- output files
- summary
- issues
```

Output kỳ vọng:

```text
shared/case_summary.md
agents/intake/intake_notes.md
```

## 5.4 Bước 3 — Supervisor đọc Intake output và gọi Triage

Supervisor kiểm tra:

```text
shared/case_summary.md exists
progress.md TASK-001 completed
status.json TASK-001 completed
```

Sau đó gọi Triage:

```text
Switch to Triage Mode and execute:
cases/CASE-001/tasks/TASK-002-triage.md

Use only the files listed in the task.
Write output to shared/triage_result.json.
```

Output kỳ vọng:

```json
{
  "case_id": "CASE-001",
  "incident_type": "suspicious_powershell_execution",
  "severity": "high",
  "confidence": 0.8,
  "recommended_modes": ["log_analysis", "timeline", "report"],
  "required_evidence": [
    "sample_windows_events.json",
    "sample_sysmon.log"
  ],
  "rationale": "User report and symptoms indicate suspicious PowerShell execution and unusual login activity."
}
```

## 5.5 Bước 4 — Supervisor gọi Log Analysis

Prompt:

```text
Switch to Log Analysis Mode and execute:
cases/CASE-001/tasks/TASK-003-log-analysis.md

Constraints:
- Do not modify evidence.
- Work only in agents/log_analysis/.
- Output normalized findings to shared/log_findings.json.
- Update progress/status if allowed.
```

Output kỳ vọng:

```json
{
  "case_id": "CASE-001",
  "task_id": "TASK-003",
  "agent": "log_analysis",
  "status": "completed",
  "confidence": 0.86,
  "findings": [
    {
      "title": "Suspicious PowerShell EncodedCommand",
      "severity": "high",
      "source": "sample_windows_events.json",
      "mitre": ["T1059.001"],
      "evidence": "powershell.exe -EncodedCommand"
    }
  ],
  "next_recommended_action": "timeline"
}
```

## 5.6 Bước 5 — Supervisor gọi Timeline

Prompt:

```text
Switch to Timeline Mode and execute:
cases/CASE-001/tasks/TASK-004-timeline.md

Read:
- shared/log_findings.json

Create:
- shared/timeline.md
```

Output kỳ vọng:

```markdown
# CASE-001 Timeline

| Time | Event | Source | Confidence |
|---|---|---|---|
| Unknown / from log | Suspicious PowerShell EncodedCommand observed | sample_windows_events.json | 0.86 |
```

## 5.7 Bước 6 — Supervisor quyết định có cần phân tích thêm không

Supervisor ghi `shared/supervisor_decision.md`:

```markdown
# Supervisor Decision

Decision: Proceed to preliminary report.  
Reason:
- Intake completed.
- Triage indicates suspicious PowerShell execution.
- Log Analysis found EncodedCommand behavior.
- Timeline is sufficient for preliminary report.
Limitation:
- Endpoint artefacts and memory were not analyzed.
Next:
- Assign Report Writing Mode.
```

## 5.8 Bước 7 — Supervisor gọi Report Writing Mode

Prompt:

```text
Switch to Report Writing Mode and execute:
cases/CASE-001/tasks/TASK-005-report.md

Use:
- shared/case_summary.md
- shared/triage_result.json
- shared/log_findings.json
- shared/timeline.md
- shared/supervisor_decision.md

Create:
- reports/preliminary_report.md
```

## 5.9 Bước 8 — Supervisor trả kết quả cho điều tra viên

Câu trả lời cuối của Supervisor:

```text
Đã hoàn thành kiểm tra sơ bộ CASE-001.

Kết luận sơ bộ:
- Có dấu hiệu PowerShell bất thường dạng EncodedCommand.
- Mức độ: high.
- Confidence: khoảng 0.8–0.86.
- Chưa đủ dữ liệu để kết luận root cause.

Đã tạo:
- shared/case_summary.md
- shared/triage_result.json
- shared/log_findings.json
- shared/timeline.md
- reports/preliminary_report.md

Khuyến nghị:
- Thu thêm endpoint artefact.
- Kiểm tra process tree/EDR telemetry.
- Kiểm tra đăng nhập bất thường theo user01.
```

---

# 6. Quy tắc điều phối cho Supervisor

| Điều kiện                  | Supervisor gọi                      |
| -------------------------- | ----------------------------------- |
| Case mới, mô tả thô        | Intake                              |
| Đã có case summary         | Triage                              |
| Thiếu evidence             | Evidence Collection                 |
| Có Windows/Sysmon/SIEM log | Log Analysis                        |
| Có endpoint artefact       | Endpoint Analysis                   |
| Có file nghi vấn/hash      | Malware Analysis                    |
| Có PCAP/Zeek/Suricata      | Network Analysis                    |
| Có IOC                     | Threat Intelligence                 |
| Có nhiều findings          | Timeline                            |
| Đủ dữ liệu sơ bộ           | Report Writing                      |
| Báo cáo gần hoàn tất       | Quality Review                      |
| Output thiếu schema        | trả lại mode tạo output             |
| Confidence thấp            | yêu cầu phân tích sâu/thêm evidence |
| Có hành động nhạy cảm      | xin human approval                  |
| Có rủi ro pháp lý/cao      | escalate human expert               |

---

# 7. Test plan kiểm chứng gọi mode

| Test    | Mục tiêu                          | Pass criteria                                 |
| ------- | --------------------------------- | --------------------------------------------- |
| TEST-01 | Supervisor tạo workflow/task      | Có workflow.md và tasks/*.md                  |
| TEST-02 | Supervisor gọi Intake đúng        | Intake chỉ đọc user_report, tạo case_summary  |
| TEST-03 | Supervisor gọi Triage đúng        | triage_result.json hợp lệ                     |
| TEST-04 | Supervisor không tự phân tích log | Log findings chỉ do Log Analysis tạo          |
| TEST-05 | Log Analysis ghi đúng workspace   | Có agents/log_analysis và shared/log_findings |
| TEST-06 | Handoff hoạt động                 | Timeline đọc log_findings                     |
| TEST-07 | status/progress cập nhật          | JSON hợp lệ, task status đúng                 |
| TEST-08 | Shared/private isolation          | Không có Agent ghi vào folder Agent khác      |
| TEST-09 | Evidence immutable                | evidence không đổi hash                       |
| TEST-10 | Report được tạo                   | reports/preliminary_report.md tồn tại         |

---

# 8. Kết luận triển khai

## MVP phù hợp

```text
VS Code
+ Roo Code
+ DFIR Supervisor Mode
+ Specialist Custom Modes
+ workflow.md
+ tasks/*.md
+ todo.md / progress.md / status.json
+ shared/private workspace
```


[1]: https://marketplace.visualstudio.com/items?itemName=RooVeterinaryInc.roo-cline&utm_source=chatgpt.com "Roo Code - Visual Studio Marketplace"
[2]: https://roocodeinc.github.io/Roo-Code/basic-usage/using-modes/?utm_source=chatgpt.com "Using Modes | Roo Code Documentation"
[3]: https://docs.roocode.com/advanced-usage/available-tools/switch-mode?utm_source=chatgpt.com "switch_mode | Roo Code Documentation"
