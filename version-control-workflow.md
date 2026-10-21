# PG-01 — Version Control Workflow

## Trạng thái và thao tác

Luồng đề xuất: draft mutable → commit immutable version → submit review cho đúng version → decisions → promotion có kiểm tra → main head. Review có các trạng thái proposed `pending/approved/rejected/withdrawn/stale/promoted`; phải map với approval hiện tại trước khi triển khai. Không đổi state machine đang chạy ở PG-01.

1. **Mở/sửa:** tạo draft từ explicit base version và branch head. UI giữ dirty state, last acknowledged revision và trạng thái save. Không coi request đang gửi là đã lưu.
2. **Save:** gửi payload với expected draft revision. Server kiểm quyền, validation và conditional update. Conflict trả trạng thái tương đương 409/412 và revision mới, không last-write-wins. UI giữ bản người dùng, hiển thị diff để họ tự chọn.
3. **Commit:** khóa thao tác UI khi save đang chờ; gửi expected revision và fingerprint nội dung người dùng thấy. Server tạo version từ đúng draft revision trong transaction, validate snapshot và ghi audit. Nếu sửa sau save/commit, bản mới là dirty draft riêng; không được gắn nhãn đã gửi.
4. **Branch commit:** cập nhật branch head bằng CAS trên expected head/revision. Main không nhận commit trực tiếp. Nếu branch đã đổi, rollback toàn bộ commit/head operation hoặc giữ version detached theo policy rõ ràng; không công bố head thành công giả.
5. **Submit:** chỉ nhận immutable version ID + server digest; review page render từ snapshot đó, không từ draft/catalog hiện tại. UI có dirty edits phải yêu cầu người dùng save/commit trước hoặc chủ động chọn gửi version cũ với nội dung được hiển thị rõ. Không tự bỏ sửa đổi hoặc submit “latest”. Pin target head và policy revision.
6. **Review:** kiểm quyền, reviewer independence/quorum theo hệ thống hiện hữu. Quyết định gắn version/digest/evidence đã pin. Comment không thay nội dung. Version mới yêu cầu review mới; reject không sửa version.
7. **Promotion:** server kiểm lại quyền hiện tại, subject/digest, policy, decisions, evidence và target head trong transaction; CAS old head → approved version, ghi promotion + audit/outbox nguyên tử. Retry dùng idempotency key. Target đổi thì review stale; không tự merge hay tự approve lại.

## Bảo vệ nội dung chưa lưu

Cảnh báo khi rời trang, đổi version/branch hoặc đóng editor nếu dirty, save pending hoặc save failed. Browser unload chỉ là best effort; recovery buffer có tenant/user scope, retention và bảo vệ dữ liệu theo chính sách, không là bản đã submit. Không lưu dữ liệu nhạy cảm vào browser storage nếu chưa đánh giá. Khi crash/reconnect, so revision trước phục hồi; không tự ghi đè server. Hai tab phải xử lý như hai writer độc lập. Hiển thị riêng “đã lưu draft”, “đã commit version” và “đã gửi review”.

## Concurrent updates và bảo vệ main

- Draft save, branch head, review state và promotion dùng conditional writes; không dựa riêng vào khóa UI hoặc timestamp.
- Hai promotion cùng expected head chỉ một thành công; giao dịch thất bại không để audit thành công hoặc head nửa chừng. Notification qua outbox có retry/dedup.
- Main protected tại service/storage permission boundary; API cũ, import, background job và admin path cũng phải tuân thủ. Không chỉ ẩn nút frontend.
- Nếu source không phải descendant target, chặn promotion theo mặc định. Người có quyền tự giải quyết diff thành version mới với parent links được ghi rõ và gửi review mới; không cung cấp tự merge. Fast-forward cũng vẫn cần approval.
- Rollback nghiệp vụ là một promotion được review tới nội dung phục hồi/new revert version theo policy; không xóa history hoặc reset main ngầm. Quy tắc lineage của rollback phải được chốt trước triển khai.
- Revoke approval trước promotion phải làm review mất hiệu lực; sau promotion tạo sự kiện và quy trình khắc phục mới, không sửa lịch sử quyết định. Race revoke/promote được serialize theo review revision/transaction.

## Task, Lab và Sensory Evaluation 14D

Task tham chiếu version cụ thể và loại quan hệ. Khi tạo sample, người dùng xác nhận version pha; đóng băng liên kết và lưu actual preparation deviations như evidence riêng, không sửa công thức snapshot. Evaluation 14D tham chiếu sample và chính version đó, giữ rubric/schema version, evaluator và thời điểm theo model thật sau audit. Không coi 14D là một lịch thời gian hay số chiều khi chưa có đặc tả.

Review hiển thị evidence đúng snapshot/reference đã gửi. Evidence bổ sung có thể được append; nếu ảnh hưởng quyết định, tạo vòng review mới hoặc yêu cầu xác nhận lại theo policy được duyệt. Task status, sample completion và AI score không thay thế approval của người.

## AI và audit

AI chỉ cung cấp analysis/diff explanations/suggestions qua quyền đọc tối thiểu. Tool/service credentials của AI không có quyền mutate draft/version/approval/head. Không cho suggestion tự chạy qua background task. Người dùng tự nhập/chấp nhận có chủ ý bằng thao tác sửa draft; mọi thay đổi sau đó vẫn qua commit/review thông thường.

Audit server ghi create/save/commit/submit/decision/revoke/promotion/conflict quan trọng, actor thực và actor hỗ trợ, request/correlation ID, subject IDs/digests và reason. Không ghi secret hay toàn bộ payload nhạy cảm không cần thiết. Hạn chế sửa/xóa log, retention và truy cập theo tenant; nếu cần chống admin tamper dùng external append-only sink theo threat model đã duyệt.
