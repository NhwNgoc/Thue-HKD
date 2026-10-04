# SPEC — App HKD: Sổ bán hàng có tính thuế (Zalo Mini App)

| | |
|---|---|
| **Tên dự án** | App HKD *(tên thương mại do chủ dự án tự đặt)* |
| **Phiên bản spec** | **v0.5 — CHỐT, sẵn sàng code** |
| **Ngày** | 30/08/2026 |
| **Loại dự án** | Dự án cá nhân, 1 người, vibe-code 100% |
| **Nền tảng** | Zalo Mini App (Android + iOS) |
| **Backend** | Railway (đã có sẵn) |
| **Thông báo & giao file** | Zalo Bot Platform chính thức |
| **Phát hành** | **Chế độ Testing** — không cần qua kiểm duyệt Zalo |
| **Kiến trúc dữ liệu** | **Online-only** — server là nguồn sự thật duy nhất |

> ⚠️ **Disclaimer bắt buộc hiển thị trong app:** App chỉ là công cụ **ghi chép và tham khảo**. Số liệu thuế chỉ là ước tính. **Hộ kinh doanh tự chịu trách nhiệm** kê khai và nộp thuế. App **không tự động gửi bất kỳ dữ liệu nào cho cơ quan thuế**.

---

## 0. Toàn bộ quyết định đã chốt

| Vấn đề | Quyết định |
|---|---|
| **Định vị** | **"Sổ bán hàng có tính thuế"** — ghi đơn là chính, thuế là phụ |
| Khách hàng | HKD doanh thu **dưới 1 tỷ/năm** (được miễn thuế) |
| Ngành | **Tạp hóa** và **Ăn uống** |
| Nền tảng | Zalo Mini App |
| Backend | Railway |
| Đăng nhập | Tài khoản Zalo (`authorize` + `getUserInfo`) |
| Thông báo | Zalo Bot Platform chính thức |
| **Dữ liệu** | **Online-only**, không offline |
| **Ghi âm** | **Có, hoạt động trên cả hai nền** |
| **Giao file** | **Bot gửi vào chat Zalo** (không dùng `downloadFile`) |
| **Ảnh hóa đơn** | Lưu trên server, không giới hạn thời gian |
| **Phát hành** | Chế độ Testing, chưa public |
| Doanh thu | Miễn phí, tính phí theo năm sau này |
| Trách nhiệm pháp lý | Không chịu trách nhiệm — công cụ tham khảo |
| Nộp tờ khai | Chỉ xuất tờ khai, user tự nộp |
| Hóa đơn điện tử | Không làm |
| Người dùng | Chỉ chủ cửa hàng, **một cửa hàng** |
| Giá vốn | Cả hai: nhập tay + tự đề xuất từ phiếu nhập |
| OCR | Đầy đủ dòng hàng |
| Dữ liệu quá khứ | Bắt đầu từ ngày cài, ghi bù tối đa **7 ngày** |
| App giao đồ ăn | Ngoài phạm vi v1 |
| Deadline | Không cố định |

### Ba quyết định ở vòng 4 làm spec gọn đi đáng kể

**1. Online-only → xóa toàn bộ cơ chế đồng bộ**
Không còn hàng đợi offline, không SyncQueue, không xử lý xung đột, không IndexedDB. Server là nguồn sự thật duy nhất. Đổi lại **trải nghiệm khi mạng yếu trở thành yêu cầu hạng nhất** — xem mục 5.4, đây là phần dễ làm ẩu nhất và cũng là phần user cảm nhận rõ nhất.

**2. Chế độ Testing → không cần kiểm duyệt**
Bỏ được rủi ro bị từ chối duyệt và vòng lặp 3–5 ngày. Iterate nhanh, sửa gì cũng deploy được ngay.
Ràng buộc còn lại: **bản Testing giới hạn 60 version** (Development 300) — đừng đốt deploy vô tội vạ, gom thay đổi rồi hãy build.
Vẫn nên viết trang chính sách riêng tư từ sớm — vừa là chuẩn mực đúng đắn, vừa để sẵn cho ngày public.

**3. Bot gửi file → bỏ `downloadFile`**
Một đường giao file duy nhất, ít code hơn, và thật ra tiện hơn cho user: file Excel/PDF nằm trong lịch sử chat Zalo, chuyển tiếp cho kế toán bằng 2 chạm. Ràng buộc: **file ≤ 5MB** — thừa sức cho sổ sách một năm.

---

## 1. Định vị & mục tiêu

> **"Sổ bán hàng có tính thuế"** — không phải "app thuế".

Khách hàng mục tiêu dưới 1 tỷ → **nộp 0 đồng thuế**. Thuế chỉ nghĩ tới 1–2 lần/năm. Thứ khiến họ mở app **mỗi ngày** là ghi bán hàng và xem hôm nay bán được bao nhiêu, lãi bao nhiêu.

- Màn hình mặc định = **màn hình ghi đơn**
- Thuế nằm ở tab riêng, không phô trương
- Thuế là **lý do cài**, sổ bán hàng là **lý do ở lại**

| # | Mục tiêu | Chỉ số |
|---|---|---|
| G1 | Ghi một đơn ≤ 5 giây | Từ mở Mini App → lưu đơn |
| G2 | Mở app mỗi ngày | ≥ 5 đơn/ngày/user active |
| G3 | Biết cách ngưỡng 1 tỷ bao xa | Hỏi trực tiếp người test |
| G4 | Voice parse đúng | ≥ 90% câu 1–3 mặt hàng |
| G5 | Xuất đúng mẫu S1a-HKD & 01/TKN-CNKD | Kế toán xác nhận |
| G6 | Chi phí vận hành | ≤ 3.000 đ/user/tháng |

**Ngoài phạm vi:** hóa đơn điện tử · nộp tờ khai qua API · phân quyền & nhân viên · HKD trên 3 tỷ · nhiều cửa hàng · tồn kho · doanh thu app giao đồ ăn · offline · công nợ · khuyến mãi.

### Người dùng thử nghiệm
Đã có người test thật, hiện **ghi chép bằng giấy**. Đây là điều kiện lý tưởng: không phải cạnh tranh với thói quen dùng app khác, và bất kỳ cải thiện nào so với sổ tay đều thấy rõ ngay.

**Ba việc nên làm với người test ngay từ v0.1:**
1. Chụp lại **cuốn sổ giấy** của họ — cách họ ghi chính là bản thiết kế UI tốt nhất
2. Đo xem một ngày họ ghi bao nhiêu dòng, mỗi dòng mất bao lâu
3. Hỏi thẳng: *"Cuối tháng anh/chị làm gì với cuốn sổ này?"* — câu trả lời quyết định tính năng báo cáo nào thật sự cần

---

## 2. Bối cảnh pháp lý (cập nhật 30/08/2026)

