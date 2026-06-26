# Case 1 - Support Ticket Triage

## Thông tin học viên

* Họ và tên: `Nguyễn Minh Đức`
* Mã học viên: `2A202600808`
* Repo: `Day22-Track1-2A202600808-NguyenMinhDuc`

---

# 1. Unit of Work

Unit of Work tôi chọn là: **một ticket support mới đi vào hệ thống → AI đọc subject, message, customer_tier → AI gán category, urgency, route_to, requires_human và lý do tóm tắt**.

Đây là một lát cắt đủ nhỏ để eval vì nó tập trung vào một quyết định vận hành cụ thể: ticket sẽ được đưa vào hàng đợi nào và có cần người thật xử lý ngay hay không. Tôi không chọn “toàn bộ hệ thống support” vì phạm vi đó quá rộng, bao gồm cả trả lời khách hàng, theo dõi SLA, cập nhật trạng thái, xử lý sau ticket. Trong case này, output cuối cùng được dùng bởi nhân viên support nội bộ, team billing, technical support hoặc support lead để ưu tiên xử lý ticket.

Nếu AI sai, hậu quả vận hành có thể là: ticket bị route sang sai team, ticket khẩn cấp bị đánh thấp, khách enterprise bị chậm xử lý, hoặc escalation quan trọng bị bỏ sót. Ví dụ ticket “payment failed and account disabled” nếu bị route thành “product question / medium / support L1” thì nhìn bề mặt có vẻ hợp lý, nhưng thực tế là sai vì khách bị khóa tài khoản và có khả năng đang bị chặn công việc.

---

# 2. Quality Question

Quality Question tôi chọn là:

> AI có gán đúng category, urgency, route_to và requires_human để ticket được đưa vào đúng hàng xử lý, không bỏ sót escalation quan trọng, đặc biệt với khách enterprise hoặc ticket đang chặn công việc không?

Câu hỏi này cụ thể hơn câu “AI có triage tốt không?” vì nó tập trung vào các quyết định làm thay đổi workflow vận hành. Trong case này, điều quan trọng nhất không phải là câu chữ tóm tắt có hay hay không, mà là ticket có được đưa đến đúng người, đúng mức ưu tiên và đúng thời điểm hay không.

Nếu fail ở đây, khách hàng có thể mất trust vì họ gửi vấn đề nghiêm trọng nhưng hệ thống lại xử lý như vấn đề bình thường. Các behavior bắt buộc là: phát hiện dấu hiệu urgency cao, route đúng team, bật cờ human review khi rủi ro cao. Các behavior bị cấm là: tự tin route khi thiếu thông tin, đánh low cho ticket có dấu hiệu “locked out / account disabled / blocking work”, hoặc route billing issue sang product_team.

---

# 3. Output Contract tối thiểu

Output Contract tối thiểu tôi đề xuất:

```json
{
  "ticket_id": "string",
  "category": "technical | billing | product_question | account_access | bug_report | unknown",
  "urgency": "low | medium | high | critical",
  "route_to": "support_l1 | technical_support | billing_ops | product_team | human_escalation | needs_review",
  "requires_human": true,
  "confidence": 0.0,
  "reason_codes": ["string"],
  "summary_reason": "string",
  "evidence_spans": ["string"]
}
```

Giải thích từng field:

| Field            | Lý do cần có                                                                                                                                                                                         |
| ---------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `ticket_id`      | Dùng để nối output AI với ticket gốc trong hệ thống. Nếu thiếu field này thì khó trace, debug hoặc review lại kết quả eval.                                                                          |
| `category`       | Dùng để render nhãn loại yêu cầu trên UI và làm cơ sở route ticket. Ví dụ billing issue phải route về billing_ops, technical issue route về technical_support.                                       |
| `urgency`        | Dùng để quyết định mức ưu tiên xử lý. Đây là field quan trọng vì ticket high/critical có thể cần escalation hoặc ưu tiên SLA.                                                                        |
| `route_to`       | Dùng để đưa ticket vào đúng hàng xử lý. Nếu field này sai thì ticket có thể đến team không xử lý được.                                                                                               |
| `requires_human` | Dùng để bật cờ người thật cần xử lý ngay. Đặc biệt quan trọng với khách enterprise, ticket critical hoặc ticket có dấu hiệu chặn công việc.                                                          |
| `confidence`     | Dùng để trigger review khi AI không chắc chắn. Confidence cũng giúp phân tích eval sau này: case nào AI tự tin nhưng sai là case rủi ro cao.                                                         |
| `reason_codes`   | Dùng để giải thích quyết định theo dạng có cấu trúc, ví dụ `enterprise_customer`, `locked_out`, `billing_failure`, `blocking_work`. Field này giúp code eval kiểm tra lý do có khớp với input không. |
| `summary_reason` | Dùng để hiển thị lý do ngắn trên UI cho nhân viên support hiểu vì sao AI gợi ý như vậy.                                                                                                              |
| `evidence_spans` | Dùng để lưu các đoạn bằng chứng trong ticket khiến AI đưa ra quyết định. Field này hỗ trợ human review và giảm nguy cơ AI bịa lý do.                                                                 |

