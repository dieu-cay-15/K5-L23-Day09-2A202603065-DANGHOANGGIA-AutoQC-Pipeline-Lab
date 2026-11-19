# Báo cáo Day 09 — Đặng Hoàng Gia, 2A202603065

## 1. Kết luận trước

Pipeline kết luận **Hold** (mã thoát `10`).

- **Số lượng ảnh đầu vào / đầu ra:** Từ lô đầu vào **224 ảnh** (`data/raw`), Bước 1 (`intake`) đã phát hiện và loại bỏ **22/224 ảnh** vi phạm lỗi bắt buộc cấp file (gồm 2 ảnh hỏng `CORRUPT_IMAGE`, 3 ảnh không đạt độ phân giải tối thiểu `BAD_SIZE`, 8 ảnh trùng từng byte `DUP_IMAGE_EXACT`, 4 ảnh trùng gần đúng `DUP_IMAGE_NEAR` và 5 ảnh rò rỉ từ tập kiểm thử `LEAK_TEST`), giữ lại **202/224 ảnh** đưa vào dataset sạch (`submission/export/clean_v1/`, tổng cộng **748 bounding boxes** hợp lệ trên 4 lớp: `car` 101, `person` 574, `truck` 41, `bus` 32).
- **Lý do chính kết luận Hold:** Toàn bộ lỗi bắt buộc cấp dòng và cấp dataset (36 lỗi ở vòng 1) đã được máy tự động sửa (`03_fixes.csv`) và người duyệt mô phỏng gán bổ sung cho 6/6 ảnh thiếu file nhãn (`MISSING_LABEL_FILE`), nên sau vòng 3 dataset còn **0 lỗi bắt buộc (`n_errors = 0`)**. Tuy nhiên, do hàng đợi cần người xem có **125/202 ảnh** trong khi ngân sách duyệt chỉ giới hạn **80 ảnh** (`review.budget_images = 80`), sau bước duyệt vẫn còn **38/202 ảnh (`warn_ratio = 0.1881 = 18.81%`)** mang cảnh báo chưa qua mắt người — vượt ngưỡng chặn `gate.warn_ratio_hold = 0.10` (`10.0%`). Vì vậy pipeline dừng ở trạng thái **Hold** (chỉ xuất bản nháp `manifest.json` với `decision = "Hold"`, chưa phát hành chính thức).

---

## 2. Lỗi và phân việc

Bảng dưới đây tổng hợp toàn bộ **21 mã lỗi/cảnh báo** (58 lỗi bắt buộc `ERROR` và 153 cảnh báo `WARN`, tổng cộng 211 dòng) ghi nhận trong `submission/reports/02_error_list_v1.txt`:

