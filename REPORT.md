# REPORT — Lab Day 12

> Điền **bằng lời và số liệu của chính bạn** (lấy từ màn hình và từ `evidence/*.json`). Mỗi câu 4–8 dòng là đủ. Số không có trong bằng chứng của bạn sẽ bị trừ điểm. Báo cáo công khai trên GitHub; giảng viên đối chiếu với `evidence/` và với bài của các bạn khác.

| | |
|---|---|
| Họ và tên | Đặng Thái Anh |
| MSSV (đúng bằng `--student-id` bạn đã chạy) | 2A202602740 |
| GitHub | thaianh20021 |
| Seed chia dữ liệu / seed mô hình (xem `evidence/closedloop.json`, mục `student`) | 2896 / [8062, 8063, 8064, 8065, 8066] |
| Số seed huấn luyện bạn dùng (`--seeds`) | 5 |

## Kết quả cuối

Điền từ `evidence/closedloop.json`, `openloop.json`, `gate.json` (cho mỗi mô hình học, lấy **dải qua các seed**).

| Bộ điều khiển | L2 vòng hở (m) | Va chạm vòng kín trên log (%) | Quyết định cổng |
|---|---|---|---|
| constant-velocity | 1.253 | 19.9% | BLOCK |
| IDM | 1.523 | 0.1% | (baseline) |
| EgoMLP | 0.758 .. 0.761 (mean 0.759) | 19.5% .. 29.9% (mean 23.8%) | BLOCK |
| PercMLP | 0.617 .. 0.620 (mean 0.619) | 0.4% .. 2.0% (mean 1.1%) | BLOCK |
| PercMLP+ (sau khi sửa) | 0.809 .. 0.849 (mean 0.829) | 0.3% .. 0.5% (mean 0.4%) | RELEASE |

## Câu 1 · Dữ liệu
Bước 0 bạn phát hiện những vấn đề gì ở dữ liệu? Vì sao không dùng cột `Acc_LV`, và bạn thay bằng gì? Bao nhiêu quỹ đạo bị loại vì trùng, và vì sao chia theo quỹ đạo?

Ở bước 0, quá trình kiểm toán dữ liệu từ file `evidence/data_audit.json` làm lộ ra ba vấn đề nghiêm trọng:
1. Cột gia tốc xe dẫn `Acc_LV` có hệ số tương quan với đạo hàm vận tốc d(Speed_LV)/dt xấp xỉ bằng 0 (`acc_lead_column_corr_with_dv_dt` = 0.0454), trong khi cột của xe theo sau đạt tương quan rất cao 0.9291. Do đó, `Acc_LV` không đáng tin cậy để sử dụng trực tiếp; ta phải thay thế bằng gia tốc xe dẫn được tính từ đạo hàm chuỗi vận tốc đã làm mượt `Speed_LV` (`v_lead_s`).
2. Tập dữ liệu gốc gồm 9.993 quỹ đạo với 992.679 dòng nhưng phát hiện tới 383 quỹ đạo bị trùng lặp hoàn toàn (`exact_duplicates_removed` = 383) tại các ID cách xa nhau, chỉ còn lại 9.610 quỹ đạo sạch (`trajectories_clean` = 9610).
3. Ta bắt buộc phải chia dữ liệu theo quỹ đạo (trajectory-level split: 6.727 train / 2.883 test) thay vì chia theo từng khung hình (frame-level). Lý do là các khung hình liên tiếp trong mỗi đoạn 7–11 giây (70–109 bước ở tần số 10 Hz, dt = 0.1 s) có tính tương quan chuỗi thời gian cực lớn; chia theo khung hình sẽ làm rò rỉ dữ liệu (data leakage) giữa tập train và test, khiến việc đánh giá không còn tính khái quát hóa độc lập.

## Câu 2 · Phát lại
Bản refactor float32 khác bit ở bao nhiêu đoạn, vượt dung sai ở bao nhiêu? Thay đổi AEB làm lệch bao nhiêu đoạn, và vì sao lệch ở đúng những khung đó (mở một đoạn ra xem)? Phát theo thời gian thật làm lộ ra điều gì mà phát nhanh nhất che đi?