Các field trên là tối thiểu vì chúng đều ảnh hưởng trực tiếp đến UI, routing, escalation hoặc eval. Tôi chưa đưa các field như `customer_sentiment`, `language`, `suggested_reply` vào contract tối thiểu vì case này chỉ đánh giá triage nội bộ, không yêu cầu AI trả lời khách hàng.

---

# 4. Eval Decision Map

| Thành phần cần chấm                                                   |    Code |   LLM | Human | Expert | Lý do                                                                                                                                                                                               |
| --------------------------------------------------------------------- | ------: | ----: | ----: | -----: | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Schema output và required fields                                      |     Yes |    No |    No |     No | Đây là kiểm tra deterministic. Code có thể xác nhận output có đủ `ticket_id`, `category`, `urgency`, `route_to`, `requires_human`, `confidence`, `reason_codes`, `summary_reason`.                  |
| Allowed enum cho `category`, `urgency`, `route_to`                    |     Yes |    No |    No |     No | Các giá trị hợp lệ đã được định nghĩa trước. Code chấm chính xác hơn LLM và tránh lỗi typo hoặc sinh nhãn ngoài taxonomy.                                                                           |
| Confidence nằm trong khoảng 0-1                                       |     Yes |    No |    No |     No | Đây là rule số học đơn giản, nên giao cho code.                                                                                                                                                     |
| Business rule enterprise + high/critical phải `requires_human = true` |     Yes |    No |    No |     No | Rule này rõ ràng và có thể kiểm tra bằng điều kiện logic. Nếu fail thì escalation bị bỏ sót.                                                                                                        |
| Billing issue không route sang product_team                           |     Yes | Maybe | Maybe |     No | Nếu category đã là billing thì code có thể kiểm tra route không được là product_team. Nhưng nếu ticket mô tả billing bằng ngôn ngữ gián tiếp, LLM/human có thể cần đánh giá category có đúng không. |
| Nhận diện đúng urgency dựa trên nội dung ticket                       |      No |   Yes |   Yes |     No | Việc phân biệt medium/high/critical cần hiểu ngữ nghĩa như “blocking work”, “locked out”, “account disabled”. LLM judge có thể chấm semantic, human review dùng cho case khó.                       |
| `reason_codes` có grounded trong input                                | Partial |   Yes |   Yes |     No | Code có thể bắt một số keyword rõ ràng, nhưng không đủ để xác nhận toàn bộ reason có đúng nghĩa hay không. LLM/human cần kiểm tra AI có bịa lý do không.                                            |
| `summary_reason` đúng, ngắn và không hallucinate                      |      No |   Yes |   Yes |     No | Summary cần đọc hiểu nội dung ticket. Code không đánh giá tốt việc tóm tắt có đúng ngữ cảnh hay không.                                                                                              |
| Case thiếu thông tin hoặc ambiguous có được route review không        | Partial |   Yes |   Yes |     No | Code có thể bắt subject/message quá ngắn, nhưng đánh giá “thiếu thông tin” thường cần semantic judgment. Human review cần xem các case AI tự tin quá mức.                                           |
| Release decision cho pilot                                            |      No |    No |   Yes |     No | Quyết định có nên pilot hay không là quyết định PM/ops dựa trên tỷ lệ lỗi, failure mode và rủi ro vận hành, không nên giao hoàn toàn cho code hoặc LLM.                                             |
| Domain expert review                                                  |      No |    No |    No |     No | Case này không cần domain expert chuyên sâu vì đây là nghiệp vụ vận hành support SaaS, không phải y tế/pháp lý/tài chính có rủi ro chuyên môn cao. Support lead hoặc ops lead là đủ.                |

---

# 5. Kiểm tra tự động bằng code

Các rule kiểm tra tự động cần có:

## 5.1. Kiểm tra schema

