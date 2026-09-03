# Runbook nộp bài Lab 28

Hướng dẫn này dành cho Windows PowerShell. Làm từ trên xuống dưới. Mỗi bước
đều nêu lệnh cần chạy, kết quả mong đợi và ảnh cần chụp.

Không chụp hoặc đưa lên Git: token, mật khẩu, nội dung file `.lab28/`, URL tạm
có quyền truy cập, database, cache hay model weights.

## 0. Chuẩn bị

Mở PowerShell và chạy:

```powershell
Set-Location D:\code\Day28-Modern-Platform-Lab-Student
$env:PYTHONUTF8 = "1"
$env:LAB28_VLLM_REQUIRE_REAL = "false"
$env:MLFLOW_SUPPRESS_PRINTING_URL_TO_STDOUT = "true"
New-Item -ItemType Directory -Force submission-screenshots | Out-Null
py -3.12 -m uv --version
docker info
docker compose --env-file ports.template --profile full ps
```

Kết quả đúng: `uv` in phiên bản, Docker daemon trả thông tin và các service
quan trọng hiện `healthy` hoặc `Up`.

Nếu stack chưa chạy:

```powershell
docker compose --env-file ports.template --profile full up -d --build --wait
```

Không chụp `docker info` nếu màn hình có registry credential.

## 1. Chạy bộ kiểm tra bắt buộc

Chạy từng lệnh; dừng lại xử lý nếu có lệnh đỏ:

```powershell
py -3.12 -m uv run ruff check .
py -3.12 -m uv run python scripts/verify_matrix.py
py -3.12 -m uv run python scripts/check_portability.py
py -3.12 -m uv run python scripts/validate_manifests.py
py -3.12 -m uv run pytest starter-tests -q
py -3.12 -m uv run pytest tests -q
py -3.12 -m uv run pytest integration-tests -m "not gpu and not langsmith" -q
```

Kết quả cần thấy:

- Ruff: `All checks passed!`
- Matrix: `245 checks passed`
- Portability và manifest validation: `OK` hoặc `passed`
- Starter suite: `4 passed`
- Fast suite: `83 passed`
- Live suite: `56 passed, 16 deselected`

Chụp terminal có các summary cuối, đặt tên `01-tests-and-matrix.png`.

## 2. Tạo evidence

```powershell
py -3.12 -m uv run lab28 evidence --out evidence
Get-ChildItem evidence -File | Sort-Object Name
```

Kết quả đúng: có `integration-report.json` và các artifact IP01-IP10. IP09 có
hai file riêng cho Prometheus và Grafana. IP07 có thể ghi `reachable=false`
khi chưa có GPU; tuyệt đối không sửa JSON thành `true`.

Chụp danh sách file, đặt tên `02-evidence-files.png`.

## 3. Sơ đồ kiến trúc

```powershell
Start-Process .\docs\images\lab28-architecture-overview.png
```

Chụp toàn bộ sơ đồ, đặt tên `03-architecture.png`.

Khi trình bày, mô tả: Envoy -> FastAPI -> Kafka -> Airflow -> Delta; Delta cấp
dữ liệu cho Feast, Qdrant và MLflow; Prometheus/Grafana cùng
OpenTelemetry/Jaeger quan sát toàn bộ luồng. Ownership nằm trong `ANSWERS.md`.

## 4. Happy path và Airflow

In các ID cần đối chiếu:

```powershell
$airflow = Get-Content evidence\ip02-airflow-run.json -Raw | ConvertFrom-Json
$kafka = Get-Content evidence\ip01-kafka-consume.json -Raw | ConvertFrom-Json
[pscustomobject]@{
  DagRunId = $airflow.dag_run_id
  DagState = $airflow.state
  TraceId = $kafka.trace_id
  KafkaTopic = $kafka.topic
  KafkaPartition = $kafka.partition
  KafkaOffset = $kafka.offset
} | Format-List
```

Chụp terminal, đặt tên `04-happy-path-ids.png`.

Mở Airflow:

