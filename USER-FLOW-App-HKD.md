# USER FLOW — App HKD (Zalo Mini App)

> Phụ lục của `SPEC-App-HKD.md` v0.5. Mọi sơ đồ viết bằng Mermaid.
> Ngày: 30/08/2026

---

## 1. Bản đồ điều hướng

```mermaid
flowchart TD
    Z(["Zalo — tìm kiếm / link chia sẻ"]) --> MA["Mini App khởi động"]
    MA --> CHK{"Đã có Store?"}
    CHK -- Chưa --> ONB["Onboarding 3 bước"]
    ONB --> S1
    CHK -- Rồi --> S1["S1 · GHI ĐƠN<br/>(màn hình mặc định)"]

    S1 -.tab bar.- S8
    S1 -.tab bar.- S9
    S1 -.tab bar.- S11

    S8["S8 · BÁO CÁO"]
    S9["S9 · THUẾ"]
    S11["S11 · CÀI ĐẶT"]

    S1 --> S3["S3 · Danh sách đơn"]
    S3 --> S4["S4 · Chi tiết đơn"]
    S1 --> S5["S5 · Danh mục sản phẩm"]

    S8 --> S6["S6 · Nhập hàng"]
    S6 --> S7["S7 · Review OCR"]

    S9 --> EXP1["Xuất sổ S1a-HKD"]
    S9 --> EXP2["Xuất tờ khai 01/TKN-CNKD"]
    S9 --> S10["S10 · Cẩm nang"]

    S11 --> BOT["Kết nối bot Zalo"]
    S11 --> PRIV["Chính sách riêng tư"]
    S11 --> DEL["Xuất / xóa dữ liệu"]

    EXP1 --> CHAT(["Chat Zalo — nhận file"])
    EXP2 --> CHAT
```

---

## 2. F0 — Onboarding lần đầu

Mục tiêu: **ghi được đơn đầu tiên trong 60 giây**.

```mermaid
flowchart TD
    A(["Mở Mini App lần đầu"]) --> B["authorize()"]
    B --> C{"User đồng ý?"}
    C -- Không --> C1["Màn hình giải thích<br/>vì sao cần quyền"] --> B
    C -- Có --> D["getUserInfo()<br/>lấy Zalo ID, tên, avatar"]
    D --> E["POST /api/auth/session<br/>tạo User"]
    E --> F["Bước 1/3<br/>Bạn bán gì?"]
    F --> F1["🏪 Tạp hóa"]
    F --> F2["🍜 Quán ăn"]
    F1 --> G
    F2 --> G["Bước 2/3<br/>Tên cửa hàng?"]
    G --> H["Bước 3/3<br/>Tải danh mục mẫu theo ngành"]
    H --> I["Tạo Store<br/>activation_date = hôm nay"]
    I --> J(["S1 · Màn hình ghi đơn"])

    J -.nhắc nhẹ sau.-> K["Banner: bổ sung MST, địa chỉ"]
    J -.nhắc nhẹ sau.-> L["Banner: kết nối bot Zalo<br/>để nhận tổng kết mỗi tối"]

    style J fill:#d4edda
```

**Điểm quan trọng:** MST, địa chỉ, `getPhoneNumber`, kết nối bot đều **không hỏi lúc onboarding**. `getPhoneNumber` chỉ xin đúng lúc user bấm xuất tờ khai.

---

## 3. F3.1 — Ghi đơn nhanh (có xử lý mạng)

