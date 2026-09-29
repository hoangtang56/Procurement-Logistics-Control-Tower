# Procurement & Logistics Control Tower

Dự án portfolio Power BI phân tích độ tin cậy giao hàng và mức độ hiển thị chi phí vận chuyển trong chuỗi cung ứng sản phẩm y tế quốc tế.

> **Công cụ:** Power BI, Power Query, DAX, Excel  
> **Bộ dữ liệu:** USAID Supply Chain Shipment Pricing Data (2006–2015)  
> **Phạm vi phân tích:** 7.030 lô hàng, phân tích ở cấp độ shipment

## 1. Bài toán kinh doanh

Bộ phận mua hàng và logistics cần bảo đảm hàng thiết yếu sẵn có, đồng thời kiểm soát chi phí vận chuyển. Báo cáo chỉ hiển thị tổng chi tiêu không trả lời được các câu hỏi vận hành quan trọng:

- Lô hàng nào giao trễ và tổng giá trị hàng đang gặp rủi ro là bao nhiêu?
- Quốc gia nhận hàng hoặc phương thức vận chuyển nào cần được ưu tiên xử lý?
- Nhà cung cấp / đơn vị hoàn tất đơn hàng nào vừa có chất lượng giao hàng thấp vừa có chi phí logistics cao?
- Dữ liệu freight cost đầy đủ đến mức nào trước khi được sử dụng để ra quyết định chi phí?

Dự án biến dữ liệu dòng hàng thô thành một dashboard Power BI tương tác, giúp người dùng đi từ bức tranh toàn cảnh về dịch vụ giao hàng đến việc rà soát theo nhà cung cấp, quốc gia và phương thức vận chuyển.

## 2. Mục tiêu và KPI

Dự án hướng đến ba mục tiêu quản trị:

1. Theo dõi độ tin cậy giao hàng so với mốc tham chiếu 90%.
2. Lượng hóa giá trị hàng có nguy cơ do giao trễ, thay vì chỉ nhìn số lô trễ.
3. Xác định đánh đổi giữa chi phí và chất lượng dịch vụ theo điểm đến, phương thức vận chuyển và nhà cung cấp / đơn vị hoàn tất đơn hàng.

| KPI | Định nghĩa |
|---|---|
| Total Shipments | Số lượng mã lô hàng phân biệt (`ASN/DN #`) |
| On-time Delivery Rate | Tỷ trọng lô giao sớm hoặc đúng ngày kế hoạch |
| Average Late Days | Số ngày trễ trung bình, chỉ tính các lô trễ |
| Late Shipment Value | Tổng giá trị hàng của các lô giao trễ |
| Freight Cost per Kg | Freight cost được ghi nhận riêng / trọng lượng được ghi nhận |
| Freight Cost Ratio | Freight cost được ghi nhận riêng / giá trị lô hàng, chỉ tính lô hợp lệ |
| Freight Data Coverage | Tỷ trọng lô có freight cost dạng số được ghi nhận riêng |

## 3. Dữ liệu và grain

Nguồn dữ liệu là USAID Supply Chain Shipment Pricing Data. Một lô hàng có thể gồm nhiều dòng sản phẩm; nếu sử dụng trực tiếp bảng gốc cho biểu đồ cấp lô, các giá trị như freight cost và ngày giao có thể bị lặp.

Mô hình tách riêng hai cấp độ chi tiết:

| Bảng | Grain | Mục đích |
|---|---|---|
| `fact_shipment_line` | Một dòng sản phẩm trong một lô | Phân tích sản phẩm, số lượng và giá trị dòng hàng |
| `fact_shipment` | Một lô hàng (`ASN/DN #`) | KPI giao hàng, freight cost và giá trị lô |
| `agg_shipment_value` | Một lô hàng | Tổng hợp giá trị các dòng hàng lên cấp lô |
| `dim_date` | Một ngày lịch | Lọc thời gian và phân tích xu hướng tháng |
| `dim_country` | Một quốc gia nhận hàng | Phân tích theo điểm đến |
| `dim_shipment_mode` | Một phương thức vận chuyển | Phân tích theo phương thức |
| `dim_vendor` | Một nhà cung cấp / đơn vị hoàn tất đơn hàng | Đánh giá hiệu suất đối tác |
| `dim_product` | Một mô tả sản phẩm | Drill-down ở cấp dòng sản phẩm |