* Kiểm tra: Output phải là JSON/object hợp lệ.
  Vì sao nên giao cho code: Đây là điều kiện kỹ thuật rõ ràng, không cần semantic judgment.

* Kiểm tra: Output phải có đủ required fields: `ticket_id`, `category`, `urgency`, `route_to`, `requires_human`, `confidence`, `reason_codes`, `summary_reason`.
  Vì sao nên giao cho code: Nếu thiếu field, UI hoặc routing có thể lỗi. Code kiểm tra được chính xác.

* Kiểm tra: `ticket_id` trong output phải trùng với `ticket_id` trong input.
  Vì sao nên giao cho code: Đây là rule đối chiếu deterministic. Nếu sai ticket_id thì trace eval và review bị hỏng.

* Kiểm tra: `requires_human` phải là boolean.
  Vì sao nên giao cho code: Kiểu dữ liệu có thể validate tự động.

* Kiểm tra: `reason_codes` phải là array/list, không phải string đơn.
  Vì sao nên giao cho code: Đây là ràng buộc schema rõ ràng.

* Kiểm tra: `summary_reason` không được rỗng nếu `confidence >= 0.5`.
  Vì sao nên giao cho code: Nếu AI đủ tự tin để route thì phải có lý do hiển thị cho người vận hành.

## 5.2. Kiểm tra allowed enums

* Kiểm tra: `category` chỉ được thuộc một trong các giá trị: `technical`, `billing`, `product_question`, `account_access`, `bug_report`, `unknown`.
  Vì sao nên giao cho code: Taxonomy cố định, code bắt lỗi sinh nhãn ngoài danh sách tốt hơn LLM.

* Kiểm tra: `urgency` chỉ được thuộc `low`, `medium`, `high`, `critical`.
  Vì sao nên giao cho code: Enum cố định, không cần LLM.

* Kiểm tra: `route_to` chỉ được thuộc `support_l1`, `technical_support`, `billing_ops`, `product_team`, `human_escalation`, `needs_review`.
  Vì sao nên giao cho code: Nếu route ngoài danh sách, hệ thống không biết đưa ticket vào hàng nào.

## 5.3. Kiểm tra confidence

* Kiểm tra: `confidence >= 0` và `confidence <= 1`.
  Vì sao nên giao cho code: Đây là kiểm tra số học.

* Kiểm tra: Nếu `category = unknown` thì `confidence` không được quá cao, ví dụ không quá `0.7`.
  Vì sao nên giao cho code: Nếu AI vừa nói unknown vừa confidence rất cao thì output không nhất quán.

* Kiểm tra: Nếu `confidence < 0.6` thì `route_to` phải là `needs_review` hoặc `human_escalation`.
  Vì sao nên giao cho code: Low confidence nên được đưa qua người thật thay vì tự route mạnh.

## 5.4. Kiểm tra business rules

* Kiểm tra: Nếu `customer_tier = enterprise` và `urgency` là `high` hoặc `critical`, thì `requires_human` phải bằng `true`.
  Vì sao nên giao cho code: Rule đã được nêu rõ trong đề, có thể kiểm tra bằng logic.

* Kiểm tra: Nếu `category = billing` thì `route_to` không được là `product_team`.
  Vì sao nên giao cho code: Đây là mapping route rõ ràng. Billing issue vào product_team sẽ gây trễ xử lý.

* Kiểm tra: Nếu `category = billing` và message/subject có dấu hiệu `payment failed`, `billing failed`, `account disabled`, thì `route_to` phải là `billing_ops` hoặc `human_escalation`.
  Vì sao nên giao cho code: Với các keyword rõ ràng, code có thể bắt failure nghiêm trọng.

* Kiểm tra: Nếu subject/message chứa `locked out`, `account disabled`, `cannot login`, `blocking work`, thì `urgency` không được là `low`.
  Vì sao nên giao cho code: Các cụm này là trigger vận hành rõ ràng, nếu bị đánh low thì có nguy cơ bỏ sót escalation.

* Kiểm tra: Nếu subject/message chứa `URGENT` và `account disabled` thì `urgency` nên là `high` hoặc `critical`.
  Vì sao nên giao cho code: Đây là rule keyword mạnh trong seed case high-risk.

* Kiểm tra: Nếu `urgency = critical` thì `requires_human` phải là `true`.
  Vì sao nên giao cho code: Critical ticket không nên chỉ để AI route tự động mà không bật cờ người xử lý.

