# AI OCR PoC — Kế hoạch Chi tiết

## 1. Giới thiệu Chức năng

### 1.1. Chức năng này là gì?

**AI OCR (Optical Character Recognition)** là module cho phép nhân viên chụp ảnh hóa đơn VAT và hệ thống **tự động đọc, trích xuất** toàn bộ thông tin trên hóa đơn thành dữ liệu có cấu trúc (JSON), sau đó **đối chiếu với ngân sách** công ty để xác định hóa đơn hợp lệ hay không.

### 1.2. Vì sao cần chức năng này?

| Vấn đề hiện tại | Giải pháp FlexiTrip |
|-----------------|---------------------|
| Nhân viên phải tự nhập thủ công từng khoản chi | AI đọc hóa đơn tự động trong vài giây |
| Kế toán phải kiểm tra từng hóa đơn bằng mắt | Policy Engine đối chiếu tự động |
| Dễ gian lận (sửa số tiền, hóa đơn giả) | AI cross-check nội dung + MST hợp lệ |
| Quy trình hoàn ứng chậm (3-5 ngày) | Kết quả tức thì → trigger Smart Contract hoàn tiền |

### 1.3. Chức năng này hoạt động ra sao trong dự án?

```
NHÂN VIÊN đi công tác
    │
    ▼
Ăn uống / Khách sạn → Nhận hóa đơn VAT
    │
    ▼
Mở app FlexiTrip → Chụp ảnh hóa đơn    ← ① INPUT
    │
    ▼
AI OCR trích xuất thông tin               ← ② XỬ LÝ
    │
    ▼
Policy Engine đối chiếu ngân sách         ← ③ ĐÁNH GIÁ
    │
    ├── ✅ Hợp lệ → Smart Contract hoàn tiền + Cộng Trust Token
    │
    └── ❌ Vượt mức → Thông báo nhân viên, chờ phê duyệt thủ công
```

> **Vai trò trong hệ thống:** Module OCR là **cầu nối** giữa thế giới thực (hóa đơn giấy) và Blockchain (Smart Contract). Không có OCR thì không thể tự động hóa hoàn ứng.

---

## 2. Tech Stack

| Thành phần | Công nghệ | Version | Vai trò |
|-----------|-----------|---------|---------|
| **Runtime** | Node.js | ≥ 18 | Môi trường chạy JavaScript |
| **AI/OCR** | Google Gemini API | `gemini-2.5-flash` | Vision AI đọc ảnh hóa đơn |
| **Gemini SDK** | `@google/generative-ai` | latest | SDK chính thức gọi Gemini API |
| **Env Config** | `dotenv` | latest | Load API key từ file `.env` |
| **Module System** | ES Modules | — | `import/export` syntax |
| **Test** | Node.js built-in `--test` | — | Unit test không cần thêm lib |

### Tại sao chọn Gemini Vision API?

- ✅ **Miễn phí** cho quota sinh viên (free tier đủ dùng)
- ✅ **Multimodal** — cùng 1 API xử lý được cả ảnh lẫn text
- ✅ **Đọc tiếng Việt** tốt — không cần tiền xử lý ngôn ngữ
- ✅ **JSON output** — prompt engineering để AI trả trực tiếp JSON
- ✅ **Cùng hệ sinh thái** — dùng lại cho AI Planner module sau

### Tại sao không dùng Tesseract / Google Cloud Vision / AWS Textract?

| Lựa chọn | Lý do bỏ |
|----------|----------|
| Tesseract OCR | Độ chính xác thấp với tiếng Việt, cần tiền xử lý ảnh phức tạp |
| Google Cloud Vision | Cần billing account, pricing phức tạp |
| AWS Textract | Cần AWS account, cấu hình IAM phức tạp |
| GPT-4 Vision | Tốn phí, không có free tier đủ dùng |

---

## 3. Luồng Xử lý Chi tiết

### 3.1. Sơ đồ Tổng quan

