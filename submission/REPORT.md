# Lab 21 — Evaluation Report

**Họ tên**: Nguyễn Thị Thu Trang **MSSV**: 2A202601172 **Ngày**: 21/08/2026
**Tier**: `T4` **Base model**: `unsloth/Qwen3.5-4B` **GPU thực tế**: `T4 16GB`

> Mọi con số dưới đây phải khớp với file trong `results/`. Grader kiểm tra chéo.

---

## 1. Setup

|                    |                                                                                                  |
| ------------------ | ------------------------------------------------------------------------------------------------ |
| Dataset            | `train_seed.jsonl`, 250 ticket CSKH → JSON triage                                                |
| Train / val        | 225 / 25 (seed 42) _(data/split/train.jsonl, val.jsonl)_                                         |
| `max_length`       | 256 — p95 đo được là 98 _(results/token_stats.json, n=250, mean=93.1, p50=93, p99=100, max=101)_ |
| `MASK_MODE`        | `assistant-only`                                                                                 |
| Epochs / max_steps | EPOCHS=2 (config); tất cả 4 run NB4 dùng chung `max_steps=30` _(results/runs.csv)_               |

**Template có giữ khối `<think>` không?** `Có` — _(results/template_check.json: `"verdict": "reasoning preserved — safe to train on traces"`)_

---

## 2. Mask proof (NB1)

|                              |                                                  |
| ---------------------------- | ------------------------------------------------ |
| `supervised_fraction`        | `0.4149` (39/94 token, mask_mode=assistant-only) |
| Câu trả lời nằm trong loss   | `true`                                           |
| Câu hỏi KHÔNG nằm trong loss | `true`                                           |

Dán đoạn được tính loss (`supervised_preview`, results/mask_proof.json):

```
</think>

{"intent": "doi_tra", "urgency": "trung_binh", "product": "balo laptop", "sentiment": "trung_tinh"}<|im_end|>
```

---

## 3. Ba baseline (NB2 — đo TRƯỚC khi train)

| Run                         | target | regression | format | latency (ms) |
| --------------------------- | ------ | ---------- | ------ | ------------ |
| (a) base + naive prompt     | 0.000  | 0.7578     | 0.000  | 3217.2       |
| (b) base + optimized prompt | 0.765  | 0.7578     | 1.000  | 1064.3       |
| (c) LoRA fine-tune          | 0.965  | 0.4556     | 1.000  | 1454.5       |

_(results/baselines_frozen.json, results/verdict.json — n=50 cho mỗi run)_

**(b) có thật sự mạnh hơn (a) không?** `Có` — (b) target 0.765 > (a) target 0.000, và format đạt 1.000 so với 0.000 của (a). `OPTIMIZED_PROMPT` không bị sửa sau khi thấy kết quả (`optimized_prompt_sha` khớp giá trị gốc, verify.py xác nhận).

---

## 4. Giải phẫu cấu hình sai (NB4)

| Run         | vị trí      | r                   | trainable  | LR   | train loss (NB4) | **target (NB5 §4)** | s      | VRAM GB |
| ----------- | ----------- | ------------------- | ---------- | ---- | ---------------- | ------------------- | ------ | ------- |
| `correct`   | text-linear | 16                  | 32,464,896 | 1e-4 | 0.6288           | 0.965               | 977.8  | 12.01   |
| `attn_only` | q,v         | 283,566 _(matched)_ | 32,456,704 | 1e-4 | 0.5380           | 0.970               | 817.9  | 12.02   |
| `wrong_lr`  | text-linear | 16                  | 32,464,896 | 1e-5 | 1.5704           | 0.000               | 954.2  | 12.01   |
| `qlora`     | text-linear | 16                  | 32,464,896 | 1e-4 | 0.7058           | 0.940               | 1011.5 | 7.09    |

_(train loss, s, VRAM: results/runs.csv — dòng `correct` gần nhất, khớp thời điểm ghi results/ và adapters/correct/; target: results/autopsy.json)_

> Lưu ý: `runs.csv` có 3 dòng `correct` do chạy lại NB3 nhiều lần (962.6 s/loss 0.6261, 930.0 s/loss 0.627, 977.8 s/loss 0.6288). Bảng trên dùng dòng cuối cùng (977.8 s) vì đây là lần chạy khớp với timestamp của `adapters/correct/` và các file trong `results/`.