* Kiểm tra: Nếu `route_to = human_escalation` thì `requires_human` phải là `true`.
  Vì sao nên giao cho code: Hai field này phải nhất quán với nhau.

* Kiểm tra: Nếu `route_to = needs_review` thì `requires_human` phải là `true` hoặc có flag review tương đương.
  Vì sao nên giao cho code: Case cần review nhưng không bật human flag sẽ dễ bị bỏ qua.

## 5.5. Kiểm tra consistency giữa category và route

* Kiểm tra: `category = technical` thì `route_to` nên là `technical_support`, `support_l1`, `human_escalation` hoặc `needs_review`, không nên là `billing_ops`.
  Vì sao nên giao cho code: Mapping category-route có thể định nghĩa trước.

* Kiểm tra: `category = account_access` thì route không được là `product_team`, vì lỗi đăng nhập/tài khoản thường cần support/technical/account team.
  Vì sao nên giao cho code: Đây là rule vận hành tương đối rõ.

* Kiểm tra: `category = product_question` thì `urgency = critical` chỉ được chấp nhận nếu có trigger enterprise/blocking/incident; nếu không thì cần review.
  Vì sao nên giao cho code: Giúp bắt case AI lạm dụng critical cho câu hỏi sản phẩm thông thường.

## 5.6. Kiểm tra reason_codes

* Kiểm tra: Nếu reason_codes có `enterprise_customer` thì input `customer_tier` phải là `enterprise`.
  Vì sao nên giao cho code: Đây là đối chiếu trực tiếp từ input.

* Kiểm tra: Nếu reason_codes có `billing_failure` thì subject/message nên có dấu hiệu billing như `payment`, `billing`, `invoice`, `charge`, `failed`.
  Vì sao nên giao cho code: Có thể bắt một số hallucination đơn giản.

* Kiểm tra: Nếu reason_codes có `locked_out` thì subject/message nên có dấu hiệu như `locked out`, `cannot login`, `account disabled`, `access denied`.
  Vì sao nên giao cho code: Các trigger này khá cụ thể, code kiểm tra được ở mức keyword.

* Kiểm tra: Nếu reason_codes có `blocking_work` thì subject/message nên có dấu hiệu `blocking`, `cannot work`, `blocked`, `team is locked out`, hoặc ý tương tự được đánh dấu trong reference.
  Vì sao nên giao cho code: Có thể kiểm tra một phần bằng keyword/reference label.

## 5.7. Kiểm tra case ambiguous / low-info

* Kiểm tra: Nếu subject là `Help` và message rất ngắn như `Please help asap`, output không nên có `confidence > 0.8`.
  Vì sao nên giao cho code: Với seed ambiguous, AI không nên tự tin quá mức.

* Kiểm tra: Với input thiếu nội dung rõ ràng, `category` nên là `unknown` hoặc `route_to = needs_review`.
  Vì sao nên giao cho code: Có thể kiểm tra bằng các case reference có expected label.

* Kiểm tra: Nếu message rỗng hoặc quá ngắn, output phải bật review thay vì route tự động sang team chuyên biệt.
  Vì sao nên giao cho code: Đây là rule an toàn cho dữ liệu thiếu.

---

# 6. Tiêu chí chấm bằng LLM

Các tiêu chí semantic cần LLM judge:

## 6.1. Category có đúng ý ticket không

* Tiêu chí: LLM judge kiểm tra AI có hiểu đúng loại vấn đề chính của ticket không.
  Vì sao code không bắt tốt: Cùng một ý có thể được diễn đạt bằng nhiều cách. Ví dụ “our team is locked out because your billing system failed” vừa liên quan billing vừa liên quan account access, cần hiểu nguyên nhân chính là billing failure gây khóa tài khoản.

## 6.2. Urgency có hợp lý với mức độ ảnh hưởng không

* Tiêu chí: LLM judge đánh giá urgency có phản ánh đúng mức độ ảnh hưởng tới khách hàng không.
  Vì sao code không bắt tốt: Không phải ticket nào có chữ “urgent” cũng thật sự critical, và không phải ticket nào không có chữ “urgent” cũng low. Cần đọc ngữ cảnh như “blocking my work”, “admin cannot access”, “team locked out”.

## 6.3. Route_to có phù hợp với nguyên nhân chính không

* Tiêu chí: LLM judge kiểm tra ticket nên route về team nào dựa trên nội dung thực tế.
  Vì sao code không bắt tốt: Một ticket có thể có nhiều tín hiệu. Ví dụ lỗi login sau password reset là technical/account access, còn account disabled vì payment failed là billing_ops hoặc human_escalation. Code keyword đơn giản có thể route sai.

