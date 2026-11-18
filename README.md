# GoatSports Workflow Service

Điều phối các quy trình kéo dài nhiều giờ bằng Camunda 8 (Zeebe). Trạng thái trong Camunda là nguồn sự thật;
venue-service chỉ lưu bản chiếu do service này ghi xuống.

## Quy trình (BPMN trong `src/main/resources/bpmn`)

| Quy trình | Nội dung |
|---|---|
| `booking-cancellation-refund.bpmn` | Hủy sân và hoàn tiền: trong 60 phút đầu tự duyệt; sau đó chờ chủ sân quyết định, nhắc chủ sân, tự duyệt khi đến giờ chơi; tạo hoàn tiền và chờ payOS báo kết quả |
| `booking-lifecycle.bpmn` | Vòng đời đơn đặt sân: hết giờ giữ chỗ, đã cọc, đã thu phần còn lại, đã nhận sân, không đến, hoàn thành |

**Job worker**: `booking-cancellation.apply-state`, `booking-cancellation.create-refund`,
`booking-cancellation.remind-owner`, `booking-lifecycle.apply-state`.

## Chức năng chính

- Khởi tạo quy trình và nhận quyết định từ venue-service (`/internal/booking-cancellations`, `/internal/booking-lifecycles`).
- Ghi trạng thái về venue-service kèm số phiên bản (job key) để venue bỏ qua cập nhật cũ.
- Nhận sự kiện hoàn tiền từ Kafka `payment.events` (`PaymentRefundEventConsumer`) rồi correlate message vào quy trình.
- `CancellationStateReconciler` định kỳ đẩy lại trạng thái các quy trình đang chạy để chống lệch.

## Cấu trúc

```
src/main/java/iuh/fit/workflowservice/
├── application/     # DTO, port
├── infrastructure/  # Camunda worker, Kafka, Feign tới venue-service, reconciler
└── presentation/    # REST controller nội bộ
src/main/resources/bpmn/
shared-kernel/       # @AuditLog, Feign interceptor dùng chung
```

## Tích hợp

- **Camunda 8** (`CAMUNDA_*`), chạy bằng `infrastructure/compose/docker-compose-camunda.yml` (Camunda + Elasticsearch, khoảng 2 GB RAM).
- **Kafka nhận**: `payment.events`. **Kafka phát**: `user-log`.
- **Feign**: venue-service.

## Chạy và kiểm thử

```bash
mvn -f shared-kernel/pom.xml clean install -DskipTests
mvn spring-boot:run -Dspring-boot.run.profiles=dev     # cần Camunda đang chạy và application-dev.properties
mvn test
```

Test quy trình (`BookingCancellationProcessTest`) dùng Camunda Process Test + Testcontainers, tự bỏ qua khi không có Docker.

## CI/CD

`.gitlab-ci.yml`: test → `jib:build` lên Docker Hub → gọi Jenkins job `GoatSports-CD`.
