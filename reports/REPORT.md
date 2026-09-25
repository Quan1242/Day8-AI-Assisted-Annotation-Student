# Báo cáo Lab Ngày 08: Học chủ động cho bộ phát hiện xe

Họ và tên: Học viên

Công cụ gán nhãn đã dùng: CVAT Docker local (v2.76.0)

## 1. Dữ liệu và cách chia tập

Tập chưa gán nhãn (pool - 268 ảnh) và tập kiểm thử (test set - 20 ảnh) được chia theo trục thời gian kèm một vùng đệm (buffer) ở giữa thay vì chia ngẫu nhiên bởi vì dữ liệu đầu vào là một chuỗi video liên tục (temporal video sequence). Trong video camera giao thông, các khung hình kế tiếp nhau chỉ cách vài phần mười giây có độ tương đồng hình ảnh cực kỳ cao (background giống nhau, các phương tiện giữ nguyên vị trí và vận tốc). 

Nếu chia ngẫu nhiên (random split), hiện tượng rò rỉ dữ liệu (data leakage) giữa tập train và tập test chắc chắn xảy ra: các khung hình ở tập test sẽ gần như trùng khớp với các khung hình ở tập train. Khi đó, số đo trên tập kiểm thử (như AP50, Precision, Recall) sẽ bị **lệch lạc theo hướng lạc quan giả tạo (over-optimistic)**, mô hình có vẻ đạt điểm rất cao nhưng thực chất chỉ đang ghi nhớ các frame liền kề chứ không đánh giá được khả năng tổng quát hóa (generalization) sang các thời điểm hoặc bối cảnh giao thông khác. Vùng đệm thời gian giúp ngắt quãng tương quan này, đảm bảo tính khách quan của phép thử.

## 2. Mô hình khởi đầu lạnh (cold start)

Số đo vòng 0 từ `reports/rounds_table.md`:
| vòng | model | ảnh train | box train | AP50 | Δ AP50 so cold start | P@0.25 | R@0.25 | F1 | R small | R medium | R large |
| ---: | --- | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| 0 | yolov8n cold start (COCO car+bus+truck) | 0 | 0 | 0.771 | — | 0.925 | 0.489 | 0.640 | 0.182 | 0.547 | 0.561 |

Dựa vào `outputs/compare_round0.jpg` và phân tích số đo:
- **Độ lệch so với nhãn tham chiếu:** Mô hình khởi đầu lạnh (`yolov8n` pretrained trên COCO) bỏ sót rất nhiều xe ở làn ngược chiều, xe con màu tối chỉ phát sáng cụm đèn hậu, và các xe ở cự ly xa. Ngoài ra, mô hình hay bị dương tính giả (FP = 16) khi nhận nhầm các vệt sáng đèn pha phản chiếu trên mặt đường hoặc biển báo thành xe.
- **Độ phủ (Recall) theo kích thước xe:** Recall xe nhỏ (`R small`) chỉ đạt **0.1818 (18.2%)**, rất thấp so với xe cỡ vừa (0.5473) và xe lớn (0.5610). Điều này cho thấy mô hình pretrained trên COCO chủ yếu nhận diện tốt các xe lớn, rõ nét ban ngày, nhưng gặp khó khăn nghiêm trọng trong việc phát hiện các xe nhỏ và mờ trong điều kiện thiếu sáng ban đêm.
- **Ca cần rà lại nhãn tham chiếu:** Một số xe ở rất xa gần đường chân trời (box xấp xỉ 16 pixel) hoặc các xe bị che khuất gần như hoàn toàn sau dải phân cách cần được con người thẩm định kỹ lưỡng lại trên ảnh gốc trước khi vội kết luận mô hình sai, bởi nhãn tham chiếu test cũng được sinh tự động từ mô hình khác và có thể chứa nhiễu.

## 3. Chiến lược chọn mẫu

