# Tt

Ok mình hướng dẫn code REST API & Flow Power Automate để bắt user mới được share vào site với quyền Contribute.

⸻

🧠 Ý tưởng chính:
	•	Lấy danh sách RoleAssignments
	•	Lọc quyền Contribute
	•	So sánh với danh sách đã lưu
	•	User nào mới xuất hiện → vừa được share quyền

⸻

🧩 BƯỚC 1 — HTTP GET trong Power Automate

🔹 Action: Send an HTTP request to SharePoint

Method:

GET

URI:

_api/web/RoleAssignments?$expand=Member,RoleDefinitionBindings

Headers:

Accept: application/json;odata=verbose

📌 Kết quả trả về dạng JSON

⸻

🧩 BƯỚC 2 — Parse JSON

Body mẫu (sample):

{
   "d":{
      "results":[
         {
            "Member":{
               "Title":"user@domain.com"
            },
            "RoleDefinitionBindings":{
               "results":[
                  {
                     "Name":"Contribute"
                  }
               ]
            }
         }
      ]
   }
}

Schema (dùng trong Parse JSON):

{
  "type": "object",
  "properties": {
    "d": {
      "type": "object",
      "properties": {
        "results": {
          "type": "array",
          "items": {
            "type": "object",
            "properties": {
              "Member": {
                "type": "object",
                "properties": {
                  "Title": { "type": "string" }
                }
              },
              "RoleDefinitionBindings": {
                "type": "object",
                "properties": {
                  "results": {
                    "type": "array",
                    "items": {
                      "type": "object",
                      "properties": {
                        "Name": { "type": "string" }
                      }
                    }
                  }
                }
              }
            }
          }
        }
      }
    }
  }
}


⸻

🧩 BƯỚC 3 — Filter chỉ lấy role Contribute

Trong Apply to each lặp d.results

Điều kiện:

contains(item()?['RoleDefinitionBindings']?['results']?[0]?['Name'], 'Contribute')


⸻

🧩 BƯỚC 4 — Check user đã tồn tại trong List chưa

Action: Get items (SharePoint list “UserPermissionsLog”)

Filter Query:

UserEmail eq '@{item()?['Member']?['Title']}'


⸻

🧩 BƯỚC 5 — Condition để bắt user mới

Condition:

length(body('Get_items')?['value']) is equal to 0

👉 Nếu đúng → user mới đc share

⸻

🧩 BƯỚC 6 — Ghi log vào List

Các trường:

Field	Value
UserEmail	@{item()?['Member']?['Title']}
Role	Contribute
TimeDetected	utcNow()


⸻

🧩 BƯỚC 7 — Gửi cảnh báo (Teams/Email)

Message:

⚠️ PHÁT HIỆN USER MỚI ĐƯỢC SHARE QUYỀN
Email: @{item()?['Member']?['Title']}
Role: Contribute
Time: @{utcNow()}


⸻

✔ MẪU FLOW HOÀN CHỈNH (Pseudo code)

Recurrence (5 phút)
   |
   V
HTTP GET: _api/web/RoleAssignments?$expand=Member,RoleDefinitionBindings
   |
   V
Parse JSON
   |
   V
Apply to each (d.results[])
    |
    +-- Condition: Name = Contribute ?
    |
    +-- If Yes:
            |
            V
         Get items from List where UserEmail = this user
            |
            V
          Condition length == 0 ?
               |
               +-- Yes: (User mới)
                        - Add item to list
                        - Send notification
               |
               +-- No: (User cũ) do nothing


⸻

🧪 TEST KẾT QUẢ

Test thực tế:
	•	Owner share site → Add member Contribute
	•	Flow chạy sau 5 phút
	•	Flow detect user mới
	•	→ Gửi cảnh báo

⸻

📌 Bạn cần mình tạo Flow file (.zip) cho bạn import?

👉 Chỉ cần gửi:
	•	Site URL
	•	Tên list log (nếu bạn chưa có mình sẽ tạo luôn)

Mình sẵn sàng làm mẫu chi tiết cho bạn 💪
