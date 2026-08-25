# Realtime Examiner — Product & Functional Requirements

**Document 1 / 2 — Product & Functional Requirements**
**Đối tượng đọc:** Product, Business Stakeholder, Operation, QA, Legal/Compliance (không yêu cầu nền tảng kỹ thuật)
**Trạng thái tài liệu:** Rev 5 — Rev 2 cập nhật 5 quyết định Stakeholder (mock test, luồng dữ liệu HeyGen, restart Part, số câu hỏi, scoring ngoài scope); Rev 3 tiếp thu BA review: chốt định vị LiveAvatar = Avatar/Streaming layer (khuyến nghị LITE Mode), tách Exam Session vs Avatar Session, nâng Restart Part thành Core Business Rule, tách latency metrics, bổ sung cost model, ưu tiên lại Open Questions theo P0/P1/P2. Rev 4: Stakeholder chốt #4 — cơ chế câu hỏi **Hybrid** (mục 4.4). Rev 5: bổ sung rà soát sơ bộ DPA HeyGen (mục 13.5).
**Ngày:** 2026-08-24

**Quy ước đánh dấu (áp dụng cho MỌI kết luận trong tài liệu):**

| Nhãn | Ý nghĩa |
|---|---|
| ✅ **Confirmed Requirement** | Đã được nêu rõ trong yêu cầu ban đầu, coi là đã chốt |
| 🟡 **Assumption** | Giả định hợp lý, cần Stakeholder xác nhận trước khi coi là chốt |
| 💡 **Recommendation** | Đề xuất của Product/BA, Stakeholder có thể chấp nhận hoặc điều chỉnh |
| ❓ **Open Question – Need Confirmation** | Chưa có quyết định, phải chốt trước/ trong quá trình development |
| ⛔ **Out of Scope / Future Phase** | Không thuộc phạm vi phase này (đã đánh dấu rõ lý do) |

---

## 0. Executive Summary

Realtime Examiner là tính năng cho phép học viên luyện/thi IELTS Speaking bằng cách hội thoại trực tiếp, theo thời gian thực, với một **AI Examiner** hiển thị dưới dạng avatar (sử dụng **LiveAvatar API của HeyGen** — định vị là **lớp Avatar/Streaming**, không phải "bộ não" bài thi; khuyến nghị dùng **LITE Mode** để IEG tự chủ STT/LLM/TTS và conversation logic). Hệ thống IEG kết hợp **STT** (chuyển giọng nói học viên thành văn bản), **LLM** (phân tích câu trả lời và đề xuất câu hỏi tiếp theo), **TTS** (tạo giọng nói Examiner), còn LiveAvatar hiển thị avatar realtime. Bài thi gồm 3 Part theo đúng format IELTS Speaking (Interview – Long Turn – Discussion), mỗi Part 3 phút, với **3 behavior profile riêng biệt** cho Examiner vì bản chất tương tác của từng Part khác nhau. Toàn bộ câu hỏi, câu trả lời, audio, transcript và metadata được lưu lại phục vụ scoring/review/audit. Topic/question do **IEG quản lý** — AI không tự quyết định đề thi.

**Các quyết định đã chốt (Stakeholder xác nhận 2026-08-24):**
1. ✅ Đây là **mock test**, không tính vào kết quả chính thức → toàn bộ mục Anti-cheat/Proctoring (mục 14) là **Out of Scope**.
2. ✅ Học viên nhiều lứa tuổi. **HeyGen chỉ cung cấp Avatar AI** (khuôn mặt hiển thị + có thể cấu hình context); **toàn bộ dữ liệu bài thi trả về và lưu tại hệ thống IEG**.
3. ✅ **Refresh hoặc mất mạng → học viên phải làm lại toàn bộ Part đang dở** (các Part đã hoàn thành được giữ).
4. ✅ Số câu hỏi: **Part 1: 6–10 câu; Part 2: 1 câu (cue card); Part 3: 4–6 câu follow-up theo topic Part 2**.
5. ✅ **Scoring thuộc một hệ thống khác của IEG** — ngoài scope tính năng này; Realtime Examiner chỉ cần lưu và bàn giao đầy đủ dữ liệu (audio, transcript, câu hỏi, metadata).
6. ✅ **Cơ chế câu hỏi: Hybrid** — IEG cung cấp câu hỏi gốc/cue card/seed question; AI được generate follow-up bám theo câu trả lời (chi tiết từng Part ở mục 4.4).

**Mức độ sẵn sàng hiện tại: PARTIALLY READY (tiệm cận READY)** — business flow, Part-specific behavior và 5 quyết định lớn đã chốt; còn lại chủ yếu là tham số cấu hình và các hạng mục Legal/consent (ghi âm, học viên dưới 18 tuổi — vẫn cần Legal review vì học viên nhiều lứa tuổi), retention, và NFR (peak concurrency, giới hạn gói HeyGen).

---

## 1. Overview

### 1.1 Realtime Examiner là gì

Realtime Examiner là một "giám khảo ảo": học viên mở bài thi trên trình duyệt, nhìn thấy một avatar giám khảo, nghe giám khảo chào hỏi và đặt câu hỏi bằng giọng nói, trả lời bằng micro của mình, và giám khảo phản hồi/hỏi tiếp một cách tự nhiên — giống trải nghiệm ngồi trước giám khảo IELTS thật. ✅ **Confirmed Requirement**

### 1.2 UX mong muốn

- Hội thoại **liên tục, độ trễ thấp**, cảm giác "đang nói chuyện với người thật".
- Avatar có hình ảnh, cử động môi khớp với lời nói (HeyGen LiveAvatar). ✅ **Confirmed Requirement**
- Học viên luôn biết mình đang ở Part nào, còn bao nhiêu thời gian, và trạng thái hệ thống (đang nghe / đang suy nghĩ / đang nói). 💡 **Recommendation**

### 1.3 Tại sao cần / khác gì Speaking test hiện tại

| | Speaking test hiện tại (giả định: ghi âm trả lời theo đề có sẵn hoặc thi với giáo viên) | Realtime Examiner |
|---|---|---|
| Tương tác | Một chiều hoặc phụ thuộc lịch giáo viên | Hai chiều, realtime, 24/7 |
| Follow-up | Không có hoặc cố định | AI sinh follow-up dựa trên câu trả lời thực tế |
| Trải nghiệm | Ít giống thi thật | Mô phỏng sát format thi thật (3 Part, giám khảo hỏi–đáp) |
| Chi phí vận hành | Cần giáo viên/giám khảo | Tự động, scale được |

🟡 **Assumption:** mô tả "Speaking test hiện tại" ở trên là giả định — cần IEG xác nhận hiện trạng để phần so sánh chính xác.

### 1.4 AI Examiner hoạt động thế nào (mức business)

Vòng lặp cơ bản: **Examiner chào hỏi → đặt câu hỏi → Student trả lời (mic) → hệ thống chuyển giọng nói thành text (STT) → AI phân tích (LLM) → quyết định follow-up hoặc câu hỏi tiếp theo → chuyển thành giọng nói (TTS) → Avatar nói → lặp lại → hết Part → chuyển Part → hết bài thi.** ✅ **Confirmed Requirement**

**Nguyên tắc quan trọng (business rule cốt lõi):** AI Examiner chỉ **đề xuất** hành động hội thoại tiếp theo (hỏi follow-up, chuyển câu, kết thúc lượt). **Quyền quyết định thay đổi trạng thái bài thi** (chuyển Part, kết thúc bài, dừng timer) thuộc về **hệ thống backend/orchestrator**, không thuộc về AI. Điều này đảm bảo bài thi công bằng, đúng giờ, không bị AI "tự ý" thay đổi. Chi tiết kỹ thuật ở Document 2. ✅ **Confirmed Requirement**

### 1.4b Định vị kiến trúc: LiveAvatar là lớp Avatar/Streaming, không phải "Exam Brain"

LiveAvatar có hai chế độ (theo tài liệu LiveAvatar):
- **FULL Mode:** LiveAvatar quản lý toàn bộ pipeline ASR → LLM → TTS → WebRTC.
- **LITE Mode:** developer tự cung cấp AI stack (STT/LLM/TTS); LiveAvatar phụ trách realtime video streaming của avatar.

💡 **Recommendation (mạnh): Phase 1 dùng LiveAvatar LITE Mode.** Lý do: yêu cầu cốt lõi của IEG là kiểm soát chặt conversation logic, question control, behavior profile, state machine và dữ liệu scoring — những thứ này phải nằm trong hệ thống IEG (Orchestrator + AI Examiner), không giao cho LiveAvatar. FULL Mode sẽ làm việc kiểm soát question/state/context khó hơn đáng kể.

Luồng kiến trúc mức business (LITE):

```
              HỆ THỐNG IEG
┌───────────────────────────────────────┐
│ Exam Frontend                             │
│     │                                     │
│ Exam Orchestrator                         │
│     ├─ Exam State / Timer                 │
│     ├─ Topic / Question Config (IEG)      │
│     ├─ Part Behavior Profile ×3           │
│     └─ Conversation Rules                 │
│     │                                     │
│ AI Examiner (STT → LLM → TTS của IEG)     │
└─────│─────────────────────────────────┘
      │ audio đã tạo sẵn
      ▼
 LiveAvatar LITE (streaming) → Avatar video → Student
```

