# REPORT — Lab Day 12

> Điền **bằng lời và số liệu của chính bạn** (lấy từ màn hình và từ `evidence/*.json`). Mỗi câu 4–8 dòng là đủ. Số không có trong bằng chứng của bạn sẽ bị trừ điểm. Báo cáo công khai trên GitHub; giảng viên đối chiếu với `evidence/` và với bài của các bạn khác.

| | |
|---|---|
| Họ và tên | Nguyễn Đức Đông |
| MSSV (đúng bằng `--student-id` bạn đã chạy) | 2A202602367 |
| GitHub | nguyenducdong22 |
| Seed chia dữ liệu / seed mô hình (xem `evidence/closedloop.json`, mục `student`) | split 9974 / model 1284 (seeds 1284–1288) |
| Số seed huấn luyện bạn dùng (`--seeds`) | 5 |

## Kết quả cuối

Điền từ `evidence/closedloop.json`, `openloop.json`, `gate.json` (cho mỗi mô hình học, lấy **dải qua các seed**).

| Bộ điều khiển | L2 vòng hở (m) | Va chạm vòng kín trên log (%) | Quyết định cổng |
|---|---|---|---|
| constant-velocity | 1.262 | 21.1 | BLOCK (R2, R3, R4, R6) |
| IDM | 1.509 | 0.03 (1/2883) | (baseline) |
| EgoMLP | 0.759 ±0.002 (0.756–0.762) | 19.8 – 31.3 | BLOCK (R2–R6) |
| PercMLP | 0.622 ±0.003 (0.618–0.625) | 0.3 – 6.5 | BLOCK (R2–R6) |
| PercMLP+ (sau khi sửa) | 0.806 ±0.013 | 0.17 – 0.28 | RELEASE (qua cả 7 quy tắc) |

## Câu 1 · Dữ liệu
Bước 0 bạn phát hiện những vấn đề gì ở dữ liệu? Vì sao không dùng cột `Acc_LV`, và bạn thay bằng gì? Bao nhiêu quỹ đạo bị loại vì trùng, và vì sao chia theo quỹ đạo?

Ba vấn đề. (1) Cột `Acc_LV` gần như không liên quan tới đạo hàm của `Speed_LV`: tương quan chỉ 0.045, trong khi cột tương ứng của xe sau là 0.929. Vì vậy tôi không dùng `Acc_LV` mà lấy gia tốc xe dẫn từ đạo hàm của `Speed_LV` đã làm mượt (`a_lead`) khi khai thác ca phanh. (2) Có 383 quỹ đạo trùng hệt nhau (9993 → 9610 quỹ đạo sạch), nằm ở các id cách xa nhau; nếu không loại, cùng một quỹ đạo có thể rơi vào cả train và test và làm điểm test đẹp giả. (3) Phải chia theo **quỹ đạo** vì các khung liền nhau trong một quỹ đạo gần như giống nhau (10 Hz); chia theo khung sẽ để mô hình "thấy trước" gần như đúng tình huống test. Ngoài ra `gap = Pos_LV − Pos_FAV − 4.5` khớp tới 3.6e-11 m nên cột khoảng cách dùng được.

## Câu 2 · Phát lại
Bản refactor float32 khác bit ở bao nhiêu đoạn, vượt dung sai ở bao nhiêu? Thay đổi AEB làm lệch bao nhiêu đoạn, và vì sao lệch ở đúng những khung đó (mở một đoạn ra xem)? Phát theo thời gian thật làm lộ ra điều gì mà phát nhanh nhất che đi?

