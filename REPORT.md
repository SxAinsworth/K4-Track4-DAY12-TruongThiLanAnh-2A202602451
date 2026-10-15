# REPORT — Lab Day 12

> Điền **bằng lời và số liệu của chính bạn** (lấy từ màn hình và từ `evidence/*.json`). Mỗi câu 4–8 dòng là đủ. Số không có trong bằng chứng của bạn sẽ bị trừ điểm. Báo cáo công khai trên GitHub; giảng viên đối chiếu với `evidence/` và với bài của các bạn khác.

| | |
|---|---|
| Họ và tên | Trương Thị Lan Anh |
| MSSV (đúng bằng `--student-id` bạn đã chạy) | 2A202602451 |
| GitHub | SxAinsworth |
| Seed chia dữ liệu / seed mô hình (xem `evidence/closedloop.json`, mục `student`) | split seed 1151 / model seeds 9163..9167 (cruise speed 12.0 m/s) |
| Số seed huấn luyện bạn dùng (`--seeds`) | 5 |

## Kết quả cuối

Điền từ `evidence/closedloop.json`, `openloop.json`, `gate.json` (cho mỗi mô hình học, lấy **dải qua các seed**).

| Bộ điều khiển | L2 vòng hở (m) | Va chạm vòng kín trên log (%) | Quyết định cổng |
|---|---|---|---|
| constant-velocity | 1.261 | 20.2 (CI95 18.8–21.7) | BLOCK (R2, R3, R4, R6) |
| IDM | 1.488 | 0.14 (CI95 0.05–0.36) | (baseline) |
| EgoMLP | 0.764 .. 0.780 (TB 0.772 ± 0.006) | 12.9 .. 30.2 | BLOCK (R2, R3, R4, R5, R6) |
| PercMLP | 0.620 .. 0.632 (TB 0.625 ± 0.004) | 0.7 .. 3.1 | BLOCK (R2, R3, R4, R5, R6) |
| PercMLP+ (sau khi sửa) | 0.822 ± 0.015 | 0.28 .. 0.35 | RELEASE (qua cả 7 quy tắc) |

## Câu 1 · Dữ liệu
Bước 0 bạn phát hiện những vấn đề gì ở dữ liệu? Vì sao không dùng cột `Acc_LV`, và bạn thay bằng gì? Bao nhiêu quỹ đạo bị loại vì trùng, và vì sao chia theo quỹ đạo?

`data_audit.json` cho thấy ba vấn đề. (1) Cột `Acc_LV` gần như không liên quan đến đạo hàm tốc độ xe dẫn: tương quan chỉ **0,045**, trong khi cột tương ứng của xe theo sau đạt **0,929**. Vì vậy khi khai thác ca phanh, tôi dùng `a_lead` tính từ `Speed_LV` đã làm mượt chứ không dùng `Acc_LV`. (2) Có **383** quỹ đạo trùng hệt nhau nằm ở các id cách xa nhau, nên số quỹ đạo giảm từ 9.993 xuống 9.610. Nếu không loại, một bản sao rơi vào train và bản kia vào test, mô hình được "chấm trên dữ liệu đã thấy" và điểm test bị thổi phồng. (3) Các khung hình trong cùng một quỹ đạo tương quan rất mạnh (10 Hz, 70–109 bước). Chia theo khung hình thì khung t ở train và khung t+1 ở test gần như giống nhau, cũng là rò rỉ. Vì vậy tôi chia theo **quỹ đạo** (split seed 1151): 167.537 cửa sổ train và 71.596 cửa sổ test, không trùng quỹ đạo. Ngoài ra `Spatial_Gap` khớp với hiệu vị trí trừ chiều dài 4,5 m (độ lệch tối đa 3,6e-11 m), nên cột khoảng cách đáng tin.

