# Báo cáo Ngày 3 — Tracking Annotation

Họ tên / nhóm: Nguyễn Hùng Minh / solo
MSSV: 2A202602069
Ngày: 15/09/2026

---

## 1. Quá trình gán nhãn

| Mục | Giá trị |
| --- | --- |
| Công cụ | CVAT |
| Thời gian gán `clip_02` (warm-up) | 20 phút |
| Thời gian gán `clip_01` | 1 tiếng |
| Số track đã vẽ trong `clip_01` | 8 |
| Số keyframe trung bình mỗi track | 23 |

Ba tình huống khó nhất khi gán clip này, và bạn xử lý thế nào:

1. Xe bị che khuất hoặc chỉ nhìn thấy một phần: giữ nguyên ID khi vẫn còn nhận diện được cùng một xe và đặt bbox ôm phần nhìn thấy được.
2. Hai xe cắt nhau/chồng lấn: rà chậm các frame trước và sau điểm giao nhau, giữ ID theo quỹ đạo liên tục thay vì đổi ID tại điểm che khuất ngắn.
3. Xe xuất hiện hoặc rời khỏi mép ảnh: bắt đầu ở frame đầu tiên xác định chắc chắn là xe bốn bánh và kết thúc ở frame cuối còn nhìn thấy; không đoán phần bbox nằm ngoài ảnh.

## 3. Pre-gold lock và chấm trước/sau rework

Artifact pre-gold (`evidence/pre-gold/clip_01/gt.txt` và `manifest.json`) đã có trong workspace và khớp hash manifest. Bản pre-gold có 566 row, 190 frame và 8 track.

| Evidence | Giá trị |
| --- | --- |
| SHA-256 từ `evidence/pre-gold/clip_01/manifest.json` | `cf237a459fdb3d2805107bc0c74199dc8ba837af7f9d5d2fe8b17a1e8a4c1049` |
| Thời điểm khóa | `2026-09-15T08:57:09.781644+00:00` |
| Số row / frame / track trước khi mở reference | 566 / 190 / 8 |

| | HOTA | DetA | AssA | LocA | IDF1 | MOTA | MOTP | FP | FN | IDSW |
| --- | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| Bản pre-gold | 0.8132 | 0.8058 | 0.8332 | 0.8834 | 0.9375 | 0.9075 | 0.8732 | 22 | 28 | 0|
| Sau rework / bản hiện có | 0.8287 | 0.8149 | 0.8444 | 0.8900 | 0.9535 | 0.9075 | 0.8832 | 23 | 30 | 0 |

Qua cổng (`IDF1 >= 0.80`, `MOTA >= 0.75`, `MOTP >= 0.70`): **có**

Các điểm lỗi còn được evaluator chẩn đoán ở bản hiện có:

| Loại lỗi | Frame | ID | Đã xử lý / ghi nhận |
| --- | --- | --- | --- |
| Bbox lỏng | 190 | 1 | IoU 0.52 với gold; cần rà lại bbox cuối clip |
| Bbox lỏng | 105, 116 | 6 | IoU lần lượt 0.545 và 0.571; cần rà lại vị trí bbox |
| Bbox lỏng | 96 | 5 | IoU 0.581; cần rà lại vị trí bbox |
| Track bị phủ một phần | 105–116 | 6 | Gold chỉ phủ 42/56 frame; cần kiểm tra các frame bị thiếu |
| Ghost bbox | 51, 53, 149, 151 | 4 | Evaluator ghi bbox xuất hiện sớm/muộn so với reference |

## 4. Kết quả model: ByteTrack control vs ReID treatment

Cấu hình từ `outputs/model_run_config.json`:

| Mục | Giá trị |
| --- | --- |
| Python / ultralytics / torch / lap | 3.13.15 / 8.4.145 / 2.11.0+cu128 / 0.5.13 |
| weights / hai tracker | `yolo26n.pt` / `bytetrack.yaml` và `configs/trackers/botsort-reid.yaml` |
| conf / IoU / imgsz / classes | 0.25 / 0.70 / 960 / `[2, 5, 7]` |
| device | `0` |

| So sánh | HOTA | DetA | AssA | LocA | IDF1 | MOTA | MOTP | FP | FN | IDSW |
| --- | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| bạn vs gold | 0.8287 | 0.8149 | 0.8444 | 0.8900 | 0.9535 | 0.9075 | 0.8832 | 23 | 30 | 0 |
| ByteTrack control vs gold | 0.7085 | 0.6487 | 0.7761 | 0.8463 | 0.8746 | 0.7487 | 0.8226 | 88 | 54 | 2 |
| BoT-SORT + ReID vs gold | 0.7635 | 0.7110 | 0.8204 | 0.8721 | 0.9001 | 0.7923 | 0.8595 | 91 | 26 | 2 |
| ReID vs bạn | 0.8228 | 0.7672 | 0.8828 | 0.9186 | 0.9103 | 0.8110 | 0.9125 | 89 | 17 | 1 |

## 5. Phân tích — năm câu hỏi

