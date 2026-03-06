# Thiết kế kiến trúc đầy đủ — DNH AI-First App (IPOD Only)

## 1. Mục tiêu tài liệu

Tài liệu này mô tả **kiến trúc vận hành** của ứng dụng DNH AI-first theo đúng mô hình **IPOD**.

Phạm vi của tài liệu này chỉ gồm:
- kiến trúc runtime của app
- ranh giới giữa các layer
- contracts dữ liệu
- luồng xử lý chính
- cách tổ chức module và trách nhiệm
- các nguyên tắc triển khai để giữ hệ ổn định, dễ test, dễ mở rộng

Tài liệu này **không** đưa vào bất kỳ framework suy luận, engine phân tích, hay module tư duy nào như một phần của kiến trúc. Chúng không thuộc architectural runtime của app này.

---

## 2. Mục tiêu sản phẩm

Ứng dụng là một **AI-first local-first analysis gateway** cho prompt text.

Người dùng nhập prompt, chọn provider AI, nhập API key của chính họ, sau đó app sẽ:
- chuẩn hóa input
- xây dựng request có cấu trúc cho AI
- yêu cầu AI phân tích theo khung DNH
- áp policy reducer cục bộ
- chạy validator hậu kiểm
- hiển thị verdict, breakdown, flags, phân tích AI, và audit log

Mục tiêu của app:
- cho AI làm semantic analysis chính
- giữ policy control ở local app
- tách rõ UI, business logic, provider integration, và data tracking
- có khả năng audit và giải thích quyết định

---

## 3. Phạm vi hệ thống

### Trong phạm vi
- web app hoặc desktop local app
- user nhập prompt text
- user chọn provider AI
- user nhập API key của chính họ
- app gọi provider trực tiếp
- app nhận structured analysis
- app áp reducer và validator
- app render kết quả
- app lưu audit/session/metrics cục bộ

### Ngoài phạm vi
- backend server trung gian
- multi-user auth
- remote database
- model training
- local semantic model riêng
- workflow orchestration nhiều agent
- plugin architecture giai đoạn đầu

---

## 4. Kiến trúc tổng thể

Kiến trúc duy nhất của hệ là **IPOD**:

```text
INPUT → PROCESS → OUTPUT → DISPLAY
                ↘ DATA
```

Ý nghĩa:
- **Input**: nhận tương tác và chuẩn hóa dữ liệu đầu vào
- **Process**: xử lý business logic, policy logic, reducer, validator logic
- **Output**: toàn bộ side effects và external integrations
- **Display**: render UI từ dữ liệu đã được xử lý
- **Data**: lưu trữ, metrics, analytics, audit records, tuning records

Mọi thành phần trong app phải map rõ vào một layer IPOD.

---

## 5. Nguyên tắc kiến trúc

1. **AI-first**
   - semantic analysis chính đến từ provider AI
   - app local không giả lập semantic understanding sâu bằng rules làm lõi

2. **Local policy authority**
   - AI được quyền phân tích
   - app local giữ quyền chốt verdict cuối thông qua reducer và validator

3. **Structured contracts trước, UI sau**
   - AI output phải parse về object chuẩn trước khi render
   - không render prose tự do như nguồn chân lý

4. **Process thuần business logic**
   - không chứa network calls
   - không mutate UI trực tiếp
   - không ghi storage trực tiếp

5. **Output cô lập side effects**
   - mọi API call, storage write, export, telemetry đều nằm ở Output

6. **Display không chứa policy logic**
   - UI chỉ hiển thị DTO đã hoàn tất
   - UI không tự classify, score, hay override verdict

7. **Data không can thiệp runtime decision theo cách ngầm**
   - metrics và analytics phục vụ quan sát và tuning
   - không tự điều chỉnh policy giữa một request đang chạy

8. **Schema-first integration**
   - tất cả dữ liệu qua ranh giới layer phải có schema rõ ràng

---

## 6. Các use case chính

### Use case 1: phân tích prompt bình thường
1. user nhập prompt
2. app chuẩn hóa input
3. app build analysis request
4. app gọi provider
5. app parse structured response
6. app chạy reducer và validator
7. app hiển thị verdict và phân tích
8. app lưu audit/session/metrics

