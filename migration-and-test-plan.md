# PG-01 — Migration and Test Plan

## Giới hạn thực hiện

PG-01 chỉ tạo tài liệu. Chưa chạy migration, backup, restore hoặc test ứng dụng; repository không có ứng dụng/test runner. Không sửa dữ liệu thật. Các acceptance tests dưới đây là kế hoạch, không phải kết quả đã pass.

## Thứ tự triển khai theo phụ thuộc

| Task | Phụ thuộc | Sản phẩm và acceptance tests |
| --- | --- | --- |
| PG-02 Source & reuse audit | Không | Xác định đúng repo/ref, file/line schema/API/UI/approval/14D; mapping reuse/extend/new được owner review; không còn giả định stack |
| PG-03 Domain & permission contract | PG-02 | Chốt snapshot scope, units/precision, branch policy, reviewers, lineage, evidence và retention; test matrix quyền gồm cross-tenant và legacy APIs |
| PG-04 Storage & migration design | PG-03 | Migration cụ thể được review; thử backup/restore và expand/rollback trên DB tổng hợp; xác minh transactions/constraints |
| PG-05 Snapshot & immutable version | PG-04 | Commit snapshot/parent/digest; catalog sửa/xóa không thay version; mutation bị từ chối; decimal/unit/metadata/attachment round-trip |
| PG-06 Draft & branch concurrency | PG-05 | Save/commit CAS và idempotency; hai writer không mất dữ liệu; protected main không nhận ghi trực tiếp |
| PG-07 Diff & editor recovery | PG-06 | Diff đầy đủ, dirty/save failure/reload/two-tab recovery; chọn version cũ phải rõ nội dung; không silently discard |
| PG-08 Existing approval adapter | PG-05, PG-06, PG-03 | Submit đúng immutable subject; permission/quorum/revoke và audit hiện hữu vẫn chạy; không approve draft |
| PG-09 Task/Sample/14D links | PG-05, PG-02 | Liên kết đúng version; sample/evaluation mismatch bị chặn; evidence pin và legacy unresolved rõ ràng |
| PG-10 Promotion transaction | PG-08, PG-09 | Atomic CAS head + promotion + audit/outbox; stale/revoked review bị chặn; retry không lặp promotion |
| PG-11 AI read-only integration | PG-07, PG-03 | Phân tích có provenance; credentials/tool calls không thể sửa/approve/promote ngay cả khi prompt yêu cầu |
| PG-12 Staging rehearsal & release gate | PG-07 đến PG-11 | E2E, security, restore và rollback đạt; reconciliation không sai khác; owner duyệt rollout riêng |

Không task nào trong bảng được triển khai trong PR tài liệu này.

## Migration an toàn được đề xuất

1. **Discovery:** kiểm tra engine/version, schema thực, kích thước bảng, lock behavior, jobs, retention và backup tooling. Chốt RPO/RTO với owner; không tự đặt con số hoặc tuyên bố có backup.
2. **Backup rehearsal:** tạo dữ liệu tổng hợp đại diện, backup có manifest/checksum bằng công cụ chính thức, kiểm tra mã hóa/quyền; restore vào DB tách biệt và so count, relationships, decimal, attachments/digests. Không sao chép production data nếu chưa có phê duyệt riêng và chính sách xử lý.
3. **Expand:** chỉ thêm cấu trúc/index có backward compatibility; FK/check/index online nếu engine hỗ trợ. Chạy dry-run và đo lock trên staging; không drop/rename/rewrite cột cũ. Feature flags mặc định tắt.
4. **Backfill thử:** có checkpoint và idempotent mapping legacy ID → baseline version. Chỉ snapshot nội dung hiện quan sát được, không bịa lịch sử, parents hoặc approvals. Dữ liệu thiếu/không hợp lệ vào báo cáo quarantine, không tự sửa. Giữ unresolved links; không gán mọi evaluation cũ cho current formula.
5. **Reconcile:** count từng formula, lossless line/metadata/unit comparison, digest ổn định qua rerun, FK/tenant/orphan checks. Old/new read paths cho cùng nội dung; rerun không nhân bản version. Đánh giá code cũ còn có thể ghi và vô hiệu hóa đường bypass trước rollout.
6. **Cutover thử:** bật theo staging/test scope, read compare trước write enable. Không dual-write hai nguồn không có transaction/reconciliation. Chỉ cho production migration bằng task riêng có runbook đã review, backup/restore được chứng minh và người chịu trách nhiệm.
7. **Rollback:** trước cutover tắt feature và quay read path cũ, giữ additive structures. Sau có version mới, ưu tiên rollback code tương thích, giữ engine read-only và dữ liệu lịch sử; không down-migrate phá bảng. Nếu buộc restore, dừng writes, archive/reconcile phần phát sinh theo RPO/RTO trước khi thực hiện; không restore đè dữ liệu mới một cách ngầm định. Thử rollback trước release.
8. **Contract:** xóa schema cũ chỉ ở đợt độc lập sau retention và xác minh mọi consumer đã chuyển; không nằm trong PG-01 hay migration ban đầu.

