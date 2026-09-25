# Báo cáo Lab Ngày 08: Học chủ động cho bộ phát hiện xe

Họ và tên: Đoàn Diệu Linh

Công cụ gán nhãn đã dùng: CVAT

Sao chép file này thành `reports/REPORT.md`. Mọi con số phải truy được
từ `reports/rounds_table.md`, `outputs/selection_round1.csv`, `outputs/metrics_round*.json` hoặc
`outputs/round*_diff.md`. Không coi nhãn test do mô hình tạo là chân lý tuyệt đối.

## 1. Dữ liệu và cách chia tập

Tại sao tập chưa gán nhãn (pool) và tập kiểm thử (test set) được chia theo trục thời gian, có vùng
đệm ở giữa, thay vì chia ngẫu nhiên? Nếu chia ngẫu nhiên, số đo trên tập kiểm thử sẽ bị lệch theo
hướng nào, và vì sao?

Việc chia pool và test theo trục thời gian và để lại một vùng đệm (buffer zone) ở giữa là để đảm bảo tập kiểm thử (test set) hoàn toàn độc lập với tập huấn luyện. Trong video, các khung hình nằm sát nhau về mặt thời gian thường có bối cảnh và các chiếc xe giống hệt nhau. Nếu ta chia ngẫu nhiên, các ảnh "gần như giống hệt nhau" này sẽ bị rải rác lọt vào cả hai tập train và test. Điều này dẫn đến sự rò rỉ dữ liệu (data leakage) - mô hình học thuộc lòng (overfit) các xe ở tập train và áp dụng nguyên si sang tập test. 
Khi đó, điểm kiểm tra (metrics) trên tập test sẽ bị lệch theo hướng **cao hơn thực tế rất nhiều (quá lạc quan)**, khiến ta lầm tưởng mô hình tổng quát hóa tốt nhưng thực chất nó chỉ đang ghi nhớ cục bộ.

## 2. Mô hình khởi đầu lạnh (cold start)

Chép dòng vòng 0 từ `rounds_table.md`. Dựa vào `outputs/compare_round0.jpg`, cho biết mô hình khởi
đầu lạnh không khớp nhãn tham chiếu ở những loại xe nào. Độ phủ (recall) theo kích thước xe cho
thấy điều gì? Một trường hợp nào cần người rà lại nhãn tham chiếu trước khi kết luận mô hình sai?

| vòng | model | ảnh train | box train | AP50 | Δ AP50 so cold start | P@0.25 | R@0.25 | F1 | R small | R medium | R large |
| ---: | --- | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| 0 | yolov8n cold start (COCO car+bus+truck) | 0 | 0 | 0.771 | — | 0.925 | 0.489 | 0.640 | 0.182 | 0.547 | 0.561 |

- **Nhóm xe chưa khớp nhãn tham chiếu**: Dựa vào ảnh so sánh, mô hình cold start chủ yếu bỏ sót (không khớp) các xe có kích thước rất nhỏ (ở xa tít đường chân trời), xe bị khuất (occlusion) bởi xe khác, hoặc xe quá tối nằm ở rìa ảnh.
- **Độ phủ (Recall) theo kích thước**: Recall của xe kích thước nhỏ (small) chỉ đạt **0.182** (tức là bỏ sót đến hơn 81% lượng xe nhỏ). Trong khi đó, recall của xe vừa (medium) đạt **0.547** và xe lớn (large) đạt **0.561**. Điều này cho thấy mô hình hoạt động tương đối tốt với các xe to, rõ, gần camera nhưng cực kỳ kém trong việc nhận diện xe nhỏ.
- **Trường hợp cần kiểm tra lại nhãn tham chiếu**: Đó là khi mô hình dự đoán (vẽ box) đúng vào một chiếc xe bị khuất hoặc quá nhỏ ở xa, nhưng do nhãn tham chiếu (pre-label ban đầu) bị thiếu sót nên dự đoán này bị tính là lỗi False Positive. Ta cần kiểm tra lại xem chiếc xe đó có thực sự tồn tại và hộp box có lớn hơn 16 pixel không, trước khi vội kết luận rằng mô hình đã nhận diện sai.

## 3. Chiến lược chọn mẫu

Giải thích bằng lời công thức `score = W_U·U + W_A·A + W_D·D` và vai trò của `MIN_GAP_S`.
Dẫn ba frame trong `reports/SELECTION.md` và một frame khác để chứng minh cách bạn cân nhắc
độ bất định, ảnh gần trùng và công gán nhãn. Điểm bất định có chứng minh ảnh đó sẽ cải thiện
mô hình không? Vì sao?