### Use case 2: prompt mơ hồ hoặc schema lỗi
1. provider trả response thiếu trường hoặc sai schema
2. app không tin response đó là kết quả hợp lệ
3. reducer hạ verdict xuống `clarify` hoặc `reframe`
4. validator đánh dấu parse issue
5. UI hiển thị guidance an toàn thay vì final answer tự do

### Use case 3: prompt có rủi ro cao
1. provider phân tích harm cao hoặc policy conflict
2. reducer force `decline`
3. validator kiểm tra text output không lộ nội dung unsafe
4. UI hiển thị decline explanation và safe alternative nếu có

### Use case 4: provider lỗi
1. API timeout, rate limit, malformed JSON, network fail
2. app tạo error-state có cấu trúc
3. UI hiển thị lỗi provider rõ ràng
4. session vẫn được audit ở mức tối thiểu

---

## 7. Layer INPUT

### 7.1. Mục tiêu
Layer Input chịu trách nhiệm nhận tương tác từ user và biến chúng thành **input packet sạch, hợp lệ, thống nhất schema**.

### 7.2. Trách nhiệm
- nhận raw prompt
- nhận provider selection
- nhận API key
- nhận các option cấu hình phiên chạy
- chuẩn hóa text
- validate input tối thiểu
- gắn request id, session id, timestamp
- phát event hoặc tạo packet cho Process

### 7.3. Những gì Input không được làm
- không quyết định verdict
- không classify frame
- không chấm harm sâu
- không gọi provider API
- không tự lưu audit business records
- không tự render policy outcome

### 7.4. Dữ liệu vào
- raw prompt text
- provider selection
- api key
- ui options

### 7.5. Dữ liệu ra
```ts
type InputPacket = {
  requestId: string
  sessionId: string
  rawText: string
  normalizedText: string
  language: 'vi' | 'en' | 'mixed' | 'unknown'
  provider: 'openai' | 'anthropic'
  hasApiKey: boolean
  options: {
    allowAiCall: boolean
    persistSession: boolean
    includeAudit: boolean
  }
  timestamp: number
}
```

### 7.6. Input modules đề xuất
- `capturePromptInput()`
- `normalizePromptText()`
- `validatePromptInput()`
- `validateProviderSelection()`
- `validateApiKeyPresence()`
- `buildInputPacket()`

---

## 8. Layer PROCESS

### 8.1. Mục tiêu
Layer Process là **trung tâm business logic** của app.

Trong hệ này, Process chịu trách nhiệm:
- dựng request contract cho AI
- xây system prompt/developer rubric
- thực hiện pre-gate cứng
- parse structured response thành object chuẩn
- áp reducer
- chạy validator
- tạo final result DTO

### 8.2. Vai trò của Process trong app AI-first
Process không tự đóng vai semantic engine chính.
Nó là **policy control plane**:
- định nghĩa AI phải trả gì
- diễn giải output theo schema
- chốt quyết định cuối
- bảo đảm kết quả an toàn để render

### 8.3. Trách nhiệm chi tiết

#### a. Pre-gate nhẹ
Dùng cho các điều kiện hiển nhiên:
- input rỗng
- input quá dài vượt limit
- provider không hợp lệ
- API key không hiện diện khi user bật AI call
- request không đủ điều kiện gửi

#### b. Build analysis contract
Tạo schema AI bắt buộc phải tuân theo.

#### c. Build system prompt và request payload logic
Bao gồm:
- định nghĩa khung phân tích
- format output bắt buộc
- rules cho decision field
- field nào là required

#### d. Parse provider response
Biến response thành object typed nội bộ.

#### e. Policy reducer
Dùng output của AI để chốt action cuối.

#### f. Output validator
Kiểm tra:
- schema completeness
- decision consistency
- unsafe elaboration
- answer/analysis conflict
- drift giữa analysis và answer

#### g. Final result assembly
Tạo object duy nhất để Display render.