| Mã lỗi | Mức độ | Số lượng (kèm mẫu số) | Ai sửa (máy/người) | Vì sao phân việc như vậy |
|---|:-:|---|---|---|
| `CORRUPT_IMAGE` | `ERROR` | 2 / 224 ảnh (`img_0010`, `img_0136`) | **Máy** (loại ở intake / `remove_image`) | File ảnh hỏng byte hoặc cắt cụt không giải mã được bằng `im.load()`, không thể dùng để train. |
| `BAD_SIZE` | `ERROR` | 3 / 224 ảnh (`img_0107`, `img_0113`, `img_0122`) | **Máy** (loại ở intake / `remove_image`) | Cạnh ngắn `< 250 px` (180 px, 226 px), vi phạm quy tắc độ phân giải tối thiểu của dự án. |
| `DUP_IMAGE_EXACT` | `ERROR` | 8 / 224 ảnh | **Máy** (loại ở intake / `remove_image`) | Trùng mã băm `md5` từng byte với ảnh trước đó; giữ ảnh đầu tiên và xoá bản sao là an toàn tuyệt đối. |
| `DUP_IMAGE_NEAR` | `ERROR` | 4 / 224 ảnh (`img_0076`, `img_0143`, `img_0186`, `img_0223`) | **Máy** (loại ở intake / `remove_image`) | Trùng gần đúng (`phash_dist <= 10`) do thu nhỏ, cắt nhẹ hoặc đổi độ sáng; giữ lại sẽ làm lệch phân bố train. |
| `LEAK_TEST` | `ERROR` | 5 / 224 ảnh (`img_0049`, `img_0101`, `img_0167`, `img_0174`, `img_0221`) | **Máy** (loại ở intake / `remove_image`) | Trùng `md5` hoặc `phash` với ảnh trong `data/test/images`; bắt buộc loại để tránh rò rỉ dữ liệu làm metric đẹp giả. |
| `BAD_CLASS_NAME` | `ERROR` | 2 / 4 lớp trong `data.yaml` (`Person`, `Bus`) | **Máy** (`autofix`: `rename_classes`) | Lỗi quy ước đặt tên/viết hoa trong `data.yaml`; máy chuẩn hoá về `[car, person, truck, bus]` chính xác 100%. |
| `BAD_FORMAT` | `ERROR` | 3 dòng / 3 ảnh (trên 202 ảnh) | **Máy** (`autofix`: `drop_line`) | Dòng thiếu cột (4 cột) hoặc chứa `nan`; máy không thể đoán toạ độ bị mất nên xoá dòng hỏng và đẩy ảnh vào hàng đợi duyệt. |
| `BAD_CLASS_ID` | `ERROR` | 6 dòng / 6 ảnh (trên 202 ảnh) | **Máy** (`autofix`: `drop_line`) | ID lớp ngoài `0..3` (`-1`, `4`, `7`, `93`); máy xoá dòng sai cú pháp rồi cộng điểm `FIXED_DROP_LINE` để người kiểm lại. |
| `BOX_ZERO` | `ERROR` | 4 dòng / 4 ảnh (trên 202 ảnh) | **Máy** (`autofix`: `drop_line`) | Box suy biến có `w <= 0` hoặc `h <= 0` (diện tích bằng 0), không mang thông tin hình học. |
| `DUP_BOX` | `ERROR` | 7 dòng / 7 ảnh (trên 202 ảnh) | **Máy** (`autofix`: `drop_line`) | Hai dòng trong cùng file gán cùng một lớp và chồng khít `IoU > 0.90`; xoá dòng lặp phía sau là an toàn. |
| `BOX_OUT_OF_IMAGE` | `ERROR` | 8 dòng / 8 ảnh (trên 202 ảnh) | **Máy** (`autofix`: `clip_box`) | Mép box tràn khỏi `[0, 1]` quá `overflow_tol = 0.002` (mức tràn `0.023–0.060`); máy cắt mép về `[0, 1]` giữ nguyên nhãn lớp. |
| `MISSING_LABEL_FILE` | `ERROR` | 6 / 202 ảnh (`img_0031`, `img_0053`, `img_0126`, `img_0135`, `img_0170`, `img_0219`) | **Người** (`autofix` ghi `needs_label` $\rightarrow$ đầu hàng đợi) | Ảnh chưa có file `.txt`; máy tuyệt đối không được tạo file rỗng (vì rỗng nghĩa là ảnh nền không có vật), phải chuyển người gán. |
| `BOX_HUGE` | `WARN` | 3 dòng / 3 ảnh (trên 202 ảnh) | **Người** (hàng đợi duyệt) | Box chiếm `> 95%` diện tích ảnh (`0.970–0.988`); có thể là xe buýt/người chụp cận cảnh thật hoặc box vẽ lỗi, cần người nhìn ảnh. |
| `BOX_TINY` | `WARN` | 6 dòng / 4 ảnh (trên 202 ảnh; lên 7 dòng ở v2 sau clip) | **Người** (hàng đợi duyệt) | Cạnh ngắn `< 12 px` (`8.9–11.9 px`); vật thể ở xa hay nhiễu phụ thuộc ngữ cảnh và guideline dự án. |
| `CLASS_RARE` | `WARN` | 4 mã lớp lạ (`-1`, `4`, `7`, `93`) trên `<dataset>` | **Máy** (tự hết sau khi `autofix` xoá `BAD_CLASS_ID`) | Thống kê cấp tập dữ liệu phát hiện các token lớp có `< 5` box; biến mất hoàn toàn ở vòng 2 sau khi xoá 6 dòng `BAD_CLASS_ID`. |
| `EMPTY_LABEL` | `WARN` | 12 / 202 ảnh ở v1 (tăng lên 15 / 202 ảnh ở v2 sau khi xoá dòng hỏng) | **Người** (hàng đợi duyệt) | File `.txt` rỗng: máy không phân biệt được đâu là ảnh nền thật (background) và đâu là ảnh bị xoá/bỏ sót hết nhãn. |
| `POSSIBLE_WRONG_CLASS` | `WARN` | 18 dòng / 18 ảnh (trên 202 ảnh) | **Người** (hàng đợi duyệt) | Nhãn khớp vị trí với `yolo26l` (`IoU >= 0.50`) nhưng khác lớp (`car` $\leftrightarrow$ `truck` $\leftrightarrow$ `bus`, `person`); cần con người quyết định ngữ nghĩa. |
| `POSSIBLE_MISSING` | `WARN` | 53 dòng / 52 ảnh (trên 202 ảnh) | **Người** (hàng đợi duyệt) | Model `yolo26l` (`conf >= 0.60`, diện tích `>= 600 px²`) thấy vật nhưng nhãn không có box nào `IoU >= 0.30`; cần người xác nhận bổ sung. |
| `POSSIBLE_LOOSE_BOX` | `WARN` | 28 dòng / 24 ảnh (trên 202 ảnh) | **Người** (hàng đợi duyệt) | Cùng lớp với dự đoán model nhưng `0.30 <= IoU < 0.50`; cần người chỉnh lại biên box cho sát vật thể. |
| `POSSIBLE_FALSE_POS` | `WARN` | 29 dòng / 28 ảnh ở v1 (còn 24 dòng / 23 ảnh ở v2) | **Người** (hàng đợi duyệt) | Box nhãn không khớp bất kỳ dự đoán nào của model (`conf >= 0.10`, `IoU < 0.30`); nghi ngờ gán thừa hoặc sai vị trí. |