### 2.1. Mốc thay đổi
| Mốc | Nội dung | Căn cứ |
|---|---|---|
| 01/01/2026 | Bỏ thuế khoán | NQ 68-NQ/TW |
| 01/01/2026 | Bãi bỏ lệ phí môn bài | NQ 198/2025/QH15 |
| 01/01/2026 | **Ngưỡng không chịu thuế: 1 tỷ đ/năm** | NĐ 141/2026/NĐ-CP |
| 01/01/2026 | Chế độ kế toán HKD mới | **TT 152/2025/TT-BTC** |
| 13/05/2026 | Biểu mẫu tờ khai mới | **TT 50/2026/TT-BTC** |
| 2026 | Tỷ lệ thuế theo ngành | NĐ 68/2026/NĐ-CP |

### 2.2. Phân nhóm
| Nhóm | Doanh thu/năm | Nghĩa vụ | App hỗ trợ |
|---|---|---|---|
| **🎯 Nhóm 1** | **≤ 1 tỷ** | **Miễn GTGT & TNCN.** Thông báo doanh thu 1 lần/năm | ✅ Đầy đủ |
| Nhóm 2 | 1 – 3 tỷ | Nộp thuế % doanh thu, khai quý | ⚠️ Ước tính + cảnh báo |
| Nhóm 3–4 | > 3 tỷ | Thuế trên lợi nhuận | ❌ "App không hỗ trợ quy mô này" |

### 2.3. Tỷ lệ % thuế *(chỉ dùng khi ước tính vượt ngưỡng)*
| Ngành | % GTGT | % TNCN | Tổng |
|---|---|---|---|
| **Bán lẻ, tạp hóa** | 1,0% | 0,5% | **1,5%** |
| **Dịch vụ ăn uống** | 3,0% | 1,5% | **4,5%** ⚠️ |
| Dịch vụ khác | 5,0% | 2,0% | 7,0% |
| Hoạt động khác | 2,0% | 1,0% | 3,0% |

> ⚠️ Nguồn không thống nhất về ăn uống (4,5% hay 7%). Cần kế toán xác nhận. Chỉ ảnh hưởng con số dự báo, không ảnh hưởng nghĩa vụ thực tế của nhóm dưới ngưỡng.

### 2.4. Nghĩa vụ của HKD dưới 1 tỷ

| Nghĩa vụ | Chi tiết |
|---|---|
| Nộp thuế | ❌ Không — miễn cả GTGT và TNCN |
| Tờ khai quý | ❌ **Không phải nộp** |
| **Thông báo doanh thu năm** | ✅ **Mẫu 01/TKN-CNKD** (TT 50/2026/TT-BTC) |
| Hạn nộp | **31/01 năm kế tiếp** |
| Hộ mới thành lập trong 2026 | Nộp **2 lần**: 31/07/2026 và 31/01/2027 |
| Nơi nộp | Cơ quan thuế địa bàn, qua eTax |
| **Sổ kế toán** | ✅ **Mẫu S1a-HKD** (TT 152/2025/TT-BTC) |
| Hóa đơn điện tử | ❌ Không bắt buộc |

**Mẫu 01/TKN-CNKD:**
```
[Thông tin chung] Kỳ tính thuế (năm) · Lần đầu / Bổ sung lần thứ...
                  Tên HKD · MST · Địa chỉ · Người được ủy quyền

[Phần A] Xác định nghĩa vụ thuế — theo TỪNG NGÀNH NGHỀ:
         Tổng doanh thu · DT không chịu GTGT · DT chịu 0%
         Thuế GTGT phải nộp · DT tính TNCN · Số thuế đã nộp

[Phần B–D] Thuế TTĐB, tài nguyên, BVMT → tạp hóa/ăn uống KHÔNG phát sinh → ẩn
[Phần E]   Đề nghị xử lý thuế nộp thừa → ẩn
```

**Sổ S1a-HKD — chỉ 3 cột:**
| Cột | Nội dung |
|---|---|
| A | Ngày, tháng ghi sổ |
| B | Diễn giải nội dung doanh thu bán hàng hóa, dịch vụ |
| 1 | Số tiền |

Cho phép ghi gộp theo kỳ → app xuất mặc định **1 dòng/ngày**, có tùy chọn chi tiết từng đơn.

### 2.5. Doanh thu tính thuế
Gồm **tất cả nguồn thu**: tiền mặt, chuyển khoản, QR, ví điện tử, và app giao đồ ăn.

App giao đồ ăn ngoài phạm vi v1 → **bắt buộc hiện cảnh báo** cho user ngành ăn uống ở màn hình thuế và trên tờ khai:
> *"Nếu bạn có bán qua GrabFood/ShopeeFood, nhớ cộng thêm phần đó khi kê khai — app chưa hỗ trợ ghi nhận."*

---

## 3. Nền tảng Zalo

### 3.1. Công nghệ Mini App
```
zmp-cli   — khởi tạo, dev server, deploy (nền Vite)
zmp-ui    — 50+ component ZaUI: Button, Input, Modal, Tabs, List, Sheet...
zmp-sdk   — cầu nối tới Zalo native
React + TypeScript
```

### 3.2. API SDK sẽ dùng

| API | Dùng để |
|---|---|
| `authorize` | Xin quyền lần đầu |
| `getUserInfo` | Tên, avatar, Zalo ID |
| `getPhoneNumber` | SĐT — **chỉ xin khi điền tờ khai**, không xin lúc onboarding |
| `getAccessToken` | Token để backend xác thực |
| `setStorage` / `getStorage` | Cache nhẹ: shop_id, cài đặt, danh mục sản phẩm |
| `chooseImage` / `openMediaPicker` | Chụp/chọn ảnh hóa đơn |
| `audio/record` | Ghi âm giọng nói |
| `openWebview` | Xem trước PDF tờ khai, mở hướng dẫn |
| `openChat` | Mở chat với bot để user gửi mã ghép |
| `request` | Gọi API Railway |
| `payment` | Để dành v2 khi thu phí |

> `downloadFile` **không dùng** — mọi file đi qua bot.

### 3.3. Ràng buộc nền tảng

| Ràng buộc | Cách xử lý |
|---|---|
| Bản Testing giới hạn **60 version** | Gom thay đổi rồi mới build, đừng deploy từng commit |
| `setStorage` dung lượng nhỏ | Chỉ cache; server là nguồn sự thật |
| Bị ràng buộc `zmp-ui` | Chấp nhận — ZaUI đủ dùng và làm app trông đúng chất Zalo |
| Cần eKYC Zalo cá nhân | Làm trước khi bắt đầu |
| File qua bot ≤ 5MB | Thừa sức cho Excel sổ sách 1 năm |

### 3.4. Zalo Bot Platform

API **kiểu Telegram** (`python-zalo-bot` phát triển dựa trên `python-telegram-bot`).

```
Token:       <numeric_id>:<secret>
Phương thức: getMe · sendMessage · sendPhoto · sendSticker · setWebhook
Điều kiện:   User phải CHAT VỚI BOT TRƯỚC (pairing)
Chi phí:     Miễn phí
Giới hạn:    Media ≤ 5MB
```