### 8.4. Những gì Process không được làm
- không gọi HTTP trực tiếp
- không write localStorage/IndexedDB trực tiếp
- không set UI state trực tiếp
- không render component
- không phụ thuộc provider SDK cụ thể

### 8.5. Core contracts trong Process

#### DnhAnalysis
```ts
type DnhAnalysis = {
  danh: {
    label: string
    evidence: string[]
  }
  nghia: {
    label: string
    confidence: number
    evidence: string[]
  }
  he: {
    primary: 'ethical' | 'legal' | 'technical' | 'mixed' | 'ambiguous'
    secondary: string[]
    confidence: number
  }
  tension: {
    detected: boolean
    type: 'hedge' | 'role' | 'goal' | 'frame' | 'none'
    explanation: string
  }
  harm: {
    severity: 'none' | 'low' | 'medium' | 'high'
    categories: string[]
    rationale: string
  }
  decision: {
    action: 'proceed' | 'clarify' | 'reframe' | 'decline'
    reason: string
  }
}
```

#### ModelResponsePayload
```ts
type ModelResponsePayload = {
  analysis: DnhAnalysis
  response: {
    userFacingText: string
    safeReframe?: string
    followupQuestion?: string
  }
}
```

#### ReducedDecision
```ts
type ReducedDecision = {
  action: 'proceed' | 'clarify' | 'reframe' | 'decline'
  reason: string
  overridden: boolean
  overrideReason?: string
}
```

#### ValidationResult
```ts
type ValidationResult = {
  schemaValid: boolean
  consistencyValid: boolean
  safeToRender: boolean
  flags: string[]
  validatorSummary: string
}
```

#### FinalResultDTO
```ts
type FinalResultDTO = {
  requestId: string
  sessionId: string
  provider: 'openai' | 'anthropic'
  verdict: ReducedDecision
  analysis: DnhAnalysis | null
  response: {
    userFacingText: string
    safeReframe?: string
    followupQuestion?: string
  }
  validation: ValidationResult
  auditRef?: string
  timestamps: {
    startedAt: number
    completedAt: number
  }
}
```

### 8.6. Decision precedence
Process phải có reducer precedence cứng:

1. schema invalid → `clarify`
2. harm high → `decline`
3. he ambiguous → `reframe`
4. tension detected nhưng AI đề xuất `proceed` → `clarify`
5. nếu không conflict → chấp nhận decision từ AI

### 8.7. Process modules đề xuất
- `preGateInput()`
- `buildAnalysisSchema()`
- `buildSystemPrompt()`
- `buildProviderRequest()`
- `parseModelResponse()`
- `reduceDecision()`
- `validateModelOutput()`
- `buildFinalResultDTO()`

---

## 9. Layer OUTPUT

### 9.1. Mục tiêu
Layer Output chịu trách nhiệm cho **mọi side effect**:
- gọi API provider
- lưu session
- lưu audit
- xuất dữ liệu
- telemetry write

### 9.2. Trách nhiệm
- gọi OpenAI hoặc Anthropic
- retry / timeout / cancel request
- parse HTTP-level response
- lưu raw provider response nếu policy cho phép
- lưu audit record
- lưu session record
- phát signal lỗi I/O có cấu trúc

### 9.3. Những gì Output không được làm
- không tự quyết định final verdict
- không tự sửa policy reducer
- không tự render UI
- không nhúng business logic của Process

### 9.4. Provider abstraction
App phải dùng abstraction chung cho provider adapters.

```ts
type ProviderRequest = {
  requestId: string
  provider: 'openai' | 'anthropic'
  apiKey: string
  systemPrompt: string
  userPrompt: string
  responseFormat: 'json'
  schemaVersion: string
}
```

```ts
type ProviderRawResult = {
  requestId: string
  provider: 'openai' | 'anthropic'
  success: boolean
  rawText?: string
  errorCode?: string
  errorMessage?: string
  latencyMs: number
  receivedAt: number
}
```

### 9.5. Output modules đề xuất
- `callOpenAI()`
- `callAnthropic()`
- `handleProviderError()`
- `saveAuditRecord()`
- `saveSessionRecord()`
- `saveMetricsSnapshot()`
- `exportResultJson()`