```powershell
Start-Process http://localhost:8082
```

Username là `airflow`. Lấy password bằng lệnh sau, nhưng không chụp màn hình:

```powershell
(Get-Content .lab28\airflow\simple-auth-passwords.json -Raw | ConvertFrom-Json).airflow
```

Trong Airflow, mở DAG `lab28_ingestion_pipeline`, chọn đúng run ID vừa in rồi
mở Grid/Graph. Ảnh phải thấy run `success` và bốn task:

- `drain_kafka_into_delta`
- `refresh_online_features`
- `index_new_documents`
- `announce_processed_batch`

Đặt tên ảnh `05-airflow-success.png`.

## 5. Delta và replay-safe

In version và row count:

```powershell
$delta = Get-Content evidence\ip03-delta-history.json -Raw | ConvertFrom-Json
[pscustomobject]@{
  FeedbackLatestVersion = $delta.feedback.time_travel.latest_version
  FeedbackLatestRows = $delta.feedback.time_travel.latest_rows
  FeedbackEarliestVersion = $delta.feedback.time_travel.earliest_version
  FeedbackEarliestRows = $delta.feedback.time_travel.earliest_rows
  DocumentLatestVersion = $delta.documents.history[0].version
} | Format-List
```

Chụp kết quả, đặt tên `06-delta-versions.png`.

Chạy riêng journey replay với tên assertion hiển thị rõ:

```powershell
py -3.12 -m uv run pytest integration-tests\test_j2_idempotent_replay.py -vv
```

Kết quả đúng: mọi test `PASSED`, bao gồm cùng idempotency key, một Delta row,
một feature fact và một Qdrant point. Chụp danh sách test cùng summary, đặt tên
`07-replay-safe.png`.

## 6. Feast và Qdrant

```powershell
$feast = Get-Content evidence\ip04-feast-online.json -Raw | ConvertFrom-Json
$qdrant = Get-Content evidence\ip05-qdrant-search.json -Raw | ConvertFrom-Json
[pscustomobject]@{
  AskerId = $feast.entity.asker_id
  FeedbackCount = $feast.features.feedback_count
  FeastDeltaVersion = $feast.features.delta_version
  FeastDegraded = $feast.degraded
  QdrantCollection = $qdrant.collection
  QdrantPoints = $qdrant.points_total
  EmbeddingModel = $qdrant.embedding_model_id
} | Format-List
```

Ảnh phải thấy Feast không degraded, có Delta version và Qdrant có points. Đặt
tên `08-feast-qdrant.png`.

## 7. MLflow promotion và rollback

```powershell
py -3.12 -m uv run pytest integration-tests\test_j3_promotion_rollback.py -m "not gpu" -vv
Start-Process http://localhost:5000
```

Chụp terminal có các test register, provenance, promotion và rollback
`PASSED`; đặt tên `09-mlflow-rollback-test.png`.

Trong MLflow, vào `Models` -> `lab28-rag-release`. Chụp danh sách version và
alias `champion`; đặt tên `10-mlflow-champion.png`.

## 8. Sự cố, recovery và no-data-loss

```powershell
py -3.12 -m uv run pytest integration-tests\test_j4_degraded_recovery.py -m "not gpu" -vv
```

Kết quả đúng:

- Feast outage tạo trạng thái degraded.
- Qdrant outage làm readiness fail closed.
- Poison message được đưa vào DLQ.
- Good record trong cùng batch vẫn vào Delta.
- Replay không tạo duplicate.
- Platform cuối cùng trở lại baseline.

Chụp các test name và summary, đặt tên `11-failure-recovery.png`.

## 9. Gateway rate limit

```powershell
$gateway = Get-Content evidence\ip08-gateway.json -Raw | ConvertFrom-Json
$gateway | Format-List gateway_url,configured_rps,requests_sent,accepted,rejected,rate_limited_stat
$gateway.sample_200 | Format-List
$gateway.sample_429 | Format-List
```

