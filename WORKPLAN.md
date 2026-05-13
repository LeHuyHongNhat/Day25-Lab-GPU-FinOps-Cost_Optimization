# Kế Hoạch Hoàn Thành Day 25 - GPU FinOps & Cost Optimization

## 1. Mục Tiêu

Hoàn thành lab GPU FinOps theo đúng flow của repo hiện tại:

- Chạy bộ mock GPU FinOps services bằng Docker trên Mac M4 Pro.
- Mở tunnel từ máy local để Kaggle/Colab gọi được API gateway.
- Chạy notebook `notebook/gpu_finops_lab.ipynb` trên môi trường có GPU NVIDIA.
- Hoàn thiện các cell TODO ở Part 8.5.
- Thu thập screenshot, chart PNG, notebook output và report để nộp GitHub/LMS.

## 2. Kết Luận Từ Codebase

Repo gồm 2 phần chính:

- Local backend bằng Docker Compose:
  - `gateway`: cổng chính `http://localhost:8000`.
  - `gpu-node-manager`: mock multi-node GPU cluster, port `8001`.
  - `billing-api`: mock billing/cost API, port `8002`.
  - `spot-manager`: mock spot/preemptible instances, port `8003`.
  - `autoscaler`: KEDA-like autoscaling simulator, port `8004`.
  - `cost-tracker`: OpenCost-like cost allocation, port `8005`.
- Notebook:
  - `notebook/gpu_finops_lab.ipynb` chạy trên Kaggle/Colab.
  - Part 1-7 gọi API về local backend thông qua tunnel.
  - Part 8 cần GPU NVIDIA thật để train ResNet-18 và đo CUDA metrics.
  - Part 8.5 hiện còn TODO ở Cell 27-31, cần tự implement trước khi nộp.

Điểm cần chú ý cho Mac M4 Pro:

- Mac M4 Pro không có CUDA/NVIDIA GPU, vì vậy không dùng để chạy Part 8 real GPU.
- Mac M4 Pro phù hợp để chạy Docker services local và tunnel.
- Nếu Docker Desktop đang chạy Apple Silicon image mặc định, các service Python trong repo vẫn phù hợp vì được build từ Dockerfile đơn giản.
- `nvidia-smi`, `pynvml`, CUDA telemetry chỉ có ý nghĩa trên Kaggle/Colab GPU, không phải trên Mac.

## 3. Flow Tổng Thể

Thứ tự làm nên đi theo 8 giai đoạn:

1. Chuẩn bị Mac M4 Pro.
2. Start Docker services local.
3. Test gateway và endpoints.
4. Mở tunnel.
5. Upload notebook lên Kaggle/Colab và cấu hình GPU.
6. Chạy Part 1-8, chụp ảnh và lưu chart.
7. Implement và chạy Part 8.5.
8. Gom bài nộp, kiểm tra đủ bằng chứng, push GitHub.

## 4. Giai Đoạn 1 - Chuẩn Bị Trên Mac M4 Pro

### Việc Cần Làm

1. Mở Docker Desktop.
2. Cài tunnel tool, ưu tiên `cloudflared`.
3. Đảm bảo terminal đang ở đúng repo.

### Lệnh

```bash
cd /Users/AI_ThucChien/Day25-Lab-GPU-FinOps-Cost_Optimization
brew install cloudflare/cloudflare/cloudflared
open -a Docker
```

### Kết Quả Mong Đợi

- Docker Desktop chạy ổn định.
- `cloudflared` có thể gọi từ terminal.
- Repo path đúng là `/Users/AI_ThucChien/Day25-Lab-GPU-FinOps-Cost_Optimization`.

### Checklist

- [x] Docker Desktop đã mở.
- [x] `cloudflared --version` chạy được.
- [x] Terminal đang ở repo Day25.

## 5. Giai Đoạn 2 - Start Docker Services Local

### Việc Cần Làm

Chạy script macOS có sẵn trong repo.

### Lệnh

```bash
./run.sh start
```

### Kết Quả Mong Đợi

Script in ra:

```text
Gateway is UP at http://localhost:8000
ALL SERVICES RUNNING
```

Các port mong đợi:

| Service | URL |
|---|---|
| Gateway | `http://localhost:8000` |
| GPU Node Manager | `http://localhost:8001` |
| Billing API | `http://localhost:8002` |
| Spot Manager | `http://localhost:8003` |
| Autoscaler | `http://localhost:8004` |
| Cost Tracker | `http://localhost:8005` |

### Nếu Lỗi

- Docker chưa chạy: mở Docker Desktop và chạy lại.
- Port bị chiếm: kiểm tra bằng `lsof -i :8000`, sau đó xử lý process đang chiếm port.
- Service không lên: xem log bằng `./run.sh logs gateway` hoặc `./run.sh logs`.

### Checklist

- [x] `./run.sh start` chạy thành công.
- [x] Gateway trả response ở `http://localhost:8000`.
- [x] Docker Compose có đủ 6 services.

## 6. Giai Đoạn 3 - Test Endpoints

### Việc Cần Làm

Chạy test endpoints trước khi mở tunnel.

### Lệnh

```bash
./run.sh test
```

### Kết Quả Mong Đợi

Các endpoint sau đều `[OK]`:

- `GET /`
- `GET /cluster/nodes`
- `GET /cluster/metrics`
- `GET /billing/pricing`
- `GET /spot/pricing`
- `GET /autoscaler/policy`
- `GET /cost/dashboard`

### Checklist

- [x] `GET /` OK.
- [x] Cluster endpoints OK.
- [x] Billing endpoint OK.
- [x] Spot endpoint OK.
- [x] Autoscaler endpoint OK.
- [x] Cost dashboard endpoint OK.

## 7. Giai Đoạn 4 - Mở Tunnel Cho Kaggle/Colab

### Việc Cần Làm

Mở tunnel từ local gateway `localhost:8000` ra public HTTPS URL.

### Lệnh

```bash
./run.sh tunnel
```

### Kết Quả Mong Đợi

Script in ra URL dạng:

```text
https://<random-subdomain>.trycloudflare.com
```

Copy URL này để đưa vào notebook:

```python
GATEWAY_URL = "https://<random-subdomain>.trycloudflare.com"
```

### Lưu Ý

- Tunnel phải tiếp tục chạy trong lúc notebook Kaggle/Colab đang gọi API.
- URL `trycloudflare.com` thay đổi mỗi lần chạy lại tunnel.
- Nếu tunnel không hiện URL, xem `.tunnel.log`:

```bash
grep 'trycloudflare' .tunnel.log
```

### Checklist

- [x] Tunnel đang chạy.
- [x] Đã copy public HTTPS URL.
- [x] Không tắt terminal tunnel trong lúc chạy notebook.

## 8. Giai Đoạn 5 - Chuẩn Bị Notebook Trên Kaggle/Colab

### Việc Cần Làm

1. Vào Kaggle hoặc Colab.
2. Upload `notebook/gpu_finops_lab.ipynb`.
3. Bật GPU accelerator.
4. Điền thông tin sinh viên.
5. Cấu hình `GATEWAY_URL`.

### Khuyến Nghị Môi Trường

Ưu tiên Kaggle:

- Accelerator: `GPU T4 x2` nếu có.
- Nếu không có T4 x2, dùng `P100` hoặc GPU miễn phí khả dụng.

### Cell Cần Sửa

Cell 2.5:

```python
STUDENT_NAME = "Ten Cua Ban"
STUDENT_ID = "MSSV Cua Ban"
```

Cell 2:

```python
GATEWAY_URL = "https://<random-subdomain>.trycloudflare.com"
```

### Kết Quả Mong Đợi

- Notebook kết nối được gateway.
- Header thông tin sinh viên hiển thị trong notebook.
- Cell kết nối gateway báo thành công.

### Checklist

- [x] Notebook đã upload lên Kaggle/Colab.
- [x] GPU accelerator đã bật.
- [x] Cell 2.5 có tên và MSSV thật.
- [x] Cell 2 có tunnel URL mới nhất.
- [x] Header sinh viên hiển thị rõ.
- [x] Notebook connect được gateway.

## 9. Giai Đoạn 6 - Chạy Part 1 Đến Part 7

### Part 1 - GPU Cluster Monitoring

Việc cần làm:

- Chạy Cell 3 và Cell 4.
- Quan sát mock GPU nodes, utilization, memory, power draw.

Kết quả mong đợi:

- Cell 3 hiển thị danh sách node và GPU.
- Cell 4 hiển thị tổng số GPU, busy GPUs, idle GPUs, average utilization.

Ảnh cần lưu:

- `screenshots/part1_cluster_monitoring.png`
- `screenshots/part1_cluster_metrics.png`

Checklist:

- [ ] Cell 3 chạy không lỗi.
- [ ] Cell 4 chạy không lỗi.
- [ ] Screenshot có header sinh viên.

### Part 2 - Workload Submission & Cost Tracking

Việc cần làm:

- Chạy Cell 5 và Cell 6.
- Submit nhiều workload vào mock cluster.
- Record billing events.

Kết quả mong đợi:

- Workload được assigned vào GPU hoặc queued nếu hết GPU.
- Billing summary có total cost, budget, cost by GPU type.

Ảnh cần lưu:

- `screenshots/part2_workload_submission.png`
- `screenshots/part2_billing_summary.png`

Checklist:

- [ ] Cell 5 có workload status.
- [ ] Cell 6 có billing summary.
- [ ] Screenshot có header sinh viên.

### Part 3 - Spot Instance Management

Việc cần làm:

- Chạy Cell 7, Cell 8, Cell 9.
- Xem spot pricing.
- Request spot instances.
- Simulate preemption.

Kết quả mong đợi:

- Bảng giá spot theo GPU type.
- Spot request có accepted/rejected status.
- Savings report có spot cost, on-demand equivalent, savings percentage.

Ảnh cần lưu:

- `screenshots/part3_spot_pricing.png`
- `screenshots/part3_spot_request.png`
- `screenshots/part3_spot_preemption.png`

Checklist:

- [ ] Cell 7 hiển thị spot pricing.
- [ ] Cell 8 request spot thành công hoặc có status rõ ràng.
- [ ] Cell 9 có savings report.
- [ ] Screenshot có header sinh viên.

### Part 4 - Autoscaling

Việc cần làm:

- Chạy Cell 10 và Cell 11.
- Xem/update policy.
- Trigger autoscaler evaluation.

Kết quả mong đợi:

- Policy trước/sau update hiển thị rõ threshold, min/max nodes, cooldown.
- 5 evaluation cycles có action hoặc reason.

Ảnh cần lưu:

- `screenshots/part4_autoscaler_policy.png`
- `screenshots/part4_autoscaler_evaluation.png`

Checklist:

- [ ] Cell 10 hiển thị policy trước/sau.
- [ ] Cell 11 chạy đủ evaluation cycles.
- [ ] Screenshot có header sinh viên.

### Part 5 - Cost Analysis & Optimization

Việc cần làm:

- Chạy Cell 12, Cell 13, Cell 14, Cell 15.
- Tạo cost snapshots.
- Xem waste report.
- Lấy optimization recommendations.
- Xem dashboard tổng hợp.

Kết quả mong đợi:

- Cell 12 có 5 snapshots với total/idle/active cost.
- Cell 13 có waste percentage và potential savings.
- Cell 14 có recommendations.
- Cell 15 có dashboard tổng hợp cluster, billing, waste.

Ảnh cần lưu:

- `screenshots/part5_cost_snapshots.png`
- `screenshots/part5_waste_report.png`
- `screenshots/part5_recommendations.png`
- `screenshots/part5_dashboard.png`

Checklist:

- [ ] Cell 12 có đủ 5 snapshots.
- [ ] Cell 13 có waste analysis.
- [ ] Cell 14 có recommendations.
- [ ] Cell 15 có dashboard.
- [ ] Screenshot có header sinh viên.

### Part 6 - Visualization

Việc cần làm:

- Chạy Cell 16 và Cell 17.
- Tạo chart từ mock FinOps data.

Kết quả mong đợi:

- Cell 16 tạo cost breakdown chart.
- Cell 17 tạo time-series cost tracking chart.

File chart cần lưu:

- `generated_charts/finops_cost_breakdown.png`
- `generated_charts/finops_timeseries.png`

Ảnh cần lưu:

- `screenshots/part6_cost_breakdown_viz.png`
- `screenshots/part6_timeseries_viz.png`

Checklist:

- [ ] Cell 16 tạo `finops_cost_breakdown.png`.
- [ ] Cell 17 tạo `finops_timeseries.png`.
- [ ] Screenshot có header sinh viên.

### Part 7 - Complete FinOps Workflow

Việc cần làm:

- Chạy Cell 18.
- Thực hiện full workflow: submit workloads, monitor, detect waste, autoscale, optimize cost.

Kết quả mong đợi:

- Output hiển thị đủ 7 steps của optimization workflow.
- Có final metrics/dashboard sau workflow.

Ảnh cần lưu:

- `screenshots/part7_full_workflow.png`

Checklist:

- [ ] Cell 18 chạy không lỗi.
- [ ] Output có đủ các step của workflow.
- [ ] Screenshot có header sinh viên.

## 10. Giai Đoạn 7 - Chạy Part 8 Real GPU Workload

### Vì Sao Không Chạy Trên Mac M4 Pro

Part 8 dùng CUDA/NVIDIA telemetry:

- Detect GPU bằng CUDA/PyTorch.
- Thu GPU metrics qua NVIDIA tooling.
- Train ResNet-18 trên GPU.
- So sánh FP32 với AMP.

Mac M4 Pro không có CUDA, nên phải chạy trên Kaggle/Colab GPU.

### Việc Cần Làm

- Chạy Cell 19 đến Cell 26 trên Kaggle/Colab.
- Đảm bảo tunnel local vẫn chạy.
- Nếu dataset hoặc package download chậm, giữ notebook session active.

### Kết Quả Mong Đợi

- Cell 19 detect được GPU NVIDIA.
- Cell 20 có diagnostic GPU metrics.
- Cell 21 chuẩn bị CIFAR-10 và ResNet-18.
- Cell 22 train FP32 thành công.
- Cell 23 train AMP thành công.
- Cell 24 so sánh FP32 vs AMP bằng bảng và chart.
- Cell 25 report real GPU cost về gateway.
- Cell 26 tạo telemetry và cost per epoch charts.

Ảnh cần lưu:

- `screenshots/part8_gpu_detection.png`
- `screenshots/part8_gpu_metrics_diagnostic.png`
- `screenshots/part8_fp32_summary.png`
- `screenshots/part8_amp_summary.png`
- `screenshots/part8_fp32_vs_amp_comparison.png`
- `screenshots/part8_real_gpu_cost_report.png`

File chart cần lưu:

- `generated_charts/real_gpu_comparison.png`
- `generated_charts/real_gpu_telemetry.png` nếu monitor có data.
- `generated_charts/cost_per_epoch.png`

Checklist:

- [ ] Kaggle/Colab detect GPU thật.
- [ ] FP32 training hoàn tất.
- [ ] AMP training hoàn tất.
- [ ] Comparison cho thấy time/cost/savings.
- [ ] Real GPU cost report gửi về gateway thành công.
- [ ] Charts đã download.
- [ ] Screenshot có header sinh viên.

## 11. Giai Đoạn 8 - Implement Part 8.5 Advanced GPU Cost Optimization

Notebook hiện còn TODO ở Cell 27-31. Phần này phải hoàn thiện trước khi nộp, không chỉ chạy output TODO.

### Cell 27 - Multi-GPU Cost Analysis

Việc cần làm:

- Implement `analyze_multi_gpu_cost`.
- Dùng `GPU_PRICING` để lấy giá GPU.
- Nếu không truyền scaling factors, dùng scaling sub-linear thực tế, ví dụ:
  - 1 GPU: speedup `1.0`.
  - 2 GPU: speedup `1.8`.
  - 4 GPU: speedup `3.2`.
  - 8 GPU: speedup `5.6`.
- Tính:
  - training time theo từng GPU count.
  - total cost.
  - speedup.
  - scaling efficiency.
  - cost-performance ratio.
  - optimal GPU count theo cost efficiency.

Kết quả mong đợi:

- Bảng so sánh 1, 2, 4, 8 GPU.
- Kết luận GPU count tối ưu.

Ảnh cần lưu:

- `screenshots/part85_multi_gpu_analysis.png`

Chart cần lưu:

- `generated_charts/multi_gpu_scaling.png`

Checklist:

- [ ] Function không còn `pass`.
- [ ] Có bảng output rõ ràng.
- [ ] Có optimal GPU count.
- [ ] Có chart `multi_gpu_scaling.png`.

### Cell 28 - Project Cost Forecasting

Việc cần làm:

- Implement `forecast_project_cost`.
- Tính cost từng phase:
  - `duration_hours * gpu_count * price_per_hour`.
- Tính uncertainty theo `uncertainty_pct`.
- Tính base total.
- Tính contingency buffer.
- Tính confidence interval hoặc best/expected/worst case.

Kết quả mong đợi:

- Phase-by-phase breakdown.
- Base cost.
- Contingency.
- Forecast total.
- Best case, expected case, worst case hoặc confidence interval.

Ảnh cần lưu:

- `screenshots/part85_project_forecast.png`

Chart cần lưu:

- `generated_charts/project_forecast.png`

Checklist:

- [ ] Function không còn `pass`.
- [ ] Có bảng phase cost.
- [ ] Có total forecast.
- [ ] Có confidence/best/worst range.
- [ ] Có chart `project_forecast.png`.

### Cell 29 - Optimization Opportunity Analysis

Việc cần làm:

- Implement `analyze_optimization_opportunities`.
- Tính current baseline cost.
- Chấm điểm từng strategy theo savings, effort, risk.
- Sắp xếp ưu tiên strategy.
- Tính cumulative savings theo thứ tự triển khai.
- Tạo roadmap: quick wins trước, high-effort sau.

Gợi ý scoring:

- Effort LOW/MEDIUM/HIGH tương ứng `1/2/3`.
- Risk LOW/MEDIUM/HIGH tương ứng `1/2/3`.
- Priority score có thể là `savings_pct / (effort_score * risk_score)`.

Kết quả mong đợi:

- Ranked list optimization strategies.
- Savings USD và savings percentage.
- Risk/effort rõ ràng.
- Cumulative savings sau khi apply nhiều strategy.
- Roadmap triển khai.

Ảnh cần lưu:

- `screenshots/part85_optimization_analysis.png`

Chart cần lưu:

- `generated_charts/optimization_roadmap.png`

Checklist:

- [ ] Function không còn `pass`.
- [ ] Có ranked recommendations.
- [ ] Có cumulative savings.
- [ ] Có roadmap.
- [ ] Có chart `optimization_roadmap.png`.

### Cell 30 - Integrated Cost Dashboard

Việc cần làm:

- Implement `create_advanced_finops_dashboard`.
- Kết hợp kết quả Cell 27-29.
- Tạo dashboard 2x3 hoặc 3x2 bằng matplotlib.

Dashboard nên có 6 panel:

- Multi-GPU cost curve.
- Scaling efficiency.
- Project forecast với confidence bands.
- Phase cost breakdown.
- Optimization priority matrix.
- Cumulative savings roadmap.

Kết quả mong đợi:

- Một figure tổng hợp có title, axis labels, legends rõ ràng.
- Lưu file `advanced_finops_dashboard.png`.

Ảnh cần lưu:

- `screenshots/part85_integrated_dashboard.png`

Chart cần lưu:

- `generated_charts/advanced_finops_dashboard.png`

Checklist:

- [ ] Function không còn `pass`.
- [ ] Dashboard có đủ 6 panel.
- [ ] Chart có labels/legend.
- [ ] File `advanced_finops_dashboard.png` được tạo.

### Cell 31 - Challenge Exercise

Việc cần làm:

- Tính baseline cost cho scenario LLM fine-tuning:
  - 8x A100.
  - 200 giờ.
  - FP32.
  - on-demand.
  - budget `5000 USD`.
- Dùng kết quả multi-GPU analysis để chọn GPU count hợp lý.
- Chọn 3-5 optimization strategies.
- Tính cumulative savings.
- Forecast final cost với uncertainty.
- Kiểm tra có dưới budget và đáp ứng constraints không.
- Viết justification rõ ràng.

Kết quả mong đợi:

- Baseline cost.
- Strategy đề xuất.
- Final expected cost.
- Confidence interval.
- Budget verdict: under/over budget.
- Risk và trade-off.

Ảnh cần lưu:

- `screenshots/part85_challenge_strategy.png`

Checklist:

- [ ] Có baseline cost.
- [ ] Có strategy cụ thể.
- [ ] Có final forecast.
- [ ] Có kết luận under/over budget.
- [ ] Có justification.