### Cơ chế xếp hạng `05_review_queue.csv` và hiệu quả của ngân sách 80 ảnh
- **Vì sao hàng đợi xếp thứ tự như vậy:** Hàm `review_queue.build_queue` tính điểm ưu tiên cho mỗi ảnh theo tổng trọng số mức độ nghiêm trọng nhân với độ tự tin của model:
  $$\text{priority}(img) = \sum_{i \in \text{WARN}} w(\text{code}_i) \cdot (1 + \text{conf}_i) + \sum_{f \in \text{fixes}} w(\text{action}_f)$$
  Trong đó `NEEDS_LABEL` có trọng số cao nhất (`10.0`) vì ảnh chưa có file nhãn là lỗi bắt buộc chặn xuất dataset; tiếp theo là `POSSIBLE_WRONG_CLASS` (`3.0`), `POSSIBLE_MISSING` (`2.0`), `POSSIBLE_FALSE_POS` (`2.0`), `FIXED_DROP_LINE` (`2.0`), `POSSIBLE_LOOSE_BOX` (`1.5`), `EMPTY_LABEL` (`1.0`), `FIXED_CLIP_BOX` (`1.0`), `BOX_HUGE` (`0.5`), `BOX_TINY` (`0.3`).
  - Ảnh đứng số 1 là `img_0020` (`priority = 11.524`) vì vừa bị máy xoá 1 dòng `BAD_CLASS_ID` (`FIXED_DROP_LINE = 2.0`), vừa bị model cờ `POSSIBLE_WRONG_CLASS` (`3.0 * (1 + 0.9474) = 5.842`) và `POSSIBLE_MISSING` (`2.0 * (1 + 0.8408) = 3.682`).
  - Ngay vị trí thứ 2 đến thứ 7 là toàn bộ **6/6 ảnh `NEEDS_LABEL`** (`img_0031`, `img_0053`, `img_0126`, `img_0135`, `img_0170`, `img_0219`, `priority = 10.0`), đảm bảo người duyệt luôn gán xong các ảnh thiếu nhãn trước khi hết ngân sách.