Từ `evidence/replay.json`, việc kiểm thử phát lại cho thấy các hiện tượng kỹ thuật then chốt:
1. Bản refactor float32 khác bit ở toàn bộ 600/600 đoạn (`not_bit_identical` = 600), nhưng có 0/600 đoạn vượt dung sai (`differ_beyond_tol` = 0) với độ lệch tuyệt đối tối đa chỉ 0.000403 m/s² (nằm gọn trong ngưỡng `atol = 1e-3, rtol = 1e-3`). Phép cộng số thực 32-bit không có tính kết hợp (ví dụ minh họa: `(0.1 + 0.2) + 0.3 = 0.6000000000000001` khác `0.1 + (0.2 + 0.3) = 0.6`, sai số tương đối 6.94e-06), do đó kiểm tra bằng nhau tuyệt đối từng bit là một sai lầm, cần phải dùng kiểm thử có dung sai kỹ thuật.
2. Thay đổi luật AEB (`aeb_ttc = 1.5 s`) làm lệch đúng 14/600 đoạn (chiếm 2.3%). Mở chi tiết đoạn 65, sự sai khác chỉ xảy ra ở 3 khung hình nguy hiểm (khung 75, 76, 77) khi khoảng cách gap co lại còn 4.998 m, 4.687 m, 4.397 m với tốc độ tiến gần 3.388 m/s, 3.205 m/s, 3.062 m/s, làm TTC giảm xuống 1.475 s, 1.462 s, 1.436 s (< 1.5 s). Tại đây IDM thường chỉ ra lệnh phanh khoảng -4.38 m/s² đến -4.35 m/s², trong khi AEB can thiệp cưỡng bức phanh khẩn cấp hết cỡ -6.0 m/s².
3. Chế độ phát nhanh nhất (`fastest`) giả định máy tính xử lý tức thì, che giấu hoàn toàn các vấn đề trễ hạn và rớt gói (0 late, 0 dropped). Ngược lại, phát theo thời gian thực (`realtime`) mô phỏng độ trễ xử lý thực tế đã làm lộ ra 579 khung hình bị trễ (late) và 574 khung hình bị rớt (dropped) trên tổng số 59.195 khung hình, khiến 369/600 đoạn bị thay đổi lệnh điều khiển thực tế.

## Câu 3 · Kịch bản
Lưới của bạn (tốc độ cruise, giới hạn khoảng cách, số ô): bao nhiêu % ô **không ai tránh được**? Độ trễ phản ứng 0 → 0,6 s làm vùng đâm của IDM tăng bao nhiêu? Log có phủ vùng khó không? (dẫn giới hạn phanh mạnh nhất của người lái trong log, và phần lưới đòi hỏi hơn thế)

Từ kết quả `evidence/scenarios.json`:
1. Lưới kịch bản có kích thước 60 mức khoảng cách gap (2.76 m – 59.31 m) và 45 mức tốc độ tiến gần closing speed (0.0 m/s – 14.0 m/s với vận tốc cruise 14.0 m/s), tổng cộng 2.700 ô kịch bản. Ở độ trễ phản ứng 0.0 s, có 3.6% số ô (chính xác 3.63%) là bất khả thi về mặt vật lý (unavoidable - ngay cả khi phanh tối đa 9.0 m/s² vẫn đâm vì gap < dv² / (2 * b_max)).
2. Khi độ trễ phản ứng tăng từ 0.0 s lên 0.6 s, vùng va chạm của IDM tăng vọt từ 3.6% lên 10.2% (tăng thêm 6.6 điểm phần trăm, tức tăng gần gấp 3 lần), do xe bị mất thêm một quãng đường dv * t_react trước khi cơ cấu phanh bắt đầu kích hoạt. Đáng chú ý, vùng đâm của IDM khớp hoàn hảo 100% với biên bất khả thi vật lý (0 ô sai lệch ở cả hai độ trễ).
3. Log lái thật hoàn toàn không phủ vùng khó: giới hạn phanh mạnh nhất của con người trong log ở phân vị 99.9% chỉ là 3.62 m/s² (`human_brake_limit_p99.9` = 3.6 m/s²), trong khi cơ cấu phanh có thể phanh tới 9.0 m/s². Có tới 12.7% (khoảng 13%) số ô trên lưới đòi hỏi mức giảm tốc vượt quá 3.62 m/s². Toàn bộ 614 sự kiện xe dẫn phanh khai thác từ log (chiếm 6.39% tổng số quỹ đạo, với độ giảm tốc p50 là -2.53 m/s² và p99 là -4.21 m/s²) đều chỉ tập trung ở vùng khoảng cách an toàn và closing speed thấp, để lại một khoảng trống kịch bản nguy hiểm mà mô hình học thuần từ log chưa từng được quan sát.