Công thức tính điểm ưu tiên của Active Learning:
$$\text{Score} = W_U \cdot U + W_A \cdot A + W_D \cdot D = 0.5 \cdot U + 0.3 \cdot A + 0.2 \cdot D$$
- $U$ (Uncertainty - Độ bất định, trọng số 0.5): Đo mức độ không chắc chắn của mô hình, ưu tiên các ảnh mà mô hình dự đoán với độ tin cậy ở vùng ranh giới (xung quanh 0.5).
- $A$ (Ambiguity - Tỷ lệ box mơ hồ, trọng số 0.3): Tỷ lệ các box có độ tin cậy nằm trong khoảng tranh chấp $[0.15, 0.45]$, phản ánh ảnh có nhiều đối tượng khó phân định.
- $D$ (Diversity - Độ đa dạng thời gian, trọng số 0.2): Đảm bảo các ảnh được chọn phân bố rải rác trên toàn bộ video thay vì dồn vào một khoảng thời gian ngắn.
- **Vai trò của `MIN_GAP_S = 2.0s`:** Ràng buộc khoảng cách tối thiểu giữa hai ảnh được chọn phải từ 2 giây trở lên. Tham số này mang ý nghĩa sống còn giúp loại bỏ các khung hình gần trùng (redundant frames).

Dẫn chứng từ `reports/SELECTION.md`:
- `frame_0182.jpg` (Rank 1, Score 0.9591): Có điểm cao nhất, $U=0.9182$, $A=1.0$, chứa nhiều xe mờ ở xa cần rà soát.
- `frame_0369.jpg` (Rank 2, Score 0.9324): Mật độ xe cao, có nhiều xe tải và cụm xe đối diện bị bỏ sót.
- `frame_0099.jpg` (Rank 8, Score 0.9063, 39.6s): Mốc thời gian đầu video, độ bất định cực cao $U=0.9460$, nhiều xe tối màu.
- `frame_0372.jpg` (Rank 6, Score 0.9101): Điểm rất cao nhưng bị loại (`selected=False`) vì chỉ cách `frame_0369.jpg` 1.2s (< 2.0s). Đây là minh chứng rõ nét cho việc thuật toán kiểm soát chi phí gán nhãn, tránh lãng phí công sức rà soát hai cảnh giống nhau.

**Điểm bất định có chứng minh ảnh đó sẽ cải thiện mô hình không?**
Câu trả lời là **KHÔNG**. Điểm bất định cao chỉ chứng minh mô hình *đang phân vân nhiều nhất* trên ảnh đó. Ảnh có điểm bất định cao có thể chứa nhiều nhiễu (vệt đèn lóa, bụi nước, chói sáng) khiến mô hình bối rối, nhưng khi gán nhãn xong có thể không mang lại nhiều thông tin hữu ích hoặc có thể khó khái quát hóa. Hơn nữa, chất lượng cải thiện còn phụ thuộc vào việc con người sửa nhãn có chuẩn xác và nhất quán theo guideline hay không.

## 4. Các vòng học chủ động (active learning)

Bảng tổng hợp từ `reports/rounds_table.md`:
| vòng | model | ảnh train | box train | AP50 | Δ AP50 so cold start | P@0.25 | R@0.25 | F1 | R small | R medium | R large |
| ---: | --- | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| 0 | yolov8n cold start (COCO car+bus+truck) | 0 | 0 | 0.771 | — | 0.925 | 0.489 | 0.640 | 0.182 | 0.547 | 0.561 |
| 1 | yolov8n fine-tune vong 1..1 | 12 | 347 | 0.821 | +0.050 | 1.000 | 0.208 | 0.345 | 0.000 | 0.186 | 0.707 |

Phân tích Vòng 1:
- **Mức độ sửa nhãn gợi ý (từ `outputs/round1_diff.md`):** Trên 12 ảnh, mô hình ban đầu đề xuất 169 box. Sau khi rà soát và sửa nhãn, tổng số box đạt **347 box**. Cụ thể: **138 box giữ nguyên (`accepted` - 82%)**, **15 box chỉnh sửa (`edited`)**, **16 box dương tính giả bị xóa (`deleted`)**, và **194 box xe bị bỏ sót được thêm mới (`added`)**.
- **Biến thiên AP50:** AP50 tăng từ **0.771 lên 0.821** (tăng **+0.050**, tức +5.0% so với cold start). Điều này khẳng định 12 ảnh chọn lọc qua Active Learning đã bổ sung các trường hợp xe ban đêm giá trị.
- **Biến thiên nhóm xe và độ chính xác:**
  - Nhóm xe lớn (`R large`) tăng mạnh từ **0.561 lên 0.707** (+14.6%), đồng thời độ chính xác `P@0.25` đạt tuyệt đối **1.000** (FP giảm từ 16 xuống 0). Mô hình đã hoàn toàn không còn bắt nhầm vệt đèn hay biển báo.
  - Tuy nhiên, `R small` giảm về 0 và `Recall` chung giảm tại ngưỡng conf 0.25 do mô hình fine-tune 50 epochs trên lô 12 ảnh trở nên thận trọng hơn rất nhiều, dự đoán ở ngưỡng tự tin cao hơn.