```mermaid
flowchart TD
    A(["S1 · Tab Nhanh"]) --> B["Gõ số tiền trên bàn phím số"]
    B --> C["Chọn PTTT<br/>💵 Mặt / 🏦 CK / 📱 Ví"]
    C --> D{"Bấm gì?"}

    D -- "🎤 Mic" --> VOICE(["→ Flow giọng nói (mục 4)"])
    D -- "Đổi ngày" --> DATE["Chọn ngày<br/>tối đa 7 ngày trước"]
    DATE --> C
    D -- "LƯU" --> E["Sinh client_uuid<br/>Optimistic UI: cộng doanh thu ngay"]

    E --> F["POST /api/orders"]
    F --> G{"Kết quả"}

    G -- "200 OK" --> H["Toast xanh 'Đã lưu'<br/>Hiện nút Hoàn tác 10 giây"]
    H --> I["Reset bàn phím<br/>sẵn sàng đơn tiếp"]
    I --> A

    G -- "Lỗi mạng / 5xx" --> J["ROLLBACK con số<br/>Toast đỏ 'Chưa lưu được'"]
    J --> K["GIỮ NGUYÊN dữ liệu đã gõ<br/>Nút chuyển thành 'Thử lại'"]
    K --> L{"User làm gì?"}
    L -- "Thử lại" --> F
    L -- "Bỏ qua" --> A

    G -- "409 trùng client_uuid" --> H

    H -.-> M{"Bấm Hoàn tác<br/>trong 10s?"}
    M -- Có --> N["DELETE /api/orders/:id<br/>trừ lại doanh thu"]
    N --> A

    style H fill:#d4edda
    style J fill:#f8d7da
```

---

## 4. F3.2 — Ghi đơn bằng giọng nói ⭐

```mermaid
flowchart TD
    A(["Giữ nút 🎤"]) --> B{"Có quyền mic?"}
    B -- Chưa --> B1["Xin quyền"]
    B1 --> B2{"User cấp?"}
    B2 -- Không --> B3["Ẩn nút mic<br/>quay về bàn phím số"] --> Z(["S1"])
    B2 -- Có --> C
    B -- Rồi --> C["audio/record bắt đầu<br/>hiện đồng hồ đếm + waveform"]

    C --> D{"Sự kiện"}
    D -- "Thả nút" --> E
    D -- "Quá 15 giây" --> E["Dừng ghi → file audio"]
    D -- "Vuốt ra ngoài" --> Z

    E --> F["POST /api/orders/voice<br/>audio + danh mục SP của shop"]
    F --> G["Gemini: audio → JSON đơn hàng<br/>(1 lần gọi)"]
    G --> H{"Parse được?"}

    H -- Không --> H1["Toast 'Chưa nghe rõ, thử lại'<br/>giữ nguyên màn hình"] --> Z
    H -- "Chỉ ra được số tiền" --> I1["Bản nháp: đơn không chi tiết"]
    H -- "Đủ dòng hàng" --> I2["Bản nháp: danh sách SP + SL + giá"]

    I1 --> J
    I2 --> J["MÀN HÌNH BẢN NHÁP<br/>hiện transcript + đơn đã parse"]

    J --> K{"User"}
    K -- "Sửa" --> L["Chỉnh SL / giá / PTTT / ngày"] --> J
    K -- "Hủy" --> Z
    K -- "Xác nhận" --> M["POST /api/orders<br/>source = voice"]
    M --> N(["Đã lưu → reset"])

    style J fill:#fff3cd
    style N fill:#d4edda
```

**Nguyên tắc bất di bất dịch:** giọng nói **không bao giờ** tự lưu thẳng. Luôn qua bản nháp.

**Ví dụ parse:**

| Câu nói | Kết quả |
|---|---|
| "Hai chai nước mắm Nam Ngư ba lăm nghìn một chai" | 2 × Nước mắm Nam Ngư @35.000 |
| "Một trăm hai mươi nghìn" | Đơn không chi tiết, 120.000 |
| "Ba tô bún bò hai cà phê sữa đá chuyển khoản" | 3 × Bún bò + 2 × Cà phê sữa đá, CK |
| "Hôm qua bán được năm trăm nghìn" | Đơn hôm qua, 500.000 |

---

## 5. F3.1b — Ghi đơn chế độ Chi tiết

```mermaid
flowchart TD
    A(["S1 · Tab Chi tiết"]) --> B["Lưới sản phẩm có emoji"]
    B --> C{"Thao tác"}
    C -- "Chạm SP" --> D["Thêm vào giỏ, SL +1"]
    C -- "Tìm kiếm" --> E["Lọc theo tên"] --> B
    C -- "SP chưa có" --> F["Tạo nhanh SP mới<br/>tên + giá"] --> D
    D --> B

    B --> G["Xem giỏ hàng"]
    G --> H{"Sửa giỏ"}
    H -- "Đổi SL" --> G
    H -- "Đổi giá lần này" --> G
    H -- "Xóa dòng" --> G
    H -- "Xong" --> I["Chọn PTTT"]
    I --> J["LƯU"]
    J --> K["Tính cost_amount<br/>từ giá vốn từng SP"]
    K --> L["POST /api/orders<br/>source = detail + items[]"]
    L --> M(["Đã lưu — đơn này TÍNH ĐƯỢC lãi gộp"])

    style M fill:#d4edda
```