## 6.4. Requires_human có được bật đúng khi có rủi ro vận hành không

* Tiêu chí: LLM judge kiểm tra AI có bỏ sót escalation trong các trường hợp khách enterprise, high/critical, hoặc bị chặn công việc không.
  Vì sao code không bắt tốt: Một số dấu hiệu chặn công việc không dùng đúng keyword. Ví dụ “our whole team cannot use the dashboard before the board meeting” là high-impact dù không có chữ “blocking”.

## 6.5. Summary_reason có đúng và đủ ngắn không

* Tiêu chí: Summary phải phản ánh đúng lý do triage, không quá dài, không thêm thông tin không có trong input.
  Vì sao code không bắt tốt: Code khó đánh giá chất lượng tóm tắt và mức độ faithful với nội dung gốc.

## 6.6. Reason_codes có grounded không

* Tiêu chí: LLM judge kiểm tra từng reason_code có bằng chứng trong input không.
  Vì sao code không bắt tốt: Code có thể bắt keyword rõ ràng nhưng không thể đánh giá tốt các cách diễn đạt gián tiếp.

## 6.7. AI có tự tin quá mức trong case thiếu thông tin không

* Tiêu chí: Với ticket mơ hồ như “Help / Please help asap”, LLM judge kiểm tra AI có tránh gán category/route quá mạnh không.
  Vì sao code không bắt tốt: Độ mơ hồ là đánh giá ngữ nghĩa, không chỉ dựa vào độ dài message.

## 6.8. AI có bỏ sót mixed-intent không

* Tiêu chí: LLM judge kiểm tra ticket có nhiều ý, ví dụ vừa hỏi billing vừa báo lỗi tài khoản, AI có chọn route chính hợp lý hoặc bật review không.
  Vì sao code không bắt tốt: Mixed-intent cần đọc hiểu thứ tự ưu tiên vận hành.

## 6.9. AI có hallucinate policy hoặc sự thật không có trong input không

* Tiêu chí: AI không được nói “SLA bị vi phạm”, “khách đã bị charge hai lần”, “đây là bug hệ thống” nếu input chưa nói.
  Vì sao code không bắt tốt: Hallucination thường nằm trong câu chữ tự nhiên của `summary_reason`, cần LLM đọc và so với input.

## 6.10. Mock outcome T-002 có bị đánh fail không

* Tiêu chí: Với ticket `URGENT: payment failed and account disabled`, LLM judge phải nhận ra output `Product question / Medium / Support L1 / requires_human = false` là sai.
  Vì sao code không bắt tốt hoàn toàn: Code bắt được một số rule, nhưng LLM giúp giải thích tại sao đây là lỗi judgment vận hành nghiêm trọng.

---

# 7. Human / Expert Review

## 7.1. Ai cần review?

Người cần review chính là **Support Lead / Support Ops / Customer Support Manager**. Đây là nhóm hiểu quy trình xử lý ticket, SLA, route team và mức độ ảnh hưởng đến khách hàng. Họ không cần là domain expert chuyên sâu như bác sĩ hay luật sư, nhưng cần hiểu vận hành support của công ty.

## 7.2. Review những case nào?

Human review nên áp dụng cho các nhóm case sau:

1. **Ticket high hoặc critical**

   * Vì nếu AI route sai hoặc không bật escalation, khách hàng có thể bị chậm xử lý nghiêm trọng.

2. **Ticket enterprise**

   * Vì khách enterprise thường có SLA, giá trị hợp đồng và kỳ vọng hỗ trợ cao hơn.

3. **Ticket có confidence thấp**

   * Vì AI không chắc thì không nên tự động route mạnh.

4. **Ticket ambiguous / thiếu thông tin**

   * Ví dụ subject “Help”, message “Please help asap”. AI có thể đoán sai category.

5. **Ticket mixed-intent**

   * Ví dụ payment failed + account disabled + locked out. Cần người thật xác nhận route ưu tiên.

6. **Ticket mà code eval fail rule quan trọng**

   * Ví dụ `billing` nhưng route sang `product_team`, hoặc `critical` nhưng `requires_human = false`.

7. **Ticket AI tự tin cao nhưng LLM judge đánh fail**

   * Đây là failure nguy hiểm vì hệ thống có vẻ tự tin nhưng thực tế sai.

## 7.3. Có cần domain expert không?