Tất cả relationships là một-nhiều, lọc một chiều từ dimension sang fact. `fact_shipment` và `fact_shipment_line` không nối trực tiếp với nhau để tránh đường lọc mơ hồ và hiện tượng tính trùng.

## 4. Tiền xử lý và business rules

Power Query được sử dụng để chuẩn hóa kiểu dữ liệu, xử lý các trường freight/weight hỗn hợp và tạo các trường đánh giá giao hàng.

### Quy tắc ngày giao thực tế

Bộ dữ liệu có hai trường ngày có thể thể hiện việc hoàn tất giao hàng. Ngày dùng để báo cáo được xác định theo thứ tự ưu tiên:

```text
Actual Delivery Date = Delivered to Client Date
                       nếu trống thì dùng Delivery Recorded Date
```

Quy tắc này ưu tiên ngày giao cho khách hàng; chỉ dùng ngày hệ thống ghi nhận giao khi trường chính bị thiếu.

```text
Delivery Delay Days = Actual Delivery Date − Scheduled Delivery Date
```

| Số ngày chênh lệch | Delivery Status |
|---:|---|
| Nhỏ hơn 0 | Early |
| Bằng 0 | On-time |
| Lớn hơn 0 | Late |
| Thiếu ngày giao thực tế hoặc ngày kế hoạch | Unavailable |

### Quy tắc freight và trọng lượng

- `Freight Included in Commodity Cost` và `Invoiced Separately` không được quy đổi thành 0; chúng được giữ là các trạng thái phi số.
- Các measure về freight chỉ sử dụng lô có freight cost dạng số được ghi nhận riêng.
- Chỉ số cost per kg chỉ dùng lô có trọng lượng dạng số lớn hơn 0.
- Giá trị một lô hàng được tính bằng tổng `Line Item Value` của tất cả dòng có cùng `ASN/DN #`.

Những quy tắc này tránh việc đánh giá thấp freight cost hoặc diễn giải quá mức mức độ bao phủ của dữ liệu chi phí.

## 5. Dashboard

Dashboard gồm ba trang liên kết. Slicer theo thời gian, quốc gia và phương thức vận chuyển hỗ trợ chuyển từ góc nhìn danh mục sang rà soát rủi ro chi tiết.

### Executive Overview

Hiển thị tổng giá trị hàng, số lô, tỷ lệ giao đúng hẹn, giá trị hàng giao trễ và freight cost được ghi nhận. Xu hướng theo tháng và phân rã rủi ro giúp phát hiện khu vực có chất lượng dịch vụ thấp hơn mốc 90%.


### Delivery & Cost Performance

So sánh chất lượng dịch vụ và hiệu quả chi phí theo quốc gia và phương thức vận chuyển. Scatter plot làm nổi bật các nhóm vừa có freight cost per kg cao vừa có tỷ lệ giao đúng hẹn thấp.


### Supplier Performance

Ưu tiên nhà cung cấp / đơn vị hoàn tất đơn hàng theo giá trị hàng giao trễ, tỷ lệ giao đúng hẹn, số ngày trễ trung bình và freight cost per kg. Trang này hỗ trợ trao đổi SLA dựa trên các KPI minh bạch, không dùng điểm tổng hợp có trọng số tùy ý.


## 6. Insight chính

Các số liệu dưới đây được đọc khi các slicer ở trạng thái **All**.

1. **Độ tin cậy giao hàng cần được ưu tiên cải thiện.** Toàn danh mục chỉ có khoảng **88,6%** lô hàng được giao sớm hoặc đúng hẹn, thấp hơn mốc tham chiếu 90%. Có **804 lô giao trễ**, vì vậy thứ tự ưu tiên nên xét đồng thời số lượng và giá trị hàng gặp rủi ro.

2. **Truck là nhóm rủi ro giao hàng lớn nhất.** Phương thức Truck chỉ đạt khoảng **79,3%** giao đúng hẹn và có xấp xỉ **143,1 triệu USD** giá trị hàng giao trễ. Cần rà soát tuyến đường, điều phối giao nhận và SLA với đơn vị vận chuyển.