> Chỉ đơn ở chế độ Chi tiết mới tính được lãi gộp. Đơn Nhanh chỉ có doanh thu.

---

## 6. F2 — Nhập hàng bằng OCR hóa đơn

```mermaid
flowchart TD
    A(["S6 · Nhập hàng"]) --> B{"Cách nhập"}
    B -- "Nhập tay" --> MANUAL["Form: ngày, NCC, tổng tiền"] --> SAVE
    B -- "📷 Chụp hóa đơn" --> C["chooseImage / openMediaPicker"]

    C --> D["Nén ảnh ~1024px, JPEG q80"]
    D --> E["POST /api/purchases/ocr"]
    E --> F["Gemini Vision<br/>structured output"]
    F --> G{"Kết quả"}

    G -- "Lỗi / timeout" --> H["Mở form nhập tay<br/>ảnh hiển thị bên cạnh"] --> MANUAL
    G -- "OK" --> I["S7 · MÀN HÌNH REVIEW"]

    I --> J["Bảng dòng hàng<br/>ô confidence thấp = highlight vàng"]
    J --> K["Khớp mờ tên hàng<br/>với danh mục shop"]
    K --> L{"Khớp được?"}
    L -- "Có" --> M["Gắn product_id"]
    L -- "Không" --> N["Gợi ý 'Tạo SP mới'"]
    M --> O
    N --> O["User sửa từng dòng nếu cần"]

    O --> SAVE["Lưu PurchaseOrder + PurchaseItem<br/>Ảnh gốc → volume server"]
    SAVE --> P{"Có dòng hàng khớp SP<br/>và giá lệch > 5%?"}
    P -- Không --> Q
    P -- Có --> R(["→ Flow cập nhật giá vốn (mục 7)"])
    R --> Q(["Xong — về S6"])

    style I fill:#fff3cd
    style Q fill:#d4edda
```

---

## 7. F1 — Cập nhật giá vốn từ phiếu nhập

```mermaid
flowchart TD
    A(["Lưu phiếu nhập xong"]) --> B["Với mỗi dòng hàng khớp SP"]
    B --> C{"cost_price_is_manual?"}
    C -- "true (user nhập tay)" --> D{"Lệch > 5%?"}
    C -- "false" --> D
    D -- Không --> E(["Bỏ qua"])
    D -- Có --> F["Hỏi: Cập nhật giá vốn<br/>Nước mắm 28.000 → 30.000?"]

    F --> G{"User chọn"}
    G -- "Có" --> H["cost_price = giá mới<br/>cost_price_is_manual = false"]
    G -- "Không" --> E
    G -- "Đừng hỏi nữa" --> I["Đánh dấu SP: bỏ qua tự cập nhật"]

    H --> J(["Xong"])
    I --> J
    E --> J

    style F fill:#fff3cd
```

> **Không bao giờ ghi đè giá vốn user nhập tay mà không hỏi.**

---

## 8. F0.4 — Kết nối bot Zalo

```mermaid
sequenceDiagram
    actor U as Chủ shop
    participant MA as Mini App
    participant API as Railway API
    participant BOT as Zalo Bot
    participant CHAT as Chat Zalo

    U->>MA: Bấm "Bật thông báo Zalo"
    MA->>API: POST /api/bot/pair
    API->>API: Sinh mã 6 số<br/>lưu BotPairing, hết hạn 10 phút
    API-->>MA: { code: "482915" }
    MA->>U: Hiện mã + nút "Mở chat bot"
    U->>MA: Bấm nút
    MA->>CHAT: openChat(bot)
    U->>BOT: Gửi "482915"
    BOT->>API: webhook { chat_id, text }
    API->>API: Khớp mã → User.bot_chat_id = chat_id
    API->>BOT: sendMessage xác nhận
    BOT-->>U: ✅ Đã kết nối!<br/>Bạn sẽ nhận tổng kết mỗi tối 21:00
    U->>MA: Quay lại app
    MA->>API: GET /api/bot/status
    API-->>MA: { linked: true }
    MA->>U: Hiện "Đã kết nối ✅"
```