```
Ảnh hóa đơn (JPG/PNG/WEBP, max 5MB)
    │
    ▼
┌──────────────────┐
│  ① Đọc file      │  fs.readFileSync() → Buffer
│     Input Layer   │  Detect MIME type (image/jpeg, image/png)
│                   │  Convert → Base64 string
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│  ② Gemini Vision │  Gửi base64 image + Prompt template
│     AI Layer     │  Model: gemini-2.5-flash
│                  │  Prompt: "Trích xuất hóa đơn VAT → JSON"
│                  │  Response: raw text chứa JSON
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│  ③ Parser &      │  Loại bỏ markdown (```json ... ```)
│     Validator    │  JSON.parse() → Object
│                  │  Validate schema (trường bắt buộc?)
│                  │  Normalize tiền (string → number)
│                  │  Cross-check: subtotal + VAT = total?
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│  ④ Policy Engine │  So sánh grand_total vs hạn mức cấp bậc
│     Logic Layer  │  Kiểm tra tax_id format (10 hoặc 13 số)
│                  │  Tổng hợp: VALID / OVER_BUDGET / INVALID
└────────┬─────────┘
         │
         ▼
   KẾT QUẢ CUỐI CÙNG
   {
     ocr: { ...dữ liệu trích xuất... },
     policy: { status, message, details }
   }
```

### 3.2. Chi tiết từng bước

#### Bước ①: Đọc file ảnh
- Nhận đường dẫn file từ command line argument
- Kiểm tra file tồn tại (`fs.existsSync`)
- Đọc file thành Buffer (`fs.readFileSync`)
- Xác định MIME type dựa trên extension (`.jpg` → `image/jpeg`)
- Convert sang Base64 (`buffer.toString('base64')`)
- **Giới hạn:** max 5MB, chỉ hỗ trợ JPG/PNG/WEBP

#### Bước ②: Gọi Gemini Vision API
- Tạo request với 2 phần: **image** (base64) + **text** (prompt)
- **Prompt yêu cầu AI đọc TẤT CẢ thông tin** có trên hóa đơn → trả về JSON
- AI tự phát hiện và trích xuất mọi trường: tên cửa hàng, MST, danh sách món, giá, VAT, v.v.
- **Không giới hạn trường cố định** — hóa đơn có gì thì AI trả về cái đó
- Gọi `model.generateContent([image, prompt])`
- Nhận response text (có thể chứa markdown code block)
- **Timeout:** 30 giây, retry 1 lần nếu lỗi mạng

#### Bước ③: Parse & Validate
- Loại bỏ ` ```json ``` ` wrapper nếu có
- `JSON.parse()` → object
- **Giữ nguyên TẤT CẢ trường** AI trả về — không cắt bớt
- Chỉ validate tối thiểu để biết OCR có thành công hay không:
  - Có `grand_total` (tổng tiền) không? → Cần cho Policy Engine
  - JSON có parse được không? → Không thì báo lỗi
- Normalize số tiền nếu AI trả string: `"1,200,000"` → `1200000`
- Cross-check nếu có đủ dữ liệu: `subtotal + vat_amount ≈ grand_total`

> **Triết lý:** AI là bộ não đọc hiểu — hóa đơn có thông tin gì thì trả về thông tin đó. Parser chỉ "dọn dẹp" format, KHÔNG giới hạn trường.

#### Bước ④: Đối chiếu Policy
- Lấy hạn mức theo `employeeRole` từ bảng `POLICY_LIMITS`
- 3 phép kiểm tra:
  1. **Budget check:** `grand_total ≤ approvedBudget`?
  2. **Policy check:** `grand_total ≤ limit_per_day`?
  3. **Tax ID check:** `tax_id` match pattern `/^\d{10}(\d{3})?$/`?
- Xếp loại cuối cùng:
  - `VALID` — Tất cả 3 check đều pass
  - `OVER_BUDGET` — Vượt ngân sách
  - `INVALID_TAX_ID` — MST không hợp lệ
  - `INVALID_DATA` — Thiếu trường bắt buộc hoặc parse lỗi

---

## 4. Dữ liệu — Input & Output

### 4.1. Input: Ảnh hóa đơn VAT Việt Nam

Hóa đơn VAT tiêu chuẩn Việt Nam gồm:
- **Header:** "HÓA ĐƠN GIÁ TRỊ GIA TĂNG"
- **Thông tin người bán:** Tên, địa chỉ, MST
- **Thông tin người mua:** Tên, địa chỉ (nếu có)
- **Bảng chi tiết:** STT, Tên hàng, ĐVT, Số lượng, Đơn giá, Thành tiền
- **Tổng kết:** Cộng tiền hàng, Thuế GTGT (8% hoặc 10%), Tổng thanh toán
- **Chữ ký:** Người mua, Người bán

### 4.2. Output: JSON từ AI