### 9.6. Error classes cần hỗ trợ
- network timeout
- network unavailable
- rate limit
- auth invalid
- malformed JSON
- provider schema mismatch
- storage write fail
- audit save fail

### 9.7. Fallback behavior
- provider fail → final result với error state có cấu trúc
- parse fail → reducer chuyển về `clarify`
- storage fail → không chặn UI render nếu verdict đã an toàn
- audit save fail → hiển thị warning riêng, không crash app

---

## 10. Layer DISPLAY

### 10.1. Mục tiêu
Display chỉ chịu trách nhiệm trình bày kết quả cho user.

### 10.2. Trách nhiệm
- render input form
- render loading state
- render provider error state
- render final verdict badge
- render analysis breakdown
- render response panel
- render validation flags
- render audit summary
- render session history view

### 10.3. Những gì Display không được làm
- không tự chạy reducer
- không parse raw provider JSON thành business object
- không re-score harm hoặc tension
- không gọi provider trực tiếp
- không tự sửa decision object

### 10.4. UI sections đề xuất
- Prompt Input Panel
- Provider Config Panel
- Submit Controls
- Status Strip
- Verdict Badge
- DNH Breakdown Panel
- User-Facing Analysis Panel
- Validation Warning Panel
- Audit Summary Panel
- Session History Panel

### 10.5. Display state model
Display chỉ nên nhận:
- `idle`
- `submitting`
- `providerError`
- `completed`
- `completedWithWarnings`

### 10.6. Display modules đề xuất
- `PromptComposerView`
- `ProviderSettingsView`
- `SubmitActionBar`
- `LoadingView`
- `ProviderErrorView`
- `VerdictBadgeView`
- `AnalysisBreakdownView`
- `ResponseTextView`
- `ValidationFlagsView`
- `AuditSummaryView`
- `SessionHistoryView`

---

## 11. Layer DATA

### 11.1. Mục tiêu
Data chịu trách nhiệm quản lý dữ liệu vận hành và quan sát hệ.

### 11.2. Trách nhiệm
- lưu audit records
- lưu session history
- lưu settings cục bộ
- lưu metrics
- lưu thống kê provider failures
- lưu validation outcomes
- lưu prompt/version metadata nếu cần

### 11.3. Những gì Data không được làm
- không tự can thiệp reducer trong runtime
- không tự sửa session hiện tại theo side effect ngầm
- không đóng vai nguồn chân lý thay cho Process

### 11.4. Nhóm dữ liệu chính

#### a. Session data
- request history
- provider used
- timestamps
- final verdict

#### b. Audit data
- input packet snapshot
- provider raw result reference
- parsed analysis snapshot
- reduced decision
- validation result

#### c. Metrics data
- request count
- provider latency
- provider failure rate
- parse failure rate
- validator warning rate
- verdict distribution

#### d. Settings data
- UI preferences
- persistence preferences
- last selected provider

### 11.5. Data stores đề xuất
- `sessionStore`
- `auditStore`
- `metricsStore`
- `settingsStore`

---

## 12. Luồng dữ liệu end-to-end

## 12.1. Happy path

```text
User nhập prompt
  ↓
INPUT chuẩn hóa và tạo InputPacket
  ↓
PROCESS pre-gate InputPacket
  ↓
PROCESS build system prompt + schema + ProviderRequest
  ↓
OUTPUT gọi provider
  ↓
OUTPUT trả ProviderRawResult
  ↓
PROCESS parse ModelResponsePayload
  ↓
PROCESS reduceDecision
  ↓
PROCESS validateModelOutput
  ↓
PROCESS build FinalResultDTO
  ↓
DISPLAY render FinalResultDTO
  ↓
DATA lưu session/audit/metrics
```

## 12.2. Provider failure path

```text
OUTPUT gọi provider
  ↓
provider lỗi / timeout / rate limit
  ↓
OUTPUT tạo structured provider error
  ↓
PROCESS build fallback result
  ↓
DISPLAY render provider error state
  ↓
DATA lưu metrics + audit tối thiểu
```