## Câu 4 · Vòng hở và vòng kín
Dán bảng L2 và tỷ lệ va chạm của bốn bộ điều khiển (kèm hạng ở mỗi cách chấm). Cặp nào đổi thứ tự? Vì sao IDM có L2 như vậy mà vẫn lái tốt? Vì sao EgoMLP có L2 tốt hơn constant-velocity mà vẫn đâm?

Bảng so sánh chi tiết giữa vòng hở và vòng kín từ `evidence/openloop.json` và `closedloop.json`:

| Bộ điều khiển | L2 vòng hở (m) | Hạng vòng hở | Tỷ lệ va chạm vòng kín (%) | Hạng vòng kín |
|---|---|---|---|---|
| constant-velocity | 1.253 m | 3 | 19.9% (575 / 2.883) | 3 |
| IDM | 1.523 m | 4 (kém nhất) | 0.1% (4 / 2.883) | 1 (tốt nhất) |
| EgoMLP | 0.759 m | 2 | 23.8% (mean 5 seeds: 19.5% .. 29.9%) | 4 (kém nhất) |
| PercMLP | 0.619 m | 1 (tốt nhất) | 1.1% (mean 5 seeds: 0.4% .. 2.0%) | 2 |

Các cặp bị đảo ngược thứ hạng hoàn toàn: `('const-vel', 'IDM')`, `('const-vel', 'EgoMLP')`, `('IDM', 'EgoMLP')`, và `('IDM', 'PercMLP')`.
- IDM có điểm L2 vòng hở kém nhất (1.523 m) vì luật toán học không cố sao chép từng dao động vi mô của người lái trên quỹ đạo tĩnh, nhưng khi đưa vào vòng kín, IDM lái xuất sắc nhất (chỉ 0.1% va chạm) nhờ có vòng lặp phản hồi kín (closed-loop feedback) liên tục điều chỉnh gia tốc dựa trên khoảng cách tức thời và vận tốc tương đối.
- Ngược lại, EgoMLP có L2 tốt hơn hẳn constant-velocity (0.759 m so với 1.253 m) vì nó học được phân bố tốc độ mượt mà của con người trong các tình huống bình thường. Tuy nhiên khi vào vòng kín nó lại đâm nhiều nhất (23.8%, riêng trên lát cắt phanh gấp đâm tới 53.8%) vì EgoMLP hoàn toàn không được cung cấp thông tin không gian phía trước (không có gap và vận tốc xe dẫn), biến nó thành một hệ thống "mù" không có khả năng phản ứng khi xe phía trước phanh dừng.

## Câu 5 · Một lần chạy chưa đủ
EgoMLP dao động bao nhiêu giữa các seed? Khoảng tin cậy Wilson nói gì? Nếu bạn chỉ chạy đúng một seed thì có thể báo kết quả nào, và vì sao điều đó nguy hiểm?

Dựa trên dữ liệu 5 seed huấn luyện từ `evidence/closedloop.json`:
1. Tỷ lệ va chạm vòng kín của EgoMLP dao động dữ dội từ 19.49% (seed 8064, với k = 562 va chạm trên n = 2.883) lên tới 29.86% (seed 8066, với k = 861 va chạm trên n = 2.883), độ chênh lệch cực đại giữa các seed lên tới 10.37 điểm phần trăm (mean là 23.83%). Ngay cả PercMLP cũng dao động từ 0.42% (seed 8064) đến 1.98% (seed 8066).
2. Khoảng tin cậy Wilson 95% ở seed tốt nhất của EgoMLP là [18.09%, 20.98%] hoàn toàn tách rời, không hề có bất kỳ điểm chồng lấn nào với khoảng tin cậy của seed xấu nhất là [28.22%, 31.56%].
3. Nếu chỉ chạy duy nhất một seed, kỹ sư có thể ngẫu nhiên chọn trúng seed 8064 và báo cáo tỷ lệ va chạm chỉ 19.5% (hoặc đối với PercMLP chỉ 0.4%), tạo ra ảo giác rằng mô hình đã đạt độ an toàn chấp nhận được. Ngược lại, nếu chạy trúng seed 8066, kết quả lại lên tới 29.9%. Điều này cực kỳ nguy hiểm trong phát triển xe tự hành vì sự ngẫu nhiên khi khởi tạo trọng số mạng nơ-ron và xáo trộn batch có thể dẫn đến việc phát hành một hệ thống không ổn định vào thế giới thực, nơi một seed kém may mắn có thể gây ra tai nạn thảm khốc.