**Chưa kết nối thì sao?** App vẫn dùng bình thường, chỉ nhắc nhẹ ở dashboard. Nhưng luồng **xuất file bị chặn mềm** — xem mục 10.

---

## 9. F6 — Bot gửi thông báo định kỳ

```mermaid
sequenceDiagram
    participant CRON as Cron worker
    participant DB as Postgres
    participant API as Tax engine
    participant BOT as Zalo Bot
    actor U as Chủ shop

    Note over CRON: 21:00 mỗi ngày
    CRON->>DB: Lấy các Store có bot_chat_id<br/>và bật tổng kết ngày
    loop Mỗi store
        CRON->>DB: Truy vấn doanh thu hôm nay,<br/>số đơn, lãi gộp, lũy kế năm
        CRON->>API: Tính % ngưỡng + trạng thái thuế
        API-->>CRON: { revenue_ytd, pct, status }
        CRON->>BOT: sendMessage(chat_id, nội dung)
        BOT-->>U: 🏪 Tổng kết ngày...
        CRON->>DB: Ghi NotificationLog
    end

    Note over CRON: Khi lũy kế chạm 70% / 90% / 100%
    CRON->>BOT: sendMessage cảnh báo ngưỡng
    BOT-->>U: ⚠️ Đã đạt 90% ngưỡng 1 tỷ...

    Note over CRON: Trước 31/01: 30 / 7 / 1 ngày
    CRON->>BOT: sendMessage nhắc hạn
    BOT-->>U: 📋 Còn 7 ngày để nộp 01/TKN-CNKD
```

**Lịch thông báo:**

| Loại | Thời điểm |
|---|---|
| Tổng kết ngày | 21:00 (cấu hình được) |
| Tổng kết tháng | Ngày 1, 08:00 |
| Tổng kết quý | Ngày đầu quý sau, 08:00 |
| Cảnh báo ngưỡng | Ngay khi chạm 70% / 90% / 100% |
| Nhắc hạn 31/01 | Trước 30 / 7 / 1 ngày |

---

## 10. F4.5 & F5 — Xuất file qua bot

```mermaid
flowchart TD
    A(["S9 Thuế hoặc S8 Báo cáo"]) --> B{"Chọn file"}
    B --> B1["📒 Sổ doanh thu S1a-HKD"]
    B --> B2["📊 Chi tiết bán hàng"]
    B --> B3["🧾 Sổ chi phí"]
    B --> B4["💵 Lãi gộp"]
    B --> B5["📋 Tờ khai 01/TKN-CNKD"]

    B1 --> C
    B2 --> C
    B3 --> C
    B4 --> C
    B5 --> P{"Đã có MST<br/>và thông tin hộ?"}

    P -- Chưa --> P1["Form bổ sung: MST, địa chỉ<br/>getPhoneNumber() ở đây"] --> C
    P -- Rồi --> C{"Đã kết nối bot?"}

    C -- Chưa --> D["Màn hình mời kết nối bot<br/>'File sẽ được gửi vào chat Zalo'"]
    D --> E(["→ Flow kết nối bot (mục 8)"])
    E --> C

    C -- Rồi --> F["POST /api/export/..."]
    F --> G["Server sinh file<br/>ExcelJS / react-pdf"]
    G --> H{"Là tờ khai?"}
    H -- Có --> I["Chèn cảnh báo:<br/>'Số liệu từ ngày [activation_date]'<br/>+ cảnh báo app giao đồ ăn nếu ngành FNB"]
    H -- Không --> J
    I --> J["bot.sendDocument(chat_id, file)"]
    J --> K["Mini App: '✅ Đã gửi vào chat Zalo'<br/>nút [Mở chat]"]
    K --> L(["User mở chat, tải hoặc<br/>chuyển tiếp cho kế toán"])

    style D fill:#fff3cd
    style I fill:#f8d7da
    style L fill:#d4edda
```