## 12.3. Invalid schema path

```text
provider trả text không parse được
  ↓
PROCESS parse fail
  ↓
PROCESS reduceDecision -> clarify
  ↓
PROCESS validation gắn flag schemaInvalid
  ↓
DISPLAY render warning + clarify guidance
```

---

## 13. Trình tự xử lý runtime chi tiết

### Bước 1: Input intake
- nhận raw text
- trim whitespace
- normalize unicode nếu cần
- detect language sơ bộ
- validate provider selection
- validate key presence
- tạo `InputPacket`

### Bước 2: Pre-gate
- chặn input rỗng
- chặn cấu hình thiếu provider/key
- quyết định có được gửi sang Output hay không

### Bước 3: Build analysis request
- tạo system prompt
- tạo response schema
- gói `ProviderRequest`

### Bước 4: Provider call
- Output gọi provider tương ứng
- theo timeout budget
- ghi latency
- nhận raw text/json

### Bước 5: Parse response
- parse JSON
- validate required fields
- map sang `ModelResponsePayload`

### Bước 6: Reduce decision
- áp precedence rules
- tạo `ReducedDecision`

### Bước 7: Validate output
- check schema validity
- check consistency
- check render safety
- tạo `ValidationResult`

### Bước 8: Build final result
- ghép tất cả thành `FinalResultDTO`

### Bước 9: Render
- Display dùng `FinalResultDTO`
- không đụng business logic nữa

### Bước 10: Persist data
- session store
- audit store
- metrics store

---

## 14. Hợp đồng dữ liệu chính

### 14.1. InputPacket
Nguồn chân lý duy nhất từ Input sang Process.

### 14.2. ProviderRequest
Nguồn chân lý duy nhất từ Process sang Output khi gọi provider.

### 14.3. ProviderRawResult
Nguồn chân lý duy nhất từ Output về Process sau khi gọi provider.

### 14.4. ModelResponsePayload
Dữ liệu có cấu trúc sau parse.

### 14.5. FinalResultDTO
Nguồn chân lý duy nhất từ Process sang Display.

### 14.6. AuditRecord
Nguồn chân lý để lưu vết xử lý.

```ts
type AuditRecord = {
  auditId: string
  requestId: string
  sessionId: string
  inputSnapshot: InputPacket
  providerResultSummary: {
    provider: 'openai' | 'anthropic'
    success: boolean
    latencyMs: number
    errorCode?: string
  }
  parsedAnalysis: DnhAnalysis | null
  reducedDecision: ReducedDecision
  validation: ValidationResult
  createdAt: number
}
```

---

## 15. Ràng buộc giữa các layer

### Input → Process
- chỉ truyền qua `InputPacket`
- không truyền UI component references

### Process → Output
- chỉ truyền qua `ProviderRequest` hoặc các storage command DTO
- không truyền closure UI state

### Output → Process
- chỉ trả `ProviderRawResult` hoặc structured storage result
- không tự gọi reducer

### Process → Display
- chỉ truyền `FinalResultDTO`
- Display không được truy cập raw provider response để tự suy luận lại

### Process/Data
- Process có thể yêu cầu lưu record
- Data không tự override Process result

---

## 16. Quy tắc bất biến kiến trúc

1. Không có provider SDK call trong Process.
2. Không có network logic trong Display.
3. Không có reducer logic trong Display.
4. Không có direct local storage write từ Display.
5. Mọi render final answer phải dựa trên `FinalResultDTO`.
6. Mọi decision cuối phải đi qua reducer.
7. Mọi answer hiển thị phải đi qua validator result.
8. Mọi API integration phải đi qua Output adapters.
9. Mọi dữ liệu qua ranh giới layer phải có schema.
10. Không dùng raw provider prose làm nguồn chân lý trực tiếp.

---

## 17. Phân tách module theo feature trong IPOD

```text
src/
  features/
    prompt-intake/
      input/
      process/
      output/
      display/
      data/

    ai-analysis/
      input/
      process/
      output/
      display/
      data/

    result-review/
      input/
      process/
      output/
      display/
      data/

  shared/
    types/
    schemas/
    utils/
    constants/
```