Bản refactor float32 khác bit ở cả 600/600 đoạn nhưng 0/600 đoạn vượt dung sai (atol = rtol = 1e-3, max diff 9.3e-05 m/s²) — phép cộng số thực không kết hợp ((0.1+0.2)+0.3 = 0.6000000000000001), nên so từng bit là phép thử sai; dung sai cỡ 1e-3 m/s² nhỏ hơn nhiều so với độ chính xác cơ cấu chấp hành. Thay đổi AEB làm lệch 11/600 đoạn (ví dụ 51, 163, 225, 299, 320), max diff 1.73 m/s². Mở một đoạn: chỉ các khung có xe đang tiến gần và TTC = gap/closing < 1.5 s mới bị lệch, vì đó là điều kiện duy nhất để luật AEB ghi đè lệnh IDM bằng −6 m/s²; ở mọi khung khác hai bản cho cùng lệnh. Phát nhanh nhất: 0 khung trễ, 0 khung rớt vì không có hạn chót. Phát theo thời gian thật: 582 khung trễ + 574 khung rớt trên 59740 khung, làm 381/600 đoạn đổi lệnh — lỗi trễ hạn chỉ lộ ra khi tôn trọng thời gian thật.

## Câu 3 · Kịch bản
Lưới của bạn (tốc độ cruise, giới hạn khoảng cách, số ô): bao nhiêu % ô **không ai tránh được**? Độ trễ phản ứng 0 → 0,6 s làm vùng đâm của IDM tăng bao nhiêu? Log có phủ vùng khó không? (dẫn giới hạn phanh mạnh nhất của người lái trong log, và phần lưới đòi hỏi hơn thế)

Lưới của tôi: cruise 12 m/s, khoảng cách 2.76–59.3 m (p2–p98 của log), tốc độ tiến gần 0–12 m/s, 60 × 45 = 2700 ô. Với độ trễ 0 s có 2.2 % ô không ai tránh được; với 0.6 s là 7.4 %. Kiểm chứng bộ mô phỏng bằng hàm `unavoidable`: 0 ô lệch xa biên ở cả hai độ trễ. Vùng đâm của IDM tăng từ 2.2 % lên 7.4 % — trùng đúng vùng bất khả thi, nên IDM có 0.0 % va chạm "do bộ điều khiển" (attributable). Nhờ biên này ta tách được "ai cũng đâm" (vật lý) khỏi "bộ điều khiển đâm" (lỗi). Log không phủ vùng khó: người lái phanh mạnh nhất (p99.9) chỉ 3.6 m/s² trong khi cơ cấu làm được 9, và 8.6 % (≈9 %) lưới đòi phanh mạnh hơn thế; 614 ca phanh khai thác được (6.4 % quỹ đạo, đỉnh p99 −4.2 m/s²) đều nằm ở vùng dễ.

## Câu 4 · Vòng hở và vòng kín
Dán bảng L2 và tỷ lệ va chạm của bốn bộ điều khiển (kèm hạng ở mỗi cách chấm). Cặp nào đổi thứ tự? Vì sao IDM có L2 như vậy mà vẫn lái tốt? Vì sao EgoMLP có L2 tốt hơn constant-velocity mà vẫn đâm?

| Bộ điều khiển | L2 vòng hở (m) | Hạng L2 | Va chạm vòng kín | Hạng va chạm |
|---|---|---|---|---|
| const-vel | 1.262 | 3 | 21.1 % | 3 |
| IDM | 1.509 | 4 | 0.0 % (1/2883) | 1 |
| EgoMLP | 0.759 | 2 | 24.2 % (TB 5 seed) | 4 |
| PercMLP | 0.622 | 1 | 1.9 % (TB 5 seed) | 2 |

Các cặp đổi thứ tự: (const-vel, IDM), (const-vel, EgoMLP), (IDM, EgoMLP), (IDM, PercMLP). IDM có L2 tệ nhất (1.509 m) vì nó không bắt chước người: nó giữ khoảng cách theo luật T = 1.2 s của riêng mình, nên vị trí dự báo lệch khỏi người lái — nhưng trong vòng kín nó luôn phản ứng với gap và tốc độ xe dẫn, nên gần như không đâm. EgoMLP có L2 0.759 m tốt hơn const-vel vì học được quán tính tốc độ của người, nhưng nó chỉ nhìn tốc độ của chính nó, không thấy gap hay tốc độ xe dẫn: khi tự lái, nó không biết xe trước đang phanh nên đâm (63.0 % trên lát xe dẫn phanh, worst seed). L2 thấp chỉ nghĩa là "giống log trung bình", không nghĩa là "phản ứng đúng".

