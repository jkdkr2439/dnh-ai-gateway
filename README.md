# DNH AI-First IPOD Gateway 🚀

*Read this in other languages: [English](#english-version) 🇺🇸 | [Tiếng Việt](#phiên-bản-tiếng-việt) 🇻🇳*

---

<a name="english-version"></a>
## English Version

This is an Ethics Semantic Analysis System based on the **DNH framework** (Danh - Nghia - He), built strictly upon the **IPOD Architecture**.

### The IPOD Architecture
The application uses a strict unidirectional data flow model:
- **INPUT**: Receives the raw prompt, auth key, and provider selection. Responsible only for standardization and packet building.
- **PROCESS**: The core business logic. Pre-gates, enforces the DNH schema, parses responses, runs the policy reducer, and validates. DOES NOT contain I/O logic.
- **OUTPUT**: Handles provider adapters (OpenAI, Anthropic) and standardizes HTTP errors. Side-effect only wrapper.
- **DISPLAY**: Completely stateless presentation UI based on the `FinalResultDTO`.
- **DATA**: Local persistence (localStorage for Web MVP) handling Session, Audit, and Metrics.

### Folder Structure
```bash
src/
├── app.ts                  # Main Orchestrator (runs I->P->O->P->D flow)
├── shared/                 # Types, Constants, Utilities
├── input/                  # Normalization, Validation, InputPacket Builder
├── process/                # Decision Reducer, Output Validator, Post-Parse
├── output/                 # OpenAI & Anthropic Adapters (Browser-safe call)
├── data/                   # Session & Audit LocalStorage
└── display/                # Static HTML Interface
```

### How to Run (Node.js Environment)
1. Install dependencies: `npm install`
2. Build via TypeScript compiler: `npx tsc`
3. Check the output in the `/dist` directory or open `src/display/index.html` to view the UI.

> **Note**: This is a pure architectural codebase using ES6/TS modules designed to be fully local-first. It can be easily integrated into any React, Vue, or Vanilla frontend without backend proxying.

---

<a name="phiên-bản-tiếng-việt"></a>
## Phiên bản Tiếng Việt

Đây là hệ thống phân tích ngữ nghĩa đạo đức (DNH - Danh Nghĩa Hệ) được xây dựng theo kiến trúc **IPOD**.

### Kiến trúc IPOD
Ứng dụng sử dụng mô hình luồng dữ liệu một chiều rõ ràng:
- **INPUT**: Tiếp nhận raw prompt, auth key, provider selection. Chỉ chuẩn hóa và build packet.
- **PROCESS**: Trái tim nghiệp vụ. Pre-gate, xây schema bắt buộc (DNH), parse phản hồi, filter (reducer) và validation. KHÔNG chứa logic I/O.
- **OUTPUT**: Xử lý các provider adapters (OpenAI, Anthropic) và chuẩn hoá lỗi HTTP. Tuyệt đối chỉ side-effect.
- **DISPLAY**: UI hiển thị hoàn toàn thụ động (stateless presentation) dựa trên FinalResultDTO.
- **DATA**: Local persistence (localStorage cho phiên bản Web MVP) đối với Session, Audit, Metrics.

### Cấu trúc Thư mục
```bash
src/
├── app.ts                  # Main Orchestrator (chạy luồng I->P->O->P->D)
├── shared/                 # Types, Constants, Utilities
├── input/                  # Normalization, Validation, InputPacket Builder
├── process/                # Decision Reducer, Output Validator, Post-Parse
├── output/                 # OpenAI & Anthropic Adapters (Browser-safe call)
├── data/                   # Session & Audit LocalStorage
└── display/                # Giao diện HTML tĩnh
```

### Cách chạy
1. Cài đặt dependency: `npm install`
2. Build typescript compiler: `npx tsc`
3. Xem kết quả ở thư mục `/dist` hoặc mở `src/display/index.html` để tham khảo UI mẫu.

> **Lưu ý**: Đây là mã nguồn cấu trúc IPOD đầy đủ. Bạn có thể sử dụng codebase module ES6/TS này tích hợp trực tiếp vào local-first App của bạn (React/Vue/Vite) sau này mà không cần sửa core logic.