**Luồng kết nối bot:**
```
Trong Mini App: [ Bật thông báo Zalo ]
  → backend sinh mã ghép 6 số, lưu kèm user_id, hết hạn sau 10 phút
  → openChat() mở chat với bot
  → user gửi mã cho bot
  → bot webhook → backend khớp mã → lưu chat_id
  → bot gửi "✅ Đã kết nối! Bạn sẽ nhận tổng kết mỗi tối lúc 21:00."
  → Mini App hiện trạng thái "Đã kết nối"
```
Chưa kết nối → nhắc nhẹ ở dashboard, **không chặn** việc dùng app.

---

## 4. Kiến trúc

```
┌──────────────────────────────────────────────────┐
│  ZALO APP (Android / iOS)                        │
│  ┌────────────────────────────────────────────┐  │
│  │  Mini App — React + TS + zmp-ui + zmp-sdk  │  │
│  │  Cache nhẹ: setStorage (shop, danh mục)    │  │
│  │  KHÔNG lưu đơn hàng ở client               │  │
│  └────────────────┬───────────────────────────┘  │
│  ┌────────────────┴───────────────────────────┐  │
│  │  Chat với Bot — thông báo + nhận file      │  │
│  └────────────────────────────────────────────┘  │
└───────────────────┬──────────────────────────────┘
                    │ HTTPS
┌───────────────────┴──────────────────────────────┐
│  RAILWAY                                         │
│  ├── API service (NestJS hoặc Fastify + TS)      │
│  ├── Bot worker (webhook + cron)                 │
│  ├── PostgreSQL                                  │
│  └── Volume — ảnh hóa đơn                        │
└───────────────────┬──────────────────────────────┘
                    │
              Gemini Flash
              ├── Vision : ảnh hóa đơn → JSON
              └── Audio  : giọng nói → JSON đơn hàng
```

**Vì sao gộp hết vào Railway:** đã có sẵn, chạy được tiến trình dài hạn (bot webhook + cron) mà serverless không làm được, có Postgres và volume tích hợp. Một nơi, ít ops — đúng với dự án 1 người.

**Vì sao Gemini xử lý audio trực tiếp:** thay vì STT rồi parse (2 bước, 2 lần lỗi), đưa thẳng file âm thanh + danh mục sản phẩm của shop vào 1 prompt → nhận về JSON đơn hàng. Model nghe được ngữ điệu nên chính xác hơn.

**Ảnh hóa đơn:** lưu vào Railway volume, đường dẫn `/{store_id}/{year}/{purchase_id}.jpg`. Nén còn ~200KB trước khi lưu → 1.000 user × 20 ảnh/tháng ≈ 4GB/năm. Khi volume căng thì chuyển sang object storage, không phải sửa code nhiều nếu tách sẵn một lớp `StorageService`.

### Ước tính chi phí

| Hạng mục | 100 user | 1.000 user |
|---|---|---|
| Railway (API + Postgres + worker + volume) | $5–10 | $25–40 |
| Zalo Mini App + Bot | $0 | $0 |
| Gemini — voice (~600 lần/user/tháng) | ~$3 | ~$30 |
| Gemini — OCR (~20 ảnh/user/tháng) | ~$1 | ~$10 |
| **Tổng** | **~$12/mo** | **~$75/mo** |

→ ~**1.900 đ/user/tháng** ở 1.000 user. Trong ngân sách 3.000 đ.

---

## 5. Danh sách màn hình

| # | Màn hình | Nội dung chính |
|---|---|---|
| S1 | **Ghi đơn** *(mặc định)* | Bàn phím số · nút mic · chọn PTTT · nút Lưu · tab Nhanh/Chi tiết |
| S2 | **Dashboard** | Doanh thu hôm nay · thanh ngưỡng 1 tỷ · chart 7 ngày · số tháng/quý |
| S3 | Danh sách đơn | Theo ngày, vuốt để sửa/xóa, tìm kiếm |
| S4 | Chi tiết đơn | Xem/sửa, lịch sử thay đổi |
| S5 | Danh mục sản phẩm | Danh sách, thêm/sửa, giá bán & giá vốn |
| S6 | Nhập hàng | Danh sách phiếu nhập, nút chụp hóa đơn |
| S7 | Review OCR | Bảng dòng hàng sửa được, ảnh gốc bên cạnh |
| S8 | Báo cáo | Chart đầy đủ, chọn kỳ, nút xuất file |
| S9 | **Tình trạng thuế** | Dưới/trên ngưỡng, nút xuất sổ & tờ khai |
| S10 | Cẩm nang | Nội dung tĩnh tải từ server |
| S11 | Cài đặt | Thông tin hộ · kết nối bot · thông báo · xuất/xóa dữ liệu · chính sách riêng tư |

**Điều hướng:** tab bar 4 mục — `Ghi đơn` · `Báo cáo` · `Thuế` · `Cài đặt`. Ghi đơn luôn là tab đầu và là màn hình mở lên đầu tiên.

---

## 6. Đặc tả chi tiết

### F0 — Khởi tạo

```
Mở Mini App
  → authorize + getUserInfo   (1 chạm, không OTP)
  → "Bạn bán gì?"  [🏪 Tạp hóa]  [🍜 Quán ăn]
  → "Tên cửa hàng?" (1 ô text)
  → VÀO THẲNG MÀN HÌNH GHI ĐƠN
```
Mục tiêu: **ghi được đơn đầu tiên trong 60 giây**.

MST, địa chỉ, `getPhoneNumber`, kết nối bot → đẩy sang sau, nhắc nhẹ ở dashboard.

**Cẩm nang (F0.5)** — nội dung tĩnh **tải từ server** (sửa được mà không tốn version Testing):
- HKD là gì, khác cá nhân kinh doanh chỗ nào
- Đăng ký HKD: hồ sơ, nộp ở đâu (UBND cấp xã/phường)
- Ngưỡng 1 tỷ nghĩa là gì
- Từ 2026 có gì đổi (bỏ thuế khoán, bỏ môn bài)
- **Nghĩa vụ hộ dưới 1 tỷ**: nộp 01/TKN-CNKD trước 31/01, giữ sổ S1a-HKD
- Nếu vượt ngưỡng thì phải làm gì

Mỗi trang có disclaimer: *"Nội dung tham khảo, không phải tư vấn pháp lý."*

---

### F1 — Danh mục

| Trường | Bắt buộc | Ghi chú |
|---|---|---|
| Tên | ✅ | |
| Emoji | ❌ | Chọn nhanh bằng mắt — quan trọng với người gõ chậm |
| Đơn vị tính | ❌ | mặc định "cái" / "suất" |
| Giá bán | ❌ | Để trống được |
| Giá vốn | ❌ | Nhập tay hoặc tự đề xuất từ phiếu nhập |
| Nhóm thuế | ✅ | Mặc định theo ngành shop, override được |

**Danh mục mẫu sẵn** (tải từ server): tạp hóa ~200 mặt hàng phổ biến; quán ăn theo loại (bún/phở, cơm, cà phê, trà sữa, ăn vặt).

