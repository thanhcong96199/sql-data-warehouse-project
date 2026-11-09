# Tìm mối quan hệ giữa các bảng (Bước 3.1: Khám phá dữ liệu)

Tài liệu này ghi lại **cách tìm ra các bảng nối với nhau qua cột nào**, kể cả khi dữ liệu giữa các hệ thống nguồn không khớp trực tiếp, cùng **kết quả áp dụng** cho 6 bảng Bronze của project.

Tất cả câu SQL viết cho SQL Server và chạy trên schema `bronze`.

---

## 1. Mục tiêu

Trước khi làm sạch dữ liệu ở tầng Silver, cần trả lời được 3 câu hỏi:

1. Mỗi bảng chứa thông tin về **cái gì** (khách hàng, sản phẩm, giao dịch…)?
2. Cột nào là **khoá**, tức là nhận diện duy nhất một dòng?
3. Các bảng **nối với nhau** qua cột nào? Giá trị có khớp trực tiếp không? Nếu không thì cần biến đổi gì?

Câu trả lời cho câu 3 chính là **danh sách việc cần làm** ở bước làm sạch: mỗi phép biến đổi sẽ trở thành một đoạn code ở tầng Silver.

---

## 2. Quy trình

```
① Đoán cặp cột  →  ② Nhìn mẫu  →  ③ Soi hình dạng  →  ④ Tìm chéo
       →  Đặt giả thuyết biến đổi  →  ⑤ Đo tỉ lệ khớp  →  Ghi kết quả
```

Áp dụng lần lượt cho **từng cặp bảng**. Nếu bước ⑤ cho tỉ lệ khớp thấp, quay lại ②–④ để tìm giả thuyết khác.

### ① Đoán cặp cột dựa trên tên và ý nghĩa

Liệt kê các cặp cột **có khả năng** nối với nhau:
- Tên giống nhau hoặc cùng phần đuôi: `_id`, `_key`, `cid`…
- Cùng mô tả một thực thể: cùng là khách hàng, cùng là sản phẩm.

Bước này chỉ cho ra **giả thuyết**, chưa phải kết luận.

### ② Đặt mẫu dữ liệu cạnh nhau

```sql
SELECT TOP 10 sls_prd_key FROM bronze.crm_sales_details;
SELECT TOP 10 prd_key     FROM bronze.crm_prd_info;
```

Đặt hai kết quả cạnh nhau và tự hỏi: **phần nào giống nhau, phần nào thừa ra, phần nào bị đổi?** Chú ý tới các ký tự `-`, `_`, khoảng trắng, tiền tố và độ dài chuỗi.

### ③ Soi "hình dạng" của toàn bộ cột

Chỉ nhìn 10 dòng mẫu dễ bỏ sót các dòng **không theo quy luật chung**. Hãy nhóm cả cột lại để xem toàn cảnh:

```sql
-- Các độ dài khác nhau và số dòng của mỗi độ dài
SELECT LEN(cid) AS do_dai, COUNT(*) AS so_dong
FROM bronze.erp_cust_az12
GROUP BY LEN(cid);

-- Các tiền tố khác nhau và số dòng của mỗi tiền tố
SELECT LEFT(cid, 3) AS tien_to, COUNT(*) AS so_dong
FROM bronze.erp_cust_az12
GROUP BY LEFT(cid, 3)
ORDER BY so_dong DESC;
```

Nếu cột có **nhiều độ dài** hoặc **nhiều kiểu tiền tố**, tức là không phải mọi dòng đều cùng định dạng. Khi đó code làm sạch phải xử lý **từng trường hợp riêng**.

### ④ Lấy 1 giá trị đi tìm ở bảng kia

Đây là kỹ thuật hiệu quả nhất khi hai bên **khác định dạng**. Lấy một giá trị cụ thể ở bảng A, rồi tìm nó **ở bất kỳ vị trí nào** trong bảng B:

```sql
SELECT prd_key
FROM bronze.crm_prd_info
WHERE prd_key LIKE '%BK-R93R-62%';     -- giá trị lấy từ crm_sales_details
-- Kết quả: BI-RB-BK-R93R-62
```