## Câu 6 · Tác nhân phản ứng
Dán bảng "xe sau phát lại" và "xe sau phản ứng". `T` bạn hiệu chỉnh là bao nhiêu, xe sau chân thực đến mức nào so với xe không phản ứng? Vì sao IDM bị tính "va chạm giả" khi xe sau chỉ phát lại, và điều này làm đổi kết luận về bộ điều khiển ra sao?

Từ `evidence/agents.json`, kết quả kiểm thử trên lưới kịch bản cắt vào ở độ trễ phản ứng t_react = 0.6 s:

| Bộ điều khiển | Xe sau phát lại (`replay`) | Xe sau phản ứng (`reactive`) | Va chạm đuôi: replay / react |
|---|---|---|---|
| constant-velocity | 80.6% | 80.6% | 0.0% / 0.0% |
| IDM | 80.9% | 10.2% | 70.6% / 0.0% |
| EgoMLP | 80.6% | 80.6% | 0.0% / 0.0% |
| PercMLP | 77.8% | 33.3% | 44.6% / 0.0% |

(Tương tự ở t_react = 0.0 s: IDM giảm từ 80.3% xuống 3.6% khi xe sau phản ứng; va chạm sau giảm từ 76.7% về 0.0%).
1. Thời khoảng cách an toàn `T` hiệu chỉnh tối ưu cho tác nhân IDM xe sau là `T = 1.4 s` (đạt khoảng cách phân bố Wasserstein W1 = 0.35 so với phân bố thời gian bám đuôi trung vị 1.86 s của người lái thật). Mức này chân thực hơn rất nhiều so với một tài xế không bao giờ phản ứng (có W1 = 0.86).
2. Khi xe sau chỉ phát lại log (`replay`), nó di chuyển mù quáng theo vận tốc ghi sẵn mà không hề biết xe thử nghiệm phía trước đang phanh. Khi IDM phản ứng chuẩn xác và phanh gấp để tránh xe cắt vào phía trước, xe sau phát lại đâm sầm vào đuôi xe IDM, tạo ra tới 70.6% "va chạm giả" từ phía sau. Hiện tượng này làm tỷ lệ va chạm của IDM bị đội lên 80.9%, trông tệ ngang ngửa một bộ điều khiển tồi như constant-velocity (80.6%).
3. Khi đưa tác nhân xe sau có phản ứng vào môi trường, va chạm phía sau của IDM biến mất hoàn toàn (0.0%), đưa tổng va chạm của IDM về đúng bản chất an toàn thực tế là 10.2% (trùng với biên vật lý). Điều này làm thay đổi căn bản kết luận: việc loại bỏ tác nhân phản ứng sẽ trừng phạt bất công các bộ điều khiển biết phanh an toàn.

## Câu 7 · Cổng phát hành và sửa
Quy tắc nào chặn PercMLP và nguyên nhân gốc là gì (dẫn số)? Bạn sửa bằng gì và vì sao nó chữa đúng nguyên nhân? Sau khi qua cổng, PercMLP+ có **tốt hơn** IDM không? Nó có nên được phát hành không, và bạn đề xuất làm gì trước khi quyết định?