- **Số lỗi ngữ nghĩa được người xem trong ngân sách 80 ảnh:** Trên tổng số **125 ảnh** trong hàng đợi (chứa **123 cảnh báo ngữ nghĩa** của `second_opinion` ở vòng 2), **80 ảnh đầu hàng đợi** đã giúp người duyệt xử lý được:
  - **6 / 6 (100%)** ảnh thiếu file nhãn (`MISSING_LABEL_FILE` / `NEEDS_LABEL`);
  - **18 / 18 (100%)** cảnh báo gán nhầm lớp (`POSSIBLE_WRONG_CLASS`);
  - **51 / 53 (96.2%)** cảnh báo thiếu box (`POSSIBLE_MISSING` — chỉ còn 2 ca có `conf` thấp trên `img_0210` và `img_0032` nằm ở vị trí 81–82);
  - **14 / 24 (58.3%)** cảnh báo thừa box (`POSSIBLE_FALSE_POS`);
  - **10 / 28 (35.7%)** cảnh báo box lỏng (`POSSIBLE_LOOSE_BOX`);
  - Tổng cộng người duyệt đã kiểm tra được **93 / 123 (75.6%)** lỗi ngữ nghĩa từ `second_opinion` (và **11 / 15** ảnh `EMPTY_LABEL`), tập trung trọn vẹn vào các lỗi sai lớp và bỏ sót vật thể có độ tự tin cao nhất.

---

## 3. Calibration (2 ngưỡng)

Kết quả sweep trên bộ `data/calib/` (39 ảnh có đáp án trong `data/calib/MANIFEST.csv`) được lưu tại `submission/calibration/calibration_report.md`:

| Tham số | Khởi đầu | Đã chọn | Precision / Recall / F1 (trước $\rightarrow$ sau) | Lý do chọn (đánh đổi Precision – Recall) |
|---|---:|---:|---|---|
| `duplicates.phash_dist` | `2` | **`10`** | - `DUP_IMAGE_EXACT`: `1.00 / 1.00 / 1.00` $\rightarrow$ `1.00 / 1.00 / 1.00`<br>- `DUP_IMAGE_NEAR`: `1.00 / 0.50 / 0.67` $\rightarrow$ **`1.00 / 0.67 / 0.80`** | Ảnh bị cờ trùng sẽ bị máy **xoá tự động** ở Bước 1, nên ưu tiên số 1 là **Precision = 1.00** (không xoá nhầm ảnh khác nhau). Tăng ngưỡng từ `2` lên `10` giúp Recall trên `calib` tăng từ `0.50` (3/6) lên `0.67` (4/6) mà vẫn giữ Precision tuyệt đối `1.00`. Trên `data/raw`, mức `10` bắt thêm đúng 2 ảnh trùng gần đúng (`img_0223` có `dist=6`, `img_0186` có `dist=10`) và 1 ảnh rò rỉ test (`img_0221` trùng `test_009.jpg` ở `dist=6`) với 0 ca xoá nhầm. |
| `second_opinion.conf` | `0.25` | **`0.60`** | - `POSSIBLE_WRONG_CLASS`: `0.56 / 0.83 / 0.67` $\rightarrow$ **`0.62 / 0.83 / 0.71`**<br>- `POSSIBLE_MISSING`: `0.50 / 1.00 / 0.67` $\rightarrow$ **`0.57 / 1.00 / 0.73`**<br>- `POSSIBLE_LOOSE_BOX`: `0.29 / 0.40 / 0.33` $\rightarrow$ `0.20 / 0.20 / 0.20`<br>- `POSSIBLE_FALSE_POS`: `0.31 / 1.00 / 0.48` $\rightarrow$ `0.31 / 1.00 / 0.48` | `second_opinion` chỉ sinh cảnh báo đưa vào hàng đợi người duyệt (ngân sách cố định 80 ảnh). Ngưỡng `0.60` là điểm cân bằng tối ưu: đây là mức cao nhất vẫn **giữ nguyên Recall tối đa** cho hai lỗi quan trọng nhất là `POSSIBLE_MISSING` (`Recall = 1.00`, 8/8) và `POSSIBLE_WRONG_CLASS` (`Recall = 0.83`, 5/6), đồng thời nâng Precision của `WRONG_CLASS` từ `0.56` lên `0.62` và `MISSING` từ `0.50` lên `0.57` (giảm hàng đợi trên `raw` từ 143 xuống 125 ảnh). Nếu nâng lên `0.70–0.80` để ép hàng đợi nhỏ lại, Recall của `POSSIBLE_MISSING` tụt xuống `0.88` rồi `0.75` (bỏ lọt vật thể bị thiếu nhãn). |

