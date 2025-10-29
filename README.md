# PromotionForm

## Mô tả
Ứng dụng web kiểm tra yêu cầu pháp lý và hồ sơ cần chuẩn bị cho các chương trình khuyến mại theo quy định Việt Nam.

## Cấu trúc Data và Mapping

### 1. Cấu trúc Data

Ứng dụng sử dụng 3 lớp cấu trúc dữ liệu chính:

#### **Layer 1: DOC_KEYS (Constants)**
Định nghĩa tất cả các documents bằng constants:

```javascript
const DOC_KEYS = {
  THONG_BAO: "Thông báo thực hiện chương trình khuyến mại được Giám đốc ký duyệt",
  PROPOSAL: "Proposal chi tiết chương trình đã được phê duyệt",
  DUTRU_NGANSACH: "Bảng dự trù ngân sách và phê duyệt ngân sách thực hiện chương trình",
  QUYET_TOAN: "Bảng quyết toán chương trình (kèm hình ảnh nghiệm thu chương trình)",
  CHUNG_TU: "Bộ chứng từ chuẩn theo quy trình thanh toán",
  DANG_KY_SO: "Hồ sơ đăng ký chương trình gửi tới Sở Công Thương...",
  // ... và các documents khác
}
```

#### **Layer 2: PROMOTION_CONFIG (Base Configuration)**
Cấu hình cơ bản cho mỗi loại hình khuyến mại, bao gồm:
- `requirement`: Yêu cầu pháp lý mặc định
- `documents`: Danh sách documents mặc định

```javascript
const PROMOTION_CONFIG = {
  "Hang mau": {
    requirement: "Không yêu cầu thủ tục.",
    documents: [DOC_KEYS.PROPOSAL, DOC_KEYS.DUTRU_NGANSACH, ...]
  },
  "Giam gia": {
    requirement: "Tổng thời gian khuyến mại không vượt quá 120 ngày/năm...",
    documents: [DOC_KEYS.THONG_BAO, DOC_KEYS.PROPOSAL, ...]
  },
  // ... các loại hình khác
}
```

#### **Layer 3: ALCOHOL_OVERRIDES (Conditional Overrides)**
Các quy tắc override dựa trên điều kiện rượu/bia ≥ 15 độ:

```javascript
const ALCOHOL_OVERRIDES = {
  yes: {
    requirements: {
      "Hang mau": "Không yêu cầu thủ tục. Hàng cho, biếu, không thu tiền"
    },
    documents: {
      add: {
        "Hang mau": [DOC_KEYS.QUYET_DINH_CHO_BIEU, DOC_KEYS.DANH_SACH_KH]
      },
      remove: {
        "Giam gia": [DOC_KEYS.THONG_BAO]
      }
    }
  },
  no: {
    documents: {
      add: { ... },
      remove: { ... }
    }
  }
}
```

### 2. Quy trình Mapping khi chọn options

Khi người dùng chọn options và click "Check Kết Quả", quy trình mapping diễn ra như sau:

#### **Bước 1: Lấy input từ user**
```javascript
const alcohol = document.querySelector('input[name="alcohol"]:checked'); // "yes" hoặc "no"
const selectedPromotions = document.querySelectorAll('input[name="promotionType"]:checked');
```

#### **Bước 2: Mapping Requirements (Yêu cầu pháp lý)**

Với mỗi promotion type được chọn:

1. **Lấy base requirement** từ `PROMOTION_CONFIG[promotionType].requirement`
2. **Kiểm tra override** nếu `alcohol === "yes"`:
   - Nếu có override trong `ALCOHOL_OVERRIDES.yes.requirements[promotionType]` → dùng override
   - Nếu không có override → giữ base requirement

```javascript
function getRequirement(promotionType, isAlcohol) {
  const config = PROMOTION_CONFIG[promotionType];
  if (isAlcohol && ALCOHOL_OVERRIDES.yes?.requirements?.[promotionType]) {
    return ALCOHOL_OVERRIDES.yes.requirements[promotionType]; // Override
  }
  return config.requirement; // Base
}
```

#### **Bước 3: Mapping Documents (Hồ sơ cần chuẩn bị)**

Với mỗi promotion type được chọn:

1. **Khởi tạo** từ base documents: `PROMOTION_CONFIG[promotionType].documents`
2. **Áp dụng removes** (nếu có):
   - Kiểm tra `ALCOHOL_OVERRIDES[yes/no].documents.remove[promotionType]`
   - Remove các documents được liệt kê
3. **Áp dụng adds** (nếu có):
   - Kiểm tra `ALCOHOL_OVERRIDES[yes/no].documents.add[promotionType]`
   - Add các documents được liệt kê
4. **Merge và deduplicate** cho tất cả promotion types được chọn

```javascript
function getDocuments(promotionType, isAlcohol) {
  const config = PROMOTION_CONFIG[promotionType];
  let docs = [...config.documents]; // Copy base documents
  
  const overrideKey = isAlcohol ? 'yes' : 'no';
  const overrides = ALCOHOL_OVERRIDES[overrideKey]?.documents;
  
  // Remove documents
  if (overrides?.remove?.[promotionType]) {
    docs = docs.filter(doc => !overrides.remove[promotionType].includes(doc));
  }
  
  // Add documents
  if (overrides?.add?.[promotionType]) {
    docs.push(...overrides.add[promotionType]);
  }
  
  return docs;
}
```

