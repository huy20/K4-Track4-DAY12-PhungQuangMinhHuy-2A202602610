# REPORT — Lab Day 12

> Điền **bằng lời và số liệu của chính bạn** (lấy từ màn hình và từ `evidence/*.json`). Mỗi câu 4–8 dòng là đủ. Số không có trong bằng chứng của bạn sẽ bị trừ điểm. Báo cáo công khai trên GitHub; giảng viên đối chiếu với `evidence/` và với bài của các bạn khác.

| | |
|---|---|
| Họ và tên | Phung Quang Minh Huy |
| MSSV (đúng bằng `--student-id` bạn đã chạy) | 2A202602610 |
| GitHub | huy20 |
| Seed chia dữ liệu / seed mô hình (xem `evidence/closedloop.json`, mục `student`) | 1714 / 9971 |
| Số seed huấn luyện bạn dùng (`--seeds`) | 3 |

## Kết quả cuối

Điền từ `evidence/closedloop.json`, `openloop.json`, `gate.json` (cho mỗi mô hình học, lấy **dải qua các seed**).

| Bộ điều khiển | L2 vòng hở (m) | Va chạm vòng kín trên log (%) | Quyết định cổng |
|---|---|---|---|
| constant-velocity | 1.247 | 19.5 | BLOCK (rớt R2, R3, R4, R6) |
| IDM | 1.525 | 0.03 (1/2883) | (baseline) |
| EgoMLP | 0.755 | 22.9–28.5 (worst 28.5) | BLOCK (rớt R2, R3, R4, R5, R6) |
| PercMLP | 0.626 | 0.6–3.4 (worst 3.4) | BLOCK (rớt R2, R3, R4, R5, R6) |
| PercMLP+ (sau khi sửa) | 0.825 | 0.17–0.24 (worst 0.24) | RELEASE (qua cả 7 quy tắc) |

## Câu 1 · Dữ liệu

Ba vấn đề bước 0 lộ ra. (1) Cột `Acc_LV` **không dùng được**: tương quan của nó với đạo hàm `d(Speed_LV)/dt` chỉ **0.045** (≈ 0, tức gần như nhiễu), trong khi `Acc_FAV` của xe theo sau tương quan **0.929** với đạo hàm `Speed_FAV`. Vì vậy tôi bỏ `Acc_LV` (trường `acc_lead_raw`) và thay bằng gia tốc suy ra từ **tốc độ xe dẫn đã làm mượt** (`a_lead`, `derive_lead` với cửa sổ 5): đây là cách duy nhất để "ca phanh" có nghĩa. (2) Có **383 quỹ đạo trùng hệt** (9993 → 9610), và chúng nằm ở các `Trajectory_ID` **cách xa nhau**, nên chia ngẫu nhiên theo khung hình có thể đưa bản sao của cùng một quỹ đạo vào cả train lẫn test → điểm bị thổi phồng (rò rỉ). (3) Phải **chia theo quỹ đạo**, không chia theo khung hình, vì các khung liền kề của một quỹ đạo gần như giống nhau; chia theo khung sẽ để test "nhìn thấy" gần đúng chuỗi của train. Kiểm tra thêm: `Spatial_Gap = Pos_LV − Pos_FAV − 4.5` khớp tuyệt đối (độ lệch trung bình 4.5 m, lệch chuẩn ~3.6e-11), và mỗi quỹ đạo dài 70–109 bước (trung vị 106).

## Câu 2 · Phát lại

Cùng đoạn log 600 đoạn, dung sai đặt `atol=rtol=1e-3` trên gia tốc lệnh. **Chạy lại cùng mã**: 0/600 đoạn lệch và 0 đoạn khác bit. **Bản refactor float32 (đổi thứ tự tính)**: 0/600 đoạn vượt dung sai, nhưng **600/600 đoạn khác từng bit**, lệch tuyệt đối lớn nhất chỉ **9.7e-05** — vì phép cộng số thực không kết hợp (`(0.1+0.2)+0.3 = 0.6000000000000001` còn `0.1+(0.2+0.3) = 0.6`; tổng 50k float32 tiến so với lùi lệch tương đối 6.9e-06). Vì vậy so bằng nhau từng bit là sai; dung sai 1e-3 là hợp lý vì log đã làm tròn 4 chữ số. **Thay đổi hành vi (thêm AEB)**: **7/600 đoạn lệch**, max diff 1.98, ở các đoạn 58, 75, 198, 235, 239, 325, 484 — đúng những đoạn có TTC thấp (AEB chỉ can thiệp khi `gap/closing < 1.5 s`), nên nó không sửa ở nơi an toàn mà chỉ ở nơi sắp đâm. **Phát theo thời gian thật**: 581 khung trễ + 578 khung rớt trên 59578 khung (giả lập chi phí 6 ms, 1% đột biến 140 ms > hạn 100 ms), làm thay đổi **370/600 đoạn**; còn phát nhanh nhất **0 trễ, 0 rớt** vì nó giả định không có hạn chót — nó che mất lỗi trễ hạn mà hệ thật sẽ có.

