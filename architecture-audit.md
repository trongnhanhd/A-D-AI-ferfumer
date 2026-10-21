# PG-01 — Mountain Lab Perfume: Architecture Audit

Ngày: 2026-10-21. Trạng thái: đề xuất thiết kế để review; chưa cho phép triển khai.

## Phạm vi và bằng chứng

Đã kiểm tra checkout `trongnhanhd/A-D-AI-ferfumer`, fetch `origin/main` và liệt kê toàn bộ cây Git tại commit `797011637383d98439d2a693ddb0208b8df0d24c`. Nhánh main chỉ chứa [README.md](README.md), gồm tiêu đề dự án và mô tả “Sử dụng AI hỗ trợ cho fefumer phát triển sản phẩm nước hoa”. Không tìm thấy AGENTS.md trong workspace tại thời điểm audit.

| Hạng mục cần audit | Bằng chứng hiện có | Kết luận |
| --- | --- | --- |
| Runtime, framework, package và CI | Không có manifest, lockfile, cấu hình build/CI | Chưa xác minh công nghệ |
| Database, schema, migrations | Không có mã hoặc tài liệu tương ứng | Chưa xác minh storage, constraints, transaction |
| API, authentication, authorization | Không có mã nguồn | Chưa xác minh endpoints hay permission model |
| Formula và ingredient UI | Không có mã nguồn | Chưa xác minh cấu trúc dòng nguyên liệu, đơn vị, autosave |
| Lab, Task, Lab Sample | Không có mã nguồn | Chưa xác minh định danh và quan hệ |
| Sensory Evaluation 14D | Không có mã nguồn hoặc đặc tả | Chưa rõ 14D là mô hình nào; không diễn giải thành 14 ngày hay 14 chiều |
| Review, approval, audit log | Không có mã nguồn | Chưa xác minh vai trò, trạng thái hoặc điểm tích hợp |

Đây là audit của repository được cung cấp, không phải xác nhận đã audit ứng dụng Mountain Lab R&D OS đang chạy. Không thể kết luận chức năng nào đã tồn tại, thiếu hoặc có thể tái sử dụng. Cần cung cấp repository/ref chứa ứng dụng, schema/migrations và tài liệu 14D trước cổng triển khai. Không cần gửi dữ liệu thật hay secret.

## Kiến trúc đề xuất, chưa phải kiến trúc hiện hữu

Engine là module nghiệp vụ trong hạ tầng ứng dụng được xác minh sau này. Công thức và lịch sử nằm trong storage ứng dụng có transaction và kiểm soát quyền; không dùng Git repository, Git CLI hoặc Git object làm kho công thức. Git chỉ dùng review tài liệu và mã nguồn.

Luồng khái niệm: UI → application authorization → draft/version/review/promotion service → storage transaction + audit/outbox → hệ thống approval và Task/Lab/Sensory hiện hữu qua adapter. Tên service là ranh giới trách nhiệm, không yêu cầu microservice hay một công nghệ cụ thể.

## Cổng xác minh và tái sử dụng

Trước thiết kế vật lý, lập bản đồ `entity thực tế → ID → ownership → lifecycle → nơi sử dụng → quyết định reuse/extend/new`. Đọc code và test cho từng mục:

1. Formula và ingredient: dùng lại định danh, unit conversion, precision, metadata và validation hiện hữu; bổ sung snapshot bất biến nếu còn thiếu. Không tạo ingredient catalog thứ hai.
2. Task, sample và 14D: dùng lại entity và quyền hiện hữu; thêm liên kết version tại thời điểm tạo/submit được xác minh. Không xây lại task board, sample tracking hoặc scoring.
3. Approval: xác định state machine, reviewer independence, quorum, revoke, expiry và audit. Adapter phải ràng buộc approval với immutable version; không thay engine approval hiện tại trước khi đánh giá tương đương.
4. Audit, identity, tenancy, file storage, notifications: mở rộng cơ chế hiện hữu nếu đáp ứng bảo toàn lịch sử, phân quyền và độ bền. Attachment cần snapshot nội dung hoặc immutable reference cùng digest.
5. Đối chiếu migrations với schema thực và chạy workflow trong môi trường thử với dữ liệu tổng hợp; ghi file/line, API và test làm bằng chứng ở bản audit cập nhật.

Nếu approval hiện tại không gắn được immutable subject và fingerprint, chặn promotion thay vì giả định adapter đủ an toàn. Không mở endpoint ghi mới trước khi mapping quyền và tenant hoàn tất.

## Ranh giới an toàn và quyết định cần review

- Draft có thể thay đổi; version đã tạo không thể sửa hoặc xóa tùy tiện. Main là con trỏ được bảo vệ, không phải bản nháp.
- Approval chỉ áp dụng version cụ thể và nội dung snapshot đã gửi. Mọi thay đổi tạo version mới và review mới.
- AI chỉ đọc dữ liệu theo quyền để giải thích diff, gợi ý và phân tích. AI không có quyền ghi công thức, approve, merge hoặc promote; output luôn là gợi ý riêng, người dùng tự thực hiện thay đổi.
- Cần chốt reviewer policy, metadata nào thuộc nội dung duyệt, quy tắc đơn vị/precision, ý nghĩa 14D, retention và nghiệp vụ rollback sau audit nguồn.

## Tiêu chí kết thúc PG-01

Bốn tài liệu có thể review ngay. Audit ứng dụng đầy đủ và khả năng tương thích vẫn bị chặn do thiếu mã nguồn; không coi proposal là bằng chứng hệ thống thật đáp ứng thiết kế. Xem [data model](data-model-proposal.md), [workflow](version-control-workflow.md) và [migration/test plan](migration-and-test-plan.md).