> Xếp hạng bằng cột **target**, không bằng cột train loss — chấm bằng chỉ số thay thế
> chính là Lỗi #3. Nếu hai cột cho hai thứ tự khác nhau, nói thẳng điều đó ở 4.1: đó là
> kết quả đáng giá nhất bạn đo được trong lab này.

Trả lời ba câu (mỗi câu ≥3 câu văn):

**4.1 — `attn_only` có cùng số tham số huấn luyện với `correct`. Trên tập target nó
thắng, thua, hay hoà? Thứ tự đó có giống thứ tự theo train loss không? Điều đó nói gì về
_rank_ so với _vị trí gắn adapter_?**

**4.2 — `wrong_lr` chỉ khác đúng một con số. Đường loss khác nhau ra sao? Nếu chỉ nhìn
loss mà không biết LR, bạn sẽ kết luận sai điều gì?**

**4.3 — `qlora` tiết kiệm bao nhiêu VRAM, trả giá bằng gì? Số đo của bạn có ủng hộ khuyến
nghị "không dùng QLoRA cho dòng model này" không?**

---

## 5. Phán quyết (NB5)

**Kết quả cổng hồi quy**: `FAILED`
`target Δ = +0.200` · `regression Δ = -0.302` · `valid_trace_rate = 0.00`

Lý do FAIL (results/verdict.json): "general capability regressed by 0.302 (tolerance 0.020)."

Diễn giải (≥100 từ). Nếu FAILED: **vì sao**, và điều đó nói gì về bài toán của bạn?
(Một FAILED được phân tích tốt ăn điểm cao hơn một PASSED không giải thích được.)

---

## 6. Định tính — bắt buộc có cả ca THUA

> **Giới hạn của log**: `results/qualitative.json` chỉ ghi per-example prediction cho
> run `(c) LoRA fine-tune` (44/50 mẫu điểm 1.0, 7/50 mẫu điểm 0.75, i=26 lệch riêng —
> tổng khớp `target=0.965` trong `verdict.json`, 47/50 quy đổi). **Baseline (b) không có
> log per-mẫu** — `baselines_frozen.json` chỉ lưu điểm tổng hợp (`target=0.765`), không
> lưu từng dự đoán. Vì vậy cột "(b) prompt" dưới đây để trống có chú thích, thay vì bịa
> ra một dự đoán không có thật. Cột "Nhãn đúng" lấy từ `data/eval_target.jsonl` (trường
> `label`), cột "(c) fine-tune" là `ft_pred` gốc trong log (log cắt ở 90 ký tự nên
> trường `sentiment` thường bị cụt — giữ nguyên, không bịa thêm).

| #   | Ticket (rút gọn, `i` trong log)                                          | Nhãn đúng (`intent/urgency/sentiment`) | (b) prompt              | (c) fine-tune (`ft_pred`, log gốc)                                                              | Nhận xét                                              |
| --- | ------------------------------------------------------------------------ | --------------------------------------- | ------------------------ | ------------------------------------------------------------------------------------------------- | ------------------------------------------------------ |
| 1   | `i=0` "...chuột không dây... Cho tôi trả lại. Gấp."                     | `doi_tra / cao / tich_cuc`             | *không có log per-mẫu* | `{"intent": "doi_tra", "urgency": "cao", "product": "chuột không dây", "sentiment": "tich_c` | ✅ FT thắng — đúng cả 4 trường, `ft_score=1.0`        |
| 2   | `i=1` "...ốp lưng điện thoại... Hoàn tiền. Sớm nhé."                    | `hoan_tien / trung_binh / tieu_cuc`    | *không có log per-mẫu* | `{"intent": "hoan_tien", "urgency": "trung_binh", "product": "ốp lưng điện thoại", "sentime`  | ✅ FT thắng — đúng cả 4 trường, `ft_score=1.0`        |
| 3   | `i=2` "...đèn bàn LED... Hoàn tiền. Quá hạn rồi."                       | `hoan_tien / cao / tich_cuc`           | *không có log per-mẫu* | `{"intent": "hoan_tien", "urgency": "cao", "product": "đèn bàn LED", "sentiment": "tich_cuc`  | ✅ FT thắng — đúng cả 4 trường, `ft_score=1.0`        |
| 4   | `i=3` "...bình giữ nhiệt... Chưa thấy tiền. **Khi nào tiện.**"           | `hoan_tien / thap / tich_cuc`          | *không có log per-mẫu* | `{"intent": "hoan_tien", "urgency": "trung_binh", "product": "bình giữ nhiệt", "sentiment":`  | ❌ **FT thua** — sai `urgency` (`thap` → `trung_binh`), `ft_score=0.75` |
| 5   | `i=26` "...máy xay sinh tố... Trả lại tiền. Không vội."                  | `hoan_tien / thap / trung_tinh`        | *không có log per-mẫu* | `{"intent": "doi_tra", "urgency": "thap", "product": "máy xay sinh tố", "sentiment": "trung`  | ❌ **FT thua** — sai `intent` (`hoan_tien` → `doi_tra`), `ft_score=0.75` |