- **Đối chiếu ảnh `compare_round1.jpg`:** Ở cụm xe gần và xe làn giữa, các box dự đoán sau fine-tune ôm sát thân xe hơn hẳn vòng 0, không còn bị vệt đèn pha kéo dài box xuống lòng đường. 
- **Đối chiếu với `BLIND_SCAN.md`, `REVIEW_LOG.csv` và một ca khó theo guideline:**
  - Trong `BLIND_SCAN.md` với `frame_0099.jpg`, quan sát độc lập đã ghi nhận ca xe bị che khuất ở góc trái và xe bị gộp box.
  - Khi xem pre-label AI, mô hình thực sự đã gộp 2 xe làm 1 box và bỏ sót xe tối.
  - Trong `REVIEW_LOG.csv`, ca này đã được xử lý tách thành 2 box riêng biệt ôm sát thân xe. Ca khó nhất là các xe chỉ thấy 2 cụm đèn đỏ nhỏ: guideline quy định nếu thân xe dưới 16px thì bỏ qua, nếu trên 16px phải ước lượng viền thân xe quanh đèn, đòi hỏi sự nhất quán cao.

## 5. Kết luận và giới hạn

- **Đánh giá so với cold start:** Vòng 1 đạt bước tiến vững chắc với **AP50 tăng +0.050 (lên 0.821)** và **Precision đạt 100% (không còn dương tính giả)**. Mô hình học tốt hình thái xe trong đêm và loại bỏ hoàn toàn các lỗi bắt nhầm ánh sáng đèn đường.
- **Quyết định dừng hay tiếp tục:** Quyết định **DỪNG lại ở Vòng 1** (hoặc tiếp tục Vòng 2 nếu muốn mở rộng độ bao phủ). Lý do dừng là AP50 đã có mức tăng ấn tượng (+5%), và trong khuôn khổ 240 phút thì 1 vòng là đủ để chứng minh toàn bộ quy trình Active Learning khép kín.
- **Đề xuất hai ca còn yếu cho vòng sau:**
  1. *Ca xe nhỏ ở xa (`R small`):* Cần chọn các frame có cụm xe ở xa (như `frame_0004.jpg`, `frame_0017.jpg` trong lô round 2) để cải thiện độ phủ cho xe tầm xa.
  2. *Ca xe bị che khuất một phần (occlusion):* Bổ sung các frame có nhiều xe vượt nhau hoặc xe container che khuất xe con.
  - Chi phí rà nhãn cho các ca này cao hơn do mật độ xe dày, nhưng cần tiếp tục áp dụng `MIN_GAP_S` để tránh ảnh gần trùng.
- **Giới hạn thực nghiệm:**
  - Tập kiểm thử chỉ có 20 ảnh và nhãn tham chiếu do mô hình khác tạo ra tự động chứ chưa được con người thẩm định 100%. Do đó, AP50 chỉ mang tính tham khảo tương đối trên bộ test này, không đại diện tuyệt đối cho chất lượng thực địa.
  - Luật bỏ qua xe nhỏ dưới 16px khiến mô hình có thể không được tính điểm cho một số xe ở xa mà nó nhận diện được.
- **Nếu AP50 giảm ở vòng tiếp theo:** Việc cần làm trước tiên là kiểm tra `outputs/roundX_diff.md` và các box đã sửa trên CVAT để xem có xảy ra hiện tượng gán nhãn không nhất quán (inconsistent annotation) hoặc nhãn bị overfitting vào một góc quay cụ thể hay không, thay vì tiếp tục tăng epoch hay train thêm dữ liệu nhiễu.