Ảnh phải thấy cả HTTP `200`, HTTP `429` và `x-request-id`. Đặt tên
`12-gateway-rate-limit.png`.

## 10. Grafana, Prometheus và Jaeger

```powershell
Start-Process http://localhost:3000/d/lab28-platform/lab-28-platform-overview
Start-Process http://localhost:9090/targets
$traceId = (Get-Content evidence\ip10-trace.json -Raw | ConvertFrom-Json).trace_id
Start-Process "http://localhost:16686/trace/$traceId"
```

Grafana dùng `admin`/`admin`. Chụp dashboard có request, latency, degraded và
component readiness; đặt tên `13-grafana.png`.

Chụp Prometheus Targets với các target bắt buộc `UP`. vLLM optional được phép
`DOWN` khi chưa làm GPU; đặt tên `14-prometheus-targets.png`.

Chụp Jaeger có cùng trace ID và các span gateway, API, Kafka, Airflow, Spark;
đặt tên `15-jaeger-trace.png`.

## 11. Load profile

```powershell
py -3.12 -m uv run python load-tests\run_profile.py --requests 50 --workers 1
py -3.12 -m uv run python load-tests\run_profile.py --requests 200 --workers 8
```

Chụp hai JSON có P50/P95/P99 và status counts; đặt tên
`16-load-profile.png`. Burst có HTTP 429 là tác dụng đúng của gateway 10 RPS,
không phải lỗi mạng.

## 12. LangSmith

Gate LangSmith đã pass. Đăng nhập LangSmith bằng tài khoản của bạn, mở project
`lab28-platform`, chọn trace gần nhất và chụp danh sách spans. Đặt tên
`17-langsmith.png`. Không chụp trang API key. Rotate khóa đã gửi trong chat.

## 13. IP07 vLLM thật

Chỉ làm khi giảng viên cấp endpoint/model ID hoặc khi bạn có GPU và tunnel được
chấp thuận. Nếu dùng Kaggle T4, chạy các cell trong
`KAGGLE_GPU_EXTENSION.md`. Kaggle không tự công khai cổng 8000; không tự dùng
tunnel không được giảng viên duyệt.

Khi có endpoint, đặt biến trong PowerShell, không ghi vào file:

```powershell
$env:LAB28_VLLM_BASE_URL = "https://ENDPOINT-DUOC-CAP/v1"
$env:LAB28_VLLM_MODEL_ID = "MODEL-ID-DUOC-CAP"
$env:LAB28_VLLM_REQUIRE_REAL = "true"
$env:LAB28_VLLM_API_KEY = Read-Host "API key nếu endpoint yêu cầu"
```

Kiểm tra danh tính vLLM:

```powershell
$root = $env:LAB28_VLLM_BASE_URL -replace '/v1/?$',''
Invoke-RestMethod "$root/version"
Invoke-RestMethod "$($env:LAB28_VLLM_BASE_URL.TrimEnd('/'))/models"
(Invoke-WebRequest -UseBasicParsing "$root/metrics").Content |
  Select-String '^vllm:' |
  Select-Object -First 10
```

Nối API và chạy GPU gate:

```powershell
docker compose --env-file ports.template --profile full up -d --force-recreate --wait api
py -3.12 -m uv run pytest integration-tests -m gpu -q
py -3.12 -m uv run lab28 ask "Nền tảng dữ liệu gồm những thành phần nào?" --via-gateway
py -3.12 -m uv run lab28 evidence --out evidence
```

Chụp `/version`, `/v1/models`, metric `vllm:`, GPU test và câu trả lời có
trace/model/release; đặt tên `18-vllm-ip07.png`.

Prometheus hiện scrape `host.docker.internal:8001`. Để toàn bộ GPU suite pass,
GPU endpoint cần được forward về local port 8001 hoặc Prometheus target phải
được cấu hình theo URL giảng viên cấp. Không đánh dấu IP07 đạt nếu test bị
skip/fail.

## 14. Kubernetes/GitOps live

Chỉ làm khi có cluster. Bật Kubernetes trong Docker Desktop rồi chạy:

```powershell
kubectl config get-contexts
kubectl cluster-info
kubectl apply --dry-run=client -k deploy/kubernetes/base
```

Static validation pass chưa phải live drift proof. Để chạy live, cần:

1. Cài Argo CD vào cluster và cài CLI `argocd`.
2. Đổi `repoURL` và `targetRevision` trong `gitops/application.yaml` sang
   private repository/revision của bài nộp.
3. Cấp repository credential cho Argo CD.
4. Đảm bảo image và các dependency mà manifest tham chiếu truy cập được.

Sau đó:

```powershell
argocd app sync lab28-platform
argocd app wait lab28-platform --health
kubectl -n lab28 annotate deployment lab28-api lab28-demo-drift=true --overwrite
argocd app get lab28-platform
argocd app history lab28-platform
argocd app rollback lab28-platform <HISTORY_ID>
argocd app wait lab28-platform --health
```

Chụp trạng thái `OutOfSync`, self-heal về `Synced` và lịch sử rollback; đặt
tên `19-argocd-drift-rollback.png`. Nếu không có cluster/Argo, giữ
`UNVERIFIED`.

## 15. Kiểm tra secret và tạo bundle

Kiểm tra phần sắp commit. Khi được hỏi, chỉ nhập 8-12 ký tự đầu của secret để
tìm kiếm, không nhập cả khóa:

```powershell
$secretPrefix = Read-Host "Nhập 8-12 ký tự đầu của secret cần kiểm tra"
rg -n --fixed-strings $secretPrefix . --hidden -g '!.git/**' -g '!.lab28/**'
git diff --check
git status --short
```

`rg` không được tìm thấy kết quả. Tên biến như `LANGSMITH_API_KEY` được phép;
giá trị key thì không.

Tạo ZIP ngoài repository:

```powershell
$bundle = Join-Path (Split-Path $PWD) "lab28-submission.zip"
Compress-Archive -Path @(
  "evidence",
  "submission-screenshots",
  "ANSWERS.md",
  "integration-report.json",
  "fast-suite-output.txt",
  "failure-recovery.md",
  "load-profile.json",
  "gitops-validation.md",
  "docs\images\lab28-architecture-overview.png"
) -DestinationPath $bundle -Force
Get-Item $bundle | Format-List FullName,Length,LastWriteTime
```

## 16. Commit và push private branch

Thay `<MSSV>` bằng mã sinh viên:

```powershell
git switch -c <MSSV>-lab28
git add src load-tests compose.yaml compose.langsmith.yaml monitoring ANSWERS.md SUBMISSION_RUNBOOK.md failure-recovery.md fast-suite-output.txt gitops-validation.md integration-report.json load-profile.json
git diff --cached --check
git diff --cached --stat
git commit -m "feat: complete Day 28 platform integration lab"
git push -u origin <MSSV>-lab28
```

`evidence/` được ignore có chủ đích, vì vậy nộp ZIP riêng qua Drive/LMS. Nếu
không có quyền push vào `origin`, tạo private repository của bạn, đổi remote
và push branch vào đó. Không push lên public repository.

## 17. Nộp LMS và dọn môi trường

Nộp:

1. URL trực tiếp đến private branch.
2. File `lab28-submission.zip` hoặc Drive link chỉ giảng viên truy cập được.
3. Ghi chú ngắn rằng core, LangSmith, tests và static GitOps đã verified.
4. Ghi rõ `IP07 UNVERIFIED` và `live Kubernetes/Argo UNVERIFIED` nếu chưa
   hoàn thành hai gate phụ thuộc môi trường.

Sau khi upload, mở lại ZIP kiểm tra đủ file và không có secret. Sau đó mới dừng
container nhưng giữ cache/volume:

```powershell
docker compose --env-file ports.template --profile full down --remove-orphans
```

Không chạy `lab28 reset --yes` trước khi nộp và kiểm tra bundle, vì lệnh đó xóa
trạng thái dùng để chứng minh demo.
