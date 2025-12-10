# Tt

Đúng rồi 👍
👉 Không bắt buộc phải dùng Parse JSON nếu bạn thao tác trực tiếp trên response bằng expressions.

Parse JSON chỉ giúp dễ nhìn hơn và có intellisense, còn về logic flow thì không cần.

⸻

✔️ TRẢ LỜI NGẮN GỌN

❓ “K có parse json đúng k?”
➡️ Đúng – bạn không cần Parse JSON.
Bạn hoàn toàn có thể:
	•	GET RoleAssignments → dùng body() để đọc
	•	EnsureUser → dùng body() để lấy d.Id
	•	Remove → không cần parse gì

⸻

✔️ FLOW RÚT GỌN (KHÔNG PARSE JSON)

1) GET RoleAssignments

_api/web/RoleAssignments?$expand=Member,RoleDefinitionBindings&$select=Member/LoginName

2) Apply to each → lặp

Value:

body('HTTP_Get_RoleAssignments')?['d']?['results']

3) Lấy LoginName (Compose)

items('Apply_to_each')?['Member']?['LoginName']

4) EnsureUser → lấy principalId

_api/web/EnsureUser()?loginName='@{outputs('Compose_LoginName')}'

5) Lấy PrincipalId (KHÔNG Parse JSON)

body('EnsureUser')?['d']?['Id']

6) Remove bằng PrincipalId (KHÔNG Parse JSON)

_api/web/RoleAssignments/RemoveByPrincipalId(@{body('EnsureUser')?['d']?['Id']})


⸻

✔️ KẾT LUẬN
	•	Không cần Parse JSON
	•	Flow chạy nhẹ hơn
	•	Không bị lỗi schema

⸻

Nếu bạn muốn, mình vẽ luôn flow / screenshot từng bước, hoặc bạn đưa mình tên các HTTP action, mình viết lại expression đúng 100% theo tên của bạn để bạn copy chạy là được.