## Câu 5 · Một lần chạy chưa đủ
EgoMLP dao động bao nhiêu giữa các seed? Khoảng tin cậy Wilson nói gì? Nếu bạn chỉ chạy đúng một seed thì có thể báo kết quả nào, và vì sao điều đó nguy hiểm?

EgoMLP dao động 19.8 % – 31.3 % va chạm qua 5 seed (1287 tốt nhất, 1286 xấu nhất). Khoảng Wilson 95 % của seed tốt nhất là [18.4, 21.3] %, của seed xấu nhất [29.6, 33.0] %: hai khoảng không chồng nhau, tức sai khác giữa seed là thật, không phải nhiễu lấy mẫu. PercMLP còn tệ hơn về độ ổn định: 0.3 % (seed 1285) đến 6.5 % (seed 1287). Nếu chỉ chạy một seed, tôi có thể báo "PercMLP chỉ đâm 0.3 %" (và CI hẹp [0.16, 0.59] % trông rất chắc chắn) — trong khi một seed khác đâm 6.5 %. Khoảng Wilson chỉ đo nhiễu của tập test, không đo phương sai huấn luyện; vì vậy phải báo dải qua seed (và dùng seed tệ nhất cho cổng, như R2/R5).

## Câu 6 · Tác nhân phản ứng
Dán bảng "xe sau phát lại" và "xe sau phản ứng". `T` bạn hiệu chỉnh là bao nhiêu, xe sau chân thực đến mức nào so với xe không phản ứng? Vì sao IDM bị tính "va chạm giả" khi xe sau chỉ phát lại, và điều này làm đổi kết luận về bộ điều khiển ra sao?

Xe sau được hiệu chỉnh theo IDM với T = 1.4 s (W1 = 0.35 so với phân bố headway người, median 1.86 s), gần thực tế hơn nhiều so với xe sau không bao giờ phản ứng (W1 = 0.87). Bảng ở t_react = 0.6 s (tỷ lệ lưới có va chạm bất kỳ):

| Bộ điều khiển | Xe sau phát lại | Xe sau phản ứng | Va chạm phía sau (replay / react) |
|---|---|---|---|
| const-vel | 77.6 % | 77.6 % | 0.0 % / 0.0 % |
| IDM | 75.7 % | 7.4 % | 68.3 % / 0.0 % |
| EgoMLP | 77.6 % | 77.6 % | 0.0 % / 0.0 % |
| PercMLP | 75.6 % | 29.4 % | 47.5 % / 0.0 % |

Khi xe sau chỉ phát lại, nó không biết xe thử vừa phanh và cứ lao tới, nên IDM — bộ phanh đúng — bị tính 68.3 % va chạm phía sau: đó là "va chạm giả", lỗi của thế giới mô phỏng chứ không của IDM. Trong thế giới phát lại IDM (75.7 %) trông gần như tệ bằng const-vel (77.6 %), tức là phạt chính hành vi đúng; với xe sau phản ứng, IDM chỉ còn 7.4 % (= vùng bất khả thi) còn const-vel vẫn 77.6 %. Kết luận về bộ điều khiển đảo ngược hoàn toàn.

## Câu 7 · Cổng phát hành và sửa
Quy tắc nào chặn PercMLP và nguyên nhân gốc là gì (dẫn số)? Bạn sửa bằng gì và vì sao nó chữa đúng nguyên nhân? Sau khi qua cổng, PercMLP+ có **tốt hơn** IDM không? Nó có nên được phát hành không, và bạn đề xuất làm gì trước khi quyết định?

