# Giải Pháp Prompting: Tự Động Hóa Xử Lý Đơn Hàng Tại Cửa Hàng

## Phần 1: Thiết kế giải pháp (I/O)

**1. Bối cảnh áp dụng:**
Cửa hàng bán lẻ nhận hàng trăm tin nhắn đặt hàng mỗi ngày qua Facebook/Zalo với đủ kiểu viết tắt, thiếu thông tin hoặc sai cú pháp. Giải pháp này giúp nhân viên tự động trích xuất thông tin khách hàng và đơn hàng từ tin nhắn thô thành dữ liệu có cấu trúc để nhập nhanh vào hệ thống.

**2. Xác định đầu vào (Input):**
* Đoạn văn bản tin nhắn đặt hàng thô của khách hàng (chứa tên, số điện thoại, địa chỉ, tên sản phẩm, số lượng, ghi chú).
* Danh sách sản phẩm hiện có của cửa hàng (để AI đối chiếu tên và giá).

**3. Cấu trúc đầu ra mong muốn (Output):**
* Ép định dạng **JSON** chuẩn để dễ dàng đưa vào phần mềm quản lý hoặc chuyển thành bảng (Table).
* Các trường dữ liệu bắt buộc: `ten_khach_hang`, `so_dien_thoai`, `dia_chi`, `danh_sach_san_pham` (tên, số lượng), `trang_thai_du_lieu` (Hợp lệ / Thiếu thông tin), và `ghi_chu_loi`.

**4. Ràng buộc & Xử lý bẫy dữ liệu:**
* **Thiếu thông tin:** Nếu khách quên ghi SĐT hoặc địa chỉ, AI không được tự bịa ra mà phải điền `"TRỐNG"` và báo lỗi ở mục `ghi_chu_loi`.
* **Sai định dạng SĐT:** Nếu SĐT không đủ 10 số hoặc chứa chữ cái, phải đánh dấu cảnh báo để nhân viên gọi xác nhận lại.
* **Sản phẩm không tồn tại:** Nếu khách đặt món không có trong menu, AI phải ghi nhận vào phần lỗi thay vì tự ý đổi sang món khác.

---

## Phần 2: Viết Prompt hoàn chỉnh và Chạy thử

### 1. Prompt hoàn chỉnh (Áp dụng khung CREATE + Few-shot + Ép JSON)

```text
[C - Character / Vai trò]
Bạn là một chuyên viên xử lý dữ liệu đơn hàng tự động cho cửa hàng trà sữa "Mùa Thu". Nhiệm vụ của bạn là đọc tin nhắn đặt hàng của khách và trích xuất thông tin chính xác tuyệt đối.

[R - Request / Yêu cầu]
Hãy phân tích tin nhắn khách hàng bên dưới, đối chiếu với Menu của quán và xuất ra kết quả dưới dạng JSON.

Menu hiện tại của quán:
- Trà sữa trân châu đường đen (Size M, L)
- Trà đào cam sả (Size M, L)
- Hồng trà macchiato (Size M, L)

[E - Examples / Ví dụ mẫu (Few-shot)]
Ví dụ 1 (Đơn chuẩn):
Input: "Cho chị 2 trà đào cam sả size L ít đá giao tới 15 Lê Lợi nhé. SĐT 0901234567, chị Lan."
Output:
{
  "ten_khach_hang": "Chị Lan",
  "so_dien_thoai": "0901234567",
  "dia_chi": "15 Lê Lợi",
  "danh_sach_san_pham": [
    {"ten_mon": "Trà đào cam sả", "size": "L", "so_luong": 2, "yeu_cau_them": "Ít đá"}
  ],
  "trang_thai_du_lieu": "Hợp lệ",
  "ghi_chu_loi": "Không có"
}

Ví dụ 2 (Đơn bị thiếu và sai dữ liệu):
Input: "Ship cho anh 1 trà sữa matcha qua 20 Nguyễn Huệ nha, số anh 098877"
Output:
{
  "ten_khach_hang": "TRỐNG",
  "so_dien_thoai": "098877",
  "dia_chi": "20 Nguyễn Huệ",
  "danh_sach_san_pham": [
    {"ten_mon": "Trà sữa matcha", "size": "TRỐNG", "so_luong": 1, "yeu_cau_them": "Không có"}
  ],
  "trang_thai_du_lieu": "Thiếu thông tin",
  "ghi_chu_loi": "Thiếu tên khách; SĐT không đủ 10 số; Món 'Trà sữa matcha' không có trong menu; Thiếu size."
}

[A - Adjustments / Ràng buộc & Xử lý lỗi]
1. Tuyệt đối không tự suy diễn hoặc bịa đặt thông tin khách không cung cấp. Nếu thiếu, bắt buộc điền chuỗi "TRỐNG".
2. Kiểm tra số điện thoại: Phải là dãy 10 chữ số bắt đầu bằng số 0. Nếu sai, ghi rõ vào `ghi_chu_loi`.
3. Kiểm tra món và size: Nếu khách không ghi size hoặc gọi món ngoài Menu, phải báo lỗi vào `ghi_chu_loi`.

[T - Type of Output / Định dạng đầu ra]
Chỉ trả về duy nhất một khối mã JSON hợp lệ. Không giải thích, không chào hỏi thêm.

[E - Extras / Dữ liệu đầu vào cần xử lý]
Input thực tế: "Em ơi giao cho bạn Minh ở ký túc xá khu B 3 ly hồng trà macchiato size M nha, gọi số 0912345678, nhớ lấy nhiều trân châu nhé."
```

### 2. Kết quả phản hồi thực tế từ AI: 

```
{
  "ten_khach_hang": "Bạn Minh",
  "so_dien_thoai": "0912345678",
  "dia_chi": "Ký túc xá khu B",
  "danh_sach_san_pham": [
    {
      "ten_mon": "Hồng trà macchiato",
      "size": "M",
      "so_luong": 3,
      "yeu_cau_them": "Nhiều trân châu"
    }
  ],
  "trang_thai_du_lieu": "Hợp lệ",
  "ghi_chu_loi": "Không có"
}
```