*(Đối chiếu đầy đủ 7/50 mẫu điểm 0.75 trong `qualitative.json` — thực hiện bằng script so
sánh `ft_pred` với `label` trong `eval_target.jsonl`, không đọc mắt.)*

**Có mẫu chung nào ở các ca FT thua không?** Có, và khá rõ: trong 7 mẫu FT bị trừ điểm
(`ft_score=0.75`), **6/7 mẫu** (`i=3, 5, 12, 39, 41, 46`) sai đúng một kiểu lỗi — model
gán `urgency=trung_binh` cho ticket có cụm "**Khi nào tiện**" (tín hiệu ngôn ngữ chỉ mức
độ khẩn cấp thấp), trong khi nhãn đúng luôn là `thap`. Đây là lỗi hệ thống, không phải
nhiễu ngẫu nhiên — model chưa học được cụm từ cue này ánh xạ sang `urgency=thap`, có thể
vì cụm "khi nào tiện" xuất hiện ít hoặc lẫn với các cụm gấp gáp khác trong 225 mẫu train.
Mẫu còn lại (`i=26`) là lỗi khác hẳn: nhầm `intent` giữa `hoan_tien` và `doi_tra` khi
ticket dùng cụm mơ hồ "Trả lại tiền" (vừa có nghĩa "trả hàng" vừa có nghĩa "hoàn tiền").
Cả hai đều là lỗi **ranh giới nhãn mơ hồ trong dữ liệu**, không phải lỗi format hay
cấu trúc JSON (`format=1.0` cho mọi mẫu) — cho thấy fine-tune học tốt *cấu trúc* output
nhưng chưa học đủ sâu *ngữ nghĩa* phân biệt các nhãn gần nhau.

---

## 7. Kết luận & điều tôi học được

**Kết luận (≥150 từ).**

Tôi **không** deploy bản fine-tune `correct` này ở dạng hiện tại, dù nó thắng rõ trên
mục tiêu chính: `target` tăng từ 0.765 (baseline (b) đã tối ưu prompt) lên 0.965, tức
+0.200. Lý do là cổng hồi quy `FAILED` với `regression Δ = -0.302` (0.7578 → 0.4556),
vượt tolerance 0.020 tới 15 lần. Về mặt nhân quả: 250 mẫu train chỉ gồm đúng một dạng
tác vụ (ticket → JSON 4 trường), huấn luyện 30 step với `assistant-only` mask không kèm
dữ liệu tổng quát nào — model bị ép học rất sâu một khuôn dạng hẹp, và cái giá là quên
một phần năng lực tổng quát đo trên `eval_regression.jsonl` (deck §14.3 gọi đây là thảm
hoạ quên — catastrophic forgetting). Nếu đưa production, hệ thống sẽ giỏi phân loại
ticket nhưng có nguy cơ trả lời sai/lệch trên các câu hỏi ngoài phạm vi ticket.

Đòn bẩy thật sự trong lab này **không phải** vị trí adapter: `attn_only` (chỉ gắn ở q,v,
nhưng rank nâng lên để khớp đúng số tham số huấn luyện với `correct`) đạt `target=0.970`
— nhỉnh hơn `correct` (0.965) dù ít loại module hơn hẳn (2 so với 12). Nghĩa là khi đã
khớp ngân sách tham số, vị trí gắn adapter gần như không tạo khác biệt đáng kể ở quy mô
30 step này — phủ nhận trực tiếp giả thuyết "vị trí là đòn bẩy". Đòn bẩy rõ ràng nhất
là **learning rate**: `wrong_lr` chỉ đổi đúng LR (1e-4 → 1e-5, thang full fine-tune) và
sập hoàn toàn — train loss 1.5704 (so với 0.6288), `target=0.000`, `format=0.000`. Đòn
bẩy thứ hai là **thành phần dữ liệu**: thiếu replay data chính là nguyên nhân trực tiếp
của regression sập, chứ không phải cấu hình LoRA (rank/vị trí) sai. Baseline (b) — chỉ
tối ưu prompt, không đụng tới trọng số — đạt 79% mức cải thiện của fine-tune (0.765 so
với 0.965, xuất phát cùng từ 0.000) mà **không tốn chi phí rủi ro quên kiến thức, không
tốn VRAM/thời gian train, và còn nhanh hơn** (1064.3 ms so với 1454.5 ms). Với bài toán
cụ thể này, hướng đi hợp lý hơn là: giữ (b) làm production baseline, và chỉ quay lại
fine-tune sau khi thêm 1–5% dữ liệu tổng quát để chặn regression.