## 12. Screenshot Matrix - Chụp Cell Nào, Lưu File Nào

Mục này là checklist chụp ảnh chính xác theo notebook. Khi chụp, mỗi ảnh phải thấy rõ:

- Header sinh viên ở đầu output cell: `GPU FinOps Lab - Student Information`, họ tên, MSSV.
- Tên cell hoặc tiêu đề output, ví dụ `Cell 3: View Cluster Nodes`, `EXPERIMENT 1`, `EXERCISE 8.5.1`.
- Kết quả chính của cell, không chụp mỗi code.
- Nếu output dài, ưu tiên chụp phần summary cuối cell; nếu cần, chụp thêm ảnh phụ nhưng giữ tên file chính theo bảng.

### Quy Tắc Chụp Ảnh

- Chạy cell xong mới chụp.
- Ảnh nên bao gồm cả header sinh viên và output chính trong cùng khung hình.
- Nếu header bị cuộn khỏi màn hình, chạy lại cell đó vì notebook đã được sửa để hiển thị header ở đầu mọi code cell.
- Không dùng ảnh crop quá sát làm mất tên metric hoặc đơn vị cost.
- Không clear output trước khi download notebook final.
- Với biểu đồ, cần nộp cả screenshot cell và file PNG được `plt.savefig(...)` tạo ra.

### Setup Và Connectivity

Các ảnh này không bắt buộc trong `SUBMISSION.md`, nhưng nên giữ để debug nếu grader hỏi.

| Thứ tự | Cell | Screenshot file đề xuất | Bắt buộc thấy trong ảnh | Ghi chú |
|---|---:|---|---|---|
| 0.1 | Cell 2.5 | `screenshots/setup_student_header.png` | Họ tên, MSSV, header sinh viên | Chỉ cần lưu nếu muốn có bằng chứng setup |
| 0.2 | Cell 2 | `screenshots/setup_gateway_connection.png` | `Connected to GPU FinOps Lab Gateway`, JSON service/routes | Dùng để chứng minh tunnel local hoạt động |

### Part 1 - GPU Cluster Monitoring

| Thứ tự | Cell | Screenshot file bắt buộc | Bắt buộc thấy trong ảnh | Kết quả mong muốn |
|---|---:|---|---|---|
| 1.1 | Cell 3 | `screenshots/part1_cluster_monitoring.png` | Header sinh viên, danh sách `node-*`, GPU id/type/status/util/memory/power | Thấy nhiều node và GPU, có GPU type như T4/A100/V100 |
| 1.2 | Cell 4 | `screenshots/part1_cluster_metrics.png` | Header sinh viên, `Total GPUs`, `Busy GPUs`, `Idle GPUs`, `Average Utilization`, `Total Power Draw` | Metrics tổng hợp có số liệu rõ ràng |

### Part 2 - Workload Submission & Cost Tracking

| Thứ tự | Cell | Screenshot file bắt buộc | Bắt buộc thấy trong ảnh | Kết quả mong muốn |
|---|---:|---|---|---|
| 2.1 | Cell 5 | `screenshots/part2_workload_submission.png` | Header sinh viên, từng workload id, status assigned/queued, node/GPU nếu assigned | Workload đã được submit vào cluster |
| 2.2 | Cell 6 | `screenshots/part2_billing_summary.png` | Header sinh viên, từng billing event, `Total Cost`, `Budget`, `Cost by GPU Type`, `Total Savings` nếu có | Billing records đã được ghi và summary có cost |

### Part 3 - Spot Instance Management

| Thứ tự | Cell | Screenshot file bắt buộc | Bắt buộc thấy trong ảnh | Kết quả mong muốn |
|---|---:|---|---|---|
| 3.1 | Cell 7 | `screenshots/part3_spot_pricing.png` | Header sinh viên, bảng `GPU Type`, `On-Demand`, `Spot Price`, `Discount`, `Availability` | Giá spot hiển thị đầy đủ |
| 3.2 | Cell 8 | `screenshots/part3_spot_request.png` | Header sinh viên, instance id, GPU type, max bid, status accepted/rejected | Spot request có kết quả rõ |
| 3.3 | Cell 9 | `screenshots/part3_spot_preemption.png` | Header sinh viên, preemption event, `Spot cost`, `On-demand equivalent`, `Savings`, `Savings %` | Thấy tác động preemption và savings report |

### Part 4 - Autoscaling

| Thứ tự | Cell | Screenshot file bắt buộc | Bắt buộc thấy trong ảnh | Kết quả mong muốn |
|---|---:|---|---|---|
| 4.1 | Cell 10 | `screenshots/part4_autoscaler_policy.png` | Header sinh viên, current policy, updated policy, thresholds, min/max nodes, cooldown | Policy được đọc và update thành công |
| 4.2 | Cell 11 | `screenshots/part4_autoscaler_evaluation.png` | Header sinh viên, evaluation decision, 5 cycles, action/reason | Autoscaler chạy đủ cycles và có quyết định |

### Part 5 - Cost Analysis & Optimization