- **Tìm thấy**: so sánh xem bên nào có **thêm** gì. Bên có thêm thông tin là bên cần xử lý.
- **Không tìm thấy**: cắt bớt giá trị, chỉ giữ phần đặc trưng nhất, rồi tìm lại.

### ⑤ Đo tỉ lệ khớp để xác nhận giả thuyết

Một vài dòng khớp chưa đủ để kết luận, phải **đếm** trên toàn bộ dữ liệu:

```sql
SELECT
    (SELECT COUNT(*) FROM bronze.erp_loc_a101) AS tong,
    (SELECT COUNT(*)
     FROM bronze.erp_loc_a101 l
     WHERE EXISTS (SELECT 1 FROM bronze.crm_cust_info c
                   WHERE c.cst_key = REPLACE(l.cid, '-', ''))) AS khop;
```

Thay `REPLACE(l.cid, '-', '')` bằng phép biến đổi theo giả thuyết của bạn: giữ nguyên cột, `SUBSTRING`, `LEFT`…

| Kết quả | Ý nghĩa |
|---|---|
| Khớp gần 100% | Giả thuyết đúng |
| Khớp 0% | Hai cột không nối trực tiếp được. Quay lại ②–④ để tìm cách biến đổi |
| Khớp một phần | Giả thuyết chỉ đúng với một phần dữ liệu, **hoặc** có những giá trị thật sự không có cặp. Cả hai trường hợp đều phải ghi lại |

Muốn xem **những dòng không khớp**, dùng `NOT EXISTS`:

```sql
SELECT l.cid
FROM bronze.erp_loc_a101 l
WHERE NOT EXISTS (SELECT 1 FROM bronze.crm_cust_info c
                  WHERE c.cst_key = REPLACE(l.cid, '-', ''));
```

### Kiểm tra khoá của bảng

Không kết luận một cột là khoá chỉ dựa vào tên cột. Hãy **so số giá trị khác nhau với tổng số dòng**:

```sql
SELECT COUNT(*) AS so_dong, COUNT(DISTINCT prd_key) AS so_gia_tri_khac_nhau
FROM bronze.crm_prd_info;

-- Liệt kê các giá trị bị lặp
SELECT prd_key, COUNT(*) AS so_lan
FROM bronze.crm_prd_info
GROUP BY prd_key
HAVING COUNT(*) > 1;
```

- Hai số bằng nhau: cột đó là khoá duy nhất.
- Không bằng: có thể dữ liệu bị **trùng** (cần làm sạch), hoặc cột này vốn **không phải khoá**, ví dụ bảng lưu nhiều phiên bản lịch sử. Muốn biết là trường hợp nào thì phải xem cụ thể các dòng bị lặp.

---

## 3. Kết quả áp dụng cho project

### Ý nghĩa của từng bảng

| Bảng | Hệ thống | Nội dung |
|---|---|---|
| `crm_cust_info` | CRM | Thông tin khách hàng |
| `crm_prd_info` | CRM | Thông tin sản phẩm, gồm cả phiên bản hiện tại và lịch sử |
| `crm_sales_details` | CRM | Giao dịch bán hàng |
| `erp_cust_az12` | ERP | Thông tin bổ sung của khách hàng: ngày sinh, giới tính |
| `erp_loc_a101` | ERP | Quốc gia nơi khách hàng sinh sống |
| `erp_px_cat_g1v2` | ERP | Danh mục sản phẩm: category, subcategory, maintenance |

CRM chứa **dữ liệu chính**; ERP **bổ sung thông tin** cho khách hàng và sản phẩm.

### Các mối liên kết