**Cơ chế giá vốn:**
```
Ưu tiên 1: giá user nhập tay (cost_price_is_manual = true)
Ưu tiên 2: giá từ phiếu nhập gần nhất
Khi phiếu nhập mới khớp SP và giá lệch > 5%:
   → hỏi "Cập nhật giá vốn [SP] 28.000 → 30.000?"  [Có] [Không] [Đừng hỏi nữa]
Không bao giờ ghi đè giá nhập tay mà không hỏi.
```

---

### F2 — Nhập hàng

```
chooseImage / openMediaPicker
  → nén client-side ~1024px, JPEG q80
  → POST /api/purchases/ocr
  → Railway → Gemini Vision (structured output)
  → JSON: { supplier, doc_no, doc_date,
             items[{name, qty, unit_price, amount}],
             total, confidence_per_field }
  → S7 Review: bảng dòng hàng, ô confidence thấp highlight vàng
  → khớp mờ tên hàng với danh mục shop → gợi ý "Tạo SP mới" nếu không khớp
  → user sửa → lưu → ảnh gốc lưu vào volume
```

**Yêu cầu:**
- ≤ 10 giây trên 4G
- **Fallback bắt buộc:** lỗi → form nhập tay, ảnh hiển thị bên cạnh
- Ảnh gốc luôn lưu (chứng từ khi thuế kiểm tra)
- Xử lý được: hóa đơn bán lẻ in kim · hóa đơn GTGT mẫu chuẩn · phiếu giao hàng viết tay · ảnh nghiêng/mờ

**Acceptance criteria**
- [ ] ≥ 90% đúng trường "tổng tiền"
- [ ] ≥ 85% dòng hàng đúng trên bộ test 100 hóa đơn thật
- [ ] Mọi trường sửa được

---

### F3 — Bán hàng ⭐

#### F3.1 Ghi đơn nhanh

**Tab "Nhanh"** (mặc định)
```
┌──────────────────────────┐
│      2 4 5 . 0 0 0  đ    │
├──────────────────────────┤
│   1     2     3          │
│   4     5     6          │
│   7     8     9          │
│  000    0    ⌫           │
├──────────────────────────┤
│  💵 Mặt │ 🏦 CK │ 📱 Ví  │
├──────────────────────────┤
│    🎤        [  LƯU  ]   │
└──────────────────────────┘
```

**Tab "Chi tiết"** — lưới sản phẩm có emoji, chạm thêm vào giỏ, thanh toán.

- Nút "Lưu" ≥ 64px, trong tầm ngón cái
- **Optimistic UI:** hiện xác nhận và cộng doanh thu ngay, không chờ server. Nếu server trả lỗi → rollback + toast "Chưa lưu được, thử lại" + nút Thử lại
- **Hoàn tác trong 10 giây**
- Ghi bù tối đa 7 ngày
- Sau khi lưu → quay lại bàn phím trống, sẵn sàng đơn tiếp

#### F3.2 Ghi đơn bằng giọng nói ⭐

```
Giữ nút mic
  → audio/record  (giới hạn 15 giây)
  → thả → file audio
  → POST /api/orders/voice  kèm danh mục SP của shop
  → Gemini: audio → JSON đơn hàng (1 call)
  → hiện BẢN NHÁP để xác nhận
  → user duyệt → lưu
```

**Ví dụ phải xử lý được:**
| Câu nói | Kết quả |
|---|---|
| "Hai chai nước mắm Nam Ngư ba lăm nghìn một chai" | 2 × Nước mắm Nam Ngư @35.000 = 70.000 |
| "Một trăm hai mươi nghìn" | Đơn không chi tiết, 120.000 |
| "Ba tô bún bò hai cà phê sữa đá chuyển khoản" | 3 × Bún bò + 2 × Cà phê sữa đá, PTTT = CK |
| "Hai lăm nghìn" | 25.000 |
| "Một trăm rưỡi" | 150.000 |
| "Năm trăm k" / "Nửa củ" | 500.000 |
| "Hôm qua bán được năm trăm nghìn" | Đơn hôm qua, 500.000 |

**Yêu cầu:**
- Giọng Bắc / Trung / Nam
- Hiểu số tiếng Việt kiểu nói: "hai lăm", "rưỡi", "chục", "k", "củ", "xị"
- Chịu ồn nền quán ăn
- Context-aware: khớp tên món với danh mục riêng của shop, không dùng từ điển chung
- **Luôn hiện bản nháp** — không tự lưu thẳng
- Nói nối nhiều đơn trong 1 lần bấm thì càng tốt
- Hiện đồng hồ đếm / waveform khi đang ghi để user biết đang hoạt động

**Acceptance criteria**
- [ ] ≥ 90% câu 1–3 mặt hàng parse đúng hoàn toàn
- [ ] Thả nút → bản nháp ≤ 4 giây
- [ ] Test thật trên **cả Android và iPhone** trước khi coi là xong

> ℹ️ Cộng đồng Zalo Mini App từng báo lỗi quyền microphone trên iOS. Chủ dự án xác nhận ghi âm hoạt động, nên đây không còn là rủi ro chặn — nhưng vẫn **test trên iPhone thật ở tuần đầu tiên** thay vì để tới cuối.

#### F3.4 Audit log
Mọi sửa/xóa ghi lại: khi nào, cũ → mới. Xóa là **soft delete** (`deleted_at`), không xóa vật lý. Lý do: sổ sách bị sửa xóa tùy tiện mất giá trị chứng minh khi cơ quan thuế kiểm tra.

---

### F4 — Thống kê

#### Dashboard (S2)
```
┌────────────────────────────────────┐
│  HÔM NAY                           │
│      2.450.000 đ                   │
│      18 đơn · lãi ~490.000 đ       │
│      ↑12% so hôm qua               │
├────────────────────────────────────┤
│  Doanh thu năm 2026                │
│  ████████████░░░░░░░░  62%         │
│  620.400.000 / 1.000.000.000 đ     │
│  ✅ CHƯA PHẢI NỘP THUẾ              │
│  Dự báo cả năm: ~830tr (an toàn)   │
├────────────────────────────────────┤
│  [Chart cột 7 ngày]                │
├────────────────────────────────────┤
│  Tháng 8:  68.500.000 đ            │
│  Quý III: 195.200.000 đ            │
└────────────────────────────────────┘
```
Màu thanh ngưỡng: 🟢 <70% · 🟡 70–90% · 🟠 90–100% · 🔴 >100%

#### Chart
Xu hướng doanh thu (7/30 ngày, 12 tháng) · Thanh ngưỡng · Top 10 sản phẩm (theo doanh thu / SL / lãi gộp) · Cơ cấu thanh toán (donut) · Doanh thu vs lãi gộp theo tháng · Dự báo năm.

**Dữ liệu chart tính ở server** (`/api/reports`), client chỉ vẽ → bundle nhẹ, không phụ thuộc dung lượng Mini App.

#### Lãi gộp
```
Lãi gộp = Σ (giá bán − giá vốn) × số lượng
```
Chỉ tính được trên đơn ở **chế độ Chi tiết** có sản phẩm gắn giá vốn.