### Gợi ý giải thích
- `prompt-intake`: quản lý form và packet đầu vào
- `ai-analysis`: build request, gọi provider, parse analysis
- `result-review`: reducer, validator, render final result, audit

Mỗi feature vẫn phải giữ cấu trúc IPOD nội bộ.

---

## 18. Phân tách module theo layer toàn cục

Nếu không muốn feature-based structure, có thể dùng layer-based structure:

```text
src/
  input/
  process/
  output/
  display/
  data/
  shared/
```

### Khuyến nghị
- bản MVP nhỏ: layer-based đủ dùng
- bản sẽ mở rộng dài hạn: feature-based + IPOD con rõ hơn

---

## 19. Thiết kế provider integration

### 19.1. Mục tiêu
Giữ provider integration thay thế được mà không làm bẩn Process.

### 19.2. Provider adapter contract
```ts
interface ProviderAdapter {
  call(request: ProviderRequest): Promise<ProviderRawResult>
}
```

### 19.3. Adapter responsibilities
- map request nội bộ sang request format của provider
- gửi HTTP request
- handle auth header
- handle timeout
- nhận raw response
- trả `ProviderRawResult`

### 19.4. Không đưa vào adapter
- decision precedence
- validator logic
- UI messaging logic
- storage policy logic

---

## 20. Thiết kế reducer

### 20.1. Mục tiêu
Reducer là authority cục bộ để chốt verdict cuối sau khi nhận phân tích từ AI.

### 20.2. Input của reducer
- parsed `DnhAnalysis`
- parse status
- provider error status

### 20.3. Output của reducer
- `ReducedDecision`

### 20.4. Reducer rules
- parse fail → `clarify`
- provider fail → `clarify` hoặc `error-state`, tùy policy hiển thị
- harm high → `decline`
- ambiguous frame → `reframe`
- tension conflict → `clarify`
- trường hợp hợp lệ → theo AI decision

### 20.5. Lý do cần reducer
- giữ app là authority policy cuối
- tránh giao toàn quyền quyết định cho provider AI
- bảo đảm consistency để audit và test

---

## 21. Thiết kế validator

### 21.1. Mục tiêu
Validator hậu kiểm kết quả để xem có an toàn và nhất quán để render không.

### 21.2. Validator checks
- đủ schema không
- analysis và response có mâu thuẫn không
- verdict và userFacingText có mâu thuẫn không
- answer có vượt quá action không
- decline mà vẫn trả nội dung actionable thì fail
- reframe mà lại có answer direct thì flag

### 21.3. Output
- `ValidationResult`

### 21.4. Chính sách render
- `safeToRender = true` → render bình thường
- `safeToRender = false` → render fallback safe text + warning

---

## 22. Error handling architecture

### 22.1. Phân loại lỗi
- Input errors
- Process contract errors
- Provider errors
- Parse errors
- Validation errors
- Data persistence errors
- Display rendering errors

### 22.2. Quy tắc xử lý
- lỗi ở layer nào thì layer đó chuẩn hóa error trước khi phát ra ngoài
- không ném raw exception xuyên qua toàn bộ app nếu có thể tránh
- mọi lỗi quan trọng phải map sang structured error object

### 22.3. Error shape đề xuất
```ts
type AppError = {
  code: string
  layer: 'input' | 'process' | 'output' | 'display' | 'data'
  message: string
  recoverable: boolean
  requestId?: string
}
```

---

## 23. Audit architecture

### 23.1. Mục tiêu
Lưu được dấu vết xử lý đủ để:
- giải thích verdict
- debug issue
- soát provider failure
- soát validator conflicts

### 23.2. Audit tối thiểu nên lưu
- input snapshot
- provider used
- parse success/failure
- reduced decision
- validation flags
- timestamps

### 23.3. Audit không nên lưu mặc định
- API key raw
- mọi dữ liệu nhạy cảm không cần thiết
- full raw transcript nếu user không bật persistence phù hợp

---

## 24. Session architecture