## Câu 2 · Phát lại
Bản refactor float32 khác bit ở bao nhiêu đoạn, vượt dung sai ở bao nhiêu? Thay đổi AEB làm lệch bao nhiêu đoạn, và vì sao lệch ở đúng những khung đó (mở một đoạn ra xem)? Phát theo thời gian thật làm lộ ra điều gì mà phát nhanh nhất che đi?

Log có 600 đoạn (9,5 MB). Chạy lại cùng mã: 0/600 đoạn khác và 0 đoạn không giống từng bit. Bản refactor float32 có đổi thứ tự phép tính: **600/600** đoạn không giống từng bit (max diff 9,3e-05), nhưng **0/600** đoạn vượt dung sai atol = rtol = 1e-3. Phép cộng số thực không có tính kết hợp ((0,1+0,2)+0,3 = 0,6000000000000001 ≠ 0,6), nên so từng bit là phép thử sai; dung sai 1e-3 m/s² nhỏ hơn rất nhiều so với tác động vật lý của lệnh. Thay đổi AEB làm lệch **10/600** đoạn. Mở đoạn 163, chỉ khung 65–67 bị lệch: gap 3,41 → 3,02 m, tốc độ tiến gần 2,32 → 2,15 m/s, nên TTC = 1,47 → 1,41 s, tức vừa xuống dưới ngưỡng AEB 1,5 s. Lệnh chuyển từ khoảng −5,7 thành −6,0. Ở mọi khung khác TTC > 1,5 s nên luật AEB không kích hoạt. Phát nhanh nhất có 0 khung trễ và 0 khung rớt vì không có hạn chót. Phát theo thời gian thật lộ ra **582 khung trễ + 577 khung rớt / 59.736 khung**, làm **371/600** đoạn đổi lệnh: đó là lỗi thời gian thực mà phát nhanh nhất che mất.

## Câu 3 · Kịch bản
Lưới của bạn (tốc độ cruise, giới hạn khoảng cách, số ô): bao nhiêu % ô **không ai tránh được**? Độ trễ phản ứng 0 → 0,6 s làm vùng đâm của IDM tăng bao nhiêu? Log có phủ vùng khó không? (dẫn giới hạn phanh mạnh nhất của người lái trong log, và phần lưới đòi hỏi hơn thế)

Lưới cắt vào của tôi có tốc độ cruise 12,0 m/s, khoảng cách lấy từ p2..p98 của log, **60 × 45 = 2.700 ô**, tốc độ tiến gần từ 0 đến 12 m/s. Biên bất khả kháng (phanh 9 m/s² sau thời gian trễ) chiếm **2,2 %** lưới khi t_react = 0 và **7,4 %** khi t_react = 0,6 s. Bộ mô phỏng khớp hàm `unavoidable` của tôi với **0 ô lệch** ở cả hai độ trễ. Vùng đâm của IDM tăng từ **2,2 % lên 7,4 %**, đúng bằng vùng bất khả kháng (attributable area 0,0 %). Nghĩa là mọi va chạm của IDM đều thuộc loại "ai cũng đâm", không phải lỗi bộ điều khiển. Tôi khai thác được **614** ca xe dẫn phanh (6,4 % số quỹ đạo; đỉnh giảm tốc p50 −2,5, p99 −4,2 m/s²). Nhưng người lái trong log phanh mạnh nhất chỉ **3,6 m/s²** (p99.9), trong khi bộ chấp hành làm được 9 m/s², và **9 %** lưới đòi phanh mạnh hơn mức đó. Log không phủ vùng khó này, nên mô hình học từ log chưa từng thấy nhãn nào ở đó.

## Câu 4 · Vòng hở và vòng kín
Dán bảng L2 và tỷ lệ va chạm của bốn bộ điều khiển (kèm hạng ở mỗi cách chấm). Cặp nào đổi thứ tự? Vì sao IDM có L2 như vậy mà vẫn lái tốt? Vì sao EgoMLP có L2 tốt hơn constant-velocity mà vẫn đâm?