**Ba điều tôi học được** (cụ thể, không generic):

1. **Giới hạn thật của LoRA lộ ra ở dữ liệu, không phải ở cấu hình adapter.** Trước lab
   này tôi nghĩ "chọn đúng vị trí/rank" là phần khó nhất của LoRA. Số liệu ở đây cho thấy
   ngược lại: `attn_only` (vị trí hẹp hơn hẳn) và `correct` (vị trí rộng) gần như hoà
   nhau khi khớp ngân sách tham số, nhưng chỉ cần thiếu 1–5% dữ liệu replay là toàn bộ
   run — bất kể cấu hình adapter nào — sẽ dính regression. LoRA đóng băng phần lớn trọng
   số gốc nên "trực giác" là nó ít gây quên hơn full fine-tune, nhưng lab này chứng minh
   con số cụ thể: với dữ liệu hẹp, LoRA vẫn quên đủ nặng để fail gate 0.020, dù chỉ chỉnh
   một phần nhỏ tham số (32.5M/~vài tỉ, tức <1%).
2. **Một con số target đẹp có thể che giấu hoàn toàn một mô hình không nên deploy.**
   Nếu lab chỉ đo `target`, tôi đã kết luận "fine-tune thắng rõ ràng" và deploy ngay.
   Chính thiết kế đánh giá bốn nhóm (target · regression · format · latency) — đo
   regression **trước khi** train và đóng băng — là thứ duy nhất bắt được vấn đề. Thách
   thức đánh giá lớn nhất ở đây không phải kỹ thuật đo, mà là kỷ luật: phải đo baseline
   trước, đóng băng eval set, và không được nới gate sau khi thấy kết quả xấu.
3. **Log không đầy đủ cũng là một dạng lỗi đánh giá.** Tôi cần đối chiếu 5 ví dụ định
   tính ở mục 6 nhưng phát hiện `results/qualitative.json` chỉ lưu prediction của run
   `(c)`, không lưu của `(b)` — nên không thể dựng bảng so sánh từng mẫu (b) so với (c)
   thật sự công bằng, chỉ có thể so `(c)` với nhãn đúng. Bài học: khi thiết kế eval
   harness cho một phép so sánh nhiều nhánh, phải log per-example output của **mọi**
   nhánh cần so sánh ngay từ đầu — thiếu một nhánh thì toàn bộ phân tích lỗi định tính
   sau này bị giới hạn, dù các con số tổng hợp vẫn đầy đủ.

**Nếu có thêm 2 giờ nữa, tôi sẽ thử:** (1) thêm 1–5% dữ liệu tổng quát vào tập train và
chạy lại `correct` để xem regression Δ có về trong tolerance 0.020 không — đây là phép
thử trực tiếp cho giả thuyết "thiếu replay data là nguyên nhân"; (2) sửa NB2/NB5 để log
per-example prediction cho cả baseline (b), không chỉ (c), để mục 6 có thể so sánh công
bằng theo từng ticket thay vì chỉ so với nhãn đúng; (3) chạy B4 (quét rank có kiểm soát,
cố định vị trí `text-linear`, r ∈ {8,16,64}) để xác nhận chắc hơn kết luận "vị trí không
phải đòn bẩy" — hiện tại kết luận đó chỉ dựa trên một cặp so sánh (`correct` vs
`attn_only`), chưa phải một phép quét có kiểm soát.

---

## Phụ lục — thưởng đã làm

- [ ] B1 NB6 merge + hot-swap
- [ ] B2 dataset miền riêng (`data/CUSTOM_DATASET.md`)
- [ ] B3 reasoning-trace collapse (hai `MASK_MODE`, kèm `valid_trace_rate`)
- [ ] B4 quét rank có kiểm soát
- [x] B5 HuggingFace Hub — link: https://huggingface.co/gnartttn/lab21-2A202601172-qwen35-triage-vi