Hiển thị: *"Lãi gộp hôm nay: ~490.000 đ (tính trên 12/18 đơn có chi tiết)"* — luôn nói rõ độ phủ để user không hiểu lầm. Và ghi rõ: *"Đây là lãi gộp, chưa trừ tiền thuê mặt bằng, điện nước, nhân công."*

#### Xuất file — sinh ở server, bot gửi vào chat

| File | Nội dung |
|---|---|
| `SoDoanhThu_S1a-HKD_[năm].xlsx` | Đúng mẫu S1a-HKD: cột A ngày, cột B diễn giải, cột 1 số tiền. Mặc định gộp theo ngày |
| `ChiTietBanHang_[kỳ].xlsx` | Sheet 1: từng đơn. Sheet 2: tổng hợp tháng + lũy kế |
| `SoChiPhi_[kỳ].xlsx` | Ngày, NCC, số CT, nội dung, số tiền |
| `LaiGop_[kỳ].xlsx` | SP, SL bán, doanh thu, giá vốn, lãi gộp, tỷ suất |

Header: tên hộ, MST, địa chỉ, kỳ báo cáo, dòng tổng cộng, chỗ ký. Số định dạng VNĐ.

**Luồng xuất file:**
```
User bấm "Xuất sổ" trong Mini App
  → POST /api/export/{type}
  → server sinh file (ExcelJS / @react-pdf)
  → bot sendDocument vào chat của user
  → Mini App hiện "✅ Đã gửi vào chat Zalo của bạn"  [ Mở chat ]
```
Nếu user **chưa kết nối bot** → hiện màn hình mời kết nối trước, giải thích ngắn gọn tại sao.

---

### F5 — Thuế

#### Rule engine (config ở server)

```json
{
  "version": "2026.02",
  "effective_from": "2026-05-13",
  "effective_to": null,
  "legal_basis": ["NĐ 141/2026/NĐ-CP", "NĐ 68/2026/NĐ-CP",
                  "TT 50/2026/TT-BTC", "TT 152/2025/TT-BTC"],
  "thresholds": { "tax_exempt_annual_revenue": 1000000000 },
  "warning_levels": [0.7, 0.9, 1.0],
  "sector_rates": [
    { "code": "RETAIL",  "label": "Bán lẻ, tạp hóa", "vat": 0.010, "pit": 0.005, "verified": true  },
    { "code": "FNB",     "label": "Dịch vụ ăn uống",  "vat": 0.030, "pit": 0.015, "verified": false },
    { "code": "SERVICE", "label": "Dịch vụ khác",     "vat": 0.050, "pit": 0.020, "verified": true  },
    { "code": "OTHER",   "label": "Hoạt động khác",   "vat": 0.020, "pit": 0.010, "verified": true  }
  ],
  "forms": {
    "annual_declaration": { "code": "01/TKN-CNKD", "circular": "50/2026/TT-BTC",
                            "deadline": "01-31", "applies_when": "annual_revenue <= 1000000000" },
    "revenue_ledger":     { "code": "S1a-HKD", "circular": "152/2025/TT-BTC" }
  },
  "unsupported_above_revenue": 3000000000
}
```

- Tính đúng cho kỳ quá khứ theo config có hiệu lực **tại kỳ đó**
- Config đổi → banner "Chính sách thuế đã cập nhật từ [ngày] — xem thay đổi"
- **Config ở server → sửa không tốn version Testing**
- Test bao phủ: dưới ngưỡng · vượt giữa năm · nhiều nhóm tỷ lệ trong 1 shop · chuyển năm · hộ mới thành lập 2 kỳ

#### Màn hình "Tình trạng thuế" (S9)

**Dưới ngưỡng (95% user):**
```
Năm 2026

Doanh thu lũy kế:      620.400.000 đ
Ngưỡng chịu thuế:    1.000.000.000 đ

✅ BẠN KHÔNG PHẢI NỘP THUẾ GTGT & TNCN

Nhưng bạn VẪN CẦN:
  📋 Nộp Mẫu 01/TKN-CNKD — hạn 31/01/2027
     [ Xuất tờ khai ]  [ Xem hướng dẫn ]
  📒 Giữ Sổ doanh thu S1a-HKD
     [ Xuất sổ ]

ℹ️ Nếu vượt 1 tỷ, thuế ước tính:
   1.000.000.000 × 1,5% = 15.000.000 đ
   [ Xem cách tính ]
```

**Vượt ngưỡng:**
```
⚠️ BẠN ĐÃ VƯỢT NGƯỠNG 1 TỶ

Doanh thu năm: 1.240.000.000 đ

Ước tính phải nộp:
  GTGT = 1.240.000.000 × 1,0% = 12.400.000 đ
  TNCN = 1.240.000.000 × 0,5% =  6.200.000 đ
  ──────────────────────────────────────────
  Tổng                          18.600.000 đ

Bạn cũng sẽ phải:
  • Kê khai theo QUÝ (không còn 1 lần/năm)
  • Sử dụng hóa đơn điện tử

❗ Con số này chỉ là ƯỚC TÍNH. Nên liên hệ
   đại lý thuế để kê khai chính xác.
```

Luôn có nút "Xem cách tính" → từng bước + căn cứ pháp lý + disclaimer. Không được là hộp đen.

#### Xuất tờ khai 01/TKN-CNKD
- PDF A4 sinh ở Railway, font tiếng Việt nhúng sẵn (Be Vietnam Pro / Roboto)
- Điền tự động: thông tin hộ + Phần A tách theo ngành nghề
- Phần B–E ẩn
- Kèm phụ lục "Bảng kê doanh thu theo tháng"
- Ô trống để ký tay
- Xử lý riêng hộ mới thành lập trong năm → 2 kỳ báo cáo
- Giao qua **bot**; `openWebview` để xem trước

> ⚠️ **Cảnh báo bắt buộc in trên tờ khai:**
> *"Số liệu trong tờ khai này được ghi nhận từ ngày [activation_date]. Nếu bạn có doanh thu trước ngày này trong năm [năm], vui lòng tự cộng thêm trước khi nộp."*
>
> Kèm dòng thứ hai cho user ngành ăn uống nếu có bán qua app giao đồ ăn. Đây là hai rủi ro tuân thủ **đã chấp nhận có ý thức**, không phải bỏ sót.

#### Nhắc hạn
Trước 31/01: **30 ngày · 7 ngày · 1 ngày**, qua bot.

---

### F6 — Bot Zalo

| Loại thông báo | Thời điểm | Mặc định |
|---|---|---|
| Tổng kết ngày | 21:00 (cấu hình được) | Bật |
| Tổng kết tháng | Ngày 1, 08:00 | Bật |
| Tổng kết quý | Ngày đầu quý sau, 08:00 | Bật |
| Nhắc hạn 31/01 | Trước 30/7/1 ngày | Bật |
| Cảnh báo ngưỡng | Đạt 70%/90%/100% | Bật |
| Gửi file khi user xuất | Ngay lập tức | — |