Không cần domain expert chuyên sâu cho case này.

Lý do: đây là bài toán **support operations triage** trong SaaS B2B. Rủi ro chính là route sai, ưu tiên sai, bỏ sót escalation, chứ không phải đưa ra quyết định chuyên môn y tế, pháp lý hoặc tài chính phức tạp. Support Lead hoặc Ops Lead là người phù hợp hơn domain expert vì họ hiểu trực tiếp hàng đợi, SLA, team ownership và hậu quả vận hành.

## 7A. Màn hình cho Domain Expert

Không áp dụng.

Vì case này không cần domain expert chuyên sâu. Nếu cần màn hình review, màn hình nên dành cho Support Lead / Ops Reviewer thay vì domain expert:

```text
+----------------------------------------------------------------+
| Support Ops Review                                             |
+----------------------------------------------------------------+
| Ticket ID: T-002                                                |
| Customer Tier: Enterprise                                       |
| Subject: URGENT: payment failed and account disabled            |
| Message: Our team is locked out because your billing system...  |
|----------------------------------------------------------------|
| AI Output                                                       |
| - Category: Billing                                             |
| - Urgency: Critical                                             |
| - Route to: Billing Ops / Human Escalation                      |
| - Requires human: Yes                                           |
| - Confidence: 0.86                                              |
| - Reason codes: billing_failure, account_disabled, locked_out   |
|----------------------------------------------------------------|
| Evidence                                                        |
| - "payment failed"                                             |
| - "account disabled"                                           |
| - "team is locked out"                                         |
|----------------------------------------------------------------|
| Reviewer Action                                                 |
| [Approve] [Change Route] [Change Urgency] [Escalate] [Reject]   |
+----------------------------------------------------------------+
```

## 7B. Tiêu chí review của Domain Expert

Không áp dụng.

Thay vào đó, Support Ops reviewer sẽ dùng các tiêu chí sau:

1. Category có phản ánh đúng vấn đề chính của ticket không?
2. Urgency có phù hợp với mức độ ảnh hưởng đến khách hàng không?
3. Route_to có đưa ticket đến đúng team chịu trách nhiệm không?
4. Requires_human có được bật đúng với ticket enterprise/high-risk không?
5. Summary_reason và reason_codes có dựa trên nội dung thật của ticket không?
6. Nếu ticket thiếu thông tin, AI có tránh tự tin quá mức không?

---

# 8. Release Gate

Tôi đề xuất release gate cho pilot như sau:

## 8.1. Điều kiện chặn tuyệt đối

Không được pilot nếu xuất hiện một trong các lỗi sau trên tập eval ban đầu:

1. Có output sai schema hoặc thiếu required field quan trọng.
2. Có `confidence` ngoài khoảng 0-1.
3. Có ticket `critical` nhưng `requires_human = false`.
4. Có ticket enterprise + high/critical nhưng không bật human review.
5. Có billing issue rõ ràng nhưng route sang `product_team`.
6. Có ticket chứa dấu hiệu `locked out`, `account disabled`, `blocking work` nhưng bị đánh `low`.
7. Có hallucination nghiêm trọng trong `summary_reason`, ví dụ bịa thêm SLA, hợp đồng, refund, charge hoặc lỗi hệ thống không có trong input.

## 8.2. Ngưỡng chất lượng tối thiểu

Trên reference dataset version đầu khoảng 80 cases:

* Schema pass rate: **100%**
* Allowed enum pass rate: **100%**
* Confidence range pass rate: **100%**
* Business rule pass rate cho escalation: **≥ 98%**
* Category accuracy: **≥ 90%**
* Route accuracy: **≥ 90%**
* Urgency accuracy: **≥ 88%**
* High-risk escalation recall: **≥ 95%**
* Hallucination rate trong `summary_reason`: **≤ 2%**
* Ambiguous case được route sang `needs_review` hoặc human review: **≥ 90%**

## 8.3. Điều kiện human review trước pilot

Các case sau luôn phải qua human review trong giai đoạn pilot:

* `urgency = high` hoặc `critical`
* `customer_tier = enterprise`
* `confidence < 0.6`
* `category = unknown`
* `route_to = needs_review` hoặc `human_escalation`
* LLM judge đánh fail semantic criterion quan trọng
* Code eval fail bất kỳ business rule nào

## 8.4. Phạm vi pilot được phép

