# Day 13 Observability Lab Report

## 1. Team Metadata

- [GROUP_NAME]: C401-X4 ;
- [REPO_URL]: https://github.com/thiennguyen37-qn/NhomX4-401-Day13.git ;
- [MEMBERS]:
  - Member 1: Võ Thanh Chung | Role: Logging, PII Redaction, Metrics, Langfuse Tracing Integration
  - Member 2: Đỗ Thế Anh | Role: streamlit_app.py owner, tạo mock data request đến APIs
  - Member 3: Nguyễn Hồ Bảo Thiên | Role: SLOs and Alerts
  - Member 4: Dương Khoa Điềm | Role: Testing, Monitoring, Dashboard Verification
  - Member 5: Lê Bảo Khang | Role: Dashboard and Observability Enhancement
  - Member 6: Hoàng Thị Thanh Tuyền | Role: update Runbook, tổng hợp report

---

## 2. Group Performance (Auto-Verified)

- [VALIDATE_LOGS_FINAL_SCORE]: 80/100 ;
- [TOTAL_TRACES_COUNT]: 37 ;
- [PII_LEAKS_FOUND]: 0 ;

---

## 3. Technical Evidence (Group)

### 3.1 Logging & Tracing

- [EVIDENCE_CORRELATION_ID_SCREENSHOT]: ![Correlation ID Screenshot](docs/screenshots/correlation_id.png)
- [EVIDENCE_PII_REDACTION_SCREENSHOT]: ![PII Redaction Screenshot](docs/screenshots/pii_redaction.jpg)
- [EVIDENCE_TRACE_WATERFALL_SCREENSHOT]: ![Trace Waterfall Screenshot](docs/screenshots/grafana-dashboard.jpg)
- [TRACE_WATERFALL_EXPLANATION]: Trong waterfall, span đáng chú ý là chuỗi child span `explain-HBA1C` -> `explain-LDL` -> `explain-GLUCOSE_F` dưới `lumina-workflow`. Ở khoảng `2026-04-20 22:17:08` đến `22:17:51`, các span `explain-*` lên `Level = ERROR`, cho thấy lỗi xảy ra khi gọi LLM để sinh diễn giải cho từng xét nghiệm. Tuy vậy span cha `lumina-workflow` vẫn kết thúc thành công (đối chiếu log `logs/2026-04-20.log:4`, `correlation_id=f11e2e76-89ff-4b57-8aa7-71167f700d5c`, `error=null`) vì `explain_node` có cơ chế fallback deterministic khi LLM lỗi. Insight chính: hệ thống đang có tính chịu lỗi tốt ở mức workflow, nhưng cần giám sát riêng error-rate của span `explain-*` để phát hiện sớm suy giảm chất lượng trả lời.

### 3.2 Dashboard & SLOs

- [DASHBOARD_6_PANELS_SCREENSHOT]: ![Dashboard 6 Panels](docs/screenshots/grafana-dashboard.jpg)
- [SLO_TABLE]:
  | SLI | Target | Window | Current Value |
  |---|---:|---|---:|
  | Latency P95 | < 3000ms | 28d | 9680ms |
  | Error Rate | < 2% | 28d | 8.1% |
  | Cost Budget | < $2.5/day | 1d | $0.2590/day |

### 3.3 Alerts & Runbook

- [ALERT_RULES_SCREENSHOT]: ![Alert Rules Screenshot](docs/screenshots/alerts.jpg)
- [SAMPLE_RUNBOOK_LINK]: [high_latency](docs/runbooks/high-latency.md); [high_error_rate](docs/runbooks/high-error-rate.md); [cost_budget](docs/runbooks/cost-budget.md); [hallucination_proxy](docs/runbooks/hallucination-proxy.md)

---

## 4. Incident Response (Group)

- [SCENARIO_NAME]: `llm_tool_span_error_but_workflow_survives`
- [SYMPTOMS_OBSERVED]:
- Trên Tracing, các span `explain-HBA1C`, `explain-LDL`, `explain-GLUCOSE_F` xuất hiện `Level = ERROR` (khung thời gian khoảng `2026-04-20 22:17:08` đến `22:17:51`).
  - Tuy nhiên `lumina-workflow` vẫn hoàn tất (không crash toàn luồng), người dùng vẫn nhận phản hồi nhưng chất lượng có lúc giảm (ví dụ câu trả lời xin lỗi/chung chung).