**Mẫu tin ngày:**
```
🏪 Tạp hóa Lan — 30/08/2026

💰 Doanh thu: 2.450.000 đ
🧾 Số đơn: 18
📈 So hôm qua: +12%
💵 Lãi gộp ước tính: ~490.000 đ

📅 Tháng 8: 68.500.000 đ
📊 Năm 2026: 620.400.000 đ (62% ngưỡng 1 tỷ)
✅ Chưa phải nộp thuế

(Số liệu tham khảo, không thay thế kê khai chính thức)
```

**Mẫu tin cảnh báo ngưỡng:**
```
⚠️ CẢNH BÁO — Tạp hóa Lan

Doanh thu năm 2026 đã đạt 900.000.000 đ
= 90% ngưỡng 1 tỷ.

Nếu vượt 1 tỷ bạn sẽ phải:
• Nộp thuế GTGT + TNCN (~1,5% doanh thu)
• Kê khai theo QUÝ thay vì 1 lần/năm
• Dùng hóa đơn điện tử

👉 Nên liên hệ đại lý thuế để được tư vấn.
```

**Lệnh bot** (chi phí gần 0, giá trị cao — user hỏi mà không cần mở app):
| Lệnh | Tác dụng |
|---|---|
| `/homnay` | Doanh thu hôm nay |
| `/thang` | Doanh thu tháng này |
| `/thue` | Tình trạng thuế |
| `/so` | Gửi file sổ doanh thu |
| `/tat` · `/mo` | Tắt / bật thông báo |

---

## 7. Xử lý mạng — vì online-only nên đây là phần quan trọng ⭐

Không có offline nghĩa là **mọi lỗi mạng đều đập thẳng vào mặt user**. Đây là phần dễ làm ẩu nhất và cũng là phần quyết định user có bỏ app hay không.

| Tình huống | Xử lý |
|---|---|
| Lưu đơn, mạng chậm | Optimistic UI: hiện đã lưu ngay. Spinner nhỏ ở góc. Xong thì im lặng |
| Lưu đơn thất bại | Rollback con số + toast đỏ **"Chưa lưu được"** + nút **Thử lại** giữ nguyên dữ liệu đã nhập. **Tuyệt đối không để user mất dữ liệu đã gõ** |
| Mất mạng hoàn toàn | Banner cố định trên cùng: "⚠️ Mất kết nối — không ghi được đơn". Bàn phím vẫn dùng được, nút Lưu chuyển thành **Thử lại** |
| Mạng trở lại | Banner tự biến mất, tự thử lại đơn đang treo (nếu user chưa rời màn hình) |
| Request timeout rồi thành công | `client_uuid` idempotency key ngăn ghi trùng |
| Mở app lúc không mạng | Hiện dữ liệu cache gần nhất từ `setStorage` + nhãn "Số liệu lúc 14:30" |
| OCR / voice thất bại | Fallback sang nhập tay, giữ nguyên ảnh/dữ liệu đã có |

**Ba nguyên tắc:**
1. **Không bao giờ để user mất thứ họ đã gõ hoặc nói.** Dữ liệu nhập dở luôn giữ lại trên màn hình cho tới khi lưu thành công
2. **Nói thật về trạng thái.** Đừng giả vờ đã lưu khi chưa lưu
3. **Luôn có đường thoát.** Mọi lỗi đều có nút để làm gì đó tiếp

`client_uuid` (UUID sinh ở client cho mỗi đơn, unique per store) là bảo hiểm rẻ tiền cho toàn bộ mục này — giữ lại dù đã bỏ offline.

---

## 8. API contract (phác thảo)

Mọi request kèm `Authorization: Bearer <zalo_access_token>`, backend verify với Zalo rồi map sang `user_id`.

```
POST   /api/auth/session          { }  → { user, store }         # tạo/lấy user từ Zalo token
PATCH  /api/store                 { name, sector_code, tax_code, address, ... }

GET    /api/products              → Product[]
POST   /api/products              { name, unit, sale_price, cost_price, emoji, ... }
PATCH  /api/products/:id
DELETE /api/products/:id
GET    /api/products/templates?sector=RETAIL   → danh mục mẫu

POST   /api/orders                { client_uuid, order_date, total_amount,
                                    payment_method, source, items[] }  → Order
GET    /api/orders?from=&to=&page=
PATCH  /api/orders/:id
DELETE /api/orders/:id
POST   /api/orders/voice          multipart: audio  → { draft: Order }   # chưa lưu

POST   /api/purchases             { doc_date, supplier_name, total_amount, items[] }
POST   /api/purchases/ocr         multipart: image  → { draft: Purchase }
GET    /api/purchases?from=&to=

GET    /api/reports/summary?period=today|month|quarter|year
GET    /api/reports/chart?type=trend|top_products|payment_mix&from=&to=
GET    /api/reports/profit?from=&to=

GET    /api/tax/status?year=2026   → { revenue_ytd, threshold, status,
                                       estimated_vat, estimated_pit, breakdown }
GET    /api/tax/rules              → TaxRuleConfig (theo ngày hiệu lực)

POST   /api/export/ledger          { year }        → gửi qua bot
POST   /api/export/sales           { from, to }    → gửi qua bot
POST   /api/export/declaration     { year }        → gửi PDF qua bot

POST   /api/bot/pair               → { code, expires_at }
GET    /api/bot/status             → { linked: bool }
POST   /api/bot/webhook            # Zalo Bot gọi vào
PATCH  /api/notifications          { type, enabled, send_at_hour }

GET    /api/content/guide          → nội dung cẩm nang (tĩnh, tải từ server)
POST   /api/account/export         → gửi ZIP qua bot
DELETE /api/account                → xóa toàn bộ dữ liệu
```

---

## 9. Mô hình dữ liệu

```
User(id, zalo_user_id UNIQUE, name, avatar, phone,
     bot_chat_id, bot_linked_at, created_at)

Store(id, owner_id, name, sector_code[RETAIL|FNB], tax_code, address,
      tax_office, established_date, activation_date, created_at)

Product(id, store_id, name, unit, sale_price, cost_price, cost_price_is_manual,
        cost_price_updated_at, category, emoji, tax_sector_code,
        is_active, sort_order)

Supplier(id, store_id, name, phone)

PurchaseOrder(id, store_id, supplier_id, supplier_name_raw, doc_no, doc_date,
              total_amount, source[ocr|manual], image_path, ocr_json,
              ocr_confidence, created_at, updated_at, deleted_at)
PurchaseItem(id, purchase_id, product_id, name_raw, qty, unit_price, amount)

SalesOrder(id, store_id, client_uuid, order_date, order_time,
           total_amount, cost_amount,
           payment_method[cash|transfer|wallet],
           source[quick|detail|voice], voice_transcript, note,
           created_at, updated_at, deleted_at)
  UNIQUE(store_id, client_uuid)          -- chống ghi trùng khi retry
SalesItem(id, order_id, product_id, name_raw, qty, unit_price, amount,
          cost_price, tax_sector_code)

TaxSnapshot(id, store_id, year, revenue_ytd, revenue_by_sector_json, threshold,
            status[under|near|over], estimated_vat, estimated_pit,
            rule_version, computed_at)

TaxDeclaration(id, store_id, year, period[full_year|h1|h2], form_code,
               revenue_by_sector_json, file_path, created_at)

TaxRuleConfig(version, effective_from, effective_to, payload_json)

BotPairing(id, user_id, code, expires_at, used_at)
NotificationSetting(store_id, type, enabled, send_at_hour)
NotificationLog(id, store_id, type, payload, scheduled_at, sent_at, status, error)

AuditLog(id, store_id, entity, entity_id, action, old_value, new_value, created_at)
```

