# Software Requirement Specification (SRS)
## Chức năng: Quản lý Sản phẩm (PIM) – Quản lý biến thể (Màu sắc, kích thước), tồn kho hiển thị realtime

---

## 1. Mô tả tổng quan (Description)
- Cung cấp chức năng quản lý biến thể sản phẩm (màu sắc, kích thước…) trong hệ thống PIM (Product Information Management).
- Cho phép tạo, cập nhật, xóa các biến thể và theo dõi tồn kho riêng cho từng biến thể.
- Hiển thị tồn kho theo thời gian thực (realtime) nhằm đảm bảo tính chính xác khi bán hàng và tránh overselling.

---

## 2. Luồng nghiệp vụ (User Workflow)

### 2.1. Quản lý biến thể
| Bước | Hành động người dùng | Phản hồi hệ thống |
|------|----------------------|-------------------|
| 1 | Truy cập trang | Hiển thị danh sách biến thể (màu, size, giá, tồn kho). |
| 2 | Nhấn "Thêm biến thể" | Hiển thị form nhập (màu sắc, kích thước, giá, tồn kho). |
| 3 | Nhập thông tin và lưu | Validate dữ liệu (bắt buộc, định dạng, trùng lặp). |
| 4 | Hệ thống xử lý | Tạo biến thể mới (kết hợp thuộc tính). |
| 5 | Thành công | Hiển thị biến thể mới trong danh sách. |
| 6 | Thất bại | Hiển thị lỗi cụ thể (trùng biến thể, thiếu dữ liệu…). |

### 2.2. Cập nhật tồn kho realtime
| Bước | Hành động người dùng | Phản hồi hệ thống |
|------|----------------------|-------------------|
| 1 | Admin thay đổi tồn kho biến thể | Gửi request cập nhật tồn kho. |
| 2 | Hệ thống xử lý | Cập nhật DB và phát sự kiện realtime (WebSocket). |
| 3 | Client (khách hàng) đang xem sản phẩm | Nhận dữ liệu mới ngay lập tức. |
| 4 | UI cập nhật | Hiển thị tồn kho mới (không cần reload trang). |

---

## 4. Ràng buộc kỹ thuật & Bảo mật (Technical Constraints)
- **Realtime:** Sử dụng WebSocket (Socket.IO) hoặc Firebase Realtime để cập nhật tồn kho.  
- **Đồng bộ dữ liệu:** Áp dụng cơ chế locking/transaction khi cập nhật tồn kho để tránh race condition.  
- **API:** RESTful (/variants, /products/{id}/variants) hoặc GraphQL.  
- **Hiệu năng:** Cache dữ liệu sản phẩm (Redis) nhưng phải invalidate khi tồn kho thay đổi.  
- **Phân quyền:** Chỉ Admin/Manager được phép chỉnh sửa biến thể và tồn kho.  

---

## 5. Trường hợp ngoại lệ & Xử lý lỗi (Edge Cases)
- **Tạo biến thể trùng (cùng màu + size):** Báo lỗi "Biến thể đã tồn tại".  
- **Tồn kho < 0:** Không cho phép lưu, hiển thị lỗi "Tồn kho không hợp lệ".  
- **Biến thể hết hàng (stock = 0):** Hiển thị "Hết hàng" trên UI, disable chọn mua.  

---

## 6. Giao diện (UI/UX)
- Hiển thị dạng bảng: **Màu | Size | Giá | Tồn kho | Hành động**  
- Cho phép chỉnh sửa inline (edit trực tiếp trong bảng).  
- Hiển thị trạng thái:  
  - Còn hàng  
  - Hết hàng  
- Realtime:  
  - Khi tồn kho thay đổi → số lượng update ngay.  
  - Hiển thị animation nhẹ (fade/update).  
- UX:  
  - Nút "Thêm biến thể" rõ ràng.  
  - Validate ngay khi nhập (real-time validation).  
  - Loading spinner khi cập nhật.  

---

## Phần 1: Mô hình hóa quy trình (Business Flow)
- Sơ đồ Use Case: 
  - Sơ đồ Use Case admin [Click here](ucAdmin.png)
  - Sơ đồ Use Case Customer [Click here](ucCustomer.png)
  - Sơ đồ Use Case Warehouse Staff [Click here](ucWarehouseStaff.png)
- Sơ đồ Activity: [Click here](Activity.png)  

---

## Phần 2: Đặc tả chức năng (Functional Requirements)
- Là một khách hàng, tôi muốn chọn màu sắc và kích thước của sản phẩm để mua đúng phiên bản mong muốn.  
- Là một khách hàng, tôi muốn xem được số lượng tồn kho còn lại theo từng biến thể để biết sản phẩm còn hay hết.  
- Là một khách hàng, tôi muốn thấy trạng thái "Hết hàng" ngay lập tức khi sản phẩm không còn để không mất thời gian đặt hàng.  
- Là một khách hàng, tôi muốn dữ liệu tồn kho được cập nhật realtime để tránh trường hợp đặt hàng nhưng bị hủy do hết hàng.  

---

## Phần 3: Đặc tả dữ liệu (Data Schema)

### Partner (Khách hàng)
| Trường           | Kiểu dữ liệu | Mô tả |
|------------------|--------------|-------|
| id               | string (PK)  | Khóa chính |
| name             | string       | Tên khách hàng |
| type             | string       | Loại khách hàng (individual/company) |
| tax_code (MST)   | string       | Mã số thuế (nếu là công ty) |
| phone            | string       | Số điện thoại |
| email            | string       | Email hợp lệ |
| shipping_address | string       | Địa chỉ giao hàng |
| created_at       | timestamp    | Thời gian tạo |

### Product (Sản phẩm)
| Trường       | Kiểu dữ liệu | Mô tả |
|--------------|--------------|-------|
| id           | string (PK)  | Khóa chính |
| name         | string       | Tên sản phẩm |
| sku          | string       | Mã sản phẩm (base SKU) |
| barcode      | string       | Mã vạch sản phẩm |
| description  | string       | Mô tả sản phẩm |
| base_price   | number       | Giá cơ bản |
| vat          | number       | Thuế VAT (%) |
| created_at   | timestamp    | Thời gian tạo |
| updated_at   | timestamp    | Thời gian cập nhật |

### Order (Đơn hàng)

| Trường       | Kiểu dữ liệu | Mô tả                          |
|--------------|--------------|--------------------------------|
| OrderID      | string (PK)  | Khóa chính                     |
| PartnerID    | string (FK)  | Khóa ngoại đến Partner         |
| order_number | string       | Mã đơn hàng                    |
| created_at   | timestamp    | Thời gian tạo đơn              |
| status       | string       | Draft, Confirmed, Cancelled    |
| total_amount | number       | Tổng tiền đơn hàng             |
| order_items  | JSON/array   | Danh sách chi tiết sản phẩm (SKU, số lượng, giá) |
| updated_at   | timestamp    | Thời gian cập nhật             |

**Constraint:** `(product_id + color + size)` phải unique (không trùng biến thể).

---