3. **Rủi ro tập trung tại một số điểm đến.** Mozambique có khoảng **53,3 triệu USD** giá trị hàng giao trễ, tương đương gần **29,3%** giá trị hàng giao đến quốc gia này. Bước phân tích tiếp theo nên tách nguyên nhân giữa chậm khâu xuất phát, hải quan, năng lực nhận hàng và giao chặng cuối.

4. **Ocean có rủi ro phục hồi dịch vụ dài nhất.** Các lô Ocean bị trễ trung bình khoảng **40,6 ngày**. Với sản phẩm thiết yếu, doanh nghiệp cần kết hợp đặt hàng sớm và chính sách tồn kho an toàn, thay vì chỉ phản ứng bằng đẩy nhanh vận chuyển sau khi trễ xảy ra.

> Các insight phản ánh dữ liệu lịch sử trong bộ dữ liệu, không đại diện cho hoạt động hiện tại của USAID hay hiệu suất hiện tại của bất kỳ nhà cung cấp nào.

## 7. Khuyến nghị

| Ưu tiên | Khuyến nghị | Kết quả kỳ vọng |
|---|---|---|
| 1 | Rà soát ngoại lệ SLA của Truck theo tuyến, nhà vận chuyển và điểm xuất phát. | Giảm giá trị hàng giao trễ ở phương thức có rủi ro cao nhất. |
| 2 | Thiết lập quy trình rà soát ngoại lệ theo quốc gia cho Mozambique và các điểm đến rủi ro cao. | Phân biệt chậm do vận chuyển với chậm do hải quan, nhận hàng hoặc giao địa phương. |
| 3 | Áp dụng cut-off đặt hàng sớm và lập kế hoạch tồn kho an toàn cho nhu cầu phụ thuộc Ocean. | Giảm nguy cơ thiếu hàng khi thời gian trễ kéo dài. |
| 4 | Chuẩn hóa việc ghi nhận freight cost giữa các lô hàng. | Tăng độ tin cậy của chỉ số cost per kg và freight ratio. |
| 5 | Dùng supplier matrix trong buổi đánh giá SLA hằng tháng. | Tập trung hành động với nhà cung cấp / đơn vị hoàn tất đơn hàng dựa trên KPI minh bạch. |

## 8. Giới hạn dữ liệu

- Dữ liệu mang tính lịch sử và được dùng cho mục đích minh họa năng lực phân tích; dashboard không phải hệ thống vận hành thời gian thực.
- Một số freight cost và weight được thể hiện bằng trạng thái văn bản thay vì giá trị số. Do đó, KPI về freight chỉ phản ánh tập lô đủ điều kiện tính toán.
- Dữ liệu không có đầy đủ thông tin về Incoterms, lead time theo hợp đồng, quãng đường, hiệu suất theo carrier hay mức tồn kho. Vì vậy không thể kết luận nguyên nhân gốc của việc giao trễ chỉ từ dashboard.
- `Delivery Recorded Date` chỉ được dùng thay thế khi thiếu ngày giao cho khách hàng. Quy tắc này cần được xác nhận với chủ quy trình trước khi áp dụng thực tế.
- Mốc 90% là mốc tham chiếu phục vụ phân tích portfolio, không phải SLA hợp đồng được công bố bởi tổ chức nguồn.

## 9. Cấu trúc dự án

```text
Procurement-Logistics-Control-Tower/
├── Procurement_Logistics_Control_Tower.pbix
├── README.md
├── data/
│   └── SCMS_Delivery_History_Dataset.csv
└── images/
    ├── data_model.png
    ├── executive_overview.png
    ├── delivery_cost_performance.png
    └── supplier_performance.png
```

## 10. Kỹ năng thể hiện

- Chuyển đổi dữ liệu và thiết kế business rules với Power Query
- Thiết kế fact/dimension và relationships trong Power BI
- Viết DAX measure cho phân tích giao hàng, giá trị hàng và freight cost
- Kiểm soát chất lượng dữ liệu qua quy tắc coverage của freight và weight
- Thiết kế dashboard quản trị và kể chuyện dữ liệu hướng đến hành động

## Nguồn dữ liệu

- USAID Supply Chain Shipment Pricing Data, được chia sẻ công khai qua các nền tảng dữ liệu như Kaggle.