## Acceptance scenarios chi tiết

- **Snapshot:** commit A, sửa tên/concentration/unit rule trong catalog, đọc A vẫn giống byte-level canonical payload/digest; quantity lớn/nhỏ, null, zero, Unicode, dòng nguyên liệu lặp và metadata mở rộng round-trip chính xác.
- **Immutability/lineage:** update/delete version hoặc parent qua mọi write path bị từ chối; child khác tenant/formula, cycle và parent không tồn tại bị chặn. Root hợp lệ không có parent.
- **Dirty state:** sửa sau save hoặc trong lúc save; submit không thể lấy revision khác người dùng thấy. Network timeout, save failure, tab close/reload, two-tab conflict giữ được nội dung và báo trạng thái chính xác.
- **CAS:** hai save cùng revision chỉ một ghi thành công; hai commit cùng branch head chỉ một đổi head; conflict giữ local edits. Không lost update; cùng idempotency key/payload trả cùng kết quả, khác payload bị từ chối.
- **Review integrity:** draft ID bị từ chối; thay client digest, version hoặc evidence revision sau submit không được dùng approval cũ. Version B không thể dùng decision của A. Self-approval/quorum áp dụng policy thực tế đã chốt.
- **Promotion:** hai reviewer/promoter thao tác đồng thời, revoke/promotion race và target head đổi đều có kết quả serialize hợp lệ. Inject crash trước/sau commit: không có head không audit, không notification cho transaction rollback; retry không nhân bản.
- **Main security:** gọi trực tiếp API cũ/import/job/admin path hoặc giả role/cross-tenant không bypass protection; quyền bị thu hồi giữa review và promotion được kiểm lại.
- **Diff:** thêm/xóa/đổi/move dòng, concentration/process/metadata/attachment thay đổi đều hiển thị; unit khác chiều hoặc thiếu density được báo không so sánh được, không tính số sai.
- **Evidence:** task liên kết nhiều version có relation rõ; sample giữ version đã pha; evaluation của sample khác version bị từ chối. Chỉnh evidence không ngầm thay nội dung review đang chờ.
- **AI:** prompt injection hoặc tool invocation không ghi draft, approve, merge hay promote; suggestion chỉ hiển thị để người dùng quyết định.
- **Migration:** chạy hai lần không duplicate; dữ liệu legacy thiếu được report; old app còn đọc được sau expand; restore/reconciliation và rollback giữ version mới. Mọi test dùng DB tách biệt và fixtures tổng hợp.

## Cổng release và báo cáo

Lưu command, exit code, số test executed/passed/failed/skipped, version môi trường và artifact của từng run; không chấp nhận zero-test pass. Test concurrency dùng barrier và fault injection thay vì chỉ chạy tuần tự. Đo performance theo dữ liệu đại diện và SLO được owner chốt sau audit; chưa có benchmark ở PG-01.

Production rollout bị chặn cho tới khi source audit, permission mapping, approval compatibility, snapshot/diff correctness, concurrency và backup/restore/rollback đều có bằng chứng. PR này chỉ cần review tài liệu; không xác nhận readiness của ứng dụng chưa được cung cấp.
