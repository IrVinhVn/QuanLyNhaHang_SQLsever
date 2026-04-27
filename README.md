## 🛠 Phân Tích Kỹ Thuật (Technical Details)

Dự án này được xây dựng tuân thủ nghiêm ngặt các tiêu chuẩn thiết kế cơ sở dữ liệu. Dưới đây là giải thích chi tiết cho từng bước triển khai:

### 1. Khởi tạo Database và Cú pháp
* **Tên Database:** Khởi tạo theo đúng định dạng `[Tên dự án]_[MaSV]`. Cụ thể ở đây là `[QuanLyNhaHang_K123456789]`.
* **Cú pháp an toàn:** Toàn bộ script DDL (Data Definition Language) đều sử dụng cặp ngoặc vuông `[ ]` để bọc tên bảng và tên trường (ví dụ: `[MonAn]`, `[ThanhTien]`). Điều này đảm bảo an toàn, tránh xung đột với các từ khóa mặc định của SQL Server.
* **Quy tắc đặt tên:** Áp dụng triệt để quy tắc **Bướu Lạc Đà (CamelCase)**, viết hoa chữ cái đầu mỗi từ và **không viết tắt** để đảm bảo tính tự giải thích của code (Ví dụ: `NgayCapNhat`, `TiLeGiamGia`, `DonGiaLucBan`).

<img width="1920" height="1080" alt="Screenshot 2026-04-27 212620" src="https://github.com/user-attachments/assets/d04c2304-b1f5-4428-8c81-fe4d5baaf07a" />

### 2. Thiết kế 4 bảng thực thể và Quan hệ (Relationships)
Hệ thống đã được chuẩn hóa (Normalization) để giải quyết triệt để bài toán: *Một khách hàng có thể gọi nhiều món ăn trong cùng một hóa đơn*. Hệ thống gồm 4 bảng liên kết chặt chẽ theo mô hình quan hệ:

* `[NhomMon]` (1) ─── (N) `[MonAn]`
* `[HoaDon]` (1) ─── (N) `[ChiTietHoaDon]` (N) ─── (1) `[MonAn]`

<img width="1920" height="1080" alt="Screenshot 2026-04-27 213817" src="https://github.com/user-attachments/assets/29aa1886-f042-4683-9032-46afac97cb44" />
<br>

Sự xuất hiện của bảng trung gian `[ChiTietHoaDon]` giúp tách biệt thông tin chung của hóa đơn và thông tin chi tiết từng món được gọi.

<img width="1920" height="1080" alt="Screenshot 2026-04-27 222901" src="https://github.com/user-attachments/assets/bf87f867-4390-49a3-9162-12dee5bb0bf1" />

### 3. Đa dạng hóa Kiểu dữ liệu (Data Types)
Để tối ưu hóa không gian lưu trữ và độ chính xác, dự án sử dụng đa dạng các kiểu dữ liệu thực tế:
* **Số nguyên:** `INT` dùng cho các khóa chính tự tăng (IDENTITY) và số lượng (`SoLuong`).
* **Số thực:** `FLOAT` dùng cho tỉ lệ phần trăm (`TiLeGiamGia`).
* **Tiền tệ:** `MONEY` chuyên dụng để lưu trữ giá trị tiền tệ, tránh sai số làm tròn (`DonGia`, `DonGiaLucBan`).
* **Chuỗi Unicode:** `NVARCHAR` lưu trữ văn bản có dấu tiếng Việt (`TenMon`, `GhiChuKhachHang`).
* **Ngày tháng:** `DATETIME` lưu vết thời gian giao dịch (`NgayTao`, `NgayCapNhat`).
* **Boolean:** `BIT` dùng cho các cờ trạng thái Đúng/Sai (`ConKinhDoanh`, `TrangThaiThanhToan`).

### 4. Giải thích Ràng buộc (Constraints)
Hệ thống sử dụng đầy đủ các ràng buộc để đảm bảo tính toàn vẹn dữ liệu:
* **Khóa chính (Primary Key - PK):** `[PK_NhomMon]`, `[PK_MonAn]`, `[PK_HoaDon]` được gán trên các trường ID để định danh duy nhất. Riêng bảng `[ChiTietHoaDon]` sử dụng khóa chính hỗn hợp (Composite Key) gồm `(MaHoaDon, MaMon)` để tránh nhập trùng lặp món.
* **Khóa ngoại (Foreign Key - FK):** Ràng buộc dữ liệu phải tồn tại. Ví dụ: `[FK_ChiTietHoaDon_MonAn]` đảm bảo món ăn được thêm vào hóa đơn phải có thật trong bảng `[MonAn]`.
* **Ràng buộc cứng (Check Constraint - CK):** Đóng vai trò là "tường lửa" chặn dữ liệu rác ở cấp độ Database:
  * `[CK_DonGia] CHECK ([DonGia] > 0)`: Đơn giá không được phép là số âm hoặc bằng 0.
  * `[CK_GiamGia] CHECK ([TiLeGiamGia] >= 0 AND [TiLeGiamGia] <= 1)`: Tỉ lệ giảm giá phải nằm trong khoảng từ 0% đến 100%.
  * `[CK_SoLuong_ToiThieu] CHECK ([SoLuong] >= 1)`: Khi gọi món, khách hàng phải gọi ít nhất 1 phần.

### 5. Điểm tối ưu kỹ thuật (Computed Column)
Trường `[ThanhTien]` trong bảng chi tiết hóa đơn được thiết lập tự động tính toán bằng công thức `([SoLuong] * [DonGiaLucBan]) PERSISTED`. Thuộc tính `PERSISTED` giúp SQL Server lưu trữ cứng kết quả xuống ổ đĩa, giúp các truy vấn báo cáo doanh thu nhanh hơn đáng kể do không cần tính toán lại phép nhân mỗi khi load dữ liệu.