> **Lưu ý quan trọng:** JSON dưới đây là **ví dụ minh họa**, KHÔNG phải schema cứng. AI sẽ tự trích xuất TẤT CẢ thông tin có trên hóa đơn. Hóa đơn nào có nhiều thông tin thì JSON sẽ có nhiều trường, hóa đơn nào ít thì JSON sẽ ít trường.

**Ví dụ: Hóa đơn nhà hàng đầy đủ thông tin**
```json
{
  "store_name": "NHÀ HÀNG HẢI SẢN BIỂN ĐÔNG",
  "store_address": "123 Trần Phú, Q. Hải Châu, TP. Đà Nẵng",
  "store_phone": "0236-1234567",
  "tax_id": "0312345678",
  "invoice_number": "0000123",
  "invoice_series": "AA/23E",
  "invoice_date": "16/05/2026",
  "cashier": "Nguyễn Văn A",
  "items": [
    {
      "name": "Cơm chiên hải sản",
      "quantity": 2,
      "unit": "phần",
      "unit_price": 150000,
      "total": 300000
    },
    {
      "name": "Lẩu thái hải sản",
      "quantity": 1,
      "unit": "nồi",
      "unit_price": 350000,
      "total": 350000
    }
  ],
  "subtotal": 650000,
  "discount": 0,
  "vat_rate": "10%",
  "vat_amount": 65000,
  "service_charge": 0,
  "grand_total": 715000,
  "payment_method": "Tiền mặt",
  "buyer_name": "CÔNG TY ABC",
  "buyer_tax_id": "0109876543",
  "notes": "Bàn 5, tầng 2"
}
```

**Ví dụ: Hóa đơn siêu thị đơn giản (ít trường hơn)**
```json
{
  "store_name": "MINIMART 24H",
  "store_address": "45 Lê Lợi, Q.1, TP.HCM",
  "invoice_date": "17/05/2026",
  "items": [
    { "name": "Nước suối Aquafina", "quantity": 2, "total": 20000 },
    { "name": "Bánh mì sandwich", "quantity": 1, "total": 35000 }
  ],
  "grand_total": 55000,
  "payment_method": "Momo"
}
```

> **Điểm khác biệt:** Hóa đơn nhà hàng có MST, VAT, buyer info. Hóa đơn minimart có thể chỉ có tên cửa hàng + danh sách mua + tổng tiền. AI trả về đúng những gì nó đọc được, **không bịa thêm trường không có**.
```

### 4.3. Output: Policy Result

```json
{
  "status": "VALID",
  "checks": {
    "budget_ok": true,
    "policy_ok": true,
    "tax_id_ok": true
  },
  "details": {
    "employee_role": "employee",
    "approved_budget": 1000000,
    "actual_amount": 825000,
    "daily_limit": 500000,
    "difference": 175000
  },
  "message": "✅ Hợp lệ - trong ngân sách được duyệt"
}
```

---

## 5. Bảng Hạn mức Chi tiêu (Policy)

| Cấp bậc | Ăn uống / ngày | Khách sạn / đêm | Tổng / chuyến |
|---------|----------------|-----------------|---------------|
| **Giám đốc** (director) | 1,000,000đ | 2,000,000đ | 20,000,000đ |
| **Trưởng phòng** (manager) | 700,000đ | 1,500,000đ | 15,000,000đ |
| **Nhân viên** (employee) | 500,000đ | 1,000,000đ | 10,000,000đ |

> Bảng này được hardcode trong `policyConfig.js`. Trong web app chính, HR/Admin sẽ có giao diện để sửa giá trị.

---

## 6. Cấu trúc Thư mục PoC

```
poc/ocr/
├── PLAN.md                    ← File này
├── package.json               ← Dependencies
├── .env                       ← GEMINI_API_KEY (KHÔNG commit)
├── .gitignore                 ← Bảo vệ .env, node_modules
│
├── src/
│   ├── geminiClient.js        ← Khởi tạo Gemini API client
│   ├── ocrService.js          ← Gọi Vision API, trích xuất hóa đơn
│   ├── ocrPrompt.js           ← Prompt template cho OCR
│   ├── ocrParser.js           ← Parse & validate JSON từ AI
│   ├── policyEngine.js        ← Đối chiếu ngân sách
│   └── policyConfig.js        ← Bảng hạn mức theo cấp bậc
│
├── test/
│   ├── test_ocr.js            ← Test end-to-end (chạy full luồng)
│   ├── test_parser.js         ← Unit test parser riêng
│   └── test_policy.js         ← Unit test policy riêng
│
└── sample-invoices/           ← Ảnh hóa đơn mẫu
    ├── valid_invoice.png      ← Hóa đơn hợp lệ (dưới ngân sách)
    ├── over_budget.png        ← Hóa đơn vượt mức
    └── invalid_format.png     ← Ảnh không phải hóa đơn (test edge case)