---

## 11. F5 — Màn hình Tình trạng thuế

```mermaid
flowchart TD
    A(["S9 · Thuế"]) --> B["GET /api/tax/status?year=2026"]
    B --> C["Server: tổng doanh thu năm<br/>tách theo nhóm ngành"]
    C --> D["Áp TaxRuleConfig<br/>có hiệu lực tại kỳ đó"]
    D --> E{"revenue_ytd vs ngưỡng 1 tỷ"}

    E -- "< 70%" --> F1["🟢 CHƯA PHẢI NỘP THUẾ<br/>thanh xanh"]
    E -- "70–90%" --> F2["🟡 CHƯA PHẢI NỘP<br/>thanh vàng + nhắc theo dõi"]
    E -- "90–100%" --> F3["🟠 SẮP VƯỢT NGƯỠNG<br/>thanh cam + gợi ý chuẩn bị"]
    E -- "> 100%" --> F4["🔴 ĐÃ VƯỢT NGƯỠNG"]
    E -- "> 3 tỷ" --> F5["❌ App không hỗ trợ quy mô này<br/>gợi ý gặp kế toán"]

    F1 --> G
    F2 --> G
    F3 --> G["Khối 'Bạn VẪN CẦN':<br/>📋 Nộp 01/TKN-CNKD hạn 31/01<br/>📒 Giữ sổ S1a-HKD"]

    F4 --> H["Bảng ước tính thuế<br/>GTGT + TNCN theo tỷ lệ ngành"]
    H --> I["Cảnh báo: sẽ phải khai QUÝ<br/>và dùng hóa đơn điện tử"]
    I --> J["❗ Chỉ là ƯỚC TÍNH —<br/>nên gặp đại lý thuế"]

    G --> K["Nút [Xuất tờ khai] [Xuất sổ]"]
    J --> K
    K --> L(["→ Flow xuất file (mục 10)"])

    G --> M["Nút [Xem cách tính]"]
    J --> M
    M --> N["Bảng từng bước +<br/>căn cứ pháp lý + disclaimer"]

    style F4 fill:#f8d7da
    style F5 fill:#f8d7da
    style J fill:#f8d7da
```

---

## 12. Trạng thái ngưỡng thuế trong năm

```mermaid
stateDiagram-v2
    [*] --> AnToan: Đầu năm, doanh thu = 0

    AnToan: 🟢 An toàn (< 70%)
    TheoDoi: 🟡 Cần theo dõi (70–90%)
    SapVuot: 🟠 Sắp vượt (90–100%)
    DaVuot: 🔴 Đã vượt ngưỡng
    NgoaiPhamVi: ❌ Ngoài phạm vi (> 3 tỷ)

    AnToan --> TheoDoi: chạm 70%<br/>bot gửi cảnh báo
    TheoDoi --> SapVuot: chạm 90%<br/>bot gửi cảnh báo
    SapVuot --> DaVuot: vượt 1 tỷ<br/>bot gửi cảnh báo
    DaVuot --> NgoaiPhamVi: vượt 3 tỷ

    TheoDoi --> AnToan: sửa/xóa đơn làm giảm
    SapVuot --> TheoDoi: sửa/xóa đơn làm giảm
    DaVuot --> SapVuot: sửa/xóa đơn làm giảm

    AnToan --> [*]: 31/12 — chốt năm
    TheoDoi --> [*]: 31/12
    SapVuot --> [*]: 31/12
    DaVuot --> [*]: 31/12

    note right of DaVuot
        Đổi nghĩa vụ:
        khai QUÝ thay vì năm,
        bắt buộc hóa đơn điện tử
    end note
```

---

## 13. Trạng thái một đơn hàng khi mạng chập chờn

Vì kiến trúc **online-only**, đây là vòng đời cần làm cho thật kỹ.