**Ràng buộc:**
- `SalesOrder.order_date >= Store.activation_date`
- `SalesOrder.order_date >= today - 7 days`
- `UNIQUE(store_id, client_uuid)`
- Index: `SalesOrder(store_id, order_date)` — mọi báo cáo đều query theo đây

---

## 10. Thứ tự build

Không có deadline cố định, nên chia theo **cột mốc chức năng** thay vì tuần.

### Mốc 0 — Nền móng
- [ ] eKYC tài khoản Zalo, tạo app trong Mini App Center
- [ ] Tạo bot ở bot.zaloplatforms.com, lấy token
- [ ] `zmp create` khởi tạo project, chạy được trên máy thật (cả Android + iPhone)
- [ ] Railway: dựng API service rỗng + Postgres, deploy được
- [ ] Xong khi: mở Mini App trên điện thoại, thấy chữ "Hello" gọi từ API Railway

### Mốc 1 — Ghi được đơn *(giá trị thật đầu tiên)*
- [ ] `authorize` + `getUserInfo` → tạo User + Store
- [ ] Onboarding 3 bước
- [ ] S1 Ghi đơn — tab Nhanh, bàn phím số, PTTT, Lưu
- [ ] S3 Danh sách đơn + sửa/xóa
- [ ] Mục 7 — xử lý mạng đầy đủ
- [ ] **Đưa người test dùng thật từ đây.** Đừng đợi hoàn thiện

### Mốc 2 — Thấy được con số
- [ ] S2 Dashboard + thanh ngưỡng 1 tỷ
- [ ] `/api/reports/summary` + chart 7 ngày
- [ ] F5 rule engine + S9 Tình trạng thuế

### Mốc 3 — Giọng nói ⭐
- [ ] `audio/record` + `/api/orders/voice` + Gemini
- [ ] Màn hình bản nháp
- [ ] Test thật trên cả Android và iPhone
- [ ] Đo tỷ lệ parse đúng trên 100 câu thật

### Mốc 4 — Danh mục & lãi gộp
- [ ] S5 Danh mục + template mẫu
- [ ] Tab Chi tiết ở S1
- [ ] Lãi gộp + chart đầy đủ ở S8

### Mốc 5 — Bot
- [ ] Luồng ghép mã 6 số
- [ ] Cron tổng kết ngày/tháng/quý
- [ ] Cảnh báo ngưỡng
- [ ] Lệnh `/homnay` `/thang` `/thue`

### Mốc 6 — Nhập hàng
- [ ] S6 + S7, OCR qua Gemini Vision
- [ ] Lưu ảnh vào volume
- [ ] Cập nhật giá vốn có xác nhận

### Mốc 7 — Xuất file & tờ khai
- [ ] Sổ S1a-HKD (Excel)
- [ ] Chi tiết bán hàng, sổ chi phí, lãi gộp
- [ ] Tờ khai 01/TKN-CNKD (PDF)
- [ ] Bot gửi file vào chat
- [ ] Nhắc hạn 31/01

### Mốc 8 — Hoàn thiện
- [ ] S10 Cẩm nang
- [ ] S11 Cài đặt, xuất/xóa dữ liệu
- [ ] Chính sách riêng tư
- [ ] Nhờ kế toán review Phần 2 và file xuất ra

> **Nguyên tắc xuyên suốt:** đưa người test dùng từ **Mốc 1**, không phải Mốc 8. Mỗi mốc xong thì hỏi họ một câu: *"Từ lần trước tới giờ có gì khó chịu không?"*

---

## 11. Rủi ro

| # | Rủi ro | Mức | Giảm thiểu |
|---|---|---|---|
| **R1** | **1 dev vibe-code → scope creep, không bao giờ ship** | 🔴 Cao | Không có deadline nên rủi ro này càng cao. Ép ship Mốc 1 rồi mới làm tiếp. Mỗi mốc phải có người dùng thật |
| **R2** | **Mất mạng giữa giờ đông khách → user mất đơn, mất niềm tin** | 🔴 Cao | Mục 7 làm cho thật kỹ. Đây là cái giá của online-only |
| R3 | Voice không đủ chính xác trong quán ồn | 🟠 TB | Luôn có bàn phím số; thu log (có đồng ý) để cải prompt |
| R4 | Tờ khai năm đầu thiếu doanh thu trước ngày cài | 🟠 TB | In cảnh báo rõ trên tờ khai. **Rủi ro đã chấp nhận có ý thức** |
| R5 | Thiếu doanh thu app giao đồ ăn với user ngành ăn uống | 🟠 TB | Cảnh báo ở màn hình thuế + trên tờ khai. Cân nhắc đưa vào v1.1 sớm |
| R6 | Chính sách thuế tiếp tục đổi | 🟠 TB | Rule engine ở server, versioned theo ngày hiệu lực |
| R7 | Đốt hết 60 version Testing | 🟡 Thấp | Gom thay đổi rồi mới build; đẩy config & nội dung ra server |
| R8 | Chi phí Gemini vượt kiểm soát | 🟡 Thấp | Quota voice/ngày; giới hạn 15s/lần; model rẻ nhất |
| R9 | Ảnh hóa đơn làm đầy volume Railway | 🟡 Thấp | Nén ~200KB; tách `StorageService` để chuyển object storage sau |
| R10 | User không kết nối bot → không nhận thông báo & không nhận được file | 🟠 TB | Nhắc ở dashboard; chặn mềm ở luồng xuất file (giải thích rồi mời kết nối) |

---

## 12. Chỉ số theo dõi

| Nhóm | Chỉ số |
|---|---|
| Kích hoạt | % ghi đơn đầu trong 24h · **% kết nối bot** |
| Gắn kết | Số đơn/ngày/user · **% đơn ghi bằng giọng nói** · % mở app từ tin bot |
| Chất lượng | % voice phải sửa · % OCR phải sửa · **tỷ lệ ghi đơn thất bại do mạng** |
| Giá trị | % xem màn hình thuế · % xuất file · % xuất tờ khai |
| Chi phí | Gemini/user/tháng · Railway/tháng |

Với vài người test thì đừng dựng dashboard analytics — log ra Postgres và query tay là đủ.

---

## 13. Checklist trước khi code

