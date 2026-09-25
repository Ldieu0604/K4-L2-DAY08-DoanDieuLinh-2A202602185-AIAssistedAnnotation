# Vì sao chọn lô này?

Trong 50 dòng đứng đầu `outputs/selection_round1.csv`, chọn năm frame bạn sẽ ưu tiên nếu chỉ có
ngân sách rà năm ảnh. Ghi tên, điểm, thời điểm, thứ tự và lý do; tối thiểu một quyết định phải xét
ảnh gần trùng hoặc trường hợp model không dự đoán được box: 
Dựa vào 50 dòng đầu tiên, 5 frame ưu tiên nhất (có thứ hạng cao và đảm bảo tính đa dạng) bao gồm:
1. **frame_0182.jpg** (Rank 1, Score: 0.9591, t_sec: 72.8, U: 0.9182, D: 1.0): Điểm bất định cực cao với 18 bounding boxes bất định, lại cách xa về thời gian so với các khung hình khác nên không bị lỗi trùng lặp (D=1.0).
2. **frame_0369.jpg** (Rank 2, Score: 0.9324, t_sec: 147.6, U: 0.9315, D: 1.0): Có tới 43 box dự đoán và 16 box bất định, bối cảnh giao thông đông đúc, khoảng cách thời gian an toàn.
3. **frame_0380.jpg** (Rank 3, Score: 0.9170, t_sec: 152.0, U: 0.9340, D: 1.0): Mức bất định cao nhất, đạt 15 box không chắc chắn. Cách 4.4 giây so với frame hạng 2, cung cấp thông tin mới.
4. **frame_0326.jpg** (Rank 4, Score: 0.9155, t_sec: 130.4, U: 0.9310, D: 1.0): Điểm bất định cao (U=0.9310) và khoảng cách thời gian xa với các ảnh khác, tạo sự đa dạng.
5. **frame_0331.jpg** (Rank 5, Score: 0.9154, t_sec: 132.4, U: 0.8308, D: 1.0): Dù U thấp hơn một chút, nhưng cảnh này chứa tới 47 boxes (nhiều xe nhất), mang lại bối cảnh rất hỗn loạn để model học hỏi.

Ba frame thuộc lô 12 ảnh model chọn và bằng chứng trong CSV/ảnh contact sheet: 
1. **frame_0182.jpg** (t_sec = 72.8s): Trong file CSV, cột selected = True. Ảnh chứa nhiều xe bị mờ/khuất nên điểm bất định U cao (0.9182), chứng minh qua 18 hộp ambiguous.
2. **frame_0369.jpg** (t_sec = 147.6s): CSV có selected = True. Số lượng hộp ambiguous cao (16 hộp), điểm bất định 0.9315 cho thấy model rất bối rối với cảnh này.
3. **frame_0380.jpg** (t_sec = 152.0s): CSV có selected = True. Mật độ xe cao và bất định nhiều (U=0.9340), nhưng D=1.0 cho thấy khoảng cách hợp lý giúp tránh redundant data.

Một frame có điểm cao nhưng không chọn hoặc một frame có điểm thấp vẫn nên xem, và lý do: 
**frame_0372.jpg** (Rank 6, Score 0.9101, selected=False). Dù có điểm bất định (U=0.9202) cực kỳ cao và nằm vị trí thứ 6, frame này không được model chọn. Lý do là thời điểm t_sec = 148.8 giây, rất gần với **frame_0369.jpg** (147.6 giây). Chọn cả 2 sẽ khiến dữ liệu huấn luyện bị trùng cảnh (redundancy), lãng phí chi phí gán nhãn, nên hệ thống đã khôn ngoan bỏ qua ảnh này.

Điều phép chọn này chưa chứng minh về chất lượng mô hình: 
Điểm số Active Learning (U, D) chỉ đánh giá được mức độ bối rối và phân bố đa dạng của cảnh, nó không chứng minh rằng mô hình đã "nhận diện đúng hay sai". Các hộp bất định có thể là do vật thể bị nhiễu, sai bối cảnh, hoặc đơn giản là xe quá xa, do đó vẫn cần đánh giá trên Test set (metrics) và kiểm tra nhãn thủ công (BLIND_SCAN) mới có thể kết luận chắc chắn.