**1. MOTA của bạn cao hơn hay thấp hơn IDF1? Nếu MOTA cao mà IDF1 thấp thì điều đó nói gì, và vì sao MOTA không phạt nặng lỗi ID?**

MOTA của tôi là 0.9075, thấp hơn IDF1 là 0.9535. Điều này cho thấy identity association tốt, nhưng vẫn còn lỗi coverage/geometry: 23 FP và 30 FN. MOTA chủ yếu tính đúng detection, false positive, false negative và ID switch; một lỗi ID đơn lẻ không bị trừ mạnh như FN/FP, nên MOTA có thể cao ngay cả khi IDF1 thấp hơn đáng kể.

**2. ByteTrack control và BoT-SORT + ReID treatment khác nhau thế nào ở IDF1, AssA và IDSW? Dẫn một frame sequence để giải thích treatment tốt hơn, tệ hơn hoặc không đổi đáng kể. Nhắc rõ đây không cô lập causal effect của ReID vì hai tracker implementation khác.**

ReID cao hơn ByteTrack ở IDF1 (0.9001 so với 0.8746) và AssA (0.8204 so với 0.7761), nhưng cả hai đều có 2 IDSW. Một sequence cho thấy khác biệt là vùng frame 85–121: ByteTrack bị chẩn đoán ID switch ở frame 94 cho GT track 5 và có fragmentation ở track 5, 6, 7; ReID chuyển các điểm switch tương ứng sang frame 87 của track 5 và frame 113 của track 6, đồng thời giảm FN từ 54 xuống 26. ReID cải thiện association/coverage tổng thể nhưng không loại bỏ switch. Đây là so sánh giữa hai implementation tracker khác nhau, không cô lập causal effect riêng của ReID.

**3. DetA, FP và FN đổi thế nào? Lỗi còn lại là detector hay association?**

So với ByteTrack, ReID làm DetA tăng từ 0.6487 lên 0.7110 và FN giảm từ 54 xuống 26, nhưng FP tăng nhẹ từ 88 lên 91. Vì vậy treatment cải thiện coverage nhưng vẫn tạo thêm một số detection/ghost. Các lỗi còn lại là cả detector/geometry và association: DetA, FP, FN phản ánh phần detection; IDSW và fragmentation phản ánh association. Bản annotation của tôi tốt hơn cả hai model về DetA (0.8149), FP (23), FN (30) và IDSW (0).

**4. Một chỗ bạn đúng và ReID sai (frame, ID, vì sao):**

Frame 110, GT track 6 là điểm ReID bị ID switch từ track 28 sang 31 theo `eval_reid_vs_me.json`; vì vậy annotation giữ một ID nhất quán trong khi ReID tách track tại điểm chuyển. Đây là lỗi association của model, không phải bằng chứng cần đổi annotation.

**5. Một chỗ ReID làm bạn xem lại annotation (frame, ID, vì sao), hoặc lý do evidence cho thấy model sai:**

Frame 170, track 8 là một điểm cần xem lại: evaluator ghi ReID có bbox lỏng (IoU 0.551) và ghost pred track 34 ở khu vực này. Tuy nhiên, metric chỉ ra rằng bbox/track của ReID không khớp reference; không có ảnh frame hoặc review log trong workspace để kết luận annotation sai. Vì vậy tôi giữ annotation và đánh dấu đây là điểm cần visual review, không sửa theo model một cách máy móc.

## 6. Nếu phải gán thêm 10 clip nữa

Tôi sẽ bổ sung vào `GUIDELINE_MINI.md` các quy tắc có ngưỡng rõ cho: số frame che khuất tối đa để giữ ID, cách xử lý xe rời rồi quay lại, điểm bắt đầu/kết thúc ở mép ảnh, và cách đặt bbox khi xe bị che. Mỗi ca mơ hồ nên ghi ngay frame và ID cụ thể.

Trong quy trình, tôi sẽ lưu thời gian bắt đầu/kết thúc cho từng clip, thực hiện ba lượt tự kiểm có checklist, xuất pre-gold và `manifest.json` trước khi xem reference, rồi lưu evidence frame-level cho mọi thay đổi sau đánh giá. Cuối cùng sẽ chạy validator và kiểm tra đủ artifact trước khi nộp.

## 7. Tệp đã nộp

- [x] `annotations/clip_01/gt.txt`
- [x] `annotations/clip_02/gt.txt`
- [x] `evidence/pre-gold/clip_01/gt.txt` và `manifest.json`
- [x] `GUIDELINE_MINI.md` đã điền
- [x] `outputs/eval_vs_gold.json`
- [x] `outputs/model_bytetrack_clip_01.txt`
- [x] `outputs/model_reid_clip_01.txt`
- [x] `outputs/model_run_config.json`
- [x] `outputs/eval_bytetrack_vs_gold.json`, `outputs/eval_reid_vs_gold.json`, `outputs/eval_reid_vs_me.json`
- [ ] `reports/review_partner.md` (không áp dụng vì làm solo)
- [x] `reports/REPORT.md` 
