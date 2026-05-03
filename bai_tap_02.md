## 🛠 Phân Tích Kỹ Thuật (Technical Details)

Dự án này được xây dựng tuân thủ nghiêm ngặt các tiêu chuẩn thiết kế cơ sở dữ liệu. Dưới đây là giải thích chi tiết cho từng bước triển khai:

## ⚙️ Phần 1: Thiết kế và Khởi tạo Cấu trúc Dữ liệu

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

---

## ⚙️ Phần 2: Xây dựng Function

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

---

## 🚀 Phần 3: Thủ Tục (Stored Procedure) 

Nếu như Hàm (Function) tập trung vào việc tính toán, thì **Stored Procedure (SP)** và **Trigger** chính là bộ não điều khiển toàn bộ hành động và sự tự động hóa của hệ thống. 

### 1. Stored Procedure (Thủ tục lưu trữ) - Sức mạnh thực thi
Trong SQL Server, các thủ tục hệ thống (như `sp_helptext` để xem mã nguồn hoặc `sp_spaceused` để kiểm tra dung lượng) giúp quản trị viên vận hành DB. Tuy nhiên, để giải quyết các bài toán kinh doanh thực tế, dự án đã xây dựng 3 SP tùy chỉnh nhằm bảo mật dữ liệu và tối ưu hiệu suất:

* **SP 1: Kiểm tra điều kiện logic (`sp_ThemMonAnMoi`)**
    * **Mục tiêu:** Đảm bảo tính toàn vẹn dữ liệu khi thêm món mới.
    * **Logic xử lý:** Trước khi chạy lệnh `INSERT`, SP sẽ quét bảng Thực Đơn. Nếu tên món đã tồn tại, nó sẽ in ra cảnh báo và từ chối thêm. Điều này giúp loại bỏ rác dữ liệu ngay từ tầng Database.

<img width="2560" height="1440" alt="image" src="https://github.com/user-attachments/assets/1f8060c5-c383-4b24-a795-668ea62e6cb6" />

* **SP 2: Sử dụng tham số OUTPUT (`sp_TinhTongChiTieuCuaBan`)**
    * **Mục tiêu:** Tính toán và trả về tổng số tiền mà một bàn ăn cụ thể đã chi tiêu.
    * **Logic xử lý:** SP nhận vào một `MaBanAn`, thực hiện phép tính tổng tiền từ các hóa đơn đã thanh toán của bàn đó, và "đẩy" kết quả ra ngoài thông qua tham số `OUTPUT`. Đây là cách tối ưu để ứng dụng phần mềm lấy dữ liệu báo cáo mà không cần viết lệnh truy vấn dài dòng.

<img width="2560" height="1440" alt="image" src="https://github.com/user-attachments/assets/211675f8-3a9a-4899-9adf-b8cc6be98967" />

* **SP 3: Truy vấn đa bảng (Result Set) (`sp_InBillThanhToan`)**
    * **Mục tiêu:** Đóng vai trò như một "máy in hóa đơn" hoàn chỉnh.
    * **Logic xử lý:** Khi truyền vào một `MaHoaDon`, SP sẽ sử dụng lệnh `INNER JOIN` để kết nối đồng thời 4 bảng (`BanAn`, `HoaDon`, `ChiTietHoaDon`, `ThucDon`). Nó trả về một tập kết quả đầy đủ bao gồm: tên bàn, ngày giờ in, danh sách món ăn, đơn giá và thành tiền của từng món để hiển thị lên màn hình thanh toán.

<img width="2560" height="1440" alt="image" src="https://github.com/user-attachments/assets/9ce46f35-ff26-48d5-829d-31579819a92c" />

---

## 🔄 Phần 4: Trigger và Xử lý Logic Nghiệp Vụ (Kiến thức 11)

### 1. Ứng dụng Thực tế: Tự động hóa cập nhật chéo (Bảng A -> Bảng B)
**Bài toán nghiệp vụ:** Trong vận hành nhà hàng, mỗi khi khách hàng gọi món, hủy món hoặc thay đổi số lượng món ăn (tương tác với bảng `[ChiTietHoaDon]`), tổng tiền của hóa đơn đó phải được cập nhật ngay lập tức. Nếu nhân viên thực hiện thao tác này bằng tay sẽ rất dễ dẫn đến sai sót tài chính.

