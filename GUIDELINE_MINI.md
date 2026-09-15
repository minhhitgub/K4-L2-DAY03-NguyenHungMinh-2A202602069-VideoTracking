# Mini annotation guideline — Ngày 3 (tracking)

> Điền file này **trong lúc gán nhãn**, không phải sau khi xong. Mỗi lần bạn dừng
> lại nghĩ "cái này tính sao nhỉ?" thì đó là một dòng phải ghi vào đây.
>
> Đây là tài liệu mà người gán nhãn tiếp theo sẽ đọc để làm giống bạn. Nếu hai
> người trong nhóm gán khác nhau, gần như luôn là vì file này chưa nói rõ — chứ
> không phải vì ai kém.

Nhóm / tên: `Nguyễn Hùng Minh` — MSSV `2A202602069` — làm solo
Clip: `clip_01`, `clip_02`

---

## 1. Phạm vi: gán cái gì, không gán cái gì

Một lớp duy nhất: **`vehicle`** — xe bốn bánh (xe con, van, xe buýt, xe tải).

| Gán | Không gán |
| --- | --- |
| xe con, SUV, taxi, xe bán tải | người đi bộ |
| van, minivan | xe đạp |
| xe buýt, minibus | **xe máy / mô tô** |
| xe tải, xe đầu kéo | xe trong ảnh quảng cáo, trong gương, dưới bóng nước |

Bổ sung của nhóm (nếu có): chỉ gán xe bốn bánh nhìn thấy trong cảnh; không suy đoán xe ngoài khung hoặc vật thể trong ảnh phản chiếu/quảng cáo.

## 2. Luật ID — phần quan trọng nhất

| Tình huống | Luật của nhóm | Vì sao |
| --- | --- | --- |
| Xe bị che một phần rồi hiện lại | giữ nguyên ID nếu bị che dưới 25 frame; kiểm tra quỹ đạo trước/sau khi hiện lại | 25 frame là ngưỡng mặc định của lab, tương đương khoảng 2 giây ở 12.5 fps |
| Xe bị che lâu hơn ngưỡng trên | tạo track mới nếu không còn đủ bằng chứng chắc chắn đó là cùng xe | tránh nối nhầm ID khi xe biến mất quá lâu |
| Xe rời khung hình rồi quay lại | tạo track mới | lần xuất hiện sau được xem là một đoạn quan sát mới |
| Hai xe cắt nhau / chồng lên nhau | giữ ID theo vị trí, hướng chuyển động và quỹ đạo trước/sau; không đổi ID chỉ vì có che khuất ngắn | giảm ID switch tại điểm giao nhau |

## 3. Luật bbox

| Tình huống | Luật của nhóm |
| --- | --- |
| Xe bị cắt bởi rìa ảnh | bbox chạm đúng rìa, không đoán phần ngoài ảnh |
| Xe bị xe khác che một phần | bbox ôm phần **nhìn thấy được** |
| Xe vừa xuất hiện, còn rất nhỏ / rất mờ | bắt đầu track từ frame đầu tiên xác định chắc chắn là xe bốn bánh; không gán khi chỉ còn vài pixel hoặc không phân biệt được vật thể |
| Xe đang đỗ, không di chuyển | vẫn giữ track nếu xe còn hiện diện; đặt bbox theo xe ở từng frame và chỉ kết thúc khi xe rời khung hoặc không còn nhìn thấy |
| Keyframe đặt dày ở đâu | đặt dày hơn ở lúc xe xuất hiện/rời khung, bị che, cắt nhau, đổi hướng hoặc bbox biến dạng; đoạn chuyển động ổn định có thể đặt thưa hơn |

## 4. Ít nhất ba ca mơ hồ đã gặp thật

Ghi **frame cụ thể** và **ID cụ thể**, không ghi chung chung.

### Ca 1
- Clip / frame / ID: `clip_01 / 105, 116 / ID 6`
- Tình huống: xe bị che hoặc nhìn thấy không đầy đủ; evaluator ghi IoU bbox lần lượt 0.545 và 0.571, đồng thời gold chỉ phủ 42/56 frame của track 6.
- Quyết định: giữ ID 6 xuyên suốt đoạn nếu vẫn nhận ra cùng xe; rà lại bbox ở các frame có IoU thấp và không tự đổi ID theo model.
- Lý do: continuity của quỹ đạo quan trọng hơn việc đổi ID tại một đoạn che khuất ngắn; mọi thay đổi phải dựa trên quan sát frame.

### Ca 2
- Clip / frame / ID: `clip_01 / 51, 53, 149, 151 / ID 4`
- Tình huống: evaluator phát hiện ghost bbox của ID 4 xuất hiện sớm trước khi track tham chiếu xuất hiện và còn tồn tại sau khi track tham chiếu rời khung.
- Quyết định: chỉ giữ bbox trong khoảng xe thực sự nhìn thấy; kiểm tra frame vào/ra và dùng `outside` khi xe rời cảnh.
- Lý do: không được kéo dài track theo dự đoán ngoài vùng quan sát.

### Ca 3
- Clip / frame / ID: `clip_01 / 190 / ID 1`
- Tình huống: bbox cuối clip có IoU 0.52 với gold, sát ngưỡng đánh giá 0.5.
- Quyết định: đặt bbox ôm đúng phần xe nhìn thấy đến frame cuối, chạm mép ảnh nếu bị cắt và không đoán phần nằm ngoài ảnh.
- Lý do: frame cuối dễ bị kéo bbox theo quán tính; cần rà riêng frame kết thúc thay vì sao chép keyframe trước đó.

## 5. Sửa gì sau khi chấm với gold và sau khi kiểm chéo

Luật nào trong file này hoá ra còn thiếu hoặc còn mơ hồ? Viết lại cho rõ:

- Khi xe rời khung rồi xuất hiện lại, mặc định tạo track mới; chỉ nối ID nếu đề bài hoặc bằng chứng liên tục trong clip cho phép, không nối chỉ vì hình dáng tương tự.
- Mọi bbox ở frame đầu/cuối, đoạn che khuất và điểm giao nhau phải được rà riêng. Validator chỉ bắt lỗi định dạng, không chứng minh bbox đúng; cảnh báo track 2 đứng yên từ frame 1–15 cần được xem bằng mắt để phân biệt xe đỗ với quên `outside`.
- Sau khi chấm, các điểm cần visual review của bản hiện tại gồm frame 96 ID 5, frame 105/116 ID 6 và frame 190 ID 1; không sửa theo model nếu không có bằng chứng từ ảnh.
