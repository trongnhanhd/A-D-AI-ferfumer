# PG-01 — Data Model Proposal

## Mức độ xác minh

Mô hình logic đề xuất; không phải schema đang tồn tại, SQL migration hay lựa chọn database. Tất cả entity/field bên dưới phải map với mã nguồn theo [audit](architecture-audit.md) trước khi tạo bảng mới. Dùng kiểu ID, tenant boundary và timestamp của ứng dụng sau khi xác minh.

## Các thực thể và invariant

| Entity khái niệm | Nội dung tối thiểu | Invariant |
| --- | --- | --- |
| Formula | ID hiện hữu, tenant/project/ownership, protected main branch | Tái sử dụng formula hiện hữu nếu có |
| Branch | ID, formula ID, tên, head version ID, revision, protected flag | Head cùng formula; main chỉ đổi qua promotion |
| Draft | ID, formula/branch, base version, expected branch head, payload, revision, owner, saved time | Mutable; ghi bằng compare-and-swap; không được approve |
| ImmutableVersion | ID, formula, schema version, full snapshot, digest, actor, created time | Append-only; snapshot và parent không sửa sau commit |
| VersionParent | child ID, parent ID, thứ tự/vai trò parent | Cùng formula/tenant; graph không chu trình; root không có parent |
| VersionLineSnapshot | stable line ID, catalog reference tùy chọn, ingredient snapshot, decimal quantity, unit snapshot, line metadata/order | Không phụ thuộc dữ liệu catalog hiện tại để tái dựng |
| ReviewRequest | ID, version ID, digest, target branch, expected target head, policy revision, state | Subject bất biến; không trỏ draft |
| ReviewDecision | review ID, reviewer, decision, comment, time, subject digest | Append-only; quyền và policy kiểm lại khi promote |
| Promotion | ID, review/version, target, old/new head, actor, time, idempotency key | Head + promotion + audit commit nguyên tử |
| VersionEvidenceLink | version ID, loại/ID Task hoặc Sample hoặc Evaluation, relation, provenance | Không gắn theo “latest”; cùng tenant và formula khi phù hợp |
| AuditEvent | actor/service, action, object IDs, prior/new revisions/digests, reason, correlation ID, time | Append-only; không chứa secret |

Quyết định vật lý (bảng riêng, document hay mapping entity hiện hữu) phụ thuộc storage thực tế. Nếu storage không có transaction đa entity, phải chứng minh cơ chế conditional write và atomic publication tương đương trước triển khai.

## Snapshot nguyên liệu và nội dung duyệt

Mỗi version chứa full snapshot để đọc độc lập; parent chỉ cung cấp lineage, không phải điều kiện tái dựng. Snapshot bảo toàn mọi dòng, kể cả lặp một nguyên liệu, thông qua stable line ID.

Mỗi dòng giữ catalog ID nếu có, mã/tên nguyên liệu tại thời điểm commit, grade, concentration/dilution, supplier/lot khi nghiệp vụ dùng, thông tin chất mang và metadata cần tính toán/duyệt. Giữ giá trị lượng gốc dạng decimal chính xác, unit ID/code/label và chiều đo; giữ conversion factor, density, reference temperature hoặc version quy tắc tính khi đã được dùng. Không tự điền các dữ kiện không có. Phân biệt null, unknown và zero. Không dùng binary floating point làm nguồn lượng chính xác.

Snapshot cấp công thức giữ tên, mục đích, batch basis/size nếu có, process instructions, ordering, tất cả metadata ảnh hưởng nội dung review và immutable attachment references + digests. Danh sách field chính thức phải lấy từ schema thật; unknown metadata phải được bảo toàn, không âm thầm bỏ qua.

Catalog thay đổi/xóa không làm thay đổi snapshot hoặc cascade-delete lịch sử. Catalog reference phục vụ truy vết, không dùng để render lại nội dung cũ. Dữ liệu external/attachment cần được lưu giữ bất biến theo retention và quyền truy cập tương ứng.

## Digest và diff

Digest tính trên serialization canonical có version: snapshot, metadata trong scope và ordered parent IDs. Quy định sorting keys, thứ tự dòng, decimal string, Unicode, null/missing và unit representation; giữ dữ liệu gốc, chỉ canonicalize khi tính digest. Máy chủ tự tính; không tin digest do client gửi. Digest không thay thế authorization, append-only enforcement hay backup.

Diff so stable line ID, nhận diện added/removed/changed/moved; trình bày cả lượng/đơn vị gốc. Chỉ so lượng chuẩn hóa khi tương thích chiều đo và đủ conversion context, nếu không báo “không thể quy đổi”. Hiển thị metadata, process, attachment và ingredient snapshot thay đổi. Diff không được tự sửa nội dung hoặc tự merge. Không suy đoán rename dựa riêng vào tên nguyên liệu.

## Ràng buộc và liên kết nghiệp vụ

- Unique branch name trong formula theo quy tắc case được chốt; FK cùng tenant/formula cho head, parent và review subject. Kiểm tra cycle và authorization phía server.
- Draft revision, branch revision/head là concurrency tokens. Idempotency key scoped theo tenant, actor và operation; cùng key khác payload phải bị từ chối.
- Lab Sample phải giữ version đã thực sự pha; Evaluation 14D liên kết sample và version tương ứng. Nếu sample thay công thức, tạo sample/version association mới theo policy, không viết đè evidence cũ.
- Task có thể yêu cầu, tạo hoặc đánh giá version qua typed relationship; hoàn tất task không tự đồng nghĩa approval.
- Evidence liên quan review phải được pin ID + revision/digest lúc submit; chỉnh sửa evidence cần tái đánh giá/review theo policy, không đổi nội dung đã gửi ngầm.
- Không gán lịch sử legacy cho version mới nếu chưa có bằng chứng. Dùng trạng thái unresolved/legacy provenance và chặn các hành động cần lineage chắc chắn.
- Index đề xuất trên formula/branch/history, parent-child, review target/state, sample-version, evaluation-sample và audit object/time; đo workload trước chốt index.
