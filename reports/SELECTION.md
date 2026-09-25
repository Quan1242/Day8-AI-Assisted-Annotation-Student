# Vì sao chọn lô này?

Trong 50 dòng đứng đầu `outputs/selection_round1.csv`, chọn năm frame bạn sẽ ưu tiên nếu chỉ có
ngân sách rà năm ảnh. Ghi tên, điểm, thời điểm, thứ tự và lý do; tối thiểu một quyết định phải xét
ảnh gần trùng hoặc trường hợp model không dự đoán được box:
- `frame_0182.jpg` (Thứ tự: Rank 1 | Thời điểm: 72.8s | Điểm: 0.9591): Có điểm số cao nhất toàn tập pool với độ bất định U=0.9182, số box mơ hồ đạt tối đa A=1.0 (18 box mơ hồ), chứa nhiều xe nhỏ mờ ở xa cần được chuẩn hóa nhãn.
- `frame_0369.jpg` (Thứ tự: Rank 2 | Thời điểm: 147.6s | Điểm: 0.9324): Độ bất định rất cao U=0.9315, mật độ phương tiện đông đúc (43 candidates), đặc biệt có nhiều cụm xe ở làn đối diện và xe tải lớn bị AI bỏ sót.
- `frame_0099.jpg` (Thứ tự: Rank 8 | Thời điểm: 39.6s | Điểm: 0.9063): Đại diện cho mốc thời gian sớm của video (39.6s), độ bất định cực cao U=0.9460; chứa nhiều xe tối màu khó phát hiện và xe bị che khuất ở góc trái.
- `frame_0270.jpg` (Thứ tự: Rank 13 | Thời điểm: 108.0s | Điểm: 0.8878): Mốc thời gian 108.0s giúp dàn đều độ đa dạng theo thời gian (giữa video), tránh tập trung dồn vào cụm 130s-150s; có xe khách lớn và xe bị cắt mép ảnh.
- `frame_0326.jpg` (Thứ tự: Rank 4 | Thời điểm: 130.4s | Điểm: 0.9155): Điểm cao đại diện cho cụm giao thông đông đúc lúc 130s; việc chọn frame này giúp gom bắt các ca khó mà cho phép bỏ qua các frame kế cận gần trùng lặp (như frame_0330, frame_0331) nhằm tối ưu chi phí gán nhãn cho ngân sách 5 ảnh.

Ba frame thuộc lô 12 ảnh model chọn và bằng chứng trong CSV/ảnh contact sheet:
- `frame_0182.jpg`: Rank 1 trong CSV, điểm 0.9591. Trên contact sheet cho thấy cụm xe ở làn đối diện bị mờ và vệt sáng kéo dài; thực tế đối chiếu trong `round1_diff.md` phải thêm tới 13 box xe thật và chỉnh 1 box lệch vệt đèn.
- `frame_0331.jpg`: Rank 5 trong CSV, điểm 0.9154, t_sec 132.4s, có 18 box mơ hồ và tới 47 box candidates. Trên contact sheet có nhiều vệt đèn chói lóa trên mặt đường; trong `round1_diff.md` phải xóa 4 box giả và thêm 21 box xe bị bỏ sót.
- `frame_0369.jpg`: Rank 2 trong CSV, điểm 0.9324, t_sec 147.6s. Contact sheet cho thấy mật độ xe dày đặc ở tầm xa; thực tế trong `round1_diff.md` đã bổ sung tới 25 box xe bị AI bỏ quên (nhiều nhất trong lô).

Một frame có điểm cao nhưng không chọn hoặc một frame có điểm thấp vẫn nên xem, và lý do:
- `frame_0372.jpg`: Có điểm số rất cao (Score: 0.9101, Rank 6, U=0.9202) nhưng không được chọn (`selected=False`). Lý do: Thời điểm xuất hiện là 148.8s, chỉ cách `frame_0369.jpg` (147.6s) đúng 1.2 giây, vi phạm ràng buộc khoảng cách thời gian tối thiểu `MIN_GAP_S = 2.0s`. Thuật toán loại bỏ frame này là hoàn toàn chính xác để tránh gán nhãn lặp lại cho hai khung hình gần như đồng nhất, giúp tiết kiệm chi phí dán nhãn.

Điều phép chọn này chưa chứng minh về chất lượng mô hình:
- Điểm số bất định kết hợp (Uncertainty, Ambiguity, Diversity) chỉ phản ánh mức độ do dự, phân vân của mô hình hiện tại trên các khung hình chưa gán nhãn trong tập pool; nó không đảm bảo hay chứng minh rằng việc gán nhãn các ảnh này chắc chắn sẽ làm tăng AP50 trên tập test. Hơn nữa, tập kiểm thử chỉ có 20 ảnh với nhãn tham chiếu do mô hình sinh ra (chưa qua thẩm định con người), do đó số đo trên tập test có thể không tăng ngay dù mô hình đã học thêm được nhiều đặc trưng xe đêm từ tập gán nhãn.
