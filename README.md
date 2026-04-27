## 🛠 Phân Tích Kỹ Thuật (Technical Details)

Dự án này được xây dựng tuân thủ nghiêm ngặt các tiêu chuẩn thiết kế cơ sở dữ liệu. Dưới đây là giải thích chi tiết cho từng bước triển khai:

### 1. Khởi tạo Database và Cú pháp
* **Tên Database:** Khởi tạo theo đúng định dạng `[Tên dự án]_[MaSV]`. Cụ thể ở đây là `[QuanLyNhaHang_K235480106100]`.
 <img width="1920" height="1080" alt="Screenshot 2026-04-27 230402" src="https://github.com/user-attachments/assets/d344475e-73ea-410d-bb1c-9021dc8a71e0" />

* **Cú pháp an toàn:** Toàn bộ script DDL (Data Definition Language) đều sử dụng cặp ngoặc vuông `[ ]` để bọc tên bảng và tên trường (ví dụ: `[BanAn]`, `[TongTienHoaDon]`). Điều này đảm bảo an toàn, tránh xung đột với các từ khóa mặc định của SQL Server.
* **Quy tắc đặt tên:** Áp dụng triệt để quy tắc **Bướu Lạc Đà (CamelCase)**, viết hoa chữ cái đầu mỗi từ và **không viết tắt** để đảm bảo tính tự giải thích của code (Ví dụ: `NgayLapHoaDon`, `PhanTramGiamGia`, `DonGiaTaiThoiDiemBan`).


### 2. Thiết kế 4 bảng thực thể và Quan hệ (Relationships)
Hệ thống đã được chuẩn hóa (Normalization) để giải quyết triệt để bài toán: *Một khách hàng có thể gọi nhiều món ăn trong cùng một hóa đơn*. Hệ thống gồm 4 bảng liên kết chặt chẽ theo mô hình quan hệ:

* `[BanAn]` (1) ─── (N) `[HoaDon]`
* `[HoaDon]` (1) ─── (N) `[ChiTietHoaDon]` (N) ─── (1) `[ThucDon]`

<img width="1920" height="1080" alt="Screenshot 2026-04-27 231550" src="https://github.com/user-attachments/assets/2b780bc6-dae0-46bd-819d-5c970299ac63" />


Sự xuất hiện của bảng trung gian `[ChiTietHoaDon]` giúp tách biệt thông tin chung của hóa đơn và thông tin chi tiết từng món được gọi, giải quyết triệt để mối quan hệ nhiều - nhiều giữa Hóa Đơn và Thực Đơn.

### 3. Đa dạng hóa Kiểu dữ liệu (Data Types)
Để tối ưu hóa không gian lưu trữ và độ chính xác, dự án sử dụng đa dạng các kiểu dữ liệu thực tế:
* **Số nguyên:** `INT` dùng cho các khóa chính tự tăng (IDENTITY), số lượng (`SoLuongMonAn`), và sức chứa (`SucChuaBanAn`).
* **Số thực:** `FLOAT` dùng cho phần trăm (`PhanTramGiamGia`).
* **Tiền tệ:** `MONEY` chuyên dụng để lưu trữ giá trị tiền tệ, tránh sai số làm tròn (`GiaBanMonAn`, `DonGiaTaiThoiDiemBan`, `TongTienHoaDon`).
* **Chuỗi Unicode:** `NVARCHAR` lưu trữ văn bản có dấu tiếng Việt (`TenMonAn`, `TrangThaiThanhToan`, `KhuVucBanAn`).
* **Ngày tháng:** `DATETIME` lưu vết thời gian giao dịch (`NgayLapHoaDon`).
* **Boolean:** `BIT` dùng cho các cờ trạng thái Đúng/Sai (`TrangThaiPhucVu`).

### 4. Giải thích Ràng buộc (Constraints)
Hệ thống sử dụng đầy đủ các ràng buộc để đảm bảo tính toàn vẹn dữ liệu:
* **Khóa chính (Primary Key - PK):** `[PK_BanAn]`, `[PK_ThucDon]`, `[PK_HoaDon]` được gán trên các trường ID để định danh duy nhất. Riêng bảng `[ChiTietHoaDon]` sử dụng khóa chính hỗn hợp (Composite Key) gồm `(MaHoaDon, MaMonAn)` để tránh việc một hóa đơn nhập trùng lặp thành hai dòng cho cùng một món.
* **Khóa ngoại (Foreign Key - FK):** Ràng buộc dữ liệu phải tồn tại. Ví dụ: `[FK_HoaDon_BanAn]` đảm bảo hóa đơn được tạo phải gắn với một bàn ăn có thật trong hệ thống.
* **Ràng buộc cứng (Check Constraint - CK):** Đóng vai trò là "tường lửa" chặn dữ liệu rác ở cấp độ Database:
  * `[CK_SucChuaBanAn] CHECK ([SucChuaBanAn] > 0)`: Bàn ăn bắt buộc phải có ít nhất 1 chỗ ngồi.
  * `[CK_GiaBanMonAn] CHECK ([GiaBanMonAn] >= 0)`: Giá bán của món ăn không được phép là số âm.
  * `[CK_PhanTramGiamGia] CHECK ([PhanTramGiamGia] >= 0 AND [PhanTramGiamGia] <= 1)`: Tỉ lệ giảm giá phải nằm trong khoảng hợp lý từ 0% đến 100%.
  * `[CK_SoLuongMonAn] CHECK ([SoLuongMonAn] > 0)`: Khi khách gọi món, số lượng tối thiểu phải là 1.
