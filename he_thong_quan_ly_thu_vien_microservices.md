# Hệ thống Quản lý Thư viện Đa chi nhánh --- Dịch vụ Mượn/Trả Sách trực tuyến

## 3 Lĩnh vực (Domain) nghiệp vụ chính

  -----------------------------------------------------------------------------
                STT Tên Domain    Mô tả chính   Đối tượng/Entity  Quy tắc
                                                cốt lõi           nghiệp vụ
                                                                  chính
  ----------------- ------------- ------------- ----------------- -------------
                  1 **Quản lý     Quản lý danh  `Book`,           Sách có sẵn
                    Sách**        mục, thông    `Category`,       mới cho mượn;
                                  tin, tồn kho, `BookStatus`      không xóa
                                  trạng thái                      sách đang
                                  sách                            mượn

                  2 **Quản lý     Quản lý thông `Member`,         Tối đa 5
                    Thành viên**  tin, hạng     `MemberRank`      cuốn/người;
                                  thành viên,                     hạng VIP được
                                  giới hạn mượn                   kỳ hạn dài
                                                                  hơn

                  3 **Quản lý     Tạo phiếu     `Borrowing`,      Ngày trả tối
                    Mượn-Trả**    mượn, nhận    `BorrowDetail`,   đa 14 ngày;
                                  trả, tính     `ReturnRecord`,   quá hạn tính
                                  phạt quá hạn, `Fine`            phạt theo
                                  thống kê                        ngày
  -----------------------------------------------------------------------------

> Các domain này **độc lập về khái niệm** nhưng **phụ thuộc logic nghiệp
> vụ**: Tạo phiếu mượn cần kiểm tra cả 2 domain Sách + Thành viên.

------------------------------------------------------------------------

## LỘ TRÌNH 4 GIAI ĐOẠN CHUYỂN ĐỔI KIẾN TRÚC

### GIAI ĐOẠN 1: Kiến trúc Đơn khối theo Domain --- Monolithic Architecture

**Kiến trúc**: Toàn bộ hệ thống xây dựng trong **1 dự án + 1 CSDL
chung**, nhưng **phân tách code theo Domain** ngay từ đầu.

#### Đặc trưng cấu trúc

``` text
ThuVienApp (1 dự án, 1 triển khai)
├── Domain Sách
├── Domain Thành viên
├── Domain Mượn-Trả
├── CSDL: 1 CSDL chung, các bảng có thể JOIN tự do
└── Giao tiếp: Gọi hàm trực tiếp trong bộ nhớ
```

#### Dấu hiệu đến giai đoạn này

-   Dự án mới bắt đầu, đội ngũ nhỏ (≤ 5 người)
-   Yêu cầu thay đổi nhanh, chưa rõ quy mô phát triển
-   Tất cả logic cần nhất quán dữ liệu ngay lập tức
-   Chưa có nhu cầu triển khai độc lập từng phần

#### Dấu hiệu đến lúc CHUYỂN SANG giai đoạn 2

-   Đội ngũ tăng lên \> 8 người → dễ xung đột code, chờ đợi triển khai
-   Tốc độ phát triển chậm: thay đổi nhỏ cũng phải biên dịch & triển
    khai toàn bộ
-   Khó mở rộng độc lập: Domain Sách cần nâng cấp nhưng phải chờ Domain
    Mượn-Trả ổn định

#### Rủi ro

-   **Chuyển quá sớm**: Tách ra khi chưa cần → phức tạp giao tiếp, giao
    dịch, chi phí vận hành tăng vọt mà không có lợi ích thực tế
-   **Chuyển quá muộn**: Code rối tung, domain lẫn lộn, sửa một chỗ ảnh
    hưởng toàn hệ thống → "lũ lụt thay đổi"

------------------------------------------------------------------------

### GIAI ĐOẠN 2: Dịch vụ Dựa trên Dịch vụ --- SOA (Service-Oriented Architecture)

**Kiến trúc**: Tách thành **các Dịch vụ Dịch vụ lớn** theo Domain, nhưng
**chia sẻ CSDL hoặc qua ESB** làm trung gian.

#### Đặc trưng cấu trúc