Từ `evidence/gate.json` và `fix.json`:
1. PercMLP bị chặn bởi 5 quy tắc: R2 (tỷ lệ va chạm trên log 2.0%, cận trên 2.6% > baseline 0.1% + 1.0%), R3 (lát cắt phanh 7.1%, cận trên 11.7% > baseline 0.5% + 5.0%), R4 (va chạm lưới kịch bản 23.0% > baseline 0.0% + 2.0%), R5 (độ phân tán qua 5 seed là 1.6% > 1.0%), và R6 (thế giới phản ứng 33.3% > baseline 10.2% + 2.0%). Hai nguyên nhân gốc là: (a) Dữ liệu log thiếu các kịch bản cắt vào hiểm hóc (out-of-distribution) dẫn tới trượt R4, R6; và (b) Sự tích tụ sai số nhỏ trong vòng kín (covariate shift) khi mô hình tự đưa mình vào các trạng thái chưa từng gặp trong log, dẫn tới trượt R2, R3, R5.
2. Ta sửa bằng hai giải pháp kết hợp: bổ sung 197.060 cửa sổ cắt vào được sinh mô phỏng và dán nhãn bởi IDM (`cutin_windows`, khắc phục nguyên nhân a); kết hợp 2 vòng thuật toán DAgger (`dagger_windows`, khắc phục nguyên nhân b) cho mô hình tự lái trên log, ghi lại các trạng thái nó tự ghé thăm và gắn nhãn phục hồi bằng dự báo của chuyên gia IDM. Sau khi sửa, PercMLP+ đã vượt qua cả 7 quy tắc và được duyệt RELEASE (va chạm trên log giảm còn 0.3..0.5%, va chạm lưới kịch bản giảm ngoạn mục từ 23.0% xuống 0.3%, và độ ổn định seed đạt 0.3..0.5% qua 5 seed).
3. Sau khi qua cổng, PercMLP+ **không hề tốt hơn** IDM: trên log thật IDM chỉ va chạm 0.1% (so với PercMLP+ là 0.4% mean); trên lưới thử nghiệm tốc độ cao 20 m/s chưa từng huấn luyện, IDM có tỷ lệ va chạm tránh được là 0.0%, trong khi PercMLP+ vẫn đâm 0.1..1.9%. Bản chất PercMLP+ học hoàn toàn từ luật chuyên gia IDM nên không thể vượt qua thầy của nó. Vì vậy, **chưa nên phát hành độc lập** PercMLP+ để thay thế hoàn toàn IDM; cần giữ IDM làm bộ điều khiển giám sát an toàn dự phòng (safety fallback), đồng thời tiến hành chạy thử nghiệm ngầm (shadow mode) song song trên xe thật trước khi đưa ra quyết định thương mại hóa.

## Câu 8 · Bài lab này không chứng minh được gì
Chọn **ít nhất ba** giới hạn quan trọng nhất đối với việc quyết định phát hành và giải thích vì sao mỗi cái làm kết luận của bạn kém chắc chắn. Điều gì cần làm tiếp để thu hẹp khoảng cách đến thế giới thật?

Ba giới hạn quan trọng nhất của bài lab đối với quyết định phát hành thực tế:
1. **Mô hình động lực học thuần 1 chiều dọc (1-D longitudinal simplification):** Bài lab chỉ xét chuyển động dọc trên một đường thẳng, bỏ qua hoàn toàn chuyển động ngang (lateral dynamics), độ bám đường của lốp, góc đánh lái, và khả năng chuyển làn tránh vật cản. Trong thực tế, khi gặp xe cắt vào ở khoảng cách gần, phản xạ tự nhiên của con người và hệ thống lái tự động tiên tiến là đánh lái né tránh sang làn thoáng chứ không chỉ phanh khẩn cấp trên cùng một làn.
2. **Giả định nhận diện cảm biến lý tưởng (Ideal Perception assumption):** Bài lab giả định hệ thống biết chính xác khoảng cách gap và vận tốc xe dẫn (hoặc chỉ thêm nhiễu Gaussian đơn giản ±1.0 m và ±0.5 m/s). Trong thực tế, các cảm biến Camera/LiDAR/Radar đối mặt với hiện tượng che khuất (occlusion), sai số trễ đồng bộ tín hiệu, đứt đoạn bám vết (tracking drop), và nhiễu thời tiết nghiêm trọng (mưa, sương mù, chói sáng), vốn là nguyên nhân chính gây ra tai nạn.
3. **Khoảng cách hiện thực của mô phỏng (Sim-to-Real gap & Non-interactive lead):** Xe dẫn trong log hoàn toàn không phản ứng với hành vi của xe thử nghiệm, và tác nhân phản ứng xe sau chỉ là một luật IDM đơn giản hóa với một giá trị T cố định. Giao thông thực tế là một hệ thống đa tác nhân phức tạp, bất định và chứa đựng nhiều hành vi phi lý trí (như xe tạt đầu rồi phanh gấp, người đi bộ băng qua đường bất ngờ).
- Bước tiếp theo cần làm để thu hẹp khoảng cách tới thế giới thật: (1) Đưa mô hình vào các bộ mô phỏng 3D độ chân thực cao (như CARLA hoặc CarSim) tích hợp mô hình quang học cảm biến và động lực học 6 bậc tự do; (2) Triển khai thử nghiệm chế độ chạy bóng (Shadow Mode) trên hạm đội xe thực tế chạy hàng triệu kilomet để so sánh hành vi mô hình với tài xế an toàn mà không tác động lên cơ cấu chấp hành; và (3) Xây dựng lớp bảo vệ an toàn dự phòng bằng luật toán học (Safety Envelope / RSS) để can thiệp cưỡng bức bất cứ khi nào mô hình học sâu vi phạm giới hạn an toàn vật lý.