| Bộ điều khiển | L2 vòng hở (m) | Hạng | Va chạm vòng kín | Hạng |
|---|---|---|---|---|
| const-vel | 1.261 | 3 | 20.2 % | 3 |
| IDM | 1.488 | 4 | 0.1 % | 1 |
| EgoMLP | 0.772 | 2 | 20.7 % (TB 5 seed) | 4 |
| PercMLP | 0.625 | 1 | 1.6 % (TB 5 seed) | 2 |

Bốn cặp đổi thứ tự: (const-vel, IDM), (const-vel, EgoMLP), (IDM, EgoMLP), (IDM, PercMLP). IDM có L2 tệ nhất (1,488 m) vì nó không bắt chước người lái. Nó giữ khoảng cách theo luật riêng (T = 1,2 s) và phản ứng sớm hơn người, nên quỹ đạo dự báo lệch khỏi log. Nhưng trong vòng kín nó luôn nhìn khoảng cách và tốc độ xe dẫn rồi phanh đủ, nên chỉ đâm 0,14 %. EgoMLP chỉ thấy lịch sử tốc độ của chính nó, nên học được "người thường giữ tốc độ khá đều". Điều đó cho L2 tốt (0,772 < 1,261 của const-vel) vì phần lớn cửa sổ là chạy đều. Nhưng nó **không có thông tin xe dẫn** (gap, v_lead), nên khi xe dẫn phanh nó không biết để phanh: va chạm 62,6 % trên lát phanh. Mặt khác, L2 trên lát phanh (4,6 % số cửa sổ) luôn lớn hơn L2 trung bình (EgoMLP 1,109 so với 0,772), nên trung bình che lỗi.

## Câu 5 · Một lần chạy chưa đủ
EgoMLP dao động bao nhiêu giữa các seed? Khoảng tin cậy Wilson nói gì? Nếu bạn chỉ chạy đúng một seed thì có thể báo kết quả nào, và vì sao điều đó nguy hiểm?

Tỷ lệ va chạm của EgoMLP dao động từ **12,9 % (seed 9165) đến 30,2 % (seed 9163)** qua 5 seed, tức hơn gấp đôi, trong khi L2 vòng hở gần như không đổi (0,764..0,780). Khoảng Wilson 95 % của seed tốt nhất là [11,7 %; 14,2 %] và của seed xấu nhất là [28,6 %; 31,9 %]: **không chồng nhau**. Vậy độ phân tán này là do seed huấn luyện, không phải nhiễu thống kê của tập 2.883 quỹ đạo. Wilson chỉ đo bất định do số mẫu hữu hạn **trong một mô hình**; nó không nói gì về phương sai giữa các lần huấn luyện. Nếu chỉ chạy một seed, tôi có thể báo "12,9 %" hoặc "30,2 %" tùy may rủi, và cả hai đều có khoảng tin cậy hẹp, trông rất chắc chắn. PercMLP cũng vậy: 0,7..3,1 %. Seed tốt nhất (0,73 %, cận trên 1,1 %) gần như qua được R2, trong khi seed xấu nhất (cận trên 3,8 %) thì rớt. Báo một seed có thể dẫn tới phát hành nhầm, nên cổng dùng **seed xấu nhất** (R2) và đòi dải ≤ 1 điểm (R5).

## Câu 6 · Tác nhân phản ứng
Dán bảng "xe sau phát lại" và "xe sau phản ứng". `T` bạn hiệu chỉnh là bao nhiêu, xe sau chân thực đến mức nào so với xe không phản ứng? Vì sao IDM bị tính "va chạm giả" khi xe sau chỉ phát lại, và điều này làm đổi kết luận về bộ điều khiển ra sao?


t_react = 0,6 s, tỷ lệ lưới có va chạm bất kỳ (trước hoặc sau):

