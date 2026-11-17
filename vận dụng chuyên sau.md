


# Thiết kế giải pháp Prompting: Tự động xử lý đơn hàng nhắn tin của cửa hàng

## Bước 1: Thiết kế giải pháp (I/O)

**Bài toán:** Khách nhắn tin tự do để đặt hàng. Cần chuyển thành đơn chuẩn, kiểm tra tồn kho, tính tiền và soạn tin xác nhận.

### Input
- Tin nhắn thô của khách.
- Bảng giá và tồn kho: Áo thun 150.000đ (còn 10), Quần jean 350.000đ (còn 5), Nón lưỡi trai 80.000đ (còn 0).
- Quy tắc: tổng tiền hàng từ 500.000đ giảm 10%, phí ship cố định 30.000đ.

### Output
JSON đơn hàng đã tính tiền và tin nhắn xác nhận gửi khách.

### Prompt Chaining (3 bước, output bước trước là input bước sau)

| Bước | Nhiệm vụ | Output |
|---|---|---|
| P1 | Trích xuất thông tin | JSON khách + món |
| P2 | Kiểm tra và tính tiền (Chain-of-Thought) | JSON đơn đã tính |
| P3 | Soạn tin xác nhận | Tin nhắn cho khách |

### Xử lý lỗi (đứt gãy)
- Thiếu SĐT hoặc địa chỉ: đặt `null`, `status: "missing_info"`.
- Món không có trong bảng giá hoặc hết hàng: đưa vào `rejected_items` kèm lý do.
- Tin nhắn trống, ký tự lạ, số lượng không hợp lệ: trả `Invalid`.
- P1 lỗi thì dừng chuỗi, không chạy P2 và P3.

## Bước 2: Viết prompt và chạy thử

### Prompt 1: Trích xuất thông tin

```
Bạn là nhân viên nhập liệu của cửa hàng thời trang.
Nhiệm vụ: trích xuất thông tin đơn hàng từ tin nhắn khách.
Quy tắc:
- name: viết hoa chữ cái đầu. phone: bỏ khoảng trắng, đổi sang +84.
- items: danh sách {product, qty}. qty là số nguyên.
- Thông tin không có trong tin nhắn: đặt null, không tự điền.
- Nếu tin nhắn trống hoặc không đọc được: trả {"status":"Invalid"}.
Chỉ trả về JSON thuần, không giải thích.
Cấu trúc: {"name":"","phone":"","address":"","items":[{"product":"","qty":0}],"status":"ok|missing_info|Invalid"}

Tin nhắn: {TIN_NHAN}
```

### Prompt 2: Kiểm tra và tính tiền (Chain-of-Thought)

```
Bạn là kế toán cửa hàng. Dữ liệu đơn: {JSON_P1}
Bảng giá/tồn kho: Áo thun 150000 (còn 10); Quần jean 350000 (còn 5); Nón lưỡi trai 80000 (còn 0).
Quy tắc: tổng tiền hàng >= 500000 giảm 10%; ship cố định 30000.
Hãy suy luận TỪNG BƯỚC trong thẻ <reasoning>:
1. Kiểm tra từng món có trong bảng giá không.
2. Kiểm tra qty <= tồn kho; món không đạt đưa vào rejected_items kèm lý do.
3. Tính thành tiền từng món, tổng tiền hàng (chỉ món hợp lệ).
4. Xét giảm giá, cộng ship, tính tổng.
5. Nếu status của đơn không phải "ok" hoặc có món bị loại: status = "needs_confirmation".
Sau đó trả JSON thuần trong thẻ <result>:
{"valid_items":[{"product":"","qty":0,"unit_price":0,"amount":0}],"rejected_items":[{"product":"","reason":""}],"subtotal":0,"discount":0,"shipping":0,"total":0,"status":""}
```

### Prompt 3: Soạn tin xác nhận

```
Bạn là nhân viên chăm sóc khách hàng, giọng lịch sự, ngắn gọn.
Dựa vào đơn đã tính: {JSON_P2} và tên khách: {NAME}.
Viết tin nhắn tiếng Việt dưới 80 từ: liệt kê món hợp lệ, tổng tiền; nếu có món bị loại hoặc status khác "ok" thì nêu rõ lý do và hỏi khách có đồng ý đặt phần còn lại không.
Không bịa thông tin ngoài dữ liệu.
```

### Chạy thử

**Tin nhắn khách:**
> Chào shop, mình là lê minh tuấn, SĐT 0988 777 666. Mình lấy 2 áo thun, 1 quần jean và 1 nón lưỡi trai nhé. Giao về 12 Nguyễn Trãi, Hà Nội.

**Kết quả P1:**
```json
{"name":"Lê Minh Tuấn","phone":"+84988777666","address":"12 Nguyễn Trãi, Hà Nội","items":[{"product":"Áo thun","qty":2},{"product":"Quần jean","qty":1},{"product":"Nón lưỡi trai","qty":1}],"status":"ok"}
```

**Kết quả P2:**
```
<reasoning>
1. Cả 3 món đều có trong bảng giá.
2. Áo thun 2 <= 10 đạt; Quần jean 1 <= 5 đạt; Nón lưỡi trai còn 0 -> loại.
3. Áo thun 2 x 150000 = 300000; Quần jean 350000; tổng hàng = 650000.
4. 650000 >= 500000 -> giảm 10% = 65000; sau giảm 585000; + ship 30000 = 615000.
5. Có món bị loại -> needs_confirmation.
</reasoning>
<result>
{"valid_items":[{"product":"Áo thun","qty":2,"unit_price":150000,"amount":300000},{"product":"Quần jean","qty":1,"unit_price":350000,"amount":350000}],"rejected_items":[{"product":"Nón lưỡi trai","reason":"Hết hàng"}],"subtotal":650000,"discount":65000,"shipping":30000,"total":615000,"status":"needs_confirmation"}
</result>
```

**Kết quả P3:**
> Chào anh Tuấn, shop xác nhận đơn gồm 2 áo thun và 1 quần jean, đã giảm 10% và cộng phí ship, tổng thanh toán 615.000đ. Rất tiếc nón lưỡi trai hiện đã hết hàng. Anh có đồng ý đặt phần còn lại không ạ?

### Test trường hợp lỗi
- **Thiếu thông tin:** "Mình là Hoa, lấy 1 áo thun" → P1 trả `phone: null`, `status: "missing_info"`.
- **Dữ liệu rác:** "@@## ???" → P1 trả `{"status":"Invalid"}`, dừng chuỗi.