Hệ quả nghiệp vụ quan trọng: **việc nhận định "Student đã nói xong" (ANSWER_COMPLETED) thuộc conversation engine của IEG** (VAD/STT + rule theo behavior profile từng Part — Part 2 xử lý im lặng khác hẳn Part 1/3), **không để LiveAvatar quyết định**. 💡 Recommendation — nguyên tắc kiến trúc cho Document 2.

### 1.5 Các thành phần hệ thống cần có (mức business)

| Thành phần | Trách nhiệm (mức business) | Nhãn |
|---|---|---|
| **Exam Frontend (web)** | Hiển thị avatar, timer, trạng thái; thu âm mic; check thiết bị | ✅ Confirmed (suy ra từ flow) |
| **Exam Orchestrator (backend)** | Nguồn chân lý về trạng thái bài thi: tạo session, chạy timer, chuyển Part, kết thúc bài, ghi log sự kiện | ✅ Confirmed (từ nguyên tắc 1.4) |
| **AI Examiner (LLM + behavior profile)** | Phân tích câu trả lời, sinh/chọn câu hỏi và follow-up theo rule từng Part | ✅ Confirmed |
| **STT service** | Chuyển giọng nói Student → text realtime | ✅ Confirmed |
| **TTS (IEG)** | Chuyển câu hỏi/response của Examiner thành audio | ✅ Confirmed; TTS nằm phía IEG nếu chọn LITE 💡 |
| **LiveAvatar (HeyGen)** | Lớp Avatar/Streaming: nhận audio/response đã tạo và hiển thị qua avatar realtime | ✅ Confirmed (định vị lại theo BA review) |
| **Topic/Question Management (IEG)** | IEG tạo/duyệt/quản lý topic, question, cue card, cấu hình bài thi | ✅ Confirmed |
| **Recording & Transcript Storage** | Lưu audio, transcript, câu hỏi, metadata sau bài thi | ✅ Confirmed |
| **Scoring system (hệ thống IEG khác)** | Chấm điểm sau bài thi — ngoài scope Realtime Examiner; chỉ cần handoff dữ liệu | ✅ Confirmed (mục 12) |
| **Monitoring/Logging** | Theo dõi lỗi, latency, sự kiện bất thường | 💡 Recommendation |

### 1.6 Data lưu trữ vs data chỉ tồn tại trong realtime session

| Data | Lưu lâu dài? | Nhãn |
|---|---|---|
| Audio câu trả lời của Student | **Có** | ✅ Confirmed |
| Câu hỏi Examiner đã hỏi (kèm thứ tự, Part, timestamp) | **Có** | ✅ Confirmed |
| Transcript Student | **Có** | ✅ Confirmed |
| Transcript Examiner | **Có** (yêu cầu ghi "nếu có" → khuyến nghị lưu vì Examiner text luôn sẵn có trước khi TTS) | 💡 Recommendation |
| Exam session ID, trạng thái, timestamp các mốc | **Có** | ✅ Confirmed |
| Video/avatar stream | ❓ Open Question (mục 10) | ❓ |
| Audio thô của Examiner (TTS output) | Không cần — có thể tái tạo từ text | 💡 Recommendation |
| Trạng thái tạm của realtime connection (WebSocket/WebRTC state, buffer audio đang stream) | Không — chỉ tồn tại trong session | 🟡 Assumption |
| Kết quả STT trung gian (partial transcript) | Không lưu bản trung gian, chỉ lưu bản final | 💡 Recommendation |

---

## 2. User Journey

### 2.1 Before exam (trước khi thi)

1. Học viên mở bài thi từ danh sách bài thi/khoá học. 🟡 Assumption về entry point — cần xác nhận.
2. Hệ thống **tạo exam session** (session ID, trạng thái `NOT_STARTED` → `INITIALIZING`). ✅ Confirmed
3. **Consent screen** (thông báo ghi âm/ghi hình, mục đích sử dụng) — hiển thị trước khi vào thi. ❓ Open Question / Need Legal Review (mục 13) — nội dung và tính bắt buộc do Legal quyết.
4. **Check thiết bị:** yêu cầu quyền micro, test mic (thanh âm lượng), khuyến nghị dùng tai nghe. ✅ Confirmed (mic check được nêu trong flow)
5. **Khởi tạo AI Examiner:** load cấu hình bài thi từ IEG (topic, question set, số câu hỏi, thời gian chuẩn bị Part 2...), khởi tạo phiên HeyGen LiveAvatar, kết nối realtime. ✅ Confirmed
6. Hiển thị **trạng thái khởi tạo** cho học viên ("Đang chuẩn bị giám khảo..."), chỉ cho bắt đầu khi mọi thành phần sẵn sàng (`READY`). 💡 Recommendation
7. Học viên bấm **Start** → chuyển `PART_1`, **timer Part 1 bắt đầu**. 💡 Recommendation: timer chỉ bắt đầu khi học viên chủ động bấm Start (không tự chạy trong lúc khởi tạo) — cần xác nhận.

**Khi nào tạo session / bắt đầu timer:**
- Tạo session: khi học viên mở bài thi và hệ thống bắt đầu khởi tạo. 🟡 Assumption
- Timer từng Part: bắt đầu khi Part đó chính thức bắt đầu (Examiner bắt đầu greeting/câu hỏi đầu của Part), do **server** ghi nhận. 💡 Recommendation (xem mục 7)

### 2.2 During exam (trong khi thi)

Vòng lặp mỗi lượt hỏi–đáp:

1. Examiner **greeting** (chỉ đầu bài thi) rồi **đặt câu hỏi** (avatar nói).
2. Student **trả lời** qua mic; hệ thống hiển thị trạng thái "đang nghe".
3. **STT** chuyển câu trả lời thành text (realtime).
4. **Conversation engine của IEG** nhận định Student **đã nói xong** (VAD/STT + ngưỡng im lặng theo behavior profile từng Part) — không giao quyết định này cho LiveAvatar. Ngưỡng im lặng cụ thể: ❓ Need Confirmation (tham số cấu hình theo Part).
5. **AI phân tích** câu trả lời (trong lúc này UI hiển thị Examiner "đang lắng nghe/suy nghĩ" một cách tự nhiên — xem mục 5).
6. AI **đề xuất**: hỏi follow-up, hoặc chuyển câu hỏi tiếp theo. Orchestrator xác nhận (còn thời gian không, còn câu hỏi không, đúng rule Part không).
7. **TTS + Avatar** nói câu tiếp theo. Lặp lại từ bước 2.

✅ Confirmed Requirement cho vòng lặp tổng thể; các ngưỡng/tham số bên trong là ❓ Need Confirmation.

### 2.3 Part transition (chuyển Part)

- **Part 1 kết thúc khi:** hết 3 phút (server timer), **hoặc** đã hoàn thành toàn bộ câu hỏi được cấu hình trước khi hết giờ (nếu cho phép kết thúc sớm — ❓ Open Question, mục 18).
- Khi hết giờ giữa chừng: Examiner **kết thúc lịch sự** ("Thank you. Let's move on to Part 2.") thay vì cắt đột ngột. 💡 Recommendation: cho phép một khoảng "grace" ngắn để Examiner nói câu chuyển tiếp; grace không cộng vào thời gian trả lời của Student. ❓ Độ dài grace cần xác nhận.
- **Part 2 bắt đầu:** Examiner giới thiệu cue card → hiển thị cue card trên màn hình → **thời gian chuẩn bị** (thông lệ IELTS: 1 phút, nhưng ở đây là tham số ❓ Configurable / Need Confirmation, và cần chốt: thời gian chuẩn bị có nằm trong 3 phút của Part 2 hay cộng thêm — ❓ Open Question) → Student nói long turn, Examiner **không ngắt lời**.
- **Part 3 bắt đầu:** Examiner chuyển sang câu hỏi thảo luận **liên quan chủ đề Part 2**. ✅ Confirmed
- **Timer khi chuyển Part:** timer Part cũ dừng và chốt; timer Part mới là bộ đếm **riêng, mới** (không cộng dồn thời gian thừa/thiếu giữa các Part). 💡 Recommendation — cần xác nhận.
- Việc chuyển Part do **orchestrator quyết định** (AI chỉ đề xuất / thực hiện lời thoại chuyển tiếp). ✅ Confirmed

### 2.4 Exam completion (kết thúc bài thi)

- Bài thi **hoàn thành** khi Part 3 kết thúc (hết 3 phút hoặc hoàn thành interaction — xem mục 11). Examiner nói lời kết thúc chuẩn ("That is the end of the speaking test. Thank you.").
- Hệ thống **finalize**: đóng realtime connection, chốt và upload các audio/transcript còn lại, ghi metadata cuối cùng.
- Session chuyển `PART_3 → COMPLETING → COMPLETED`. Nếu finalize lỗi (ví dụ upload audio fail), session ở `COMPLETING` và retry nền — học viên vẫn thấy màn hình "Hoàn thành". 💡 Recommendation
- **Scoring bắt đầu** (nếu trong scope) sau khi session `COMPLETED` — asynchronous (mục 12). 💡 Recommendation
- Học viên thấy màn hình hoàn thành: xác nhận đã nộp, thông tin khi nào có kết quả/feedback. 💡 Recommendation