| Bộ điều khiển | Xe sau phát lại | Xe sau phản ứng | Va chạm sau: phát lại / phản ứng |
|---|---|---|---|
| const-vel | 77.6 % | 77.6 % | 0.0 % / 0.0 % |
| IDM | 75.7 % | 7.4 % | 68.3 % / 0.0 % |
| EgoMLP | 77.8 % | 77.8 % | 0.0 % / 0.0 % |
| PercMLP | 77.9 % | 25.6 % | 53.7 % / 0.0 % |

(Ở t_react = 0: IDM 75,3 % → 2,2 %, va chạm sau 73,0 % → 0,0 %.)

Người lái trong log giữ thời khoảng cách trung vị 1,87 s. Tôi quét T và chọn **T = 1,4 s**, vì phân bố khoảng cách của nó gần người nhất (Wasserstein W1 = **0,34**). Một xe sau không bao giờ phản ứng có W1 = **0,84**, tức kém chân thực hơn rất nhiều. Khi xe sau chỉ phát lại, nó không biết xe thử vừa phanh và cứ lao tới. Vì vậy IDM, bộ điều khiển **phanh đúng** trước xe cắt vào, bị tính tới **68,3 %** lưới là "đâm từ phía sau". Đó là va chạm giả do thế giới mô phỏng, không phải lỗi bộ điều khiển. Nghịch lý là const-vel không bao giờ phanh nên không bị đâm từ sau, và trong thế giới phát lại trông nó ngang IDM (77,6 % so với 75,7 %). Với xe sau phản ứng, va chạm sau về 0 % cho mọi bộ, và IDM chỉ còn 7,4 %, đúng bằng vùng bất khả kháng. Kết luận đảo hẳn: IDM vượt trội const-vel.

## Câu 7 · Cổng phát hành và sửa
Quy tắc nào chặn PercMLP và nguyên nhân gốc là gì (dẫn số)? Bạn sửa bằng gì và vì sao nó chữa đúng nguyên nhân? Sau khi qua cổng, PercMLP+ có **tốt hơn** IDM không? Nó có nên được phát hành không, và bạn đề xuất làm gì trước khi quyết định?

PercMLP bị chặn bởi 5 quy tắc. R2: seed xấu nhất 3,09 % (cận trên 3,78 %) > 0,14 % + 1 điểm. R3: 14,3 % trên lát phanh (cận trên 20,1 %) > 1,6 % + 5. R4: 18,2 % lưới "tránh được mà vẫn đâm" > 0 % + 2. R5: dải 0,7..3,1 %. R6: 25,6 % so với 7,4 %. Có hai nguyên nhân gốc. (a) **Kịch bản log không chứa**: R4 và R6 rớt trên lưới cắt vào, đúng phần mà bước 2 cho thấy log không phủ (người phanh tối đa 3,6 m/s², 9 % lưới đòi hơn thế). Ở tốc độ cruise 20 m/s chưa từng thấy khi huấn luyện, PercMLP đâm 23,1..32,8 % lưới. (b) **Sai số tích luỹ khi tự lái** (covariate shift): L2 vòng hở rất tốt (0,625 m), nhưng vòng kín trên chính phân phối log vẫn đâm 0,7..3,1 % và dao động theo seed (R2, R3, R5). Mô hình trôi vào các trạng thái người chưa từng đi qua và không biết cách hồi phục. Tôi sửa bằng hai cách tương ứng: thêm **197.180 cửa sổ từ 20.000 ca cắt vào mô phỏng**, gắn nhãn bằng IDM, để chữa (a). Thêm **2 vòng DAgger**, gắn nhãn IDM ngay tại trạng thái mô hình tự đi vào, để chữa (b). Tổng cộng khoảng 895.700 cửa sổ mỗi seed. Kết quả của PercMLP+: va chạm trên log **0,28..0,35 %** (cận trên 0,64 %), lát phanh 2,2 %, lưới 0,6 %, dải seed 0,3..0,3 %, thế giới phản ứng 8,0 % so với 7,4 %. Nó qua cả **7/7** quy tắc. L2 vòng hở lại **tệ đi** (0,625 → 0,822 m), vì mô hình giờ bắt chước IDM chứ không bắt chước người: lại thêm một bằng chứng rằng L2 không đo an toàn.