## Câu 3 · Kịch bản

Lưới của tôi: **60 khoảng cách × 45 tốc độ tiến gần = 2700 ô**, khoảng cách 2.76–59.31 m (p2–p98 của log), tốc độ tiến gần 0–12 m/s (bằng v_ego), cruise 12 m/s. Vùng **không ai tránh được**: **2.2 % số ô ở t_react = 0** và **7.4 % ở t_react = 0.6 s**; độ trễ 0.6 s làm vùng đâm của IDM tăng từ 2.2 % lên 7.4 % (thêm 5.2 điểm), nhưng diện tích "đâm dù tránh được" của IDM vẫn **0.0 %** ở cả hai độ trễ — IDM nằm đúng trên biên cyan. Kiểm chứng bộ mô phỏng: full-brake đâm 2.2 %/7.4 % khớp physics và **0 ô lệch** khi bỏ 5 % ô sát biên ở cả hai độ trễ. Giới hạn phanh mạnh nhất của người lái trong log là **3.62 m/s²** (p99.9), vậy **8.6 % lưới đòi phanh mạnh hơn mức con người từng làm**; khai thác được 614 ca xe dẫn phanh (6.4 % của 9610 quỹ đạo). Các chấm trắng (ca từ log) tập trung ở tốc độ tiến gần nhỏ và khoảng cách vừa — log **không phủ** vùng cắt vào khó (dv lớn, gap nhỏ), đúng vùng mà mô hình học sẽ hụt.

## Câu 4 · Vòng hở và vòng kín

| Bộ | L2 vòng hở (m) | hạng | va chạm vòng kín | hạng |
|---|---|---|---|---|
| const-vel | 1.247 | 3 | 19.5 % | 3 |
| IDM | 1.525 | 4 | 0.03 % | 1 |
| EgoMLP | 0.755 | 2 | 25.7 % | 4 |
| PercMLP | 0.626 | 1 | 1.8 % | 2 |

Có **4 cặp đảo thứ hạng**: (const-vel, IDM), (const-vel, EgoMLP), (IDM, EgoMLP), (IDM, PercMLP). Cặp đáng chú ý nhất là **IDM vs PercMLP**: PercMLP tốt nhất vòng hở (0.626 m) nhưng IDM lại tốt nhất vòng kín (0.03 % so với 1.8 %). IDM có L2 **tệ nhất** (1.525 m) vì dự báo 3 giây của nó giả định xe dẫn giữ nguyên tốc độ rồi tự phanh/tăng tốc theo luật, không khớp "phong cách" của người lái; nhưng khi tự lái, vòng phản hồi liên tục sửa sai nên nó giữ khoảng cách an toàn. EgoMLP **chỉ nhìn lịch sử tốc độ của chính nó**, không thấy khoảng cách và tốc độ xe dẫn, nên không biết mình đang tiến gần; nó có L2 trung bình tốt (0.755 m) trên đa số cửa sổ dễ nhưng đâm 25.7 %.

## Câu 5 · Một lần chạy chưa đủ

EgoMLP qua 3 seed đâm **22.9 %, 25.5 %, 28.5 %** — dao động **5.6 điểm phần trăm**; seed xấu nhất có CI95 Wilson **[26.9 %, 30.2 %]**, seed tốt nhất **[21.4 %, 24.5 %]**, hai khoảng **không chồng nhau**. PercMLP dao động **0.6–3.4 %** (2.8 điểm), còn PercMLP+ rất chặt **0.17–0.24 %**. Khoảng tin cậy Wilson cho thấy ngay cả 2883 quỹ đạo cũng chỉ ước lượng tỷ lệ trong khoảng ~±1.5 điểm. Nếu chỉ chạy đúng một seed, tôi có thể báo EgoMLP "chỉ 22.9 %" hoặc "tận 28.5 %" và kết luận đổi hẳn; quy tắc R5 (dao động ≤1 điểm) sẽ **qua hay rớt tùy may mắn**. Điều đó nguy hiểm vì một seed may mắn có thể khiến bộ điều khiển nguy hiểm trông như đạt cổng — nên mọi kết luận phải dựa trên **dải qua nhiều seed**, không phải một lần chạy.

## Câu 6 · Tác nhân phản ứng

