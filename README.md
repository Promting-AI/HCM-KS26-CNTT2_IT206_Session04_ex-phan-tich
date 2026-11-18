# Yêu cầu đầu ra:

## BƯỚC 1: HAI PROMPT MẪU
 
### Giải pháp 1: Đơn giản, ngắn gọn
 
```
[Context] Bạn là trợ lý tính hóa đơn cho một cửa hàng online.
 
[Request] Hãy tính tổng tiền khách phải trả cho đơn hàng này:
- 3 cái áo, giá 200.000đ/cái
- Mã giảm 10%
- VAT 8%
- Phí ship 30.000đ
 
[Explanation] Hãy suy nghĩ từng bước rồi mới đưa ra kết quả.
 
[Action] Tính từng bước và cho biết tổng tiền cuối cùng.
 
[Tone] Ngắn gọn, dễ hiểu.
 
[Extras] Không có.
```
 
### Giải pháp 2: Chi tiết, có nhiều ràng buộc
 
```
[Context]
Bạn là hệ thống tính hóa đơn của cửa hàng online. Dữ liệu khách nhập có thể
bị thiếu, sai hoặc vô lý. Quy trình gồm 4 bước nối nhau (Prompt Chaining):
Bước A (Kiểm tra dữ liệu) → Bước B (Tính tiền hàng và giảm giá)
→ Bước C (Tính VAT và ship) → Bước D (Tổng hợp).
Kết quả của bước trước là đầu vào của bước sau.
 
[Request]
Xử lý đơn hàng: {số lượng} cái áo, giá {đơn giá}đ/cái, giảm {x}%,
VAT 8%, phí ship 30.000đ. Cho biết tổng tiền phải thanh toán.
 
[Explanation]
- Dùng CoT để ghi công thức và kết quả từng bước, như vậy dễ kiểm tra lại.
- Chia thành nhiều bước để nếu lỗi ở bước nào thì dừng và báo đúng bước đó.
- Công thức: Tiền hàng = SL × giá; Sau giảm = Tiền hàng × (1 − x%);
  VAT = Sau giảm × 8%; Tổng = Sau giảm + VAT + ship.
 
[Action]
Bước A: Kiểm tra SL là số nguyên lớn hơn 0, giá lớn hơn 0, giảm từ 0 đến 100.
  - Nếu sai hoặc thiếu thì DỪNG và trả về: {"status":"error","step":"A","reason":"..."}
Bước B: Tính tiền hàng và tiền sau giảm, ghi công thức.
Bước C: Tính VAT rồi cộng phí ship, ghi công thức.
Bước D: Tự cộng ngược lại để kiểm tra, rồi xuất JSON:
  {"status":"ok","subtotal":..,"discount":..,"vat":..,"shipping":..,"total":..}
Nếu có bước bị lỗi thì không được đoán số, phải báo bước lỗi và hỏi lại khách.
 
[Tone]
Chính xác, lịch sự, rõ ràng. Làm tròn đến đồng.
 
[Extras]
- Đầu ra gồm phần giải thích từng bước và một khối JSON ở cuối.
- Các lỗi cần xử lý: SL = 0, SL âm, giảm giá 150%, thiếu đơn giá.
- Không được bịa số liệu, thiếu thông tin thì phải hỏi lại.
```
 
---
 
## BƯỚC 2: BẢNG SO SÁNH TRADE-OFF
 
| Tiêu chí | Giải pháp 1 (Đơn giản) | Giải pháp 2 (Chi tiết) |
|---|---|---|
| Độ dễ hiểu | Cao, viết nhanh | Trung bình, dài hơn nên khó đọc hơn |
| Tốn token | Ít | Nhiều hơn (khoảng 4-5 lần) |
| Chặn lỗi biên | Trung bình, dễ tính sai hoặc tự bịa số | Cực tốt, có bước kiểm tra và dừng sớm |

 
Nhận xét của em: Giải pháp 2 tốn nhiều token và viết lâu hơn, nhưng đổi lại được độ tin cậy cao hơn và kiểm soát lỗi tốt hơn.
 
---
 
## BƯỚC 3: LỰA CHỌN VÀ CHẠY THỬ
 
### Em chọn Giải pháp 2
 
Lý do em chọn:
 
1. Đề bài có nhắc đến chặn lỗi biên và xử lý đứt gãy, mà Giải pháp 2 làm được còn Giải pháp 1 thì không.
2. Bài toán tính tiền mà sai một bước là sai hết, nên tốn thêm token em thấy vẫn chấp nhận được.
3. Kết quả ra dạng JSON nên sau này muốn đưa vào hệ thống tự động cũng dễ.
### Chạy thử
 
Test 1: Dữ liệu đúng (3 áo, 200.000đ, giảm 10%)
 
- Bước A: Dữ liệu hợp lệ
- Bước B: Tiền hàng = 3 × 200.000 = 600.000; Giảm = 60.000; Sau giảm = 540.000
- Bước C: VAT = 540.000 × 8% = 43.200; Ship = 30.000
- Bước D: Tổng = 540.000 + 43.200 + 30.000 = 613.200; cộng ngược lại thấy đúng
```json
{"status":"ok","subtotal":600000,"discount":60000,"vat":43200,"shipping":30000,"total":613200}
```
 
Test 2: Dữ liệu sai (SL = 0, giảm 150%)
 
```json
{"status":"error","step":"A","reason":"Số lượng phải lớn hơn 0 và mức giảm phải từ 0 đến 100%. Vui lòng nhập lại."}
```
 
Hệ thống dừng ngay ở bước A, không tính tiếp và không tự bịa số.
 
So với Giải pháp 1 ở Test 2: prompt đơn giản có thể trả ra tổng tiền âm hoặc chỉ còn 30.000đ mà không báo lỗi gì, như vậy là sai.
 