Nếu pass các gate trên, AI chỉ được dùng để **gợi ý triage nội bộ**, không tự động gửi phản hồi cho khách hàng và không tự đóng ticket. Nhân viên support vẫn là người xác nhận cuối cùng với ticket high-risk.

---

# 9. Kế hoạch chạy thử và dự toán chi phí

## 9.1. Mục tiêu pilot

Mục tiêu pilot là trả lời 3 câu hỏi:

1. Hướng AI triage hiện tại chính xác tới đâu?
2. AI có đủ an toàn để dùng như gợi ý nội bộ cho support team chưa?
3. Với budget nhỏ, team có thể chứng minh được AI giúp phân loại và ưu tiên ticket mà không bỏ sót escalation quan trọng không?

## 9.2. Quy mô pilot

Tôi đề xuất pilot với:

* **80 reference cases**
* **40 lần chạy/lặp lại**
* Tổng lượt inference triage: `80 cases x 40 runs = 3,200 runs`
* Mỗi run gồm:

  * 1 lần model tạo output triage
  * 1 lần LLM judge chấm semantic
* Tổng số model calls ước tính: `3,200 triage calls + 3,200 judge calls = 6,400 calls`

## 9.3. Giả định token

Giả định trung bình:

* Một triage call:

  * Input: 700 tokens
  * Output: 180 tokens
* Một LLM judge call:

  * Input: 1,000 tokens
  * Output: 250 tokens

Tổng token ước tính:

* Triage input: `3,200 x 700 = 2,240,000 tokens`
* Triage output: `3,200 x 180 = 576,000 tokens`
* Judge input: `3,200 x 1,000 = 3,200,000 tokens`
* Judge output: `3,200 x 250 = 800,000 tokens`

Tổng:

* Input tokens: `5,440,000`
* Output tokens: `1,376,000`

## 9.4. Giá API dùng để tính

Giả định dùng **GPT-5.4 mini** cho cả triage model và LLM judge.

Giá API tham khảo:

* Input: **$0.75 / 1M tokens**
* Output: **$4.50 / 1M tokens**

Chi phí API ước tính:

* Input cost: `5.44M x $0.75 = $4.08`
* Output cost: `1.376M x $4.50 = $6.19`
* Tổng API cost: khoảng **$10.27**

Làm tròn và cộng buffer 50% cho retry, prompt dài hơn hoặc log/debug:

* API budget đề xuất: **$15–20**

## 9.5. Giờ công dự kiến

Tôi đề xuất giờ công như sau:

* PM / thiết kế eval: **8 giờ**

  * Chốt Unit of Work, Quality Question, output contract, release gate.
* Support Ops / Support Lead: **8 giờ**

  * Gắn nhãn reference cases, xác nhận route taxonomy, review failure.
* Kỹ thuật / vận hành eval: **6 giờ**

  * Chuẩn bị file test cases, chạy batch thủ công hoặc script đơn giản, tổng hợp kết quả.
* Human review: **6 giờ**

  * Review high-risk, ambiguous, low-confidence, LLM-judge-fail cases.
* Domain expert: **0 giờ**

  * Không cần domain expert cho case support triage.

Tổng giờ công: **28 giờ**.

Nếu quy đổi nội bộ đơn giản:

* PM: $20/giờ x 8 = $160
* Support Ops: $15/giờ x 8 = $120
* Kỹ thuật: $20/giờ x 6 = $120
* Human review: $15/giờ x 6 = $90
* API: khoảng $20

Tổng pilot budget sơ bộ:

> **$510**, làm tròn thành **$500–600** để có buffer.

## 9.6. Timeline

Timeline đề xuất: **5 ngày làm việc**

* Ngày 1:

  * Chốt output contract, taxonomy category/route/urgency, release gate.
* Ngày 2:

  * Tạo 80 reference cases, bao gồm seed cases và edge cases.
* Ngày 3:

  * Chạy 40 vòng thử nghiệm, thu output, chạy rule code và LLM judge.
* Ngày 4:

  * Human review các case fail, high-risk, ambiguous.
* Ngày 5:

  * Tổng hợp kết quả, quyết định pass/fail pilot, đề xuất cải thiện prompt/taxonomy.

## 9.7. Vì sao plan này đủ để chứng minh pilot?

Plan này đủ nhỏ để chạy nhanh trong một tuần nhưng vẫn đủ để bắt các failure quan trọng: sai schema, route sai team, urgency sai, bỏ sót escalation, hallucination reason và overconfidence trong case thiếu thông tin. Với 80 cases và 40 lần chạy, team có thể quan sát cả chất lượng trung bình lẫn độ ổn định qua nhiều lần chạy. Budget khoảng $500–600 là hợp lý cho giai đoạn chứng minh hướng làm trước khi đầu tư triển khai thật.