### Điểm đánh lừa của Calibration và bằng chứng trên dữ liệu thực (`data/raw`)
- **Bẫy khi chỉ nhìn chỉ số F1 trên bộ `calib` nhỏ (39 ảnh):** Trong bảng sweep `duplicates.phash_dist` trên tập `calib`, giá trị `phash_dist = 16` đạt F1 cao nhất là **`0.91`** (`Precision = 1.00, Recall = 0.83`, bắt được 5/6 ca `DUP_IMAGE_NEAR` và có `FP = 0`). Nếu chọn máy móc theo F1 cao nhất trên `calib`, ta sẽ chọn `phash_dist = 16`.
- **Bằng chứng sai lệch trên lô `data/raw` (224 ảnh):** Tập `calib` chỉ có 39 ảnh ($\binom{39}{2} = 741$ cặp ảnh), trong khi lô `raw` có 224 ảnh ($\binom{224}{2} = 24.976$ cặp nội bộ và $224 \times 140 = 31.360$ cặp so với kho `test`/`gold_val` — gấp hơn **75 lần** số phép so sánh cặp). Khi chạy thử `phash_dist = 16` trên `data/raw`, số ảnh bị cờ trùng/rò rỉ nhảy vọt từ **17 ảnh** (ở `phash_dist = 10`) lên **26 ảnh** — tức có **9 ảnh hoàn toàn khác nhau bị xoá oan**:
  1. Mở xem trực tiếp cặp `img_0011.jpg` và `img_0008.jpg` (bị cờ `DUP_IMAGE_NEAR` ở `dist = 16`): `img_0008.jpg` là ảnh **hai người trượt tuyết trên đỉnh núi tuyết trắng**, còn `img_0011.jpg` là ảnh **nữ vận động viên quần vợt đang vung vợt trên sân xanh** — hai bức ảnh hoàn toàn không liên quan nhưng tình cờ có bố cục gradient tần số thấp cách nhau 16 bit Hamming.
  2. Tương tự ở `dist = 16`, có thêm 2 cặp ảnh khác nhau bị gộp nhầm (`img_0084` với `img_0074`, `img_0177` với `img_0113`) và **6 ảnh hợp lệ** (`img_0013`, `img_0017`, `img_0021`, `img_0033`, `img_0038`, `img_0115`) bị báo nhầm là `LEAK_TEST` với tập `data/test/images`.
  3. Ngược lại, ở `phash_dist = 10`, kiểm tra trực quan toàn bộ 4 cặp `DUP_IMAGE_NEAR` (`img_0076` – `img_0042` ở `dist=0`; `img_0143` – `img_0037` ở `dist=0`; `img_0223` – `img_0077` tháp đồng hồ & xe buýt bị tăng sáng ở `dist=6`; `img_0186` – `img_0061` người ngồi ghế công viên bị thu nhỏ ở `dist=10`) và 5 cặp `LEAK_TEST` (như `img_0221` trùng ảnh trụ cứu hoả `test/test_009.jpg` ở `dist=6`) đều là ảnh trùng thật 100%.

---

## 4. Cửa chặn và điều pipeline KHÔNG bắt được