```

---

## 7. Chi tiết Từng File

### 7.1. `src/geminiClient.js`

| Thuộc tính | Giá trị |
|-----------|---------|
| **Chức năng** | Khởi tạo Gemini client, export sẵn model |
| **Dependencies** | `@google/generative-ai`, `dotenv` |
| **Export** | `getModel(name?)` — trả về GenerativeModel instance |

```
Load .env → Lấy GEMINI_API_KEY
    → new GoogleGenerativeAI(key)
    → Return model instance
```

---

### 7.2. `src/ocrPrompt.js`

| Thuộc tính | Giá trị |
|-----------|---------|
| **Chức năng** | Export prompt text tiếng Việt |
| **Export** | `OCR_PROMPT` — string |

**Triết lý prompt:**
- Yêu cầu AI **đọc TẤT CẢ thông tin** hiển thị trên hóa đơn
- KHÔNG liệt kê trước danh sách trường cố định
- AI tự quyết định hóa đơn này có những trường gì → trả về đầy đủ
- Chỉ gợi ý format output (JSON), format ngày (`DD/MM/YYYY`), số tiền là number
- Yêu cầu KHÔNG bịa thêm thông tin không có trên hóa đơn

**Ví dụ ý tưởng prompt (không phải code cuối):**
> *"Hãy đọc ảnh hóa đơn này và trích xuất TẤT CẢ thông tin bạn nhìn thấy. Trả về JSON. Số tiền dùng kiểu number. Ngày dùng format DD/MM/YYYY. Chỉ trả về những gì thực sự có trên hóa đơn, không bịa thêm."*

---

### 7.3. `src/ocrService.js`

| Thuộc tính | Giá trị |
|-----------|---------|
| **Chức năng** | Đọc ảnh + gọi Gemini Vision API |
| **Input** | `filePath: string` |
| **Output** | `{ rawText: string, processingTime: number }` |

```
Nhận filePath
    → Kiểm tra file tồn tại
    → Đọc file → Buffer → Base64
    → Detect MIME type
    → Gọi model.generateContent([imagePart, promptPart])
    → Return raw response text + thời gian xử lý
```

---

### 7.4. `src/ocrParser.js`

| Thuộc tính | Giá trị |
|-----------|---------|
| **Chức năng** | Parse raw AI text → structured JSON, giữ nguyên tất cả trường |
| **Input** | `rawText: string` |
| **Output** | `{ success: boolean, data: object, warnings: string[] }` |

**Nguyên tắc:** Giữ nguyên TẤT CẢ trường AI trả về — không cắt, không lọc.

**Công việc của parser:**
- AI trả markdown ` ```json ... ``` ` → strip wrapper
- AI trả thêm text giải thích trước/sau JSON → regex tìm `{ ... }`
- Tìm tất cả trường có giá trị tiền (dạng string `"800,000"`) → convert thành number `800000`
- Kiểm tra tối thiểu: JSON parse được? Có trường `grand_total` (hoặc tương đương) không?
- Nếu thiếu tổng tiền → `warnings: ["Không tìm thấy tổng tiền"]` (vẫn trả data)
- JSON parse lỗi → return `{ success: false, errors: ["Invalid JSON"] }`

> **Không phải việc của parser:** Quyết định hóa đơn cần có những trường gì. Đó là việc của AI.

---

### 7.5. `src/policyConfig.js`

| Thuộc tính | Giá trị |
|-----------|---------|
| **Chức năng** | Export bảng hạn mức theo cấp bậc |
| **Export** | `POLICY_LIMITS: object` |

*(Xem bảng ở mục 5)*

---

### 7.6. `src/policyEngine.js`

| Thuộc tính | Giá trị |
|-----------|---------|
| **Chức năng** | Đối chiếu OCR result với ngân sách |
| **Input** | `{ ocrData, employeeRole, approvedBudget }` |
| **Output** | `{ status, checks, details, message }` |

**Logic xếp loại:**
```
IF ocrData thiếu trường bắt buộc → INVALID_DATA
IF tax_id không match /^\d{10}(\d{3})?$/ → INVALID_TAX_ID
IF grand_total > approvedBudget → OVER_BUDGET
IF grand_total > policy_limit → OVER_BUDGET  
ELSE → VALID
```

---

### 7.7. `test/test_ocr.js`

**Cách chạy:**
```powershell
cd poc/ocr
npm install
node test/test_ocr.js                                # Ảnh mẫu mặc định
node test/test_ocr.js "C:\path\to\hoadon.png"        # Ảnh tùy chọn
node test/test_ocr.js --role=director --budget=2000000  # Tùy chỉnh role + budget
```

**Output kỳ vọng:**
```
═══════════════════════════════════════
  FlexiTrip - AI OCR PoC