- **Công thức score**: Tổng điểm (score) được tính dựa trên ba yếu tố có trọng số: Điểm bất định (U - Uncertainty) phản ánh mức độ không chắc chắn của mô hình; Điểm bất đối xứng/mật độ (A - Asymmetry) đại diện cho số lượng vật thể và sự phức tạp của bối cảnh; Điểm đa dạng (D - Diversity) dùng để đánh giá mức độ khác biệt/không trùng lặp của khung hình so với các khung đã chọn.
- **Vai trò của `MIN_GAP_S`**: Đây là khoảng thời gian tối thiểu giữa các ảnh. Nếu các ảnh nằm quá sát nhau về thời gian (nhỏ hơn khoảng này), điểm đa dạng (D) sẽ bị phạt nặng. Điều này giúp loại bỏ ảnh trùng cảnh, tối ưu hóa ngân sách và công sức gán nhãn.
- **Cách cân nhắc (Dẫn chứng từ SELECTION.md)**: 
  Ba ảnh được chọn là `frame_0182.jpg`, `frame_0369.jpg` và `frame_0380.jpg` đều có điểm bất định (U) rất cao (trên 0.91), đồng thời có điểm D=1.0 do khoảng cách thời gian giữa chúng xa nhau, đảm bảo bối cảnh đa dạng. Ngược lại, `frame_0372.jpg` dù có điểm bất định rất cao (U=0.9202), nhưng lại nằm ở `t_sec=148.8`, chỉ cách `frame_0369.jpg` đúng 1.2 giây. Nhằm tiết kiệm công gán nhãn cho các ảnh gần trùng, hệ thống đã thông minh loại bỏ frame này.
- **Điểm bất định có cải thiện mô hình không?**: Không chắc chắn. Điểm bất định cao chỉ chứng tỏ mô hình đang gặp khó khăn. Khó khăn này có thể là do bối cảnh mới lạ (tốt cho học tập), nhưng cũng có thể do ảnh bị nhiễu (noise), loá sáng hoặc vật thể biến dạng mà ngay cả mắt người cũng không nhận ra. Nếu đưa nhãn mờ mịt/rác này vào, mô hình sẽ học sai và chất lượng giảm xuống.

## 4. Các vòng học chủ động (active learning)

Chép bảng từ `rounds_table.md`. Với mỗi vòng, trình bày:

- mức độ bạn đã sửa nhãn gợi ý (số box giữ nguyên, chỉnh sửa, xoá, thêm mới, lấy từ
  `outputs/round*_diff.md`);
- AP50 thay đổi bao nhiêu so với khởi đầu lạnh và so với vòng trước;
- nhóm xe nào tốt lên hoặc xấu đi theo số đo trên cùng tập test.

Dựa vào các ảnh `compare_round*.jpg`, chỉ ra một ca kết quả đổi sau fine-tune (tốt hơn hoặc xấu
đi), cùng lý do có thể kiểm. Dùng `BLIND_SCAN.md`, `REVIEW_LOG.csv` và `round1_diff.md` phân biệt
quan sát độc lập, lỗi pre-label đã sửa và kết quả mô hình sau train. Mô tả một ca khó theo guideline.

| vòng | model | ảnh train | box train | AP50 | Δ AP50 so cold start | P@0.25 | R@0.25 | F1 | R small | R medium | R large |
| ---: | --- | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| 1 | yolov8n fine-tune vong 1..1 | 12 | 356 | 0.543 | -0.228 | 1.000 | 0.164 | 0.281 | 0.000 | 0.125 | 0.707 |

- **Mức độ sửa nhãn**: Từ 12 ảnh, model gợi ý 169 box. Tôi đã giữ nguyên 132 box, chỉnh sửa 20 box, xoá 17 box và đặc biệt thêm mới tới 204 box (chủ yếu là xe bị sót) (dữ liệu từ `outputs/round1_diff.md`).
- **AP50 thay đổi**: So với vòng khởi đầu lạnh, AP50 giảm thê thảm, giảm 0.228 (từ 0.771 xuống 0.543).
- **Nhóm xe**: Nhóm xe nhỏ (Small) và vừa (Medium) xấu đi nghiêm trọng (R small từ 0.182 về 0.0, R medium từ 0.547 về 0.125). Nhóm xe lớn (Large) lại có xu hướng tốt lên, tăng từ 0.561 lên 0.707.