### 24.1. Mục tiêu
Quản lý chuỗi request trong một phiên làm việc local.

### 24.2. Session responsibilities
- nhóm request theo session id
- cho phép xem lại lịch sử verdict
- tách session khỏi audit internals

### 24.3. Session record đề xuất
```ts
type SessionRecord = {
  sessionId: string
  requestIds: string[]
  startedAt: number
  updatedAt: number
}
```

---

## 25. Metrics architecture

### 25.1. Mục tiêu
Cho phép quan sát chất lượng vận hành của app.

### 25.2. Metrics chính
- request count
- success rate theo provider
- failure rate theo provider
- average latency
- parse failure rate
- validation warning rate
- verdict distribution

### 25.3. Nơi dùng metrics
- dashboard nội bộ local
- debugging
- tuning prompt/schema/provider settings ở các phiên sau

---

## 26. Security và privacy architecture

### 26.1. Nguyên tắc
- API key thuộc user
- không lưu raw key nếu không thật sự cần
- không gửi dữ liệu đến backend riêng của app
- provider call đi trực tiếp từ app nếu môi trường cho phép
- dữ liệu local persistence phải là opt-in hoặc tối thiểu hóa

### 26.2. Quy tắc tối thiểu
- không log API key
- không ghi key vào audit record
- không render raw errors làm lộ key
- không dùng raw response làm nguồn chân lý trước khi parse

---

## 27. Testability theo kiến trúc IPOD

### 27.1. Input tests
- normalize text
- validate provider
- validate key presence
- build packet

### 27.2. Process tests
- build schema
- reduce decision
- validate output
- build final DTO

### 27.3. Output tests
- provider adapter mapping
- timeout handling
- malformed response handling
- storage failure handling

### 27.4. Display tests
- render completed state
- render provider error
- render validation warnings
- render decline/reframe/clarify/proceed variants

### 27.5. Data tests
- save/load session
- save audit
- metrics aggregation

---

## 28. MVP triển khai tối thiểu

### Input
- prompt form
- provider select
- api key input
- normalize cơ bản

### Process
- pre-gate
- build system prompt
- build schema
- parse response
- reducer
- validator

### Output
- OpenAI adapter
- Anthropic adapter
- save audit local

### Display
- verdict badge
- breakdown
- analysis text
- warnings
- provider error state

### Data
- session store
- audit store
- metrics store đơn giản

---

## 29. Khả năng mở rộng sau MVP

- thêm provider mới mà không chạm Process core
- thêm export JSON/PDF mà không chạm reducer
- thêm session compare view mà không chạm provider adapters
- thêm prompt presets mà không phá Input contracts
- thêm validator checks mà không sửa Display

---

## 30. Kiến trúc chốt cuối cùng

Ứng dụng này có kiến trúc IPOD như sau:

### INPUT
Nhận và chuẩn hóa prompt, provider config, API key, options.

### PROCESS
Xây contract cho AI, build request, parse kết quả có cấu trúc, reduce decision, validate output, build final DTO.

### OUTPUT
Gọi provider API, lưu audit, lưu session, lưu metrics, xử lý mọi side effects.

### DISPLAY
Hiển thị form, trạng thái, verdict, breakdown, response, flags, audit summary.

### DATA
Quản lý session records, audit records, metrics, settings, history cục bộ.

---

## 31. Kết luận

Kiến trúc của app này chỉ là **IPOD**.

- **Input** chịu trách nhiệm intake và chuẩn hóa.
- **Process** là policy control plane và business logic lõi.
- **Output** là vùng side effects và external integrations.
- **Display** là UI thuần render.
- **Data** là lớp lưu trữ và quan sát hệ.

Trong mô hình này:
- AI provider là capability bên ngoài, được gọi qua Output.
- DNH logic thuộc domain logic trong Process.
- verdict cuối chỉ hợp lệ sau reducer và validator.
- UI chỉ hiển thị kết quả đã được chuẩn hóa thành DTO.

Đây là bản kiến trúc đầy đủ, đúng boundary, đủ để đi tiếp sang spec cấp module, interface, file structure, và implementation plan.