Cổng chặn PercMLP ở R2 (seed tệ nhất 6.45 %, cận trên 7.41 % > 1.03 %), R3 (19.8 % trên lát phanh), R4 (21.9 % lưới cắt vào tránh được mà vẫn đâm, so với 0.0 % + 2), R5 (0.3–6.5 % qua seed) và R6 (29.4 % so với 7.4 %). Hai nguyên nhân gốc: (a) **kịch bản log không chứa** — cắt vào gần với tốc độ tiến gần lớn, đòi phanh > 3.6 m/s² (R4, R6); (b) **sai số tích luỹ** khi tự lái — mô hình học trên trạng thái của người, khi tự lái lệch dần vào trạng thái chưa thấy (R2, R3, R5, seed dao động). Sửa bằng hai cách: thêm 196950 cửa sổ từ 20000 cảnh cắt vào mô phỏng do IDM lái (chữa (a)), và DAgger 2 vòng — ghi trạng thái chính mô hình đi vào và gắn nhãn bằng IDM (chữa (b)). PercMLP+ qua cả 7 quy tắc: 0.17–0.28 % trên log (cận trên 0.55 %), 0.0 % lát phanh, 0.1 % lưới, 7.6 % thế giới phản ứng. Nhưng nó **không tốt hơn** IDM: IDM 0.03 % trên log, 0.0 % lưới, 7.4 % thế giới phản ứng — PercMLP+ chỉ "không kém hơn trong biên". Nó học từ IDM nên không thể vượt thầy trên các phép đo này; L2 vòng hở còn tăng 0.622 → 0.806 m (điểm vòng hở xấu đi khi hành vi tốt lên). Tôi chưa thay IDM: cần chạy thêm ở tốc độ ngoài phân phối (ở 20 m/s PercMLP+ còn 0.0–1.0 % lưới vs IDM 0.0 %), kiểm thử 2D/cảm biến thật, shadow mode, và lý do để thay (lợi ích ngoài an toàn, ví dụ độ êm).

## Câu 8 · Bài lab này không chứng minh được gì
Chọn **ít nhất ba** giới hạn quan trọng nhất đối với việc quyết định phát hành và giải thích vì sao mỗi cái làm kết luận của bạn kém chắc chắn. Điều gì cần làm tiếp để thu hẹp khoảng cách đến thế giới thật?

(1) **Một chiều dọc, không cảm biến**: mô hình nhận gap và tốc độ xe dẫn hoàn hảo (chỉ thêm nhiễu ±1 m / ±0.5 m/s). Thật ra nhận thức có trễ, mất dấu, nhận nhầm; không có đổi làn, giao lộ. Kết luận "PercMLP+ an toàn" chỉ đúng trong không gian quá hẹp này. (2) **Xe dẫn trong log không phản ứng với xe thử** và xe sau chỉ là IDM T = 1.4 s: ta đã thấy giả định về tác nhân đảo ngược kết luận (IDM 75.7 % → 7.4 %), nên mọi con số còn phụ thuộc mô hình tác nhân của ta. (3) **Thế giới mô phỏng là giả định của chúng ta**, và PercMLP+ được huấn luyện trên chính các cảnh cắt vào do cùng bộ mô phỏng và cùng IDM tạo ra rồi được chấm trên lưới cắt vào giống thế — có rủi ro "học đề thi"; khoảng cách sim-to-real chưa đo. (4) **Dữ liệu**: chỉ 9610 quỹ đạo Argoverse 2, 7–11 s, phanh người tối đa 3.6 m/s²; sự kiện hiếm (tỷ lệ va chạm cỡ 1e-4) không thể ước lượng với 2883 quỹ đạo test (CI của IDM đã là [0.006, 0.20] %). Cần làm tiếp: mô phỏng 2D có cảm biến và tác nhân đa dạng, dữ liệu từ nhiều thành phố/tốc độ, ước lượng sự kiện hiếm (importance sampling), chạy shadow mode trên xe thật trước khi phát hành.