``` text
Dịch vụ Sách ─────────┐
Dịch vụ Thành viên ──→ ESB / Dịch vụ Trung tâm ──→ CSDL Chung
Dịch vụ Mượn-Trả ─────┘
```

-   Tách triển khai & phát triển độc lập theo Domain
-   **ESB = trung gian giao tiếp** chuẩn hóa định dạng, bảo mật, luồng
-   **Dữ liệu vẫn chia sẻ CSDL** → vẫn dùng JOIN, giao dịch ACID bình
    thường

#### Dấu hiệu đến giai đoạn này

-   Đội ngũ chia thành các nhóm nhỏ chuyên trách
-   Cần triển khai độc lập từng domain
-   Cần chuẩn hóa giao tiếp, bảo mật, giám sát chung
-   Cần tái sử dụng logic chung cho nhiều ứng dụng (Web, App, Báo cáo)

#### Dấu hiệu đến lúc CHUYỂN SANG giai đoạn 3

-   ESB trở thành **điểm nghẽn duy nhất**: mọi yêu cầu đều đi qua →
    chậm, dễ tắc
-   CSDL chung thành **nút cổ chai**: khóa bảng, tranh chấp dữ liệu khi
    tăng tải
-   Không thể nâng cấp/cải tiến CSDL từng phần
-   Hiệu năng giảm nghiêm trọng khi số yêu cầu tăng

#### Rủi ro

-   **Chuyển quá sớm**: Chi phí xây dựng & vận hành ESB cao trong khi
    lưu lượng nhỏ → lãng phí
-   **Chuyển quá muộn**: ESB + CSDL chung chậm lại → không thể mở rộng
    theo chiều ngang, tăng tải thì toàn hệ thống chậm

------------------------------------------------------------------------

### GIAI ĐOẠN 3: Kiến trúc Vi Dịch vụ --- MSA (Microservices Architecture)

**Kiến trúc**: Tách hẳn thành **Dịch vụ nhỏ, độc lập hoàn toàn**, giao
tiếp trực tiếp qua API/Message Queue, **bỏ ESB trung tâm**.

#### Đặc trưng cấu trúc

``` text
Book Service  ←→  Member Service
     ↓  API/Queue       ↓  API/Queue
Borrowing Service ←→ Thống kê...
     ↓
CSDL Chung (chỉ chia sẻ dữ liệu, mỗi dịch vụ chỉ dùng bảng của mình)
```

-   Mỗi dịch vụ **triển khai, nâng cấp, co giãn độc lập**
-   Giao tiếp trực tiếp qua REST/gRPC/RabbitMQ → **không đi qua trung
    gian ESB**
-   Mỗi dịch vụ **chỉ chịu trách nhiệm dữ liệu thuộc domain của mình**,
    truy vấn chéo qua API
-   Chấp nhận **nhất quán cuối** thay vì ACID tức thời

#### Dấu hiệu đến giai đoạn này

-   ESB đã trở thành điểm nghẽn về hiệu năng & phát triển
-   Cần tốc độ phát triển & triển khai độc lập cao
-   Khối lượng giao dịch tăng mạnh → cần co giãn độc lập
-   Nhiều công nghệ/ngôn ngữ khác nhau phù hợp từng domain

#### Dấu hiệu đến lúc CHUYỂN SANG giai đoạn 4

-   Tranh chấp khóa & truy vấn chéo giữa dịch vụ ngày nhiều → hiệu năng
    giảm
-   Thay đổi cấu trúc bảng phải phối hợp nhiều đội → chậm & dễ lỗi
-   Một dịch vụ nặng ảnh hưởng đến CSDL chung → kéo theo dịch vụ khác
    chậm
-   Không thể tối ưu/hiệu chỉnh CSDL phù hợp từng loại dữ liệu

#### Rủi ro

-   **Chuyển quá sớm**: Phức tạp vượt nhu cầu, chi phí vận hành cao,
    chưa có vấn đề thực tế để giải quyết → lãng phí
-   **Chuyển quá muộn**: CSDL chung vẫn còn JOIN chéo nhiều → tách CSDL
    sau này sẽ cực khó, phải viết lại toàn bộ luồng giao tiếp

