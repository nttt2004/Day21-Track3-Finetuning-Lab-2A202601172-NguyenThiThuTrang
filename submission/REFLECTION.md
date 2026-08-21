# Reflection — Lab 21

*Ngắn gọn, thành thật. Phần này chấm theo độ cụ thể, không theo độ dài.*

**1. Điều gì làm bạn ngạc nhiên nhất?**

`attn_only` — chỉ gắn LoRA vào 2 module (q, v) thay vì 12 module như `correct` —
đạt `target=0.970`, nhỉnh hơn cả `correct` (0.965), miễn là rank được nâng lên để
khớp đúng số tham số huấn luyện. Tôi vào lab với giả định "gắn adapter càng nhiều
lớp càng tốt", nhưng khi đã khớp ngân sách tham số thì vị trí gần như không tạo
khác biệt ở quy mô 30 step này. Ngạc nhiên thứ hai là `regression` sập tới -0.302
(0.7578 → 0.4556) dù chỉ train 32,464,896/~vài tỉ tham số (<1%) — tôi từng nghĩ
LoRA "nhẹ tay" như vậy thì ít gây quên hơn hẳn full fine-tune, nhưng số liệu cho
thấy thiếu dữ liệu replay là đủ để phá vỡ giả định đó.

**2. Bạn mất nhiều thời gian nhất ở đâu? Nó có phải chỗ bạn dự đoán không?**

Không phải ở NB3/NB4 (train) như tôi dự đoán, mà ở việc làm sạch kết quả `make
verify` trên Windows: `eval sets unmodified` báo FAIL dù `git status` sạch —
hoá ra do `core.autocrlf=true` chuyển file sang CRLF khi checkout, làm sha256
lệch với `data/checksums.json` tính trên bản LF gốc, một false-positive môi
trường chứ không phải tôi sửa dữ liệu. Việc thứ hai tốn thời gian là `runs.csv`
có 3 dòng `correct` trùng label do chạy lại NB3 nhiều lần (loss 0.6261 / 0.627 /
0.6288) — không có cột nào đánh dấu "đây là lần chạy cuối", phải suy ra bằng
cách so timestamp file với `adapters/correct/`. Đây đúng là bài học: log nên có
run_id hoặc timestamp tường minh, không nên chỉ append và để người đọc tự đoán.

**3. Trước lab này bạn tin điều gì về fine-tuning mà giờ bạn không còn tin?**

Tôi từng tin "target tăng là fine-tune thắng, chấm hết" — vì LoRA chỉ chỉnh một
phần nhỏ trọng số nên nghĩ rủi ro thấp, không cần đo thêm gì ngoài độ chính xác
trên tác vụ đích. Lab này cho tôi một trường hợp cụ thể: `target` tăng +0.200
(0.765 → 0.965) nhưng verdict vẫn `FAILED` vì `regression` giảm -0.302, vượt
tolerance 0.020 tới 15 lần. Nếu không có gate 4 nhóm (target/regression/format/
latency) đo trước khi train, tôi đã kết luận sai và đề xuất deploy một mô hình
thực ra không nên dùng.

**4. Bạn dùng AI assistant vào việc gì trong lab? Chỗ nào nó sai?**

Tôi dùng AI assistant (Claude Code) để: chạy `make verify`/`pytest`, đọc và diễn
giải log lỗi (chỉ ra nguyên nhân CRLF của lỗi checksum, và lỗi quyền
`AppData\Local\Temp` trên Windows khiến `pytest`'s `tmp_path` fixture fail);
viết script đối chiếu từng `ft_pred` trong `results/qualitative.json` với nhãn
gốc trong `data/eval_target.jsonl` để tìm đúng 2 ca fine-tune thua thay vì chọn
ngẫu nhiên; điền số liệu vào `REPORT.md` trực tiếp từ file trong `results/`; và
chạy script `push_to_hub` để đẩy adapter lên HuggingFace Hub.

Chỗ nó chưa chắc đúng: khi chọn dòng `correct` nào trong 3 dòng trùng ở
`runs.csv` để đưa vào báo cáo, nó suy luận dựa trên **thời điểm ghi file** (dòng
cuối cùng khớp timestamp với `adapters/correct/`), chứ không có cách nào xác
nhận chắc chắn 100% đó đúng là lần chạy tạo ra `results/*.json` hiện tại — đây
là một suy luận hợp lý nhưng vẫn là suy luận, không phải sự thật được xác nhận
từ hệ thống, và nó đã tự ghi chú rõ điều này trong báo cáo thay vì giấu đi.

**5. Nếu ngày mai phải fine-tune cho một khách hàng thật, bước đầu tiên bạn làm là gì?**

Đóng băng một eval set (kèm checksum) và đo đủ ba baseline — (a) prompt mộc,
(b) prompt đã tối ưu, và thiết kế sẵn gate 4 nhóm (target/regression/format/
latency) — **trước khi** viết một dòng code train nào. Lab này cho tôi bằng
chứng cụ thể rằng nếu bỏ qua bước này, một con số target đẹp một mình nó đủ để
đánh lừa quyết định deploy.