---

# 10. Dataset Edge Cases đề xuất thêm

## Edge Case 1 - Happy path rõ ràng

Input:

```json
{
  "ticket_id": "E-001",
  "subject": "Cannot access admin dashboard",
  "message": "I cannot access the admin dashboard after resetting my password. I need to approve invoices today.",
  "customer_tier": "business"
}
```

Kỳ vọng:

* `category = account_access` hoặc `technical`
* `urgency = high`
* `route_to = technical_support`
* `requires_human = true`

Failure muốn bắt:

> Bắt lỗi AI đánh đây là câu hỏi sản phẩm thông thường hoặc urgency medium/low dù khách không truy cập được dashboard và có công việc cần xử lý.

---

## Edge Case 2 - Ambiguous input

Input:

```json
{
  "ticket_id": "E-002",
  "subject": "Help",
  "message": "Please help asap",
  "customer_tier": "standard"
}
```

Kỳ vọng:

* `category = unknown`
* `urgency = medium` hoặc cần review, không tự động critical nếu thiếu bằng chứng
* `route_to = needs_review` hoặc `support_l1`
* `confidence` thấp

Failure muốn bắt:

> Bắt lỗi AI tự tin quá mức và tự gán category/route mạnh khi input thiếu thông tin.

---

## Edge Case 3 - Missing information

Input:

```json
{
  "ticket_id": "E-003",
  "subject": "Payment issue",
  "message": "",
  "customer_tier": "standard"
}
```

Kỳ vọng:

* `category = billing`
* `urgency = medium` hoặc `needs_review`
* `route_to = billing_ops` hoặc `needs_review`
* `confidence` không được quá cao
* `summary_reason` phải nói rõ thiếu thông tin chi tiết

Failure muốn bắt:

> Bắt lỗi AI bịa thêm chi tiết như “payment failed”, “account disabled”, “customer locked out” khi message rỗng.

---

## Edge Case 4 - High-risk / escalation

Input:

```json
{
  "ticket_id": "E-004",
  "subject": "URGENT: account disabled for entire team",
  "message": "Our entire team is locked out after a failed payment attempt. We cannot use the product at all.",
  "customer_tier": "enterprise"
}
```

Kỳ vọng:

* `category = billing` hoặc `account_access`
* `urgency = critical`
* `route_to = billing_ops` hoặc `human_escalation`
* `requires_human = true`
* `reason_codes` gồm các ý như `enterprise_customer`, `locked_out`, `billing_failure`, `blocking_work`

Failure muốn bắt:

> Bắt lỗi AI route sai sang product_team/support_l1 hoặc không bật human escalation cho khách enterprise bị khóa tài khoản.

---

## Edge Case 5 - Regression case

Input:

```json
{
  "ticket_id": "E-005",
  "subject": "Question about pricing page",
  "message": "I am comparing plans and want to know whether the Pro plan includes SSO.",
  "customer_tier": "standard"
}
```

Kỳ vọng:

* `category = product_question`
* `urgency = low` hoặc `medium`
* `route_to = support_l1` hoặc `product_team`, tùy taxonomy công ty
* `requires_human = false`
* Không route escalation

Failure muốn bắt:

> Bắt lỗi AI lạm dụng escalation hoặc đánh high/critical cho câu hỏi sản phẩm bình thường chỉ vì có từ liên quan pricing/plan.

---

# 11. Kết luận

Trong Case 1, phần cần eval chặt nhất không phải là AI viết lý do hay, mà là AI có đưa ticket vào đúng hàng xử lý và không bỏ sót escalation hay không. Các phần deterministic như schema, enum, confidence, mapping route cơ bản nên giao cho code. Các phần cần đọc hiểu như urgency, grounded reason, mixed-intent và ambiguity nên giao cho LLM judge và human review. Case này không cần domain expert chuyên sâu, nhưng cần Support Lead/Ops reviewer để xác nhận taxonomy vận hành và các case high-risk trước khi pilot.

Release gate nên chặn tuyệt đối các lỗi làm ticket đi sai hàng hoặc bỏ sót human escalation. Với pilot khoảng 80 cases, 40 runs và budget khoảng $500–600, team có thể chứng minh được AI triage có đủ ổn định và an toàn để dùng như gợi ý nội bộ hay chưa.