- [ROOT_CAUSE_PROVED_BY]:
  - **Trace evidence**: nhiều child span `explain-*` bị `ERROR` trong cùng phiên chạy, cho thấy lỗi xảy ra tại bước gọi LLM để giải thích từng chỉ số.
  - **Log evidence**: `logs/2026-04-20.log:4` có `event=workflow.complete`, `correlation_id=f11e2e76-89ff-4b57-8aa7-71167f700d5c`, `latency_ms=57196`, `error=null` -> chứng minh workflow chính vẫn sống nhờ cơ chế fallback.
  - **Code evidence**: `src/nodes/explain_node.py` bắt lỗi trong `_llm_explanation(...)` và `pass` để giữ deterministic fallback, nên span con lỗi nhưng workflow không fail.
- [FIX_ACTION]:
  - Cấu hình/kiểm tra lại provider LLM (API key, endpoint, hạn mức), đồng thời bật fallback provider (`azure` <-> `groq`) khi một nhà cung cấp lỗi.
  - Bổ sung logging chi tiết trong khối `except` của `explain_node` (ghi loại lỗi + test_code + correlation_id) để truy vết nhanh hơn thay vì chỉ thấy `ERROR` trên trace.
- [PREVENTIVE_MEASURE]:
  - Thiết lập alert theo tỷ lệ `ERROR` của span `explain-*` (ví dụ >5% trong 5 phút thì cảnh báo).
  - Thêm circuit breaker/retry có backoff cho gọi LLM theo từng test.
  - Theo dõi SLI riêng cho `explain-*` (error-rate, latency) và runbook xử lý sự cố provider.

---

## 5. Individual Contributions & Evidence

### Võ Thanh Chung

- [TASKS_COMPLETED]: Logging/PII redaction, metrics va Langfuse tracing integration.
- [EVIDENCE_LINK]: [Commit 53bc88a](https://github.com/thiennguyen37-qn/NhomX4-401-Day13/commit/53bc88a)

### Đỗ Thế Anh

- [TASKS_COMPLETED]: Tracing enrichment, thiết kế UI app cho phép người dùng chỉnh sửa json body request đến APIs (để cố tình tạo lỗi, mock data test). Cho phép stress test trong khoảng được thiết lập. Cập nhật đến live dashboard
- [EVIDENCE_LINK]: [Commit 6049b53] (https://github.com/thiennguyen37-qn/NhomX4-401-Day13/commit/6049b53)

### Nguyễn Hồ Bảo Thiên

- [TASKS_COMPLETED]: Đóng góp cho SLO và Alerts flow, config và trace Langfuse.
- [EVIDENCE_LINK]: [Commit cf41ba0](https://github.com/thiennguyen37-qn/NhomX4-401-Day13/commit/cf41ba06b021cdb1d78b4e4e7edcce04ba2f798f)

### Dương Khoa Điềm

- [TASKS_COMPLETED]: Testing, monitoring, theo dõi dashboard
- [EVIDENCE_LINK]: [Commit bd4b4 ](https://github.com/thiennguyen37-qn/NhomX4-401-Day13/commit/b0e9b4ad8ce0edb4374ba5550e848e35308bd4b4) ; [Commit 8a4fff ](https://github.com/thiennguyen37-qn/NhomX4-401-Day13/commit/5e9f24ff766b6161321700725116babcee8a4fff)

### Lê Bảo Khang

- [TASKS_COMPLETED]: Dashboard/observability enhancement
- [EVIDENCE_LINK]: [Commit e195a62](https://github.com/thiennguyen37-qn/NhomX4-401-Day13/commit/e195a62)

### Hoàng Thị Thanh Tuyền

- [TASKS_COMPLETED]: Runbook/hallucination updates, prototype va demo/report support.
- [EVIDENCE_LINK]: [Commit 8e4ee94](https://github.com/thiennguyen37-qn/NhomX4-401-Day13/commit/8e4ee94)

---