**Giải pháp:** Xây dựng Trigger `trg_TinhTongTienTuDong` gắn trên bảng `[ChiTietHoaDon]`.
* **Cơ chế:** Bất cứ khi nào có lệnh `INSERT`, `UPDATE`, hoặc `DELETE` xảy ra trên chi tiết hóa đơn, Trigger sẽ tự động sử dụng bảng ảo `inserted` và `deleted` để xác định chính xác `MaHoaDon` đang bị tác động.
* **Thực thi:** Tính toán lại công thức `SUM(Số Lượng * Đơn Giá)` và ghi đè tự động vào cột `[TongTienHoaDon]` của bảng `[HoaDon]`. Đảm bảo dữ liệu chi tiết và tổng hợp luôn khớp nhau 100%.

<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/d17254b6-7e87-43b5-aef1-5ffb6e049e53" />

### 2. Thử nghiệm Hiện tượng "Vòng lặp vô hạn" (Circular Reference)
Để hiểu rõ hơn về cơ chế hoạt động ngầm của hệ thống, dự án đã thực hiện một thử nghiệm cố tình tạo ra xung đột Trigger giữa 2 bảng:
* **Trigger 1 (`trg_Loop_HoaDon`):** Gắn trên bảng `[HoaDon]`, hễ hóa đơn bị sửa -> tự động chạy sang tăng số lượng món ở `[ChiTietHoaDon]`.
* **Trigger 2 (`trg_TinhTongTienTuDong` - Hệ thống gốc):** Gắn trên bảng `[ChiTietHoaDon]`, hễ số lượng món bị sửa -> tự động chạy sang tính lại tiền ở `[HoaDon]`.
  
<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/a9e23f56-1a47-4182-afb5-1f51ebe3f0f7" />

**Hành động kích hoạt:** Thực hiện một lệnh `UPDATE` nhỏ trên bảng `[HoaDon]`.

**Kết quả hệ thống trả về:**

<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/e2d55752-4be2-412d-9535-ec26ce933008" />

### 3. Giải thích Lỗi và Nhận xét chuyên môn
Ngay khi kích hoạt lệnh `UPDATE`, hệ thống lập tức báo lỗi dòng chữ đỏ: `Maximum stored procedure, function, trigger, or view nesting level exceeded (limit 32).`

**Giải thích thông báo:**
SQL Server thông báo rẳng hệ thống đã vượt quá giới hạn lồng nhau (nesting level) tối đa là 32 cấp. Diễn biến thực tế như sau:
1. Sửa Hóa Đơn (Cấp 0) -> Đánh thức Trigger sửa Chi Tiết (Cấp 1).
2. Chi Tiết bị sửa -> Đánh thức Trigger cập nhật lại Hóa Đơn (Cấp 2).
3. Hóa Đơn vừa bị cập nhật -> Lại đánh thức Trigger sửa Chi Tiết (Cấp 3).

Quá trình "Ping-Pong" này diễn ra liên tục. Khi vòng lặp dội qua dội lại đến lần thứ 32, SQL Server nhận ra đây là một **Vòng lặp vô hạn (Infinite Loop)** gây treo RAM và CPU, nên hệ thống đã kích hoạt "cầu chì bảo vệ", ngắt lệnh và văng ra lỗi trên.

**Nhận xét cuối cùng:**
* Việc để 2 bảng liên tục nã lệnh cập nhật chéo vào nhau là một "tối kỵ" trong thiết kế Cơ sở dữ liệu.
* **Cách khắc phục thực tế:** Nếu bắt buộc phải có Trigger trên cả 2 bảng, ta phải sử dụng hàm `IF UPDATE(Tên_Cột)` ở bên trong thân Trigger. Hàm này giúp Trigger nhận biết chính xác cột nào vừa bị thay đổi, nếu đúng cột mục tiêu thì mới thực thi lệnh, từ đó bẻ gãy được vòng lặp vô hạn một cách triệt để.

> **Lưu ý thao tác:** Chạy lệnh `DROP TRIGGER` đối với các Trigger gây vòng lặp thử nghiệm ngay sau khi ghi nhận lỗi để đảm bảo Database tiếp tục hoạt động bình thường.

<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/d79af55a-0454-4956-832c-cb7a1190cca4" />

---

## 🔍 Phần 5: Cursor và Kỹ thuật Duyệt Dữ Liệu (Kiến thức 11)