- [ ] eKYC tài khoản Zalo cá nhân
- [ ] Đăng ký Mini App Center, tạo app chế độ Testing
- [ ] Tạo bot, lấy token `id:secret`, thử `sendMessage` bằng curl
- [ ] Thử bot gửi 1 file `.xlsx` vào chat — xác nhận luồng giao file chạy
- [ ] Test `audio/record` trên **iPhone thật** ở tuần đầu
- [ ] API key Gemini + đặt cảnh báo chi phí
- [ ] Railway: Postgres + volume
- [ ] Lấy file mẫu 01/TKN-CNKD và S1a-HKD bản gốc để đối chiếu layout
- [ ] Chụp lại cuốn sổ giấy của người test

---

## 14. Từ điển thuật ngữ

| Thuật ngữ | Giải thích |
|---|---|
| HKD | Hộ kinh doanh |
| GTGT / TNCN | Thuế giá trị gia tăng / Thuế thu nhập cá nhân |
| Thuế khoán | Nộp thuế cố định theo mức ấn định — **đã bỏ từ 01/01/2026** |
| Ngưỡng 1 tỷ | Doanh thu năm dưới đó thì miễn GTGT & TNCN |
| **01/TKN-CNKD** | Tờ khai/thông báo doanh thu năm cho HKD ≤1 tỷ (TT 50/2026) |
| **S1a-HKD** | Sổ doanh thu bán hàng hóa, dịch vụ (TT 152/2025) |
| Mini App | Ứng dụng chạy bên trong Zalo, không cần cài từ store |
| zmp-cli / zmp-ui / zmp-sdk | Bộ công cụ phát triển Zalo Mini App |
| eKYC | Định danh điện tử — điều kiện để cá nhân tạo Mini App |
| VLM | Vision Language Model — AI đọc ảnh, dùng thay OCR |
| Optimistic UI | Hiện kết quả ngay trước khi server xác nhận, sai thì rollback |

---

## 15. Nguồn tham khảo

**Pháp lý**
- [Mẫu 01/TKN-CNKD — TT 50/2026/TT-BTC](https://thuvienphapluat.vn/phap-luat-doanh-nghiep/bai-viet/to-khai-thue-ho-kinh-doanh-duoi-1-ty-2026-mau-01-tkn-cnkd-thong-tu-50-2026-tt-btc-21027.html)
- [Quy định về HKD dưới 1 tỷ năm 2026 (thuế, kế toán, hóa đơn)](https://thuvienphapluat.vn/phap-luat-doanh-nghiep/bai-viet/quy-dinh-ve-ho-kinh-doanh-duoi-1-ty-nam-2026-thue-ke-toan-hoa-don-20598.html)
- [Mẫu số S1a-HKD — TT 152/2025/TT-BTC](https://thuvienphapluat.vn/phap-luat/mau-so-s1ahkd-theo-thong-tu-1522025ttbtc-mau-so-doanh-thu-ban-hang-hoa-dich-vu-moi-nhat-2026-ra-sao-250054.html)
- [Hướng dẫn kê khai 01/TKN-CNKD — EasyPOS](https://easypos.vn/mau-01-tkn-cnkd/)
- [HKD dưới 1 tỷ không phải nộp tờ khai thuế quý](https://thuvienphapluat.vn/phap-luat-doanh-nghiep/bai-viet/ho-kinh-doanh-duoi-1-ty-khong-phai-nop-to-khai-thue-quy-2-2026-21022.html)
- [Nghị định 141/2026/NĐ-CP — ngưỡng 1 tỷ](https://xaydungchinhsach.chinhphu.vn/toan-van-nghi-dinh-so-141-2026-nd-cp-nang-nguong-doanh-thu-khong-phai-chiu-thue-len-1-ty-dong-119260504154326455.htm)
- [Bảng tỷ lệ thuế theo ngành — NĐ 68/2026/NĐ-CP](https://thuvienphapluat.vn/phap-luat-doanh-nghiep/bai-viet/bang-tra-cuu-ty-le-thue-ho-kinh-doanh-2026-theo-tung-nganh-nghe-theo-nghi-dinh-68-2026-nd-cp-19602.html)
- [Toàn văn TT 152/2025/TT-BTC](https://nhanh.vn/toan-van-thong-tu-1522025ttbtc-huong-dan-che-do-ke-toan-cua-ho-kinh-doanh-ca-nhan-kinh-doanh-n168240.html)
- [HKD ăn uống chịu thuế suất bao nhiêu 2026](https://sobanhang.com/ho-kinh-doanh-an-uong-chiu-thue-suat-bao-nhieu-2026/)

**Nền tảng Zalo**
- [Zalo Mini App — Tài liệu API chính thức](https://miniapp.zaloplatforms.com/docs/api)
- [Zalo Mini App — ZaUI](https://mini.zalo.me/docs/zaui/)
- [zmp-ui](https://www.npmjs.com/package/zmp-ui) · [zmp-cli](https://www.npmjs.com/package/zmp-cli)
- [Bộ skill phát triển Zalo Mini App (GitHub)](https://github.com/suminhthanh/zalo-mini-app-skills)
- [Giới hạn bản Testing và Development (60 / 300 version)](https://miniapp.zaloplatforms.com/community/2803533029779252036/gioi-han-ban-testing-va-development)
- [Cộng đồng: vấn đề quyền audio trên iOS](https://miniapp.zaloplatforms.com/community/5687569621185547586/khong-the-yeu-cau-cap-quyen-audio-tren-thiet-bi-ios)
- [python-zalo-bot — Bot API kiểu Telegram](https://pypi.org/project/python-zalo-bot/)
- [Zalo Bot Platform — ghi chú tích hợp](https://docs.openclaw.ai/channels/zalo)

> Số liệu pháp lý cần kế toán/đại lý thuế xác nhận trước khi đưa vào sản phẩm thật.

---

## 16. Điểm còn mở

Spec đã đủ để bắt đầu code. Bốn điểm dưới đây **không chặn** Mốc 0–2, có thể quyết sau:

1. **Tỷ lệ thuế ngành ăn uống — 4,5% hay 7%?** Cần kế toán xác nhận. Chỉ ảnh hưởng con số dự báo khi vượt ngưỡng, không ảnh hưởng nhóm mục tiêu. Cần trước Mốc 2.
2. **Doanh thu app giao đồ ăn.** Hiện ngoài phạm vi, nhưng với quán ăn có thể chiếm 30–50% doanh thu, và đây là phần cơ quan thuế **dễ đối chiếu nhất** vì có dữ liệu từ nền tảng. Nếu người test có bán qua app giao hàng thì nên kéo lên v1.1 ngay sau Mốc 7. Cách rẻ nhất: một ô "Doanh thu app giao hàng hôm nay" nhập gộp cuối ngày.
3. **Backend framework** — NestJS (cấu trúc rõ, nhiều boilerplate) hay Fastify (gọn, tự do hơn). Với 1 dev vibe-code thì NestJS cho code sinh ra nhất quán hơn, Fastify ship nhanh hơn. Quyết ở Mốc 0.
4. **Xuất dữ liệu khi user rời app.** Đã có `/api/account/export`, nhưng chưa quyết định lưu bao lâu sau khi user xóa tài khoản. Không gấp.