### (a) Chứng minh cửa chặn (`gate`) hoạt động đúng
Khi chạy pipeline với ngân sách người duyệt bằng 0 trên thư mục báo cáo/xuất riêng:
```bash
python -m pipeline.run --budget 0 --reports work/rep_nb --export-dir work/exp_nb
```
- **Diễn biến & Kết quả:** Ở vòng 1 và bước `autofix`, máy tự sửa 29 lỗi cú pháp dòng/file nhưng giữ nguyên **6 lỗi bắt buộc `MISSING_LABEL_FILE`** (`img_0031`, `img_0053`, `img_0126`, `img_0135`, `img_0170`, `img_0219`) dưới dạng `needs_label` để chờ người gán. Vì `--budget 0` (không có người duyệt), cả 6 ảnh này vẫn không có file `.txt` ở vòng QC thứ 3 (`work/rep_nb/07_error_list_v3.txt` còn 6 dòng `[ERROR] MISSING_LABEL_FILE`).
- **Quyết định của Gate:** Hàm `gate.decide` ghi nhận `n_errors = 6 >= 1`, lập tức kết luận **`Rework`** (` razones: ["còn 6 lỗi bắt buộc chưa xử lý (cảnh báo còn 112/202 ảnh)"]`), pipeline kết thúc với **mã thoát `20`** (`BUDGET0_EXIT_CODE=20`).
- **Chặn xuất dữ liệu:** Bước 5 in rõ `Bước 5/5 KHÔNG xuất: còn lỗi bắt buộc (Rework) — trả về bước 3`; trong `work/rep_nb/08_summary.json` trường `"exported"` có giá trị `null` và thư mục `work/exp_nb/clean_v1` **không tồn tại**. Cửa chặn đảm bảo dữ liệu còn lỗi bắt buộc không bao giờ lọt sang khâu huấn luyện.

### (b) Các điểm mù mà pipeline KHÔNG bắt được (kèm bằng chứng)
1. **Ảnh trùng gần đúng nhưng bị cắt (crop) sâu hoặc biến đổi hình học mạnh:**
   - *Bằng chứng:* Trong tập calibration (`data/calib/MANIFEST.csv`), ảnh `calib_038.jpg` là bản sao của `calib_024.jpg` ở `mức biến đổi = 4` (cắt phần lớn khung hình rồi thu phóng lại). Vì perceptual hash (`phash`) dựa trên biến đổi DCT 8x8 của toàn ảnh, việc cắt sâu làm thay đổi hoàn toàn bố cục không gian toàn cục khiến khoảng cách Hamming vượt quá 20. Ngay cả khi quét `phash_dist` lên tới `20`, `calib_038.jpg` vẫn là False Negative (`FN`).
2. **Box gán lỏng ở mức trung bình (`0.50 <= IoU < 0.85`):**
   - *Bằng chứng:* Ngưỡng dự án quy định `second_opinion.iou_loose = 0.50` (cố định). Do đó, mọi box nhãn tuy vẽ rộng hoặc hẹp hơn vật thể thật khá nhiều nhưng vẫn có độ chồng lấp `IoU >= 0.50` với box của `yolo26l` đều được coi là "đã phủ đúng vật" và không bị cờ. Trên `data/calib`, trong 5 lỗi `LOOSE_BOX` chèn cố ý (`scale = 1.45–1.58`, như `calib_002`, `calib_006`, `calib_012`, `calib_023`, `calib_024`), có tới **4/5 ca** vẫn có `IoU >= 0.50` với dự đoán của model checker nên Recall của `POSSIBLE_LOOSE_BOX` chỉ đạt `0.20` (1/5). Khi xem ảnh trực quan bằng `scripts/viz_labels.py img_0001 img_0030`, một số box người đi bộ trên `img_0030` hơi dư khoảng trống trên đầu/dưới chân nhưng `IoU > 0.50` nên pipeline không cảnh báo.