**Phân biệt quan sát và phân tích ca kết quả đổi sau fine-tune**:
1. *Quan sát độc lập (`BLIND_SCAN.md`)*: Bằng mắt thường, tôi nhận thấy rất nhiều xe ở tít xa cuối tầm nhìn bị chìm vào bóng tối hoặc bị che khuất. Những xe tối ở sát mép ảnh cũng rất khó nhận diện vì thiếu đặc trưng.
2. *Lỗi pre-label đã sửa (`REVIEW_LOG.csv`)*: Dựa trên quan sát đó, tôi đã chủ động "added" rất nhiều box mới cho các xe nhỏ ở xa và các xe tối lọt ở rìa trái (ví dụ ở `frame_0182.jpg` và `frame_0227.jpg`).
3. *Kết quả sau train*: Thay vì học được cách bắt xe nhỏ, mô hình sau khi train lại mất hoàn toàn khả năng này (R small = 0.0). Lý do có thể kiểm chứng: Tập huấn luyện quá bé (12 ảnh) bị nhồi quá nhiều nhãn khó/nhỏ, gây nhiễu loạn gradient hoặc overfit vào các đặc trưng mờ mịt, khiến mô hình "quên" (catastrophic forgetting) kỹ năng bắt xe từ COCO.

**Xử lý ca khó theo GUIDELINE_LABEL.md**:
Một ca khó là khi hai xe đứng sát nhau hoặc xe phía trước che khuất một phần xe phía sau. Theo `GUIDELINE_LABEL.md`, cách xử lý chuẩn là **vẽ hai box riêng rẽ cho từng chiếc xe** và **chỉ vẽ box cho phần thân xe nhìn thấy được** của chiếc bị che khuất, tuyệt đối không gộp chung hai xe làm một (như đã ghi chép việc "edited" trong `REVIEW_LOG.csv` cho `frame_0182.jpg`).

## 5. Kết luận và giới hạn

Kết quả vòng này so với cold start ra sao? Vì sao bạn dừng hoặc tiếp tục? Đề xuất hai ca còn yếu
hoặc bất định cho vòng sau, kèm chi phí rà nhãn và nguy cơ ảnh gần trùng. Tập kiểm thử chỉ 20 ảnh,
có luật bỏ qua xe quá nhỏ và nhãn tham chiếu do mô hình tạo chưa được rà thủ công; các giới hạn đó
ảnh hưởng thế nào đến kết luận? Nếu AP50 giảm, bạn sẽ kiểm tra điều gì trước khi train thêm?

- **So với cold start**: Kết quả giảm mạnh (AP50 giảm 0.228). Do sự sụt giảm hiệu năng đáng kể và mất khả năng bắt xe nhỏ, tôi quyết định **tạm dừng** để kiểm tra lại dữ liệu thay vì làm vòng 2 ngay lập tức.
- **Đề xuất 2 ca còn yếu cho vòng sau**:
  1. Các xe bị che khuất một phần (occlusion).
  2. Các xe ở khoảng cách rất xa (chỉ thấy đường viền hoặc đèn mờ).
  - *Chi phí sửa nhãn*: Việc căng mắt tìm và căn chỉnh viền (bounding box) cho các xe mờ/nhỏ tốn chi phí thời gian rất lớn.
  - *Nguy cơ ảnh gần trùng*: Nếu cố gắng thu thập thêm ảnh có xe nhỏ để dạy mô hình, rất dễ bốc phải các frame liên tiếp nhau của cùng một chiếc xe ở xa, dẫn đến tốn công sửa nhãn mà giá trị mang lại không cao. Cần tinh chỉnh `MIN_GAP_S`.
- **Giải thích giới hạn của tập test**:
  Tập test chỉ 20 ảnh là vô cùng nhỏ, khiến sai số thống kê rất lớn; chỉ cần mô hình bắt hụt vài box là AP50 dao động mạnh. Việc quy định bỏ qua xe dưới 16px và nhãn tham chiếu chưa được con người rà soát chéo (vẫn do mô hình tạo) gây ra tình trạng bất công: Nếu ở tập train ta ép mô hình học xe nhỏ, nhưng tập test lại không chứa nhãn xe nhỏ (do mô hình tự động tạo ra ban đầu bỏ sót), thì khi test mô hình bắt đúng xe nhỏ lại bị đánh giá là sai (False Positive). Điều này làm giảm AP50 một cách ảo tạo.
- **Kiểm tra gì khi AP50 giảm?**:
  1. Kiểm tra tập test: Phải rà soát và gán nhãn thủ công cẩn thận toàn bộ 20 ảnh test để đảm bảo nhãn tham chiếu (Ground Truth) thực sự chuẩn xác và bao gồm cả các xe nhỏ.
  2. Kiểm tra chất lượng các nhãn "added": Rà lại xem các nhãn mới thêm ở tập train có tuân thủ đúng `GUIDELINE_LABEL.md` không, hay ta đã vô tình dạy mô hình học rác/nhiễu.
  3. Kiểm tra hyperparameter: Số epoch quá lớn (50) cho tập 12 ảnh có thể đã gây overfit. Cần tinh chỉnh lại tốc độ học (learning rate) và epoch.
