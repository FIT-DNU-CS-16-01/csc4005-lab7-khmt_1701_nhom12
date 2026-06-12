# CSC4005 Lab 7 Report – Compression: KD + Quantization Trade-offs

## 1. Thông tin

* Họ tên: ...
* Mã sinh viên: ...
* Lớp: ...
* Link GitHub repo: ...
* Kỹ thuật chọn: Quantization
* Link W&B nếu dùng KD: Không sử dụng
* Link model nếu không commit trực tiếp: ...

---

# 2. Mô tả baseline model

| Nội dung                 | Giá trị                           |
| ------------------------ | --------------------------------- |
| Bài toán                 | Smart Campus Scene Classification |
| Dataset                  | MIT Indoor Scenes 67 subset       |
| Số lớp                   | 5                                 |
| Baseline model           | Vision Transformer (ViT-B/16)     |
| Baseline format          | ONNX                              |
| Baseline checkpoint/ONNX | models/vit_smartcampus.onnx       |
| Baseline model size      | 327.72 MB                         |

---

# 3. Kỹ thuật nén đã chọn

## Quantization

| Thông tin            | Giá trị                            |
| -------------------- | ---------------------------------- |
| Loại quantization    | Dynamic Quantization               |
| Input model          | vit_smartcampus.onnx               |
| Output model         | vit_smartcampus_dynamic_uint8.onnx |
| Dạng dữ liệu sau nén | UINT8                              |
| Công cụ              | onnxruntime.quantization           |

Mô tả ngắn:

```text
Nhóm sử dụng Dynamic Quantization của ONNX Runtime để giảm kích thước
mô hình Vision Transformer. Quá trình quantization chuyển trọng số
mô hình từ floating point sang UINT8 nhằm giảm bộ nhớ sử dụng và
tăng tốc inference trên CPU.

Ban đầu nhóm thử nghiệm với QInt8 nhưng gặp lỗi ConvInteger operator
trên ONNX Runtime CPU. Sau đó nhóm chuyển sang QUInt8 để đảm bảo
tính tương thích khi inference và benchmark.
```

---

# 4. Kết quả đánh giá

| Model      | Accuracy | Macro-F1 | Model size (MB) |
| ---------- | -------: | -------: | --------------: |
| Baseline   |   0.9708 |   0.9623 |          327.72 |
| Compressed |   0.9785 |   0.9708 |           83.24 |

Nhận xét:

* Accuracy không giảm sau quantization mà còn tăng nhẹ.
* Macro-F1 cũng tăng nhẹ sau khi nén.
* Kích thước mô hình giảm đáng kể (~74.6%).
* Mức thay đổi accuracy rất nhỏ nên hoàn toàn chấp nhận được.
* Quantization giúp tăng hiệu quả triển khai mà không làm ảnh hưởng đáng kể đến chất lượng mô hình.

---

# 5. Kết quả benchmark

| Model      | Batch size | Mean latency (ms) | P95 latency (ms) | Throughput (img/s) | Size (MB) |
| ---------- | ---------: | ----------------: | ---------------: | -----------------: | --------: |
| Baseline   |          1 |            175.21 |           203.29 |               5.71 |    327.72 |
| Compressed |          1 |            142.71 |           166.95 |               7.01 |     83.24 |
| Baseline   |          4 |            696.97 |           782.98 |               5.74 |    327.72 |
| Compressed |          4 |            585.90 |           677.69 |               6.83 |     83.24 |
| Baseline   |          8 |           1432.97 |          1613.78 |               5.58 |    327.72 |
| Compressed |          8 |           1176.15 |          1353.84 |               6.80 |     83.24 |

---

# 6. Bảng trade-off

| Model      | Accuracy | Macro-F1 | Mean latency @bs=1 | Throughput @bs=1 |      Size | Nhận xét                                                        |
| ---------- | -------: | -------: | -----------------: | ---------------: | --------: | --------------------------------------------------------------- |
| Baseline   |   0.9708 |   0.9623 |          175.21 ms |             5.71 | 327.72 MB | Accuracy cao nhưng model lớn và inference chậm hơn              |
| Compressed |   0.9785 |   0.9708 |          142.71 ms |             7.01 |  83.24 MB | Model nhỏ hơn, inference nhanh hơn và accuracy vẫn được giữ tốt |

---

# 7. Phân tích

## 1. Mô hình sau nén nhỏ hơn bao nhiêu phần trăm?

Mô hình sau quantization giảm từ 327.72 MB xuống còn 83.24 MB,
tương đương giảm khoảng 74.6% kích thước.

## 2. Latency giảm hay tăng?

Latency giảm đáng kể sau quantization.
Ví dụ ở batch size 1, latency giảm từ 175.21 ms xuống còn 142.71 ms.

## 3. Throughput thay đổi thế nào?

Throughput tăng sau quantization.
Ở batch size 1, throughput tăng từ 5.71 img/s lên 7.01 img/s.

## 4. Accuracy/F1 giảm nhiều không?

Accuracy và Macro-F1 gần như không giảm.
Thậm chí kết quả còn tăng nhẹ do sai số làm tròn và ảnh hưởng của quantization.

## 5. Nếu triển khai trên CPU hoặc edge device, bạn có chọn compressed model không?

Có. Vì compressed model nhỏ hơn nhiều, inference nhanh hơn,
tiết kiệm tài nguyên bộ nhớ và vẫn giữ được chất lượng dự đoán tốt.

## 6. Nếu không chọn, lý do là gì?

Trong bài lab này nhóm chọn compressed model vì trade-off đạt được là rất tốt.
Tuy nhiên trong một số hệ thống yêu cầu độ chính xác cực cao,
có thể cần giữ lại mô hình gốc để tránh rủi ro mất accuracy.

---

# 8. Khi nào chọn KD, khi nào chọn Quantization?

* Quantization phù hợp khi cần giảm kích thước mô hình và tăng tốc inference nhanh chóng mà không cần train lại mô hình.
* Knowledge Distillation phù hợp khi muốn xây dựng một student model nhỏ hơn nhưng vẫn học được kiến thức từ teacher model lớn.
* Nếu được làm lại cho hệ thống Smart Campus, nhóm vẫn ưu tiên Quantization vì dễ triển khai, tiết kiệm thời gian và đạt hiệu quả tốt trên CPU.

---

# 9. Kết luận

Trong bài lab này, nhóm sử dụng kỹ thuật Dynamic Quantization trên mô hình Vision Transformer ONNX cho bài toán Smart Campus Scene Classification. Sau quá trình quantization, kích thước mô hình giảm từ 327.72 MB xuống còn 83.24 MB, tương đương giảm khoảng 74.6%.

Kết quả benchmark cho thấy mô hình quantized có latency thấp hơn và throughput cao hơn so với baseline model. Accuracy và Macro-F1 vẫn được giữ ở mức rất cao và gần như không bị ảnh hưởng.

Ngoài ra, nhóm gặp lỗi ConvInteger khi sử dụng QInt8 trên ONNX Runtime CPU nên đã chuyển sang QUInt8 để đảm bảo khả năng tương thích khi inference.

Qua bài lab, nhóm nhận thấy quantization là một kỹ thuật hiệu quả giúp tối ưu mô hình cho triển khai thực tế trên CPU hoặc edge device mà vẫn đảm bảo chất lượng dự đoán.