| Thứ tự | Cell | Screenshot file bắt buộc | Bắt buộc thấy trong ảnh | Kết quả mong muốn |
|---|---:|---|---|---|
| 5.1 | Cell 12 | `screenshots/part5_cost_snapshots.png` | Header sinh viên, 5 dòng `Snapshot`, `Total`, `Idle`, `Waste` | Có đủ 5 snapshots |
| 5.2 | Cell 13 | `screenshots/part5_waste_report.png` | Header sinh viên, `Waste Report`, idle cost, total cost, waste percentage, monthly savings | Waste analysis có số liệu |
| 5.3 | Cell 14 | `screenshots/part5_recommendations.png` | Header sinh viên, danh sách recommendations, severity/description/savings nếu có | Có gợi ý optimize cost |
| 5.4 | Cell 15 | `screenshots/part5_dashboard.png` | Header sinh viên, dashboard gồm cluster, billing, waste, recommendations | Dashboard tổng hợp đủ thông tin |

### Part 6 - Visualization

| Thứ tự | Cell | Screenshot file bắt buộc | File PNG phải download | Bắt buộc thấy trong ảnh | Kết quả mong muốn |
|---|---:|---|---|---|---|
| 6.1 | Cell 16 | `screenshots/part6_cost_breakdown_viz.png` | `generated_charts/finops_cost_breakdown.png` | Header sinh viên, biểu đồ cost breakdown 3 panels/subplots | Chart hiển thị cost by GPU/spot/savings hoặc tương đương |
| 6.2 | Cell 17 | `screenshots/part6_timeseries_viz.png` | `generated_charts/finops_timeseries.png` | Header sinh viên, time-series/stackplot, waste percentage line | Chart time-series cost tracking được tạo |

### Part 7 - Complete FinOps Workflow

| Thứ tự | Cell | Screenshot file bắt buộc | Bắt buộc thấy trong ảnh | Kết quả mong muốn |
|---|---:|---|---|---|
| 7.1 | Cell 18 | `screenshots/part7_full_workflow.png` | Header sinh viên, `FULL FINOPS OPTIMIZATION WORKFLOW`, các step monitor/submit/autoscale/cost/recommendations/spot/cleanup | Output thể hiện full cycle end-to-end |

### Part 8 - Real GPU Workload

| Thứ tự | Cell | Screenshot file bắt buộc | File PNG phải download | Bắt buộc thấy trong ảnh | Kết quả mong muốn |
|---|---:|---|---|---|---|
| 8.1 | Cell 19 | `screenshots/part8_gpu_detection.png` | Không có | Header sinh viên, GPU name, memory, type, compute capability, PyTorch, CUDA build, `CUDA Probe Passed` | GPU NVIDIA thật và CUDA chạy được |
| 8.2 | Cell 20 | `screenshots/part8_gpu_metrics_diagnostic.png` | Không có | Header sinh viên, `GPU METRICS DIAGNOSTIC`, pynvml status, memory, power/temp nếu có | GPU metrics collector hoạt động |
| 8.3 | Cell 21 | `screenshots/part8_dataset_model_ready.png` | Không bắt buộc | Header sinh viên, CIFAR-10 train/test size, batches per epoch, training device | Không bắt buộc trong submission gốc nhưng nên giữ |
| 8.4 | Cell 22 | `screenshots/part8_fp32_summary.png` | Không có | Header sinh viên, `EXPERIMENT 1: FP32 Training`, epoch logs, `FP32 Summary`, total time, peak memory, estimated cost | FP32 train hoàn tất |
| 8.5 | Cell 23 | `screenshots/part8_amp_summary.png` | Không có | Header sinh viên, `EXPERIMENT 2: Mixed Precision`, epoch logs, `AMP Summary`, total time, peak memory, estimated cost | AMP train hoàn tất |
| 8.6 | Cell 24 | `screenshots/part8_fp32_vs_amp_comparison.png` | `generated_charts/real_gpu_comparison.png` | Header sinh viên, comparison table, speedup, cost saving, 3 charts | So sánh FP32 vs AMP rõ ràng |
| 8.7 | Cell 25 | `screenshots/part8_real_gpu_cost_report.png` | Không có | Header sinh viên, FP32/AMP workload reported, updated billing summary, final dashboard | Cost report gửi về gateway thành công |
| 8.8 | Cell 26 | `screenshots/part8_real_gpu_telemetry.png` | `generated_charts/real_gpu_telemetry.png`, `generated_charts/cost_per_epoch.png` | Header sinh viên, telemetry charts nếu có, cost per epoch chart | Nếu không có telemetry sample, vẫn phải có `cost_per_epoch.png` |

Ghi chú cho Part 8:

- Nếu Cell 19 báo lỗi CUDA kernel/P100 không tương thích, đổi runtime sang T4/L4/A100, restart runtime, chạy lại từ Cell 1.
- Nếu `real_gpu_telemetry.png` không được tạo vì monitor không có sample, ghi rõ trong report và vẫn nộp `cost_per_epoch.png`.
- Screenshot bắt buộc theo `SUBMISSION.md` không nhắc Cell 21 và Cell 26 screenshot riêng, nhưng nên lưu thêm để có evidence đầy đủ.

### Part 8.5 - Advanced GPU Cost Optimization