| Liên kết | Khớp trực tiếp? | Cần biến đổi | Tỉ lệ khớp sau biến đổi |
|---|---|---|---|
| `crm_sales_details.sls_cust_id` → `crm_cust_info.cst_id` | ✅ Có | Không cần | 60,398 / 60,398 dòng |
| `crm_sales_details.sls_prd_key` → `crm_prd_info.prd_key` | ❌ 0 / 130 mã | Lấy `prd_key` từ ký tự thứ 7 trở đi: `SUBSTRING(prd_key, 7, LEN(prd_key))` | 130 / 130 mã |
| `erp_px_cat_g1v2.id` → `crm_prd_info.prd_key` | ❌ | Lấy 5 ký tự đầu của `prd_key` và đổi `-` thành `_`: `REPLACE(SUBSTRING(prd_key, 1, 5), '-', '_')` | 36 / 37 danh mục (`CO_PD` không có sản phẩm) |
| `erp_cust_az12.cid` → `crm_cust_info.cst_key` | ⚠️ 7,442 / 18,484 | Bỏ tiền tố `NAS` **nếu có** (chỉ 11,042 dòng có): `CASE WHEN cid LIKE 'NAS%' THEN SUBSTRING(cid, 4, LEN(cid)) ELSE cid END` | 18,484 / 18,484 dòng |
| `erp_loc_a101.cid` → `crm_cust_info.cst_key` | ❌ 0 / 18,484 | Bỏ dấu `-`: `REPLACE(cid, '-', '')` | 18,484 / 18,484 dòng |

**Phát hiện quan trọng:** `crm_prd_info.prd_key` thực chất là **2 thông tin ghép thành 1 cột**:

```
BI-RB  -  BK-R93R-62
  ↑           ↑
category id   product key
(nối với      (nối với
 px_cat.id)    sales.sls_prd_key)
```

Ở tầng Silver, cột này sẽ được **tách thành 2 cột**.

### Khoá của từng bảng

| Bảng | Khoá | Số liệu kiểm tra | Ghi chú cho Silver |
|---|---|---|---|
| `crm_cust_info` | `cst_id` / `cst_key` (theo thiết kế) | 18,494 dòng; 18,485 `cst_id` và 18,488 `cst_key` khác nhau | Dữ liệu **bị trùng**, phải lọc |
| `crm_prd_info` | `prd_id` | 397 dòng; 397 `prd_id` và 295 `prd_key` khác nhau | `prd_key` lặp vì mỗi sản phẩm có **nhiều phiên bản lịch sử** |
| `crm_sales_details` | (`sls_ord_num`, `sls_prd_key`) | 60,398 dòng; 27,659 `sls_ord_num` khác nhau | Một đơn hàng gồm nhiều dòng sản phẩm |

Sơ đồ: [data_integration.drawio](data_integration.drawio)

---

## 4. Bài học rút ra

1. **Đừng tin tên cột.** `sls_ord_num` nghe như khoá nhưng không duy nhất; `prd_key` có chữ "key" nhưng bị lặp.
2. **Khi hai bên không khớp, hãy xem bên nào chứa nhiều thông tin hơn.** Ở đây `sls_prd_key` không sai, chính `prd_key` mới chứa thêm mã danh mục.
3. **Không phải dòng nào cũng sai theo cùng một kiểu.** Chỉ khoảng 60% số dòng của `erp_cust_az12` có tiền tố `NAS`. Nếu cắt 3 ký tự đầu của mọi dòng, các dòng vốn đã đúng sẽ bị hỏng.
4. **Luôn đo tỉ lệ khớp trên toàn bộ dữ liệu.** Vài dòng mẫu khớp chưa đủ để kết luận.
5. **Dùng `NOT EXISTS` thay cho `NOT IN` khi tìm dòng không khớp.** Nếu cột trong câu truy vấn con có giá trị `NULL` (ví dụ `cst_id` có `NULL` ở dòng rác), `NOT IN` sẽ trả về **rỗng** dù thực tế có dòng không khớp.
6. **Trong SQL Server, không đặt câu truy vấn con bên trong hàm tổng hợp.** Viết `SUM(CASE WHEN x IN (SELECT ...) ...)` sẽ báo lỗi *"Cannot perform an aggregate function on an expression containing an aggregate or a subquery"*. Hãy đếm bằng `WHERE EXISTS` như ở bước ⑤.