---

## 3. IELTS Speaking Structure & Part-specific Behavior

### 3.1 Bảng tổng quan

| Part | Purpose | Duration | Interaction Style | Examiner Behavior |
|---|---|---|---|---|
| Part 1 | Introduction & Interview | 3 phút ✅ | AI ↔ Student, câu hỏi ngắn, đổi chủ đề nhanh | Hỏi nhiều câu ngắn, follow-up ít, chuyển topic con thường xuyên |
| Part 2 | Long Turn | 3 phút ✅ | AI → Student, monologue | Đưa cue card, cho thời gian chuẩn bị, **KHÔNG ngắt lời** trong lúc Student nói, chỉ dừng khi hết giờ |
| Part 3 | Discussion | 3 phút ✅ | AI ↔ Student, câu hỏi trừu tượng/phân tích | Hỏi sâu, follow-up nhiều hơn Part 1, câu hỏi so sánh/quan điểm, liên quan chủ đề Part 2 |

### 3.2 Phân tích hành vi Examiner từng Part

#### Part 1 — Interview
- **Nhịp độ:** nhiều câu hỏi ngắn, mỗi câu trả lời kỳ vọng 20–40 giây; Examiner chủ động chuyển topic con (2–3 topic con trong 3 phút theo thông lệ IELTS). 🟡 Assumption dựa trên format IELTS thật — số topic con/câu hỏi là ❓ Configurable / Need Confirmation.
- **Số câu hỏi:** **6–10 câu** ✅ Confirmed (Stakeholder chốt) — số cụ thể trong khoảng này do IEG cấu hình theo topic set.
- **Preparation time:** Không có. 🟡 Assumption theo format IELTS.
- **Ngắt lời:** Examiner **được phép** kết thúc lượt một cách lịch sự nếu Student trả lời quá dài so với tính chất Part 1 (ví dụ chèn "Thank you." khi Student vượt ngưỡng thời gian trả lời — ngưỡng ❓ Need Confirmation), giống giám khảo thật.
- **Follow-up:** Được dùng nhưng **hạn chế** — khuyến nghị tối đa 1 follow-up/câu hỏi gốc. 💡 Recommendation; số tối đa là ❓ Configurable / Need Confirmation.

#### Part 2 — Long Turn
- **Số câu hỏi:** **1 câu (cue card)** ✅ Confirmed (Stakeholder chốt).
- **Cue card:** luôn là đề tĩnh do IEG chuẩn bị (mục 4). Hiển thị cả trên màn hình và được Examiner đọc.
- **Preparation time:** Có — thông lệ IELTS là 1 phút, học viên được ghi chú. ❓ Configurable / Need Confirmation: độ dài, có cho ghi chú không, và **prep time nằm trong hay ngoài 3 phút Part 2**.
- **Ngắt lời:** Examiner **gần như tuyệt đối KHÔNG ngắt lời**. Chỉ hai ngoại lệ: (a) hết giờ Part (do orchestrator ra lệnh, Examiner kết thúc lịch sự); (b) Student im lặng kéo dài — Examiner được nhắc nhẹ ("You can start whenever you're ready" / gợi ý theo cue card). ✅ Confirmed cho nguyên tắc không ngắt; ngưỡng im lặng ❓ Need Confirmation.
- **Student nói xong sớm:** Examiner có thể hỏi 1 rounding-off question ngắn (thông lệ IELTS) hoặc chuyển Part sớm — ❓ Open Question (mục 16, 18).
- **Follow-up:** Không dùng follow-up phân tích trong lúc long turn; tối đa 1 rounding-off question sau khi Student nói xong. 💡 Recommendation.

#### Part 3 — Discussion
- **Nhịp độ:** câu hỏi sâu hơn, trừu tượng hơn, câu trả lời kỳ vọng dài hơn Part 1; ít câu hỏi hơn Part 1.
- **Số câu hỏi:** **4–6 câu**, follow-up theo topic Part 2 ✅ Confirmed (Stakeholder chốt) — số cụ thể trong khoảng do IEG cấu hình.
- **Liên kết chủ đề:** câu hỏi phải liên quan chủ đề cue card Part 2. ✅ Confirmed
- **Preparation time:** Không có. 🟡 Assumption theo format IELTS.
- **Ngắt lời:** tương tự Part 1 — được kết thúc lượt lịch sự khi câu trả lời quá dài; được đào sâu ("Why do you think that?") khi câu trả lời quá ngắn.
- **Follow-up:** Dùng **nhiều hơn Part 1** — khuyến nghị tối đa 2 follow-up/câu hỏi gốc. 💡 Recommendation; ❓ Configurable / Need Confirmation.

### 3.3 Kết luận: cần 3 Examiner Behavior Profile riêng biệt

Vì bản chất tương tác của 3 Part khác nhau căn bản (hai chiều nhanh / độc thoại không ngắt / thảo luận sâu), hệ thống **phải có 3 "Examiner behavior profile" riêng biệt** — *Interview Profile*, *Long Turn Profile*, *Discussion Profile* — quy định: quyền ngắt lời, mức dùng follow-up, độ dài câu trả lời kỳ vọng, cách xử lý im lặng, cách chuyển câu hỏi. Không dùng một conversation logic chung cho cả 3 Part.

➡️ **Đánh dấu: ✅ Confirmed Requirement** — yêu cầu gốc đã nêu rõ "AI Examiner không nên dùng cùng một conversation logic cho cả 3 Part"; tài liệu này chuyển hoá thành yêu cầu 3 profile riêng biệt. (Chi tiết tham số bên trong từng profile là Configurable / Need Confirmation.)

---

## 4. Topic & Question Management

### 4.1 Nguyên tắc

- Topic/question do **IEG quản lý**. AI **không được tự quyết định topic/question cố định** nếu chưa được cấu hình từ IEG. ✅ **Confirmed Requirement**
- Mỗi bài thi được gán một cấu hình đề (topic set) từ IEG trước khi session bắt đầu. 🟡 Assumption về cách gán (theo lớp/level/random) — ❓ Need Confirmation.

### 4.2 Thuộc tính cần quản lý cho mỗi Question/Topic (mức business)

Topic, Part, Question text, Follow-up question (nếu tĩnh), Topic ID, Question ID, Version, Difficulty, Status (draft/active/retired), Language, Effective date, Metadata (tag kỹ năng, ghi chú cho reviewer). ✅ Confirmed Requirement (danh sách trường được yêu cầu). Quy trình duyệt đề (ai tạo, ai approve) — ❓ Need Confirmation.

### 4.3 Static vs Dynamic Question

| | Static Question | Dynamic Question |
|---|---|---|
| Nguồn | IEG soạn trước | AI sinh realtime dựa trên: Topic, Part, câu hỏi trước, câu trả lời của Student, conversation context, IELTS rules |
| Ưu điểm | Kiểm soát chất lượng, đúng chuẩn, dễ audit, so sánh giữa các học viên | Tự nhiên, bám sát câu trả lời, không lặp |
| Nhược điểm | Cứng, không phản ứng theo câu trả lời | Rủi ro lệch chuẩn/lạc đề, khó audit, chất lượng dao động |

### 4.4 Cơ chế câu hỏi theo từng Part — ✅ Confirmed: Hybrid (Stakeholder chốt 2026-08-24)

| Part | Cơ chế | Lý do |
|---|---|---|
| **Part 1** | **Static question (IEG) + AI-generated follow-up** trong phạm vi topic đã cấu hình | Câu hỏi gốc chuẩn hoá để công bằng; follow-up sinh động theo câu trả lời |
| **Part 2** | **Static cue card 100%** (IEG); rounding-off question có thể AI sinh nhưng trong khuôn mẫu ngắn | Cue card là "đề thi" đúng nghĩa — bắt buộc kiểm soát |
| **Part 3** | **Static seed question (IEG) theo chủ đề Part 2 + AI-generated follow-up** với độ tự do cao hơn Part 1 | Cần đào sâu theo câu trả lời nhưng vẫn giữ khung chủ đề |