3. **Lỗi ngữ nghĩa ở vật thể nhỏ (`< 600 px²`) hoặc nơi model checker (`yolo26l`) cũng nhầm/bỏ sót:**
   - *Bằng chứng:* `second_opinion` chỉ đối chiếu với một model `yolo26l` duy nhất và bỏ qua mọi dự đoán có diện tích `< min_area_px = 600 px²` hoặc `conf < 0.60`. Chẳng hạn trên `img_0004.jpg` và `img_0030.jpg`, các người đi bộ/xe hơi ở rất xa có diện tích dự đoán chỉ `150–450 px²` hoặc `conf = 0.11–0.36`; nếu người gán nhãn bỏ sót các vật thể nhỏ này, pipeline hoàn toàn im lặng. Ngược lại, sau khi người duyệt thay nhãn chuẩn COCO cho 80 ảnh đầu (`07_error_list_v3.txt`), `yolo26l` vẫn báo `POSSIBLE_WRONG_CLASS` trên `img_0078` và `img_0083` hay `POSSIBLE_FALSE_POS` trên 21 dòng nhãn người chuẩn — chứng tỏ bản thân model lớn cũng có điểm mù và định kiến riêng.

---

## 5. Train: raw vs clean (tùy chọn, chấm riêng)

- **Trạng thái:** **Chưa chạy** (bài nộp hoàn thành và kiểm tra theo gói phần lõi `make check-core` / `python scripts/check_submission.py --no-train` trên môi trường local không có GPU).
- **Quy trình và nguyên tắc đánh giá trung thực khi chạy thực nghiệm trên Colab:**
  - Huấn luyện công bằng cùng kiến trúc `yolo26n`, cùng tập `gold_val` (40 ảnh) và tập `test` (100 ảnh), cùng siêu tham số (`epochs = 20`) trên tối thiểu 2 seed (`seed 0` và `seed 1`) cho hai biến thể `raw` (224 ảnh gốc) và `clean` (202 ảnh đã qua pipeline).
  - Khi đọc kết quả `comparison.md`, phải so sánh mức chênh lệch trung bình $\Delta \text{mAP50} = \overline{\text{mAP50}}_{\text{clean}} - \overline{\text{mAP50}}_{\text{raw}}$ với **độ lệch chuẩn (std) giữa các seed**. Nếu $|\Delta \text{mAP50}|$ nhỏ hơn hoặc tương đương độ lệch giữa các seed, phải kết luận trung thực là **khác biệt nằm trong nhiễu thống kê (chưa kết luận được biến thể nào tốt hơn)**.
  - **Hai giả thuyết giải thích nếu `raw` có mAP50 tương đương hoặc cao hơn `clean` và cách kiểm chứng:**
    1. *Giả thuyết 1 — Rò rỉ tập test (`LEAK_TEST`) trong `raw`:* Lô `raw` chứa **5 ảnh trùng trực tiếp với tập `data/test/images`** (`img_0049` $\equiv$ `test_033`, `img_0101` $\equiv$ `test_039`, `img_0167` $\equiv$ `test_044`, `img_0174` $\equiv$ `test_085`, `img_0221` $\equiv$ `test_009`). Model train trên `raw` được học thuộc lòng 5% tập test nên điểm mAP trên `test` bị thổi phồng giả tạo, trong khi `clean` đã loại bỏ 5 ảnh rò rỉ này. *Cách kiểm:* Đánh giá lại cả 2 model trên tập `test` đã loại bỏ 5 ảnh rò rỉ (`test_009, test_033, test_039, test_044, test_085`) hoặc so sánh trên `gold_val`.
    2. *Giả thuyết 2 — Độ nhiễu của tập test nhỏ (100 ảnh) và 45 ảnh trong hàng đợi chưa được người duyệt:* Tập `clean` ít hơn `raw` 22 ảnh và vẫn còn 38 ảnh mang cảnh báo ngữ nghĩa chưa qua mắt người (do ngân sách duyệt dừng ở 80 ảnh), đồng thời các lớp ít mẫu như `bus` (32 box) và `truck` (41 box) rất nhạy cảm với vài box trên tập test 100 ảnh. *Cách kiểm:* So sánh AP theo từng lớp (`per-class AP`) và chạy thêm biến thể tăng ngân sách duyệt lên 125 ảnh.

---

## 6. Quyết định và ghi chú AI