------------------------------------------------------------------------

### GIAI ĐOẠN 4: MSA + Tách CSDL theo Dịch vụ --- Database-per-Service

**Kiến trúc**: Mỗi dịch vụ có **CSDL hoàn toàn riêng biệt**, không chia
sẻ chút nào. Dữ liệu chỉ đồng bộ qua sự kiện/API.

#### Đặc trưng cấu trúc

``` text
Book Service     → CSDL_Sách (riêng)
Member Service   → CSDL_ThànhViên (riêng)
Borrowing Service → CSDL_MuonTra (riêng)

Giao tiếp: REST API / Message Queue (không bao giờ JOIN chéo CSDL)
Dữ liệu chéo: gọi API hoặc sao chép dữ liệu tham chiếu theo sự kiện
Giao dịch phân tán: dùng Saga / sự kiện → nhất quán cuối
```

#### Dấu hiệu đến giai đoạn này

-   Đã ở MSA, mỗi dịch vụ phát triển & vận hành hoàn toàn độc lập
-   CSDL chung trở thành ràng buộc cuối cùng → không thể co giãn & tối
    ưu riêng
-   Đội ngũ đã thành thạo xử lý giao tiếp phân tán, lỗi mạng, nhất quán
    cuối
-   Lưu lượng dữ liệu & yêu cầu lớn → mỗi CSDL cần cấu hình & tối ưu
    riêng

#### Dấu hiệu quá sớm / quá muộn

-   **Chuyển quá sớm**: Chưa có kinh nghiệm xử lý phân tán → dữ liệu
    không nhất quán, lỗi khó gỡ, chi phí khắc phục cao hơn lợi ích nhiều
    lần
-   **Chuyển quá muộn**: Phụ thuộc vào JOIN chéo quá nhiều → khi bắt
    buộc tách sẽ phải viết lại hoàn toàn logic truy vấn, rủi ro lỗi rất
    lớn

#### Rủi ro chính

-   Xử lý giao dịch phân tán phức tạp (Saga, hoàn tác thủ công)
-   Dữ liệu tạm không nhất quán → cần cơ chế thông báo/thử lại
-   Phải xây dựng cơ chế giám sát & xử lý lỗi trên toàn luồng

------------------------------------------------------------------------

## SƠ ĐỒ TỔNG QUAN LỘ TRÌNH 4 GIAI ĐOẠN

> Phần sơ đồ minh họa bằng text/ASCII đã được loại bỏ. Có thể vẽ lại sơ
> đồ này trên draw.io hoặc PowerPoint dựa trên cấu trúc của 4 giai đoạn
> ở trên.

------------------------------------------------------------------------

## BẢNG TÓM TẮT NHẬN DIỆN & RỦI RO

  -----------------------------------------------------------------------
  Giai đoạn         Dấu hiệu đến lúc  Rủi ro chuyển QUÁ Rủi ro chuyển QUÁ
                    chuyển            SỚM               MUỘN
  ----------------- ----------------- ----------------- -----------------
  **1→2**           Đội \>8 người,    Phức tạp không    Code rối, phụ
                    triển khai chậm,  cần, chi phí cao  thuộc lẫn nhau,
                    xung đột code     không giá trị     sửa 1 chỗ ảnh
                                                        hưởng toàn hệ
                                                        thống

  **2→3**           ESB chậm, CSDL    Chưa đủ kinh      CSDL chung cổ
                    chung nghẽn, cần  nghiệm → rối giao chai, không thể
                    co giãn độc lập   tiếp, lỗi khó     mở rộng, mọi thứ
                                      theo dõi          chậm cùng lúc

  **3→4**           Tranh chấp dữ     Chưa thành thạo   Phụ thuộc JOIN
                    liệu, thay đổi    phân tán → dữ     chéo quá nhiều →
                    bảng phối hợp     liệu không nhất   tách sau phải
                    khó, tối ưu riêng quán, chi phí     viết lại toàn bộ
                    cần               khắc phục khổng   luồng dữ liệu
                                      lồ                
  -----------------------------------------------------------------------