| Thứ tự | Cell | Screenshot file bắt buộc | File PNG phải download | Bắt buộc thấy trong ảnh | Kết quả mong muốn |
|---|---:|---|---|---|---|
| 8.5.1 | Cell 27 | `screenshots/part85_multi_gpu_analysis.png` | `generated_charts/multi_gpu_scaling.png` | Header sinh viên, `EXERCISE 8.5.1`, table 1/2/4/8 GPUs, lowest cost, fastest run, best cost/performance | Multi-GPU analysis hoàn chỉnh |
| 8.5.2 | Cell 28 | `screenshots/part85_project_forecast.png` | `generated_charts/project_forecast.png` | Header sinh viên, phase-by-phase forecast, base total, contingency, expected total, 95% interval | Project forecast có uncertainty |
| 8.5.3 | Cell 29 | `screenshots/part85_optimization_analysis.png` | `generated_charts/optimization_roadmap.png` | Header sinh viên, ranked strategies, baseline cost, cumulative roadmap, total savings | Prioritization rõ savings/effort/risk |
| 8.5.4 | Cell 30 | `screenshots/part85_integrated_dashboard.png` | `generated_charts/advanced_finops_dashboard.png` | Header sinh viên, dashboard 6 panels: cost curve, speedup, forecast, phase breakdown, priority matrix, savings roadmap | Dashboard tổng hợp đầy đủ |
| 8.5.5 | Cell 31 | `screenshots/part85_challenge_strategy.png` | Không có | Header sinh viên, challenge scenario, baseline cost, selected compute plan, strategy roadmap, final forecast, budget verdict, justification | Có strategy under/over budget và giải thích |

### Danh Sách File Ảnh Cuối Cùng Cần Có

Screenshots bắt buộc/nên có:

- [ ] `screenshots/part1_cluster_monitoring.png`
- [ ] `screenshots/part1_cluster_metrics.png`
- [ ] `screenshots/part2_workload_submission.png`
- [ ] `screenshots/part2_billing_summary.png`
- [ ] `screenshots/part3_spot_pricing.png`
- [ ] `screenshots/part3_spot_request.png`
- [ ] `screenshots/part3_spot_preemption.png`
- [ ] `screenshots/part4_autoscaler_policy.png`
- [ ] `screenshots/part4_autoscaler_evaluation.png`
- [ ] `screenshots/part5_cost_snapshots.png`
- [ ] `screenshots/part5_waste_report.png`
- [ ] `screenshots/part5_recommendations.png`
- [ ] `screenshots/part5_dashboard.png`
- [ ] `screenshots/part6_cost_breakdown_viz.png`
- [ ] `screenshots/part6_timeseries_viz.png`
- [ ] `screenshots/part7_full_workflow.png`
- [ ] `screenshots/part8_gpu_detection.png`
- [ ] `screenshots/part8_gpu_metrics_diagnostic.png`
- [ ] `screenshots/part8_fp32_summary.png`
- [ ] `screenshots/part8_amp_summary.png`
- [ ] `screenshots/part8_fp32_vs_amp_comparison.png`
- [ ] `screenshots/part8_real_gpu_cost_report.png`
- [ ] `screenshots/part8_real_gpu_telemetry.png`
- [ ] `screenshots/part85_multi_gpu_analysis.png`
- [ ] `screenshots/part85_project_forecast.png`
- [ ] `screenshots/part85_optimization_analysis.png`
- [ ] `screenshots/part85_integrated_dashboard.png`
- [ ] `screenshots/part85_challenge_strategy.png`

Generated chart files:

- [ ] `generated_charts/finops_cost_breakdown.png`
- [ ] `generated_charts/finops_timeseries.png`
- [ ] `generated_charts/real_gpu_comparison.png`
- [ ] `generated_charts/real_gpu_telemetry.png` nếu có telemetry data
- [ ] `generated_charts/cost_per_epoch.png`
- [ ] `generated_charts/multi_gpu_scaling.png`
- [ ] `generated_charts/project_forecast.png`
- [ ] `generated_charts/optimization_roadmap.png`
- [ ] `generated_charts/advanced_finops_dashboard.png`

## 13. Cấu Trúc Thư Mục Nộp Bài

Tạo thư mục nộp bài theo format:

```text
[StudentName]_GPU_FinOps_Submission/
├── report.pdf
├── screenshots/
│   ├── part1_cluster_monitoring.png
│   ├── part1_cluster_metrics.png
│   ├── part2_workload_submission.png
│   ├── part2_billing_summary.png
│   ├── part3_spot_pricing.png
│   ├── part3_spot_request.png
│   ├── part3_spot_preemption.png
│   ├── part4_autoscaler_policy.png
│   ├── part4_autoscaler_evaluation.png
│   ├── part5_cost_snapshots.png
│   ├── part5_waste_report.png
│   ├── part5_recommendations.png
│   ├── part5_dashboard.png
│   ├── part6_cost_breakdown_viz.png
│   ├── part6_timeseries_viz.png
│   ├── part7_full_workflow.png
│   ├── part8_gpu_detection.png
│   ├── part8_gpu_metrics_diagnostic.png
│   ├── part8_fp32_summary.png
│   ├── part8_amp_summary.png
│   ├── part8_fp32_vs_amp_comparison.png
│   ├── part8_real_gpu_cost_report.png
│   ├── part85_multi_gpu_analysis.png
│   ├── part85_project_forecast.png
│   ├── part85_optimization_analysis.png
│   ├── part85_integrated_dashboard.png
│   └── part85_challenge_strategy.png
├── generated_charts/
│   ├── finops_cost_breakdown.png
│   ├── finops_timeseries.png
│   ├── real_gpu_comparison.png
│   ├── real_gpu_telemetry.png
│   ├── cost_per_epoch.png
│   ├── multi_gpu_scaling.png
│   ├── project_forecast.png
│   ├── optimization_roadmap.png
│   └── advanced_finops_dashboard.png
└── notebook/
    └── gpu_finops_lab.ipynb
```