```mermaid
stateDiagram-v2
    [*] --> DangNhap: User gõ số tiền

    DangNhap: Đang nhập
    DangGui: Đang gửi (optimistic UI)
    DaLuu: Đã lưu ✅
    LoiChoThuLai: Lỗi — chờ thử lại
    DaHuy: Đã hủy

    DangNhap --> DangGui: bấm LƯU<br/>sinh client_uuid<br/>cộng doanh thu ngay
    DangGui --> DaLuu: 200 OK
    DangGui --> DaLuu: 409 trùng client_uuid<br/>(coi như thành công)
    DangGui --> LoiChoThuLai: lỗi mạng / 5xx<br/>ROLLBACK con số

    LoiChoThuLai --> DangGui: bấm Thử lại<br/>(giữ nguyên client_uuid)
    LoiChoThuLai --> DaHuy: user bỏ qua
    LoiChoThuLai --> LoiChoThuLai: tự retry khi có mạng lại

    DaLuu --> DaHuy: bấm Hoàn tác trong 10s
    DaLuu --> [*]
    DaHuy --> [*]

    note right of LoiChoThuLai
        DỮ LIỆU ĐÃ GÕ
        LUÔN GIỮ NGUYÊN
        trên màn hình
    end note

    note right of DangGui
        client_uuid = idempotency key
        chống ghi trùng khi
        timeout rồi lại thành công
    end note
```

---

## 14. Vòng đời một năm thuế

```mermaid
flowchart LR
    A(["01/01<br/>Bắt đầu năm"]) --> B["Ghi đơn hằng ngày<br/>bot tổng kết mỗi tối"]
    B --> C["Cuối mỗi tháng<br/>bot tổng kết tháng"]
    C --> D["Cuối mỗi quý<br/>bot tổng kết quý"]
    D --> B

    B -.chạm mốc.-> E["70% / 90% / 100%<br/>bot cảnh báo ngưỡng"]

    D --> F(["31/12<br/>Chốt số liệu năm"])
    F --> G["01/01 năm sau<br/>bot nhắc: còn 30 ngày"]
    G --> H["24/01 — còn 7 ngày"]
    H --> I["30/01 — còn 1 ngày"]
    I --> J["User xuất tờ khai 01/TKN-CNKD<br/>bot gửi PDF vào chat"]
    J --> K["User tự nộp qua eTax Mobile"]
    K --> L(["31/01<br/>HẠN NỘP"])

    style E fill:#fff3cd
    style L fill:#f8d7da
```

**Trường hợp đặc biệt — hộ mới thành lập trong năm:** nộp 2 lần, hạn 31/07 (6 tháng đầu) và 31/01 (6 tháng cuối). App phải tự nhận biết từ `Store.established_date`.

---

## 15. Lệnh bot — flow phụ không cần mở app

```mermaid
flowchart TD
    A(["User nhắn cho bot"]) --> B{"Nội dung"}
    B -- "/homnay" --> C["Doanh thu hôm nay + số đơn"]
    B -- "/thang" --> D["Doanh thu tháng + lũy kế năm"]
    B -- "/thue" --> E["Tình trạng thuế + % ngưỡng"]
    B -- "/so" --> F["Sinh và gửi file sổ doanh thu"]
    B -- "/tat" --> G["Tắt thông báo định kỳ"]
    B -- "/mo" --> H["Bật lại thông báo"]
    B -- "Mã 6 số" --> I(["→ Flow kết nối bot (mục 8)"])
    B -- "Khác" --> J["Gợi ý danh sách lệnh<br/>+ nút mở Mini App"]

    C --> K(["Trả lời trong chat"])
    D --> K
    E --> K
    F --> K
    G --> K
    H --> K
    J --> K
```

---

## 16. Ba luồng dễ làm ẩu nhất

| Luồng | Vì sao dễ hỏng | Bắt buộc |
|---|---|---|
| **Ghi đơn khi mạng yếu** (mục 3, 13) | Online-only nên mọi lỗi mạng đập thẳng vào user | Không bao giờ để mất dữ liệu đã gõ · nói thật về trạng thái · luôn có nút thoát |
| **Bản nháp giọng nói** (mục 4) | Cám dỗ tự lưu thẳng cho nhanh | Luôn qua bản nháp, kể cả khi model tự tin 100% |
| **Cảnh báo trên tờ khai** (mục 10) | Dễ quên vì không ai kiểm tra | Luôn in dòng "số liệu từ ngày X" và cảnh báo app giao đồ ăn |