### 1. Ứng dụng CURSOR để xử lý logic trên từng bản ghi
**Bài toán nghiệp vụ:** Quản lý nhà hàng chạy chương trình "Ngày hội Đồ Nướng". Yêu cầu hệ thống duyệt qua danh sách thực đơn và áp dụng mức giảm giá:
* Giảm 10% (0.1) cho các món Nướng cao cấp (Giá > 500.000đ).
* Giảm 5% (0.05) cho các món Nướng tiêu chuẩn (Giá từ 100.000đ - 500.000đ).
* Báo cáo (In ra màn hình thông báo `PRINT`) chi tiết từng món vừa được xét duyệt giảm giá.

**Cách giải quyết bằng CURSOR:**
Hệ thống khởi tạo một Cursor chứa danh sách các món Nướng. Tiếp theo, sử dụng vòng lặp `WHILE` kết hợp lệnh `FETCH NEXT` để trích xuất tuần tự từng món ăn vào biến tạm. Tại mỗi vòng lặp, logic `IF...ELSE` được thực thi để kiểm tra giá tiền, cập nhật `%` giảm giá tương ứng và in ra thông báo cho riêng món ăn đó.

**Kết quả thực thi vòng lặp Cursor:**

<img width="1920" height="1080" alt="cruseh" src="https://github.com/user-attachments/assets/dbdbc1c7-12c7-47cc-afde-cdbc764f8491" />

### 2. Giải pháp thay thế (Set-based) và So sánh hiệu năng
Bài toán cập nhật giá ở mục 1 hoàn toàn có thể được giải quyết bằng một câu lệnh SQL thuần túy mà không cần dùng đến vòng lặp CURSOR. 

**Giải pháp Set-based:** Sử dụng một câu lệnh `UPDATE` kết hợp biểu thức `CASE WHEN` để đánh giá điều kiện và cập nhật đồng loạt trên toàn bộ bảng `[ThucDon]`.

**So sánh tốc độ (Performance Comparison):**
Bằng cách sử dụng lệnh `SET STATISTICS TIME ON` để đo lường, kết quả trả về cho thấy sự chênh lệch rõ rệt về thời gian xử lý của CPU:

<img width="1920" height="1080" alt="ko cru" src="https://github.com/user-attachments/assets/7f4345c4-90a5-46d0-90a7-19e7a856a5d9" />

**Nhận xét chuyên môn:**
* Lệnh truy vấn dạng tập hợp (Set-based) xử lý **nhanh hơn gấp nhiều lần**. SQL Server gom tất cả dữ liệu thỏa mãn điều kiện, khóa bảng một lần duy nhất, tính toán và ghi xuống đĩa cực nhanh.
* Ngược lại, CURSOR xử lý theo cơ chế **RBAR (Row By Agonizing Row - Xử lý từng dòng một)** gây lãng phí tài nguyên CPU và RAM nghiêm trọng do phải chuyển đổi ngữ cảnh liên tục. 

### 3. Bài toán "Giới hạn của SQL" - Nơi CURSOR là giải pháp duy nhất
Nếu Set-based ưu việt như vậy, tại sao CURSOR vẫn tồn tại? Lý do là SQL thông thường **không thể thực thi một thủ tục (Stored Procedure) cho từng dòng dữ liệu lấy ra từ một tập kết quả**.

**Tình huống nghiệp vụ:** Cuối ca làm việc, quản lý yêu cầu hệ thống tự động kết xuất/in lại toàn bộ chi tiết các hóa đơn *đã thanh toán trong ngày hôm nay* để kẹp vào hồ sơ kế toán (Sử dụng lại Stored Procedure `sp_InBillThanhToan` đã viết ở Phần 3).

**Tại sao SQL Set-based "bó tay"?**
Chúng ta KHÔNG THỂ viết cú pháp truyền một tập hợp trực tiếp vào lệnh EXEC (Ví dụ: `EXEC sp_InBillThanhToan (SELECT MaHoaDon...)` là sai cú pháp).

**Vai trò của CURSOR:**
Trong trường hợp này, **CURSOR là giải pháp duy nhất và bắt buộc**. Nó giúp trích xuất riêng rẽ từng `MaHoaDon` từ danh sách, nạp vào biến, và tuần tự kích hoạt thủ tục in hóa đơn nhiều lần cho đến khi kết thúc danh sách.

**Kết quả hệ thống tự động in hàng loạt Hóa Đơn bằng Cursor:**

<img width="1920" height="1080" alt="bắt buộc cru" src="https://github.com/user-attachments/assets/2fcfd561-5528-4513-830e-9563d65a8242" />