### (a) Quyết định cuối cùng và điều kiện để chuyển sang Release
- **Quyết định cuối:** Giữ nguyên kết luận **Hold** cho lô dữ liệu hiện tại — **chưa phát hành chính thức** cho sản xuất, chỉ sử dụng bản xuất `submission/export/clean_v1/` (`decision: Hold` trong `manifest.json`) làm bản nháp để thử nghiệm.
- **Điều kiện để chuyển từ Hold sang Release:**
  1. Hiện tại dataset có `202` ảnh, không còn lỗi bắt buộc (`n_errors = 0`), nhưng còn `38` ảnh có cảnh báo chưa được người duyệt (`38 / 202 = 18.81% >= 10%`).
  2. Để tỉ lệ cảnh báo chưa duyệt xuống dưới ngưỡng `gate.warn_ratio_hold = 10.0%` (tức số ảnh còn cảnh báo `< 20.2` $\Rightarrow$ tối đa **20 ảnh**), cần phân bổ thêm ngân sách người duyệt cho ít nhất **$38 - 20 = 18$ ảnh có cảnh báo** nữa (tương ứng duyệt thêm khoảng **19–25 ảnh** tiếp theo trong `05_review_queue.csv`, nâng tổng ngân sách duyệt từ `80` lên khoảng **`100–105` ảnh**), sau đó chạy lại `python -m pipeline.run --resume-after-review` để vượt qua cửa chặn với kết luận **Release** (mã thoát `0`).

### (b) Ghi chú sử dụng AI
1. **Prompt chính (đủ 4 ý theo `docs/PROMPTING.md`):** *"Viết hàm `run(ds, cfg)` và `fix(ds, issues, cfg)` bằng Python 3 cho dữ liệu nhãn YOLO 5 cột chuẩn hoá `id cx cy w h` trong `Dataset` (1 - định dạng vào); kiểm tra đủ các mã lỗi theo bảng docstring (`BAD_FORMAT`, `BAD_CLASS_ID`, `BOX_ZERO`, `BOX_OUT_OF_IMAGE`, `DUP_BOX`, `BOX_TINY`, `BOX_HUGE`, và đối chiếu dự đoán `preds` trong `second_opinion`) với mọi ngưỡng đọc từ `cfg` (2 - quy tắc kiểm); trả về `list[Issue]` giữ nguyên số dòng gốc `line` hoặc `list[dict]` cho `03_fixes.csv` (3 - định dạng ra); tuyệt đối không sửa `data/raw`, không hardcode ngưỡng, và gom lỗi theo file để ghi lại đúng 1 lần (4 - giới hạn an toàn)."*
2. **Cách kiểm tra code AI:** Đọc rà soát từng dòng đảm bảo không chạm vào `data/raw` hay các file được ghim trong `scripts/_protected.json`, chạy toàn bộ 79 unit test và e2e test (`python -m pytest -q` đạt `79 passed`), chạy sweep `pipeline.calibrate` đối chiếu với đáp án `data/calib/MANIFEST.csv`, và kiểm tra trực quan các cặp ảnh trùng cũng như box lỗi trên `data/raw` bằng `scripts/viz_labels.py`.
3. **Một lần AI sai / thiếu và cách sửa:** Khi viết `autofix.py` cho lỗi `BOX_OUT_OF_IMAGE`, bản nháp logic ban đầu chỉ cắt mép box về `[0, 1]` mà quên hai ca biên: (1) một dòng nhãn dính đồng thời 2 lỗi (`BOX_OUT_OF_IMAGE` + `DUP_BOX`) sẽ bị ghi trùng 2 lệnh sửa nếu không gộp theo số dòng gốc `line_no`, và (2) box nằm hoàn toàn ngoài ảnh (ví dụ `cx = -0.18, w = 0.04`) sau khi clip về `[0, 1]` sẽ có chiều rộng `nw = 0.0 <= 0` tạo ra box rỗng `BOX_ZERO`. Mình đã sửa bằng cách gom toàn bộ mã lỗi của từng file vào `defaultdict(set)` theo `line_no`, ưu tiên `drop_line` nếu tập mã lỗi giao với `DROP_LINE_CODES`, và tự động chuyển `clip_box` thành `drop_line` khi `nw <= 0` hoặc `nh <= 0`.