═══════════════════════════════════════

📷 Ảnh: valid_invoice.png (149 KB)
🤖 Đang gọi Gemini Vision API...
⏱️  Xử lý: 8.5s

📋 KẾT QUẢ OCR:
─────────────────────────────────────
  Cửa hàng:    NHÀ HÀNG HẢI SẢN BIỂN ĐÔNG
  MST:         0312345678
  Ngày:        16/05/2026
  Số hóa đơn:  0000123
  ─────────────
  Items:
    1. Cơm chiên hải sản  x2  = 300,000đ
    2. Lẩu thái hải sản   x1  = 350,000đ
    3. Nước ngọt           x4  = 100,000đ
  ─────────────
  Cộng tiền:   750,000đ
  VAT 10%:     75,000đ
  TỔNG:        825,000đ

📋 ĐÁNH GIÁ POLICY:
─────────────────────────────────────
  Cấp bậc:     Nhân viên
  Ngân sách:    1,000,000đ
  Thực tế:      825,000đ
  Trạng thái:   ✅ HỢP LỆ

═══════════════════════════════════════
```

---

## 8. Xử lý Lỗi & Edge Cases

| Tình huống | Xử lý |
|-----------|-------|
| File không tồn tại | Báo lỗi rõ ràng: "Không tìm thấy file: {path}" |
| File quá lớn (> 5MB) | Từ chối, gợi ý giảm kích thước |
| File không phải ảnh | Báo lỗi: "Chỉ hỗ trợ JPG, PNG, WEBP" |
| Ảnh mờ / nghiêng | AI vẫn cố đọc, `confidence_score` sẽ thấp |
| Ảnh không phải hóa đơn | AI trả JSON rỗng / lỗi, parser báo INVALID_DATA |
| API key hết hạn / sai | Báo lỗi rõ: "Kiểm tra GEMINI_API_KEY trong .env" |
| Gemini API timeout | Retry 1 lần, sau đó báo lỗi mạng |
| JSON parse lỗi | Parser trả `success: false` + error message |

---

## 9. Checklist Thực hiện

- [ ] Khởi tạo project: `npm init -y`, set `"type": "module"`
- [ ] Cài dependencies: `npm install @google/generative-ai dotenv`
- [ ] Tạo `.env` với `GEMINI_API_KEY=...`
- [ ] Tạo `.gitignore` (bảo vệ `.env`, `node_modules/`)
- [ ] Code `src/geminiClient.js`
- [ ] Code `src/ocrPrompt.js`
- [ ] Code `src/ocrService.js`
- [ ] Code `src/ocrParser.js`
- [ ] Code `src/policyConfig.js`
- [ ] Code `src/policyEngine.js`
- [ ] Code `test/test_ocr.js` (end-to-end)
- [ ] Chuẩn bị ≥2 ảnh hóa đơn mẫu trong `sample-invoices/`
- [ ] Test thành công với ảnh mẫu
- [ ] Test thành công với ảnh hóa đơn thật (chụp bằng điện thoại)
- [ ] Code `test/test_parser.js` (unit test parser riêng)
- [ ] Code `test/test_policy.js` (unit test policy riêng)

---

## 10. Tiêu chí Hoàn thành PoC

| # | Tiêu chí | Yêu cầu |
|---|----------|---------|
| 1 | OCR trích xuất đúng | Tên, MST, items, tổng tiền — chính xác ≥ 90% |
| 2 | Parser validate | Phát hiện trường thiếu, kiểu sai, cross-check tổng |
| 3 | Policy phân loại đúng | Hợp lệ / Vượt mức / MST sai — đúng 100% |
| 4 | Chạy độc lập | `node test/test_ocr.js` OK mà không cần web app |
| 5 | Module hóa | Mỗi file 1 chức năng, dễ import vào Next.js sau |
| 6 | Xử lý lỗi | Không crash khi input sai, luôn có thông báo rõ ràng |
