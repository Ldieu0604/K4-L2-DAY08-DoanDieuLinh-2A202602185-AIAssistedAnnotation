# Quét độc lập trước khi xem pre-label

Frame: `frame_0107.jpg`

Số xe nhìn thấy bằng mắt: 26

Hai vị trí dễ bị AI bỏ sót hoặc vẽ sai, kèm mô tả xe: 
- Phía xa tít ở cuối tầm nhìn (gần đường chân trời trên cao tốc) Rất nhiều xe đang di chuyển ở khoảng cách xa, hình dáng thân xe hoàn toàn chìm vào bóng tối. Chúng chỉ xuất hiện dưới dạng những chấm sáng rất nhỏ. Trong điều kiện ban đêm, ánh sáng đèn hậu màu đỏ của chúng rực lên và hòa vào nhau (glare). Xe chạy sau bị xe chạy trước che khuất (occlusion) một phần lớn. AI thường gặp khó khăn trong việc phân tách ranh giới giữa các xe này và có thể gom chúng lại thành một đối tượng duy nhất hoặc bỏ sót chiếc xe bị che khuất phía sau.
- Khu vực mép dưới cùng của khung hình (đặc biệt là góc dưới bên trái, phải và chính giữa) Mô tả xe: Có những chiếc xe chỉ mới lọt vào khung hình một phần rất nhỏ (chỉ thấy mui xe hoặc một chút bóng đen của nóc xe). Những xe này rất tối, không có đèn pha hoặc đèn hậu lọt vào khung ảnh. Việc xe bị cắt cúp (truncated) bởi viền ảnh và thiếu độ tương phản khiến AI không đủ đặc trưng (features) để nhận diện đó là một chiếc xe hoàn chỉnh.
Chạy `python3 tools/lock_blind.py` ngay sau khi điền. Sau đó giữ file này nguyên vẹn.