Tôi hiệu chỉnh xe sau IDM khớp người lái được **T = 1.4 s** (Wasserstein-1 = **0.348**), so với xe sau **không bao giờ phản ứng** W1 = **0.844** và headway trung vị thật của người lái **1.86 s**; T = 1.2 s và 1.4 s gần như ngang nhau (0.351 vs 0.348), nên 1.4 s là lựa chọn tốt. Ở t_react = 0.6 s, tỷ lệ ô lưới có va chạm (trước hoặc sau): IDM **75.7 % → 7.4 %**, PercMLP **72.8 % → 27.6 %**, PercMLP+ **75.4 % → 7.8 %**, còn const-vel **77.6 % → 77.6 %** (không đổi vì nó không phanh nên xe sau không bao giờ bị "đâm giả"). Với xe sau **phát lại**, IDM bị tính **68.3 %** va chạm **phía sau** — đây là **"va chạm giả"**: xe sau giữ nguyên tốc độ log, không biết IDM vừa phanh, nên nó tông vào đuôi; khi cho xe sau **phản ứng**, va chạm sau về **0 %**. Điều này **đảo kết luận**: trong thế giới phát lại IDM trông tệ ngang phần còn lại (75.7 %), nhưng trong thế giới biết phản ứng IDM là an toàn nhất (7.4 %) — "va chạm" đó không phải lỗi của bộ điều khiển.

## Câu 7 · Cổng phát hành và sửa

PercMLP bị chặn bởi **5 quy tắc**, hai nguyên nhân gốc. (a) R4 (20.1 % > 0 + 2) và R6 (27.6 % > 7.4 % + 2): thất bại ở vùng cắt vào mà **log chưa từng chứa** (vùng đòi phanh mạnh hơn người lái). (b) R2 (cận trên 95 % = 4.2 % > 0 + 1) và R3 (8.2 %, cận trên 13.3 % > + 5): **sai số mô phỏng tích lũy** khi mô hình tự lái trong vòng kín; thêm R5 (0.6–3.4 %, dao động 2.8 điểm > 1). Tôi sửa bằng đúng hai nguyên nhân: (1) `augment.cutin_windows` thêm **197470 cửa sổ** từ 20000 ca cắt vào mô phỏng, nhãn là việc IDM làm ở đó; (2) `augment.dagger_windows` — **2 vòng DAgger**, cho mô hình tự lái rồi gắn nhãn bằng dự báo 3 giây của IDM tại **chính các trạng thái nó tự đi vào** (tổng **894816 cửa sổ**). Sau sửa, **PercMLP+ qua cả 7 quy tắc**: R2 cận trên 0.5 %, R3 0.0 %, R4 0.3 %, R5 0.2 %, R6 7.8 % (so baseline 7.4 %). Nhưng PercMLP+ **không tốt hơn IDM**: IDM đâm 0.03 % còn PercMLP+ 0.24 %, diện tích lưới 0.0 % so với 0.3 %; nó học từ nhãn IDM nên **không thể vượt thầy**, bằng chứng chỉ cho thấy **không kém hơn**. Đáng chú ý: L2 vòng hở của PercMLP+ **tệ đi** (0.825 m so với 0.626 m của PercMLP) dù vòng kín tốt hơn nhiều — cho thấy điểm vòng hở không dùng được để ra quyết định. Tôi **không thay IDM ngay**: giữ IDM làm hệ chính, chạy PercMLP+ ở chế độ bóng (shadow), và kiểm tra thêm ở tốc độ cruise xa lạ (20 m/s) — nơi PercMLP+ vẫn đâm 0.6–1.7 % còn IDM 0.0 %.

## Câu 8 · Bài lab này không chứng minh được gì

Ba giới hạn quan trọng nhất với quyết định phát hành. (1) **Một chiều, không cảm biến, trạng thái hoàn hảo**: bài toán chỉ có khoảng cách và tốc độ dọc, mô hình nhận gap/tốc độ chính xác; thế giới thật có nhiễu cảm biến và chiều ngang. Chính thử nghiệm nhiễu (gap ±1 m, tốc độ dẫn ±0.5 m/s) đã đẩy PercMLP lên 0.6–3.3 % — kết luận "an toàn" vì thế mong manh. (2) **Xe dẫn trong log không phản ứng với xe thử**: nó phát lại một quỹ đạo cố định; bước 5 cho thấy kết luận **đảo hẳn** khi xe sau biết phản ứng (IDM 75.7 % → 7.4 %), nên tỷ lệ va chạm đo được **không phải rủi ro thật**. (3) **Vật lý và dữ liệu là giả định của chúng ta**: mô phỏng dùng dt = 0.1 s, phanh tức thời 9 m/s², biên "bất khả thi" do chính ta định nghĩa; và 9610 quỹ đạo Argoverse chỉ là đường ngoại ô, người lái hiếm khi phanh mạnh (p99.9 = 3.62 m/s²) nên mô hình **chưa từng thấy phanh gấp**. Thêm nữa, chỉ 171/2883 quỹ đạo giữ lại có xe dẫn phanh, nên khoảng tin cậy của lát cắt phanh rất rộng. Để thu hẹp khoảng cách tới thế giới thật cần: thử trên sa hình (closed course), hardware-in-the-loop, tác nhân phản ứng thật, bổ sung cảm biến, phủ kịch bản OOD/đối kháng, rồi phát hành theo giai đoạn (shadow → giới hạn) thay vì tin vào một cổng chạy trên dữ liệu log.