PercMLP+ **không tốt hơn** IDM, nó chỉ **không kém hơn** trong biên cho phép. Nó vẫn đâm nhiều hơn IDM trên log (0,35 % so với 0,14 %), trên lát phanh (2,2 % so với 1,6 %), trên lưới (0,6 % so với 0 % lưới tránh được) và trong thế giới phản ứng (8,0 % so với 7,4 %). Ở 20 m/s, PercMLP+ vẫn đâm tới 2,9 % lưới, trong khi IDM 0 %. Nó học từ IDM nên khó vượt thầy. Đề xuất của tôi: **chưa thay IDM**. Trước hết nên chạy shadow mode (chạy song song, không điều khiển) trên dữ liệu mới, mở rộng lưới sang tốc độ ngoài phân phối và cảm biến có nhiễu hoặc trễ, rồi xác định rõ PercMLP+ mang lại lợi ích gì mà IDM không có (ví dụ độ êm, giống người). Nếu không có lợi ích như vậy thì không có lý do để chấp nhận thêm rủi ro.

## Câu 8 · Bài lab này không chứng minh được gì
Chọn **ít nhất ba** giới hạn quan trọng nhất đối với việc quyết định phát hành và giải thích vì sao mỗi cái làm kết luận của bạn kém chắc chắn. Điều gì cần làm tiếp để thu hẹp khoảng cách đến thế giới thật?

1. **Xe dẫn không phản ứng và mô phỏng chỉ một chiều dọc.** Xe dẫn phát lại từ log nên không đổi hành vi theo xe thử; không có chuyển làn, người đi bộ, giao lộ. Va chạm 0,35 % của PercMLP+ chỉ đúng trong thế giới thu hẹp này, nên không suy ra được rủi ro trên đường thật.
2. **Không có cảm biến thật.** Gap và v_lead là giá trị lấy thẳng từ log. Phép thử nhiễu chỉ là nhiễu Gauss nhỏ (±1 m, ±0,5 m/s); không có mất dấu, nhận nhầm đối tượng hay trễ cảm biến thay đổi. Mô hình học có thể nhạy hơn IDM với các lỗi này, nhưng cổng không đo được.
3. **Thế giới mô phỏng là giả định của chúng ta.** Xe sau phản ứng là IDM với T = 1,4 s (W1 0,34, vẫn khác người). Lưới cắt vào giả định xe cắt vào giữ tốc độ không đổi, và giới hạn phanh 9 m/s² là giả định. R4, R6 và DAgger đều dùng cùng mô hình thế giới này, nên PercMLP+ có thể chỉ "học thuộc" giả định của bộ mô phỏng.
4. **Dữ liệu và độ phủ hạn chế.** Chỉ Argoverse 2, tốc độ p99 khoảng 15,6 m/s, 2.883 quỹ đạo test và 182 ca phanh. Với n = 182, Wilson của lát phanh rộng khoảng ±3 điểm, không đủ để chứng minh tỷ lệ sự cố hiếm (1e-6/giờ) như yêu cầu phát hành thật.

Việc cần làm tiếp: kiểm thử với tác nhân phản ứng đa dạng (không chỉ IDM) và kịch bản 2D; đưa mô hình cảm biến thật hoặc phát lại dữ liệu cảm biến thô vào vòng kín; thu thêm dữ liệu ở tốc độ cao và ca phanh gấp; chạy shadow mode trên xe thật; kiểm chứng bộ mô phỏng với log va chạm hoặc suýt va chạm thật trước khi tin cổng phát hành.
