# Day 23 Lab Reflection

> Fill in each section. Grader reads the "What I'd change" paragraph closest.

**Student:** Nguyễn Tiến Huân - 2A202600855
**Submission date:** 29/06/2026
**Lab repo URL:** [_public GitHub URL_](https://github.com/HuanNT03/Day23-2A202600855-NguyenTienHuan.git)

---

## 1. Hardware + setup output

Paste output of `python3 00-setup/verify-docker.py`:

```
Docker:        OK  (29.5.3)
Compose v2:    OK  (5.1.4)
RAM available: 1.8 GB (NEED >= 4.0 GB)
Ports free:    BOUND: [8000, 9090, 9093, 3000, 3100, 16686, 4317, 4318, 8888]
Report written: /home/huan/Develop/Github/Day23-Lab/Day23-2A202600855-NguyenTienHuan/00-setup/setup-report.json
```

---

## 2. Track 02 — Dashboards & Alerts

### 6 essential panels (screenshot)

Drop `submission/screenshots/dashboard-overview.png`.

### Burn-rate panel

Drop `submission/screenshots/slo-burn-rate.png`.

### Alert fire + resolve

| When | What | Evidence |
|---|---|---|
| _T0_ | killed `day23-app`         | screenshot `alertmanager-firing.png` |
| _T0+90s_ | `ServiceDown` fired   | screenshot `slack-firing.png` |
| _T1_ | restored app              | — |
| _T1+60s_ | alert resolved        | screenshot `slack-resolved.png` |

### One thing surprised me about Prometheus / Grafana

Điều khiến tôi bất ngờ nhất là khả năng tích hợp liền mạch giữa PromQL và LogQL trong cùng một hệ sinh thái Grafana Explore, cho phép nhanh chóng nhảy từ việc quan sát các đỉnh dị thường (spikes) trên biểu đồ metric Prometheus sang việc rà soát từng dòng log cụ thể của Loki chỉ qua các label dùng chung. Bên cạnh đó, kiến trúc pull-based của Prometheus thể hiện sự nhẹ nhàng và mạnh mẽ trong việc tự động gom (aggregating) dữ liệu từ nhiều dịch vụ khác nhau mà không gây áp lực hiệu năng lên ứng dụng bị giám sát.

---

## 3. Track 03 — Tracing & Logs

### One trace screenshot from Jaeger

Drop `submission/screenshots/jaeger-trace.png` showing `embed-text → vector-search → generate-tokens` spans.

### Log line correlated to trace

Paste the log line and the trace_id it links to:

```json
{"model": "llama3-mock", "input_tokens": 4, "output_tokens": 54, "quality": 0.82, "duration_seconds": 0.1564, "trace_id": "10d772268c823d020f9c883414de891f", "event": "prediction served", "level": "info", "timestamp": "2026-06-29T21:07:34.210036Z"}
```

### Tail-sampling math

Hệ thống sử dụng bộ lọc composite tail-sampling để quyết định giữ lại trace:
- Giữ lại 100% trace lỗi (`status_code == ERROR`)
- Giữ lại 100% trace chậm (`duration > 2s`)
- Giữ lại 1% trace khỏe mạnh bình thường (probabilistic 1%)

Nếu hệ thống tạo ra $N$ traces/giây, tỷ lệ lưu giữ thực tế (Retention Rate) được tính bằng công thức:
$$\text{Retention Rate} = P(\text{error}) \times 1.0 + P(\text{slow} \land \neg\text{error}) \times 1.0 + P(\text{healthy}) \times 0.01$$

**Giả sử trong điều kiện vận hành thực tế:**
* Tỷ lệ lỗi hệ thống là 1% ($P(\text{error}) = 0.01$)
* Tỷ lệ phản hồi chậm (> 2s) là 1% ($P(\text{slow}) = 0.01$)
* Tỷ lệ request thành công bình thường là 98% ($P(\text{healthy}) = 0.98$)

**Ta có:**
$$\text{Retention Rate} = 0.01 \times 1.0 + 0.01 \times 1.0 + 0.98 \times 0.01 = 0.0298 \approx 3\%$$

Bộ thu thập OTel Collector chỉ giữ lại khoảng **3%** tổng số trace gửi lên để lưu trữ vào Jaeger. Điều này giúp giảm thiểu dung lượng lưu trữ tới **~97%**, tối ưu hóa chi phí vận hành hạ tầng quan sát đáng kể.

---

## 4. Track 04 — Drift Detection

### PSI scores

Paste `04-drift-detection/reports/drift-summary.json`:

```json
{
  "prompt_length": {
    "psi": 3.461,
    "kl": 1.7982,
    "ks_stat": 0.702,
    "ks_pvalue": 0.0,
    "drift": "yes"
  },
  "embedding_norm": {
    "psi": 0.0187,
    "kl": 0.0324,
    "ks_stat": 0.052,
    "ks_pvalue": 0.133853,
    "drift": "no"
  },
  "response_length": {
    "psi": 0.0162,
    "kl": 0.0178,
    "ks_stat": 0.056,
    "ks_pvalue": 0.086899,
    "drift": "no"
  },
  "response_quality": {
    "psi": 8.8486,
    "kl": 13.5011,
    "ks_stat": 0.941,
    "ks_pvalue": 0.0,
    "drift": "yes"
  }
}
```

### Which test fits which feature?

*   **`prompt_length`**: Chọn **PSI (Population Stability Index)** hoặc **KS (Kolmogorov-Smirnov) Test**. 
    *   *Lý do:* Độ dài prompt là dữ liệu dạng số rời rạc hoặc liên tục (không phân phối chuẩn). KS Test là kiểm định phi tham số cực kỳ nhạy với sự thay đổi hình dạng phân bố (shape) của dữ liệu mà không cần giả định trước phân phối. PSI giúp đánh giá sự thay đổi hành vi người dùng (ví dụ: người dùng đột ngột nhập prompt quá dài).
*   **`embedding_norm`**: Chọn **KS Test**.
    *   *Lý do:* Norm của vector embedding đại diện cho độ lớn ngữ nghĩa của câu đầu vào. Sự dịch chuyển phân phối của trị số liên tục này phản ánh chủ đề người dùng hỏi đã thay đổi. KS Test giúp phát hiện nhanh chóng sự dịch chuyển này trong production.
*   **`response_length`**: Chọn **PSI** hoặc **KS Test**.
    *   *Lý do:* Giúp phát hiện nhanh các thay đổi lớn của câu trả lời từ mô hình (ví dụ: mô hình đột ngột trả lời quá ngắn do lỗi cắt chuỗi hoặc lặp từ vô hạn gây ra câu quá dài).
*   **`response_quality`**: Chọn **PSI** hoặc **KL Divergence**.
    *   *Lý do:* Điểm chất lượng (thường giả lập qua mô hình chấm điểm tự động từ [0, 1] hoặc phân nhóm). PSI là lựa chọn thực tế tốt nhất vì nó trực quan hóa sự suy giảm tỷ lệ phần trăm của các nhóm chất lượng cao (ví dụ: tỷ lệ câu đạt chất lượng tốt giảm từ 80% xuống 30%).

---

## 5. Track 05 — Cross-Day Integration

### Which prior-day metric was hardest to expose? Why?

Metric khó tích hợp nhất là của **Day 20 (Model Serving - Llama.cpp)**. Máy chủ phục vụ inference C++ của Llama.cpp không xuất bản sẵn các metric định dạng Prometheus. Để thu thập dữ liệu, chúng ta phải viết một scraper sidecar bằng Python để gọi endpoint `/stats` định dạng JSON của Llama.cpp, chuyển đổi thủ công các metric này thành các Gauge/Counter của Prometheus client rồi mới expose cổng cho Prometheus server scrape.

---

## 6. The single change that mattered most

> **Grader reads this closest.** What one thing about your stack design — a metric you added, a label you dropped, a panel you reorganized, an alert threshold you tuned — made the biggest difference between "works" and "useful"? Write 1-2 paragraphs. Connect it to a concept from the deck.

Thay đổi quan trọng nhất mang lại giá trị vận hành thực tế trong hệ thống này là việc **cấu hình cơ chế nạp động Slack Webhook cho Alertmanager bằng cách tối ưu hóa tệp docker-compose**. Ban đầu, cấu hình Alertmanager cố gắng nạp biến môi trường bằng cú pháp Go template (`{{ env "VAR" }}`) trong file tĩnh `alertmanager.yml` - điều không được hỗ trợ mặc định bởi Alertmanager v0.27.0 và dẫn tới crash dịch vụ ngay khi khởi động. Việc chuyển đổi cấu hình sang sử dụng biến môi trường tiêu chuẩn `${SLACK_WEBHOOK_URL}` kết hợp với lệnh `sed` tiền xử lý tự động ở entrypoint container đã giúp hệ thống vừa khởi chạy trơn tru, vừa bảo mật được token webhook nhạy cảm.

Điều này kết nối trực tiếp với khái niệm **Separation of Concerns (Phân tách các mối quan tâm)** và **Secret Management** trong MLOps: File cấu hình code (`alertmanager.yml`) cần được giữ tĩnh và độc lập khỏi môi trường chạy để đẩy lên Git an toàn, trong khi các thông tin nhạy cảm chỉ được inject vào ở runtime qua biến môi trường của container. Một hệ thống quan sát (Observability Stack) hữu ích trước hết phải là một hệ thống đáng tin cậy và không làm lộ bí mật bảo mật của doanh nghiệp.
