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

## ⚙️ Phần 2: Xử Lý Nghiệp Vụ bằng Hàm (User-Defined Functions - UDFs)

Trong SQL Server, các hàm có sẵn (Built-in Functions như `SUM`, `COUNT`, `COALESCE`) giải quyết rất tốt các bài toán tính toán cơ bản. Tuy nhiên, hệ thống mặc định không thể hiểu được các "quy luật kinh doanh" (Business Logic) đặc thù của một nhà hàng. 

Để giải quyết vấn đề này, dự án đã xây dựng hệ thống **Hàm tự định nghĩa (UDFs)**. Việc này giúp đóng gói các logic tính toán phức tạp thành các module độc lập, giúp tái sử dụng dễ dàng ở nhiều nơi (Báo cáo, Hóa đơn, Ứng dụng) mà không cần viết lại quy trình, đồng thời che giấu sự phức tạp của cơ sở dữ liệu bên dưới.

Dự án áp dụng 3 cấp độ Function để xử lý 3 loại bài toán khác nhau:
<img width="2560" height="1440" alt="image" src="https://github.com/user-attachments/assets/bf8d8e0c-b794-40b2-9a3a-ff97fd623eca" />

### 1. Scalar Function (Hàm Vô Hướng) - Xử lý tính toán đơn lẻ
* **Tên hàm:** `fn_TinhTongTienHoaDon`
* **Bài toán thực tế:** Khi khách hàng yêu cầu thanh toán, thu ngân cần biết chính xác tổng số tiền của hóa đơn đó là bao nhiêu.
* **Cách giải quyết:** Hàm nhận đầu vào là một Mã Hóa Đơn. Nó sẽ tự động quét toàn bộ chi tiết món ăn mà khách đã gọi, nhân số lượng với **đơn giá tại thời điểm bán** (để tránh sai lệch nếu giá món ăn thay đổi trong tương lai), sau đó cộng dồn lại. Hàm cũng được thiết kế an toàn để trả về giá trị `0` nếu hóa đơn đó chưa có món nào, ngăn chặn triệt để lỗi hệ thống do giá trị `NULL`.
<img width="2560" height="1440" alt="image" src="https://github.com/user-attachments/assets/59961ed3-6985-4df2-a448-b5b0c2636b35" />

### 2. Inline Table-Valued Function (ITVF) - Bộ lọc dữ liệu tốc độ cao
* **Tên hàm:** `fn_TimMonAnTheoTaiChinh`
* **Bài toán thực tế:** Nhân viên order thường xuyên nhận được yêu cầu từ khách: *"Cho anh xem các món Nướng có giá dưới 150.000đ"*.
* **Cách giải quyết:** ITVF hoạt động như một "khung nhìn linh hoạt" (Parameterized View). Hàm nhận vào hai biến số: Loại món và Mức giá tối đa. Nó lập tức quét qua bảng Thực Đơn, loại bỏ các món đang tạm hết hàng và trả về một danh sách (bảng) các món ăn phù hợp. Điểm mạnh của ITVF là nó được SQL Server tối ưu hóa tốc độ truy xuất cực nhanh, rất phù hợp cho các tính năng tìm kiếm trên ứng dụng.
<img width="2560" height="1440" alt="image" src="https://github.com/user-attachments/assets/dfd4c923-48c9-4675-b532-59be9edeac39" />

### 3. Multi-statement Table-Valued Function (MSTVF) - Thống kê và Xếp loại KPI
* **Tên hàm:** `fn_DanhGiaHieuSuatBanAn`
* **Bài toán thực tế:** Cuối mỗi tháng, người quản lý cần một báo cáo đánh giá xem vị trí bàn nào (ví dụ: Bàn VIP, Bàn Sân Vườn) mang lại doanh thu tốt nhất để có chiến lược sắp xếp khách hàng.
* **Cách giải quyết:** Hàm này thực hiện một chuỗi xử lý phức tạp qua nhiều bước:
  1. Tạo ra một bảng báo cáo ảo trong bộ nhớ.
  2. Gom nhóm toàn bộ hóa đơn trong tháng/năm được chỉ định. Tái sử dụng lại hàm `fn_TinhTongTienHoaDon` (ở mục 1) để tính tổng doanh thu cho từng bàn.
  3. Áp dụng quy tắc xếp loại tự động: Bàn có doanh thu trên 1.000.000đ được gắn mác "Năng Suất Cao", bàn có khách nhưng chưa đạt mức này là "Tiêu Chuẩn", và nhận diện cả những bàn "Không Có Khách".
  4. Trả về bảng báo cáo KPI hoàn chỉnh.
<img width="2560" height="1440" alt="image" src="https://github.com/user-attachments/assets/f8acb41e-47e9-4770-9029-cc6a7793cdbc" />