✅ **Confirmed (Stakeholder chốt): mô hình Hybrid** — IEG cung cấp câu hỏi gốc/cue card/seed question theo topic; AI chỉ được generate **follow-up** dựa trên câu trả lời của Student, trong phạm vi topic và behavior profile của Part. Mọi câu hỏi AI sinh ra đều phải được **lưu lại kèm liên kết câu hỏi gốc** để IEG audit (mục 5). Tham số còn mở: giới hạn follow-up depth (#5) và độ tự do cụ thể của AI ở Part 3 — ❓ Need Confirmation (không chặn, có default đề xuất).

---

## 5. Question Lifecycle (mức Business)

Mỗi câu hỏi trong một session đi qua vòng đời:

```
QUESTION_CREATED → QUESTION_PRESENTED → STUDENT_ANSWERING
→ ANSWER_COMPLETED → AI_ANALYZING → FOLLOW_UP or NEXT_QUESTION
→ QUESTION_COMPLETED
```

**Ý nghĩa business từng bước:**

| Bước | Ý nghĩa | Ai quyết định |
|---|---|---|
| `QUESTION_CREATED` | Câu hỏi được chọn từ đề IEG hoặc AI sinh ra (kèm nguồn gốc: static/dynamic, câu hỏi cha nếu là follow-up) | Orchestrator chọn từ config; AI đề xuất nếu dynamic |
| `QUESTION_PRESENTED` | Avatar đã đọc xong câu hỏi cho Student — mốc bắt đầu "lượt trả lời" | Hệ thống |
| `STUDENT_ANSWERING` | Student đang nói; STT chạy realtime | — |
| `ANSWER_COMPLETED` | Câu trả lời được coi là **xong** khi: Student im lặng vượt ngưỡng cấu hình, hoặc Examiner kết thúc lượt (quá dài/hết giờ), hoặc Student chủ động báo xong. Ngưỡng ❓ Need Confirmation | Hệ thống (theo rule), không phải AI tuỳ ý |
| `AI_ANALYZING` | AI phân tích câu trả lời để **đề xuất** follow-up hay câu mới | AI đề xuất |
| `FOLLOW_UP / NEXT_QUESTION` | Orchestrator **quyết định** dựa trên đề xuất của AI + rule (còn giờ? còn quota follow-up? đúng profile Part?) | **Orchestrator** ✅ |
| `QUESTION_COMPLETED` | Câu hỏi đóng lại, dữ liệu (audio, transcript, timestamp) được chốt cho câu đó | Hệ thống |

**UX khi AI_ANALYZING:** học viên **không nên** thấy trạng thái "máy đang xử lý" một cách lộ liễu. 💡 Recommendation: avatar thể hiện hành vi tự nhiên của giám khảo (gật đầu, "Mm-hmm", nhìn xuống ghi chú) trong lúc phân tích; chỉ hiển thị indicator kỹ thuật nếu độ trễ vượt ngưỡng bất thường.

**Liên kết follow-up với câu hỏi cha:** ✅ **Confirmed Requirement (nghiệp vụ)** — mỗi follow-up question phải được liên kết với câu hỏi gốc đã sinh ra nó, để IEG/giáo viên khi review thấy được mạch hỏi–đáp (câu gốc → follow-up 1 → follow-up 2...). Cách lưu cụ thể (schema) thuộc Document 2.

---

## 6. Conversation Rules

### 6.1 AI Examiner PHẢI (✅ Confirmed Requirement)

1. Giữ đúng topic đã cấu hình.
2. Giữ đúng Part hiện tại (không hỏi kiểu Part 3 khi đang ở Part 1...).
3. Không hỏi câu không phù hợp format IELTS Speaking (và không hỏi nội dung nhạy cảm/không phù hợp).
4. Không tự ý chuyển Part.
5. Không đưa feedback/nhận xét chất lượng câu trả lời trong lúc thi.
6. Không tiết lộ điểm số hoặc ám chỉ điểm số.
7. Không tranh luận đúng/sai với Student.
8. Không trả lời thay Student, không gợi ý đáp án.
9. Không tạo hội thoại ngoài phạm vi bài thi (kể cả khi Student cố tình kéo AI ra ngoài — ví dụ hỏi AI về bản thân nó, nhờ nó làm việc khác).

### 6.2 Xử lý Student trả lời ngoài topic 💡 Recommendation (logic cụ thể cần xác nhận)

- **Lần lạc đề thứ 1:** Examiner không sửa lỗi, đặt lại câu hỏi hoặc paraphrase câu hỏi một cách lịch sự ("That's interesting — but let me ask again: ...").
- **Lần lạc đề thứ 2 liên tiếp cùng một câu hỏi:** Examiner redirect rõ hơn, hoặc chấp nhận câu trả lời và chuyển câu tiếp theo (giống giám khảo thật — không kéo dài).
- **Ghi log:** mỗi sự kiện off-topic/redirect được **ghi nhận vào event log của session** (phục vụ scoring — coherence — và review). 💡 Recommendation.
- Ngưỡng "bao nhiêu lần thì redirect" là tham số cấu hình ❓ Need Confirmation.

### 6.3 Nguyên tắc phân quyền AI vs Backend ✅ Confirmed Requirement

**AI Examiner có thể đề xuất (suggest) hành động tiếp theo trong hội thoại — hỏi follow-up, chuyển câu, kết thúc lượt — nhưng KHÔNG có quyền tự quyết định thay đổi trạng thái bài thi**: chuyển Part, kết thúc bài thi, dừng/kéo dài timer. Các quyền đó thuộc **backend/orchestrator**, hoạt động theo rule và timer đã cấu hình. Stakeholder cần hiểu rõ nguyên tắc này trước khi duyệt tài liệu, vì nó là nền tảng cho tính công bằng và khả năng audit của bài thi. Chi tiết kỹ thuật ở Document 2.

---

## 7. Timing & Exam State

### 7.1 State machine (mức business)

```
NOT_STARTED → INITIALIZING → READY → PART_1 → PART_2 → PART_3
→ COMPLETING → COMPLETED
Bất kỳ trạng thái nào → FAILED (lỗi không phục hồi được)
NOT_STARTED/INITIALIZING/READY → CANCELLED (user thoát trước khi thi)
```

| Transition | Ý nghĩa |
|---|---|
| `NOT_STARTED → INITIALIZING` | Học viên mở bài thi; hệ thống tạo session, load config, khởi tạo avatar |
| `INITIALIZING → READY` | Mic OK, avatar OK, config OK — chờ học viên bấm Start |
| `READY → PART_1` | Học viên bấm Start; timer Part 1 chạy (server) |
| `PART_1 → PART_2 → PART_3` | Orchestrator chuyển Part khi hết giờ hoặc hoàn thành interaction (nếu cho phép sớm — ❓) |
| `PART_3 → COMPLETING` | Examiner nói lời kết; finalize dữ liệu |
| `COMPLETING → COMPLETED` | Toàn bộ audio/transcript đã lưu thành công |
| `* → FAILED` | Lỗi kỹ thuật không phục hồi (mục 16); dữ liệu đã có vẫn được lưu |
| `* → CANCELLED` | Học viên huỷ trước khi bài thi bắt đầu |

### 7.1b Core Business Rule: Restart Part khi refresh/mất mạng

**✅ Confirmed — Stakeholder chốt (đây là business rule cốt lõi, không chỉ là edge case):** refresh hoặc mất mạng → học viên **làm lại toàn bộ Part đang dở** (restart Part với timer 3 phút mới của Part đó); các Part đã hoàn thành trước đó được giữ nguyên dữ liệu; **không có resume giữa chừng Part**.

Mô hình trạng thái mức Part (chi tiết ở Document 2):

```
PART_x (IN_PROGRESS) ──refresh/mất mạng──► PART_x_ABORTED ──reconnect──► PART_x_RESTART (timer 3 phút mới)
```

Quy định kèm theo:
- (a) Dữ liệu audio/transcript của lần làm dở vẫn lưu kèm cờ **"aborted attempt"** để audit, không dùng cho scoring. 💡 Recommendation.
- (b) **Khi restart Part, dùng bộ câu hỏi mới/randomized** — nếu giữ nguyên đề, học viên có thể lợi dụng refresh để nghe lại đề + có thêm thời gian chuẩn bị (đặc biệt Part 2). 💡 Recommendation mạnh — ❓ IEG xác nhận (P0).
- (c) **Giới hạn số lần restart Part** trong một session (ví dụ tối đa N lần, vượt → session FAILED/flag Operation) — ❓ Need Confirmation (P0). Lưu ý: mỗi lần restart tạo thêm phút LiveAvatar → **tăng chi phí** (mục 10), đây cũng là lý do cần giới hạn.
- (d) Mỗi lần restart đóng kết nối avatar cũ và mở kết nối mới → **một Exam Session có thể gồm nhiều LiveAvatar Session** (phân biệt ở Glossary mục 17; lưu `liveavatar_session_id` theo từng attempt để đối soát chi phí/debug). 💡 Recommendation.

### 7.2 Timer rules

| Câu hỏi | Đề xuất | Nhãn |
|---|---|---|
| Timer bắt đầu lúc nào? | Khi Part chính thức bắt đầu (Examiner bắt đầu nói câu đầu của Part), server ghi mốc | 💡 Recommendation — Need Confirmation |
| Có chạy khi AI đang nói không? | **Có** — 3 phút là tổng thời gian gồm cả AI hỏi và Student trả lời (yêu cầu gốc đã nêu) | ✅ Confirmed |
| Có chạy khi Student đang nói không? | Có | ✅ Confirmed (suy ra từ trên) |
| Có chạy trong lúc AI_ANALYZING (độ trễ xử lý)? | Có — nhưng hệ thống phải giữ latency thấp để không "ăn" thời gian của Student; nếu latency bất thường vượt ngưỡng, xem xét bù giờ | 💡 Recommendation — chính sách bù giờ ❓ Need Confirmation |
| Prep time Part 2 có tính vào 3 phút? | ❓ Open Question — ảnh hưởng trực tiếp trải nghiệm Part 2 | ❓ |
| Có pause được không? | Mặc định **không** (đây là bài thi); ngoại lệ duy nhất cân nhắc: sự cố kỹ thuật do hệ thống gây ra | 💡 Recommendation — ❓ Need Confirmation |
| Network disconnect có pause không? | **Không có khái niệm pause/resume giữa Part** — mất mạng → Part đang dở bị huỷ, học viên làm lại Part đó từ đầu (timer Part reset về 3 phút cho lần làm lại) | ✅ Confirmed (Stakeholder chốt) |
| Refresh có reset timer không? | Refresh → làm lại toàn bộ Part đang dở: timer của Part đó bắt đầu lại (3 phút), các Part đã xong không bị ảnh hưởng | ✅ Confirmed (Stakeholder chốt) |
| Client hay server là source of truth? | **Server-side timer là source of truth.** Client chỉ hiển thị. Dù là mock test, timer vẫn không được phép bị can thiệp từ client hay bị "trì hoãn" bởi AI đang xử lý — đảm bảo trải nghiệm sát thi thật và dữ liệu nhất quán | 💡 Recommendation (mặc định theo yêu cầu gốc) |

---

## 8. Audio Recording

| Câu hỏi | Phân tích & đề xuất | Nhãn |
|---|---|---|
| Có record audio Student không? | **Có** — phục vụ scoring/review/audit | ✅ Confirmed |
| Answer-level hay session-level? | **Cả hai**: (a) file theo **từng câu trả lời** (gắn question_id — phục vụ scoring từng câu, review theo mạch hỏi–đáp) và (b) bản ghi **toàn session** (gồm cả giọng Examiner nếu khả thi — phục vụ audit, khiếu nại, nghe lại trải nghiệm đầy đủ). Nếu buộc chọn một vì chi phí: ưu tiên answer-level | 💡 Recommendation |
| Format / sample rate | Đề xuất chuẩn phổ biến cho speech (nén có kiểm soát, sample rate đủ cho STT/scoring, tối thiểu 16kHz). Con số chính thức: ❓ Need Confirmation với team AI/scoring | 💡 + ❓ |
| File naming | Theo quy ước chứa: exam_session_id, part, question_id/answer_id, thứ tự, timestamp — đủ để truy vết không cần mở DB | 💡 Recommendation |
| Storage | Object storage riêng cho dữ liệu học viên, phân quyền chặt (mục 13); vị trí lưu (VN residency?) ❓ Need Legal Review | ❓ |
| Metadata kèm theo | session ID, student ID, part, question ID, answer ID, thứ tự câu hỏi, timestamp bắt đầu/kết thúc, duration, thiết bị/trình duyệt | ✅ Confirmed (yêu cầu gốc liệt kê metadata phục vụ scoring/review/audit) |
| Upload timing | Upload **dần theo từng câu trả lời** ngay khi câu đó hoàn thành (không đợi hết bài) — giảm rủi ro mất dữ liệu khi sự cố cuối bài | 💡 Recommendation |
| Retry | Upload lỗi → retry tự động nền; hàng đợi phía client giữ lại đến khi thành công; quá ngưỡng retry → flag session để Operation xử lý, không âm thầm mất dữ liệu | 💡 Recommendation |

---

## 9. Transcript

- **Lưu cả hai transcript:** Student (từ STT) và Examiner (từ text trước khi TTS — luôn sẵn có, chính xác 100%). Yêu cầu gốc ghi Examiner transcript "nếu có" → 💡 Recommendation: **có**, vì chi phí gần bằng 0 và cần cho review/audit/scoring context.
- **Source of truth của transcript là hệ thống IEG** 💡 Recommendation: Student transcript lấy từ STT pipeline của IEG; Examiner transcript lấy từ response text gốc của IEG. **Không dùng transcript do LiveAvatar/provider cung cấp làm nguồn chính thức** (nếu có, chỉ dùng đối soát/debug) — nhất quán với nguyên tắc IEG sở hữu dữ liệu (mục 13.5).
- Mỗi dòng transcript gắn: speaker, text, timestamp, part, question_id, answer_id. ✅ Confirmed
- Transcript được ghi **realtime trong khi thi** và **finalize sau khi kết thúc** (bản final là source chính thức; partial STT không lưu). 💡 Recommendation — realtime hay hậu kỳ là ❓ Open Question ở yêu cầu gốc, đề xuất realtime để hỗ trợ AI phân tích và giảm rủi ro mất dữ liệu.

Định dạng đề xuất (mức minh hoạ — schema chi tiết ở Document 2):

```json
{
  "speaker": "student",
  "text": "...",
  "timestamp": "...",
  "part": 1,
  "question_id": "...",
  "answer_id": "..."
}
```

---

## 10. Avatar / Video

**Phân biệt hai loại nội dung:**

| | Realtime Avatar (HeyGen LiveAvatar) | Pre-generated Video |
|---|---|---|
| Dùng cho | Nội dung phụ thuộc hội thoại: câu hỏi động, follow-up, phản hồi theo câu trả lời | Nội dung cố định, lặp lại mọi session: greeting, hướng dẫn luật thi, câu chuyển Part chuẩn, lời kết thúc chuẩn |
| Chi phí | Cao (phút realtime session của HeyGen), phụ thuộc concurrent limit | Trả một lần, phát lại miễn phí, không tốn concurrent slot |
| Độ trễ/rủi ro | Có độ trễ và rủi ro disconnect | Ổn định tuyệt đối |

**Recommendation 💡:**
- **Pre-generate:** greeting mở bài, hướng dẫn cách thi, câu dẫn chuyển Part cố định, lời kết thúc — vừa giảm chi phí HeyGen, vừa tăng độ ổn định ở các mốc quan trọng.
- **Realtime:** toàn bộ hỏi–đáp trong Part (câu hỏi, follow-up, redirect, prompt khi im lặng).
- **Lưu video avatar của session để xem lại?** ❓ Open Question — video Examiner có thể tái tạo/không cần cho scoring (scoring dựa trên audio + transcript Student). Đề xuất mặc định: **không lưu video stream**, chỉ lưu **HeyGen session ID + metadata** để đối soát chi phí và debug. Cần xác nhận thêm việc HeyGen có lưu gì phía họ không (mục 13).
- Cần kiểm tra **giới hạn concurrent session theo gói cước HeyGen** trước khi chốt NFR concurrency (mục 15). ❓ Need Confirmation.

**Cost model LiveAvatar (business consideration):** theo BA review, LiveAvatar tính phí theo phút (tham khảo: LITE ~1 credit/phút, FULL ~2 credit/phút — 🟡 Assumption, **phải verify với pricing/plan HeyGen hiện hành**). Một bài thi ≈3 Part × 3 phút = ~9 phút streaming; **mỗi lần restart Part cộng thêm phút** → cần theo dõi metric **"average LiveAvatar minutes per completed exam"** và giới hạn restart (mục 7.1b). 💡 Recommendation.

**Chọn mode:** LITE vs FULL — khuyến nghị **LITE** (mục 1.4b); quyết định chính thức + verify capability theo tài liệu LiveAvatar hiện hành: ❓ Need Confirmation (P0, trước Document 2).

---

## 11. Exam Completion

Bài thi kết thúc trong các tình huống:

| Tình huống | Behavior đề xuất | Nhãn |
|---|---|---|
| Part 3 hết 3 phút | Kết thúc chuẩn: Examiner nói lời kết, finalize, `COMPLETED` | ✅ Confirmed |
| Tất cả required interaction hoàn thành trước giờ | ❓ Open Question: có cho kết thúc sớm không. 💡 Đề xuất: cho phép nếu đã hết câu hỏi cấu hình của Part 3 (Examiner không nên "câu giờ" bằng câu hỏi ngoài đề) | ❓ + 💡 |
| User chủ động submit/end sớm | ❓ Open Question: có cho phép không. 💡 Đề xuất: cho phép với xác nhận 2 bước + cảnh báo ảnh hưởng kết quả; ghi nhận là "ended early by student" | ❓ + 💡 |
| System force-end khi timeout | Khi session vượt tổng thời gian tối đa (ví dụ treo do lỗi), orchestrator force-end, lưu mọi dữ liệu đã có, đánh dấu lý do | 💡 Recommendation |
| Technical failure không recover được | Session → `FAILED`; dữ liệu đã có vẫn lưu; học viên được thông báo rõ. Theo chính sách đã chốt: học viên làm lại **Part đang dở** (các Part đã xong được giữ) | ✅ (theo quyết định restart Part) |

**Lưu ý đặc biệt Part 2 (✅ Confirmed):** không được cắt ngang long turn **trừ khi hết giờ**. Khi hết giờ giữa lúc Student đang nói: Examiner chờ điểm ngắt tự nhiên trong khoảng grace ngắn rồi kết thúc lịch sự ("Thank you."), audio được ghi trọn đến lúc dừng. Grace bao nhiêu giây ❓ Need Confirmation.

---

## 12. AI Scoring

- **Scope: ⛔ Out of Scope cho tính năng Realtime Examiner — ✅ Confirmed (Stakeholder chốt):** scoring thuộc **một hệ thống khác của IEG**. Realtime Examiner không chấm điểm, không hiển thị điểm trong lúc thi (trùng rule mục 6).
- **Nghĩa vụ của Realtime Examiner đối với scoring:** lưu và bàn giao đầy đủ, đúng cấu trúc dữ liệu đầu vào mà hệ thống scoring cần: audio recording, transcript (Student + Examiner), câu hỏi đã hỏi, conversation history, metadata Part. ✅ Confirmed — đây chính là lý do phải lưu đầy đủ dữ liệu ở mục 8–9 ngay từ phase 1.
- **Interface với hệ thống scoring:** cách bàn giao (hệ thống scoring đọc chung storage, hay có event/API thông báo khi session `COMPLETED`), định dạng dữ liệu đầu vào hệ thống đó yêu cầu — ❓ Need Confirmation với team scoring của IEG (chi tiết kỹ thuật ở Document 2).
- **Trigger:** 💡 Recommendation: khi session chuyển `COMPLETED`, hệ thống phát tín hiệu để hệ thống scoring của IEG chủ động xử lý (asynchronous).
- SLA trả kết quả cho học viên: thuộc hệ thống scoring, ngoài scope tài liệu này.

---

## 13. Compliance, Consent & Data Privacy

> Toàn bộ mục này ở trạng thái **❓ Open Question / Need Legal Review** trừ khi ghi khác. Tài liệu **không giả định "không cần consent/compliance review"**.

### 13.1 Consent
- Thông báo ghi âm (và ghi hình nếu có webcam/proctoring) **trước khi thi** — 💡 Recommendation: bắt buộc có consent screen, học viên phải chủ động đồng ý mới vào thi. Nội dung pháp lý ❓ Need Legal Review.
- Học viên **dưới 18 tuổi**: có cần consent phụ huynh không, thu thập thế nào (khi đăng ký khoá học hay trước từng bài thi)? ❓ Need Legal Review — lưu ý tập học viên IELTS có tỷ lệ vị thành niên cao.
- Học viên **từ chối ghi âm**: có được thi không? Về mặt sản phẩm, không ghi âm thì không thể scoring/review → 💡 Recommendation: từ chối = không thể làm bài thi này (đề xuất hình thức thay thế nếu có). Quyết định cuối: ❓ Need Legal/Business Confirmation.

### 13.2 Data Ownership & Access
- Ai được truy cập audio/transcript: học viên (bản của mình), giáo viên phụ trách, IEG (admin/QA), hệ thống AI scoring, support (khi xử lý khiếu nại)? ❓ Need Confirmation cho từng nhóm — đề xuất nguyên tắc least-privilege.
- **Audit log truy cập** (ai xem/tải bản ghi nào, khi nào): 💡 Recommendation: **có**, tối thiểu cho dữ liệu audio/video của học viên.
- Học viên có quyền **xem/tải lại bản ghi của mình**: ❓ Need Confirmation (đề xuất: có, qua trang kết quả).

### 13.3 Data Retention & Deletion
- Lưu bao lâu (audio, transcript, video nếu có)? ❓ Need Legal/Business Confirmation.
- Tự động xoá / chuyển cold storage sau X tháng? ❓ Need Confirmation.
- **Right to be forgotten:** học viên yêu cầu xoá dữ liệu — quy trình tiếp nhận, thời hạn xử lý, ngoại lệ (dữ liệu cần giữ cho khiếu nại điểm)? ❓ Need Legal Review.

### 13.4 Regulatory Compliance
- **Nghị định 13/2023/NĐ-CP** (bảo vệ dữ liệu cá nhân VN): giọng nói/bản ghi của học viên là dữ liệu cá nhân; cần rà soát nghĩa vụ (consent, đánh giá tác động, thông báo xử lý dữ liệu). ❓ Need Legal Review.
- **GDPR** nếu có học viên quốc tế: ❓ Need Confirmation (có học viên EU không?).
- **Data residency VN:** có yêu cầu lưu dữ liệu trong lãnh thổ VN không? ❓ Need Legal Review — ảnh hưởng lựa chọn storage/provider.
- **Chuyển dữ liệu xuyên biên giới sang third-party** (HeyGen, STT/TTS/LLM provider — hầu hết là dịch vụ nước ngoài): cần đánh giá rủi ro/thủ tục theo NĐ 13. ❓ Need Legal Review.

### 13.5 Third-party Data Handling
- **✅ Confirmed (Stakeholder chốt):** HeyGen chỉ cung cấp **Avatar AI** (khuôn mặt hiển thị, có thể nhận thêm thông tin cấu hình context); **toàn bộ dữ liệu bài thi (audio, transcript, câu hỏi, metadata) trả về và lưu trên hệ thống của IEG** — HeyGen không phải nơi lưu trữ dữ liệu học viên.
- Vẫn cần xác nhận (với vendor, người chốt là Legal IEG): trong quá trình realtime, HeyGen và STT/LLM/TTS provider có **tạm giữ/log** dữ liệu phía họ không (kể cả transient), bao lâu, có dùng để train model không, có opt-out được không? ❓ Need Confirmation — rà soát **DPA (Data Processing Agreement)** với từng provider trước khi go-live. 💡 Recommendation: đưa việc rà soát DPA vào checklist bắt buộc trước launch.

**Rà soát sơ bộ DPA HeyGen** (https://www.heygen.com/data-processing-addendum, bản hiệu lực 10/2024 — đây là review sơ bộ của Product/BA, **không thay thế Legal review**):

| Câu hỏi 13.5 | DPA HeyGen nói gì | Đánh giá |
|---|---|---|
| Vai trò dữ liệu | HeyGen = *data processor*, Customer (IEG) = *data controller* | ✅ Đúng mô hình IEG sở hữu dữ liệu |
| Dùng dữ liệu train model? | §3a: chỉ xử lý *"solely for the purpose of providing the Services"* — về nguyên tắc loại trừ training, nhưng **không có câu "no model training" tường minh** | 🟡 Yêu cầu HeyGen xác nhận bằng văn bản |
| Subprocessor | Phải liệt kê (Exhibit A/B); thông báo trước khi thêm mới, Customer có 10 ngày phản đối | ✅ Có cơ chế; Legal đọc danh sách Exhibit |
| Chuyển dữ liệu xuyên biên giới | EU SCCs / UK Addendum / Data Privacy Framework | 🟡 Viết cho GDPR/CPRA — **không đề cập luật VN**; Legal đánh giá riêng theo NĐ 13/2023 |
| Quyền chủ thể dữ liệu | §3e: HeyGen hỗ trợ yêu cầu truy cập/xoá | ✅ |
| Retention/log realtime session | **Không nêu con số cụ thể** | ❓ Phải hỏi HeyGen |
| Trẻ vị thành niên | **Không có điều khoản riêng** | ❓ Phải hỏi HeyGen |
| Data residency | HeyGen là công ty Mỹ; không cam kết vùng lưu trữ | ❓ Hỏi nếu Legal yêu cầu residency |
| Phạm vi áp dụng | DPA gắn với Order Form/Enterprise Agreement | ❓ Xác nhận **gói IEG mua có kèm DPA này không** (gói self-serve có thể chỉ theo ToS) |

---

## 14. Identity Verification & Anti-cheat (Proctoring)

> **✅ ĐÃ CHỐT (Stakeholder xác nhận): đây là MOCK TEST, không tính vào kết quả chính thức → toàn bộ mục 14 là ⛔ Out of Scope / Not Applicable.** Chỉ giữ log vận hành cơ bản: số lần reconnect/refresh/restart Part — phục vụ vận hành và phân tích sản phẩm, không phục vụ kỷ luật. Bảng dưới đây giữ lại để tham khảo cho Future Phase nếu sau này có phiên bản thi chính thức.

**Nếu tương lai có bài thi tính điểm chính thức** (Future Phase), cần phân tích/quyết định từng hạng mục:

| Hạng mục | Mô tả | Nhãn |
|---|---|---|
| Face verification | Xác thực danh tính qua webcam trước/trong khi thi, đối chiếu ảnh hồ sơ | ❓ Need Confirmation |
| Phát hiện đọc bài / người trả lời hộ | Phân tích bất thường (giọng đổi, nhịp đọc, mắt nhìn tài liệu — cần webcam) | ❓ Need Confirmation |
| Tab-switching detection | Phát hiện chuyển tab/cửa sổ trong khi thi, ghi log | ❓ Need Confirmation |
| Giới hạn reconnect | Quá N lần reconnect → flag vi phạm/huỷ kết quả (N ❓) | ❓ Need Confirmation |
| Yêu cầu webcam & không gian yên tĩnh | Điều kiện dự thi, kiểm tra trước khi bắt đầu | ❓ Need Confirmation |
| Flag session nghi vấn | Cơ chế đánh dấu session để người thật review, quyền huỷ/giữ kết quả thuộc IEG | 💡 Recommendation nếu thi chính thức |

**Ảnh hưởng tới NFR (mục 15) nếu tương lai đưa proctoring vào scope:** thêm video stream webcam → tăng bandwidth yêu cầu tối thiểu, tăng chi phí storage đáng kể (video), tăng độ phức tạp device support (mobile khó đáp ứng), thêm yêu cầu consent ghi hình (mục 13). Hiện tại (mock test) NFR **không** phải tính các chi phí này.

---

## 15. Non-Functional Requirements (NFR)

> Theo Important Rule: **không tự đặt con số chính thức** — dưới đây là benchmark tham khảo ngành, mọi con số chính thức đều ❓ Need Confirmation.

### 15.1 Latency
- **Benchmark tham khảo ngành (voice AI hội thoại):** phản hồi của AI (từ lúc Student ngừng nói đến lúc Examiner bắt đầu nói) trong khoảng **~1–2 giây** được cảm nhận là tự nhiên; trên **~3 giây** bắt đầu gây khó chịu; trên **~5 giây** phá vỡ cảm giác realtime. 💡 Benchmark tham khảo — con số cam kết chính thức ❓ Need Confirmation.
- **Tách latency thành các metric thành phần** (💡 Recommendation — cần để debug và đặt SLO ở Document 2): T1 = STT (student nói xong → text), T2 = LLM (text → quyết định câu hỏi), T3 = TTS (text → audio), T4 = LiveAvatar render/stream, **T_total = end-to-end** (student ngừng nói → avatar bắt đầu nói). SLO từng thành phần: ❓ Need Confirmation.
- **Fallback threshold:** 💡 Recommendation: có ngưỡng latency kích hoạt hành vi che (avatar nói câu đệm tự nhiên: "Let me see...") và ngưỡng cao hơn kích hoạt fallback kỹ thuật (mục 16). Con số ❓ Need Confirmation.

### 15.2 Concurrency & Scalability
- **Số học viên thi đồng thời tối đa (peak):** ❓ Need Confirmation từ Business (theo lịch thi/số lớp).
- **Giới hạn concurrent của HeyGen theo gói cước:** ❓ Need Confirmation — phải đối chiếu với peak; nếu peak > limit, cần cơ chế **hàng đợi/đặt lịch slot thi**. 💡 Recommendation: có cơ chế queue + thông báo chờ thay vì lỗi.
- Giới hạn tương tự với STT/LLM/TTS provider (rate limit): ❓ Need Confirmation.

### 15.3 Availability
- SLA mong muốn cho tính năng: ❓ Need Confirmation (ví dụ giờ thi cao điểm cần cam kết cao hơn).
- Multi-region / failover cho third-party (HeyGen down thì sao — xem fallback mục 16/18): ❓ Need Confirmation.

### 15.4 Device & Browser Support
- Danh sách browser hỗ trợ chính thức: ❓ Need Confirmation. 💡 Đề xuất tối thiểu: Chrome/Edge bản mới trên desktop (nền tảng ổn định nhất cho mic + realtime media).
- **Mobile browser (iOS Safari/Android Chrome):** ❓ Need Confirmation — lưu ý rủi ro: quyền mic khác biệt, tab bị suspend khi rời app, bandwidth dao động. 💡 Đề xuất: phase 1 desktop-first, mobile là phase sau nếu Business cần.
- App native: ❓ Need Confirmation — 💡 đề xuất chưa cần ở phase 1.
- **Bandwidth tối thiểu:** ❓ Need Confirmation — cần đo thực tế với HeyGen stream; công bố yêu cầu tối thiểu + bước kiểm tra mạng trong màn hình chuẩn bị. 💡 Recommendation.

---

## 16. Exception / Edge Cases

| Scenario | Expected Behavior | Nhãn |
|---|---|---|
| User refresh page | **Làm lại toàn bộ Part đang dở** (Stakeholder chốt): Part hiện tại bị huỷ, học viên quay lại thi Part đó từ đầu với timer 3 phút mới; các Part đã xong được giữ. Ghi log sự kiện; đề xuất tráo bộ câu hỏi khi làm lại + giới hạn số lần restart (❓) | ✅ Confirmed; chi tiết 💡/❓ |
| Internet disconnect | Tương tự refresh: Part đang dở bị huỷ, học viên làm lại Part đó từ đầu sau khi kết nối lại; các Part đã xong và dữ liệu đã upload được giữ. 💡 Cho phép disconnect rất ngắn (vài giây, tự khôi phục stream) không tính là mất mạng — ngưỡng ❓ Need Confirmation | ✅ Confirmed; ngưỡng ❓ |
| Avatar disconnect (HeyGen) | Thử reconnect avatar nền trong ngưỡng ngắn (ví dụ 5–10s — ❓); nếu không khôi phục được → áp chính sách restart Part (mục 7.1b). **Fallback audio-only (Examiner nói bằng TTS, màn hình ảnh tĩnh) là ❓ Open Question — Product phải quyết** vì thay đổi trải nghiệm cốt lõi (thi với avatar); không mặc định bật | ❓ (Product) + 💡 |
| STT timeout | Retry tự động; nếu vẫn lỗi: Examiner lịch sự xin Student nhắc lại ("Sorry, could you repeat that?") tối đa N lần (❓); vượt ngưỡng → tạm dừng kỹ thuật/flag session, không âm thầm bỏ qua câu trả lời | 💡 |
| LLM timeout | Dùng câu hỏi static tiếp theo trong đề IEG (bỏ qua follow-up động) để hội thoại không đứng hình; ghi log degradation | 💡 |
| TTS failure | Retry; nếu vẫn lỗi: hiển thị câu hỏi dạng text trên màn hình để Student đọc và trả lời (degraded mode); ghi log | 💡 |
| User không nói (im lặng) | Examiner nhắc theo profile Part: Part 1/3 — lặp lại/diễn giải câu hỏi; Part 2 — động viên bắt đầu, gợi lại cue card. Sau N lần nhắc không phản hồi (❓): chuyển câu tiếp/kết thúc Part, ghi nhận "no answer" | 💡; ngưỡng ❓ |
| User nói quá dài | Part 1/3: Examiner kết thúc lượt lịch sự khi vượt ngưỡng thời gian trả lời (❓) — giống giám khảo thật. Part 2: **không ngắt**, chỉ dừng khi hết giờ Part | ✅ Part 2 Confirmed; ngưỡng Part 1/3 ❓ |
| User nói ngoài topic | Politely redirect theo logic mục 6.2; ghi log sự kiện | 💡 |
| Browser đóng | Tương đương refresh/disconnect: khi quay lại, học viên làm lại Part đang dở; các Part đã xong được giữ | ✅ Confirmed (theo chính sách restart Part) |
| Microphone permission denied | Chặn trước khi bắt đầu thi (không tạo được trạng thái READY); hướng dẫn cấp quyền; không tính là attempt | 💡 |
| Session expired (đăng nhập hết hạn giữa bài) | Không đá học viên ra giữa bài thi: giữ realtime session sống, yêu cầu re-auth sau khi hoàn thành hoặc silent-refresh; chi tiết kỹ thuật ở Document 2 | 💡 |
| Audio upload failed | Retry nền theo mục 8; giữ hàng đợi phía client đến khi thành công; quá ngưỡng → flag cho Operation, thông báo học viên nếu ảnh hưởng kết quả | 💡 |
| Part 2: Student nói xong trước khi hết giờ | Examiner chờ ngưỡng im lặng xác nhận đã xong → hỏi 1 rounding-off question hoặc chuyển Part sớm. Chọn phương án nào: ❓ Open Question (liên quan "cho kết thúc Part sớm không") | ❓ + 💡 |
| Part 2: Student vẫn đang nói khi hết giờ | Cho grace ngắn để dừng ở điểm tự nhiên, Examiner kết thúc lịch sự ("Thank you."), audio ghi trọn tới lúc dừng, chuyển Part 3 | ✅ nguyên tắc; grace ❓ |

---

## 17. Glossary

| Thuật ngữ | Giải thích đơn giản |
|---|---|
| **STT (Speech-to-Text)** | Công nghệ chuyển giọng nói thành văn bản — dùng để "nghe" câu trả lời của học viên |
| **TTS (Text-to-Speech)** | Công nghệ chuyển văn bản thành giọng nói — dùng để giám khảo AI "nói" |
| **LLM (Large Language Model)** | Mô hình AI ngôn ngữ — "bộ não" phân tích câu trả lời và nghĩ ra câu hỏi tiếp theo |
| **AI Avatar / LiveAvatar** | Nhân vật ảo có hình ảnh, cử động môi khớp lời nói; LiveAvatar là dịch vụ realtime avatar của HeyGen |
| **WebSocket / WebRTC** | Công nghệ kết nối hai chiều liên tục giữa trình duyệt và máy chủ — giúp âm thanh/hình ảnh truyền tức thời thay vì tải từng trang |
| **Exam Session (IEG)** | Đối tượng nghiệp vụ của IEG: một lần làm bài thi của học viên, từ lúc tạo đến hoàn thành — có `exam_session_id`, trạng thái, và toàn bộ dữ liệu được lưu kèm |
| **LiveAvatar Session (provider)** | Phiên kết nối realtime với HeyGen (`liveavatar_session_id`) — chỉ là kết nối kỹ thuật. **Một Exam Session có thể gồm nhiều LiveAvatar Session** (ví dụ do restart Part) |
| **LiveAvatar FULL / LITE Mode** | FULL: HeyGen lo cả ASR→LLM→TTS; LITE: IEG tự lo AI stack, HeyGen chỉ stream avatar. Khuyến nghị LITE (mục 1.4b) |
| **Transcript** | Bản ghi văn bản của toàn bộ lời nói trong bài thi (của học viên và giám khảo) |
| **Follow-up question** | Câu hỏi nối tiếp dựa trên câu trả lời vừa rồi của học viên (khác câu hỏi mới trong đề) |
| **Cue card** | Thẻ đề Part 2: chủ đề + gợi ý các ý cần nói, học viên chuẩn bị ngắn rồi nói một mình |
| **Orchestrator** | Thành phần hệ thống "trọng tài": giữ giờ, chuyển Part, quyết định trạng thái bài thi (AI chỉ đề xuất) |

---

## 18. Open Questions / Decisions Required

**✅ Đã chốt (đóng):** #1 Mock test (không tính kết quả chính thức) · #3 Số câu hỏi: Part 1: 6–10, Part 2: 1, Part 3: 4–6 · **#4 Cơ chế câu hỏi: Hybrid (IEG cung cấp câu gốc/cue card, AI generate follow-up — mục 4.4)** · #9 Scoring ở hệ thống IEG khác (ngoài scope) · #10/#11/#12 Refresh/mất mạng → làm lại Part đang dở · #19 Anti-cheat: Out of Scope (vì mock test).

**Còn mở — đã ưu tiên lại theo P0 (chặn Document 2/dev core) / P1 (chặn go-live) / P2 (không chặn):**

| # | Câu hỏi | Ai quyết | Ưu tiên |
|---|---|---|---|
| 26 | **Chốt LiveAvatar LITE vs FULL Mode** (khuyến nghị LITE — mục 1.4b); verify capability & pricing theo tài liệu/gói HeyGen hiện hành | Eng + Business | **P0** — quyết định kiến trúc nền cho Document 2 |
| 21 | Prep time Part 2: bao lâu, có nằm trong 3 phút Part không, có cho ghi chú không? | IEG + Product | **P0** |
| 24 | Giới hạn số lần restart Part? Có dùng bộ câu hỏi mới/randomized khi làm lại Part không (khuyến nghị mạnh: có)? | Product + IEG | **P0** |
| 2 | Cho phép kết thúc Part sớm / skip question không? | Product + IEG | **P0** |
| 6 | Topic có bắt buộc IEG cấu hình 100% không (AI không bao giờ tự chọn topic)? | IEG | **P0** — đề xuất: bắt buộc (✅ theo yêu cầu gốc, xác nhận lại) |
| 25 | Interface bàn giao dữ liệu cho hệ thống scoring của IEG (định dạng, cơ chế trigger)? | Eng + team scoring IEG | **P0** (trước khi build storage schema) |
| 13 | Fallback khi LiveAvatar down: có cho phép audio-only không hay luôn restart Part? (Product phải quyết — mục 16) | Product + Business | **P1** (nên chốt sớm) |
| 14 | Audio retention bao lâu? Cold storage/xoá tự động? | Legal + Business | **P1** (trước go-live) |
| 15 | Ai được xem/tải recording? | Legal + IEG | **P1** (trước go-live) |
| 22 | Consent flow: nội dung, dưới 18 tuổi, quyền từ chối (mục 13.1)? | Legal | **P1** (trước go-live) |
| 23 | Data residency VN & đánh giá chuyển dữ liệu xuyên biên giới (NĐ 13); DPA với HeyGen/STT/LLM/TTS provider (mục 13.5)? | Legal | **P1** (trước go-live) |
| 20 | Concurrent session tối đa cần hỗ trợ? Gói HeyGen giới hạn bao nhiêu? Cost/credit thực tế theo phút? | Business + Eng | **P1** (trước load test/go-live) |
| 5 | Giới hạn follow-up depth mỗi câu hỏi gốc (Part 1 / Part 3)? | IEG + Product | P2 (có default đề xuất) |
| 7 | Có lưu video avatar / conversation video / raw audio Examiner không? | Product + Legal | P2 |
| 8 | Transcript ghi realtime hay sau khi kết thúc? (đề xuất: realtime) | Product + Eng | P2 |
| 16 | Cần audit log truy cập dữ liệu không? (đề xuất: có) | Legal | P2 |
| 17 | Cần hỗ trợ đa ngôn ngữ giao diện/hướng dẫn không? (bài thi luôn tiếng Anh) | Product | P2 |
| 18 | Cần support mobile browser không? (đề xuất: desktop-first phase 1) | Business | P2 |

---

## 19. Implementation Readiness

### Đánh giá tổng thể: **PARTIALLY READY (tiệm cận READY)**

Sau khi Stakeholder chốt 5 quyết định lớn (mock test, restart Part, số câu hỏi, scoring ngoài scope, luồng dữ liệu HeyGen), đã **đủ thông tin để bắt đầu thiết kế kỹ thuật (Document 2) và development core flow**. Điều kiện tiên quyết cho Document 2: chốt **#26 LiveAvatar LITE vs FULL** (khuyến nghị LITE — mục 1.4b) và các P0 ở mục 18; các hạng mục P1 chặn **go-live**, không chặn dev.

| Area | Status | Blocking? | Required Decision |
|---|---|---|---|
| Business flow (session, vòng lặp hỏi–đáp, completion, restart Part) | **Ready** | No | — (đã chốt, gồm chính sách refresh/mất mạng) |
| IELTS Part-specific rules (Part 1/2/3 behavior) | **Ready** (nguyên tắc + số câu hỏi đã chốt) / Partial (tham số nhỏ) | No | IEG chốt: follow-up depth, prep time Part 2 (#5, #21) trong lúc dev |
| Topic/Question management (IEG) | **Partial** (cơ chế Hybrid đã chốt #4) | **Yes** | IEG chốt quy trình tạo/duyệt đề và xác nhận lại topic 100% do IEG cấu hình (#6), cần trước khi tích hợp |
| Recording (retention, format) | **Partial** | **Yes** (retention trước go-live; format trước khi build) | Eng/AI chốt format & sample rate; Legal chốt retention (#14) |
| Consent / Legal | **TBD** | **Yes** (trước go-live, không chặn dev core) | Legal review mục 13 (#22, #23, #15) — vẫn cần dù là mock test vì có ghi âm + học viên dưới 18 tuổi |
| Scoring scope | **Ready** (đã chốt: ngoài scope, ở hệ thống IEG khác) | Interface bàn giao: **Yes** | Team scoring IEG chốt định dạng/cơ chế bàn giao dữ liệu (#25) |
| Anti-cheat scope | **Ready** (đã chốt: Out of Scope — mock test) | No | — |
| Concurrency/NFR targets | **TBD** | **Yes** (trước load test/go-live, không chặn dev core) | Business đưa peak concurrent; Eng đối chiếu gói HeyGen (#20); chốt latency target chính thức |

---

## 20. Definition of Done

Feature được coi là hoàn thành khi tất cả các mục sau được nghiệm thu (QA test được toàn bộ critical flow):

1. Học viên **start exam** được từ đầu đến cuối (tạo session → READY → Part 1→2→3 → COMPLETED).
2. AI Avatar thực hiện **greeting** và hiển thị/nói ổn định.
3. AI **đặt câu hỏi** đúng đề IEG cấu hình, đúng Part, đúng behavior profile.
4. Học viên **trả lời bằng mic**; speech → **transcript** chính xác ở mức nghiệm thu.
5. AI **generate follow-up** đúng rule (đúng Part, đúng giới hạn, liên kết câu hỏi cha).
6. AI Avatar **trả lời bằng voice** trong ngưỡng latency đã chốt.
7. **Timer đúng** (server-side, đúng 3 phút/Part).
7b. **Restart Part đúng chính sách:** refresh/mất mạng → làm lại toàn bộ Part đang dở, các Part đã xong được giữ nguyên.
8. **Part transition đúng** — đặc biệt Part 2 không bị cắt ngang sai cách (chỉ dừng khi hết giờ, kết thúc lịch sự).
9. **Exam completion đúng** (lời kết, finalize, trạng thái COMPLETED).
10. **Audio, transcript, question, conversation history được lưu đầy đủ** kèm metadata (session ID, part, question ID, thứ tự, timestamp) và liên kết follow-up → câu hỏi cha.
11. **Error/restart Part** được xử lý theo bảng edge case (mục 16) với chính sách đã chốt.
12. **Bàn giao dữ liệu cho hệ thống scoring của IEG** hoạt động (trigger/định dạng theo interface đã chốt ở #25). Việc chấm điểm tự thân là Out of Scope — không tính vào DoD.
13. **Consent/compliance flow** hoạt động — *nếu trong scope theo quyết định Legal* (tối thiểu: thông báo ghi âm hiển thị trước khi thi).
14. **Logging/monitoring** hoạt động: event log của session (câu hỏi, redirect, disconnect, degradation), latency metrics, lỗi third-party.

---

*Hết Document 1. Document 2 (Technical Specification) sẽ chi tiết hoá kiến trúc, schema dữ liệu, API, cơ chế orchestrator/timer, và các quyết định kỹ thuật — nhất quán với các Confirmed/Assumption/Open Question đã đánh dấu trong tài liệu này.*