### 3. Ví dụ cụ thể

#### **Ví dụ 1: Hàng mẫu + Rượu/Bia ≥ 15 độ (Yes)**

**Input:**
- `alcohol = "yes"`
- `selectedPromotions = ["Hang mau"]`

**Mapping Requirements:**
```
Base: "Không yêu cầu thủ tục."
Override (yes): "Không yêu cầu thủ tục. Hàng cho, biếu, không thu tiền"
→ Kết quả: "Không yêu cầu thủ tục. Hàng cho, biếu, không thu tiền"
```

**Mapping Documents:**
```
Base: [PROPOSAL, DUTRU_NGANSACH, QUYET_TOAN, CHUNG_TU]
Add (yes): [QUYET_DINH_CHO_BIEU, DANH_SACH_KH]
Remove (yes): Không có
→ Kết quả: [PROPOSAL, DUTRU_NGANSACH, QUYET_TOAN, CHUNG_TU, QUYET_DINH_CHO_BIEU, DANH_SACH_KH]
```

#### **Ví dụ 2: Giảm giá + Rượu/Bia ≥ 15 độ (Yes)**

**Input:**
- `alcohol = "yes"`
- `selectedPromotions = ["Giam gia"]`

**Mapping Requirements:**
```
Base: "Tổng thời gian khuyến mại không vượt quá 120 ngày/năm..."
Override (yes): Không có
→ Kết quả: "Tổng thời gian khuyến mại không vượt quá 120 ngày/năm..." (giữ nguyên)
```

**Mapping Documents:**
```
Base: [THONG_BAO, PROPOSAL, DUTRU_NGANSACH, QUYET_TOAN, CHUNG_TU]
Add (yes): Không có
Remove (yes): [THONG_BAO]
→ Kết quả: [PROPOSAL, DUTRU_NGANSACH, QUYET_TOAN, CHUNG_TU]
```

#### **Ví dụ 3: Hàng mẫu + Rượu/Bia < 15 độ (No)**

**Input:**
- `alcohol = "no"`
- `selectedPromotions = ["Hang mau"]`

**Mapping Requirements:**
```
Base: "Không yêu cầu thủ tục."
Override (no): Không có (chỉ có override cho requirements khi yes)
→ Kết quả: "Không yêu cầu thủ tục."
```

**Mapping Documents:**
```
Base: [PROPOSAL, DUTRU_NGANSACH, QUYET_TOAN, CHUNG_TU]
Add (no): [THONG_BAO]
Remove (no): Không có
→ Kết quả: [PROPOSAL, DUTRU_NGANSACH, QUYET_TOAN, CHUNG_TU, THONG_BAO]
```

#### **Ví dụ 4: Multiple selections**

**Input:**
- `alcohol = "yes"`
- `selectedPromotions = ["Hang mau", "Giam gia"]`

**Mapping Requirements:**
```
Hang mau: "Không yêu cầu thủ tục. Hàng cho, biếu, không thu tiền"
Giam gia: "Tổng thời gian khuyến mại không vượt quá 120 ngày/năm..."
```

**Mapping Documents:**
```
Hang mau docs: [PROPOSAL, DUTRU_NGANSACH, QUYET_TOAN, CHUNG_TU, QUYET_DINH_CHO_BIEU, DANH_SACH_KH]
Giam gia docs: [PROPOSAL, DUTRU_NGANSACH, QUYET_TOAN, CHUNG_TU]

→ Merge và deduplicate:
→ [PROPOSAL, DUTRU_NGANSACH, QUYET_TOAN, CHUNG_TU, QUYET_DINH_CHO_BIEU, DANH_SACH_KH]
```

### 4. Flow Diagram

```
User Input (alcohol + selectedPromotions)
         ↓
    checkResult()
         ↓
┌─────────────────────────────┐
│  For each promotion type:   │
│                             │
│  1. getRequirement()        │
│     Base → Override? → Result│
│                             │
│  2. getDocuments()          │
│     Base → Remove → Add     │
│     → Final Documents       │
└─────────────────────────────┘
         ↓
    Merge all results
         ↓
    Display in textarea
```

### 5. Lưu ý khi mở rộng

Khi thêm promotion type mới hoặc thay đổi rules:

1. **Thêm promotion type mới:**
   - Thêm vào `PROMOTION_CONFIG` với base requirement và documents
   - Thêm vào HTML checkboxes với `value` khớp với key

2. **Thay đổi rules cho rượu/bia:**
   - Sửa `ALCOHOL_OVERRIDES.yes` hoặc `ALCOHOL_OVERRIDES.no`
   - Thêm vào `requirements` nếu muốn override requirement
   - Thêm vào `documents.add` hoặc `documents.remove`

3. **Thêm document mới:**
   - Thêm constant vào `DOC_KEYS`
   - Sử dụng constant trong `PROMOTION_CONFIG` hoặc `ALCOHOL_OVERRIDES`