Nếu `real_gpu_telemetry.png` không được tạo vì monitor không thu được sample, ghi rõ trong report.

## 14. Nội Dung Report Nên Có

Nếu cần viết `report.pdf`, nên có các phần:

### 1. Giới Thiệu

- Mục tiêu GPU FinOps.
- Kiến trúc lab: local Docker gateway + remote GPU notebook.
- Lý do dùng Kaggle/Colab cho real GPU trên Mac M4 Pro.

### 2. Kết Quả Part 1-7

- Cluster monitoring: số GPU, trạng thái idle/busy, utilization.
- Workload và billing: workload cost, cost by GPU type.
- Spot: discount, preemption, savings.
- Autoscaling: policy, scale decision.
- Cost tracker: idle waste, recommendations.
- Visualization: breakdown và time-series insight.
- Full workflow: trước/sau optimization.

### 3. Kết Quả Part 8

- GPU thật được dùng trên Kaggle/Colab.
- So sánh FP32 vs AMP:
  - training time.
  - accuracy/loss nếu có.
  - cost per epoch.
  - cost savings.
- Nhận xét GPU utilization và telemetry.

### 4. Kết Quả Part 8.5

- Multi-GPU scaling efficiency.
- Project cost forecast.
- Ranked optimization roadmap.
- Integrated dashboard.
- Challenge strategy và budget verdict.

### 5. Kết Luận

- Kỹ năng FinOps đã học.
- Chiến lược tiết kiệm cost hiệu quả nhất.
- Trade-off giữa performance, cost, risk, preemption.

## 15. Checklist Nộp Bài Cuối Cùng

### Local Backend

- [ ] Docker services chạy thành công.
- [ ] `./run.sh test` tất cả endpoint OK.
- [ ] Tunnel URL hoạt động trong lúc chạy notebook.

### Notebook

- [ ] Đã điền `STUDENT_NAME`.
- [ ] Đã điền `STUDENT_ID`.
- [ ] Đã thay `GATEWAY_URL`.
- [ ] Notebook có output đầy đủ, không clear output.
- [ ] Part 1-7 chạy không lỗi.
- [ ] Part 8 chạy trên GPU NVIDIA thật.
- [ ] Part 8.5 không còn TODO/pass trong output.

### Screenshots

- [ ] Mọi screenshot có header thông tin sinh viên.
- [ ] Tên file screenshot đúng checklist.
- [ ] Screenshot rõ, không bị cắt mất output quan trọng.

### Charts

- [ ] `finops_cost_breakdown.png`.
- [ ] `finops_timeseries.png`.
- [ ] `real_gpu_comparison.png`.
- [ ] `cost_per_epoch.png`.
- [ ] `multi_gpu_scaling.png`.
- [ ] `project_forecast.png`.
- [ ] `optimization_roadmap.png`.
- [ ] `advanced_finops_dashboard.png`.
- [ ] `real_gpu_telemetry.png` nếu có telemetry data.

### Report

- [ ] Có phân tích Part 1-7.
- [ ] Có phân tích FP32 vs AMP.
- [ ] Có phân tích Part 8.5.
- [ ] Có kết luận và bài học FinOps.
- [ ] Nếu thiếu GPU telemetry, có ghi chú rõ.

### GitHub/LMS

- [ ] Notebook final đã được commit.
- [ ] Screenshots và generated charts đã được commit.
- [ ] Report đã được commit.
- [ ] README hoặc submission note có link/đường dẫn bài nộp.
- [ ] Push repository lên GitHub.
- [ ] Nộp link GitHub lên LMS trước deadline.

## 16. Thứ Tự Thao Tác Đề Xuất Trong Ngày Nộp

1. Start Docker:

```bash
./run.sh start
```

1. Test endpoints:

```bash
./run.sh test
```

1. Mở tunnel:

```bash
./run.sh tunnel
```

1. Upload notebook lên Kaggle/Colab, bật GPU, set student info và gateway URL.

2. Chạy Cell 1-26, chụp screenshot theo từng part.

3. Implement Cell 27-31 nếu chưa làm, sau đó chạy lại Part 8.5.

4. Download notebook đã có output và tất cả PNG từ Kaggle/Colab.

5. Gom file vào folder submission.

6. Viết report nếu LMS yêu cầu.

7. Commit và push GitHub.

8. Nộp link GitHub lên LMS.

## 17. Rủi Ro Cần Tránh

- Không dùng Mac M4 Pro để chứng minh Part 8 real GPU, vì không có CUDA/NVIDIA.
- Không tắt tunnel giữa lúc notebook đang chạy.
- Không nộp notebook còn output TODO ở Part 8.5.
- Không clear notebook output trước khi nộp.
- Không chụp screenshot thiếu header sinh viên.
- Không quên download chart PNG từ Kaggle/Colab.
- Không dùng tunnel URL cũ sau khi restart `./run.sh tunnel`.
