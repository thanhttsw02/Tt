# Tt

Hướng dẫn Flow đầy đủ (có Parse JSON) — Từ GET → Detect → EnsureUser → Remove → Log → Notify

Mình sẽ mô tả từng action trong Power Automate, copy/paste trực tiếp vào flow của bạn. Flow này chạy theo Recurrence (ví dụ 5 phút) và xử lý được nhiều user cùng lúc. Mình dùng tên action mẫu — bạn có thể đổi tên nhưng nếu đổi thì sửa lại các expression tương ứng.

Yêu cầu quyền: account chạy Flow cần Full Control / ManagePermissions trên site để gọi EnsureUser và RoleAssignments/RemoveByPrincipalId.

⸻

1) Trigger — Recurrence
	•	Action: Recurrence
	•	Frequency: Minute
	•	Interval: 5 (hoặc 1/15 tùy bạn)

⸻

2) Lấy RoleAssignments (HTTP GET)
	•	Action: Send an HTTP request to SharePoint
	•	Site Address: https://TENANT.sharepoint.com/sites/YourSite
	•	Method: GET
	•	URI:

_api/web/RoleAssignments?$expand=Member,RoleDefinitionBindings&$select=Member/LoginName,Member/Title,RoleDefinitionBindings/Name

	•	Headers:

Accept: application/json;odata=verbose

	•	Tên action mình dùng: HTTP_Get_RoleAssignments

⸻

3) Parse JSON — RoleAssignments
	•	Action: Parse JSON
	•	Content: body('HTTP_Get_RoleAssignments')
	•	Schema: (copy/paste toàn bộ)

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
                  "LoginName": { "type": "string" },
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

	•	Tên action: Parse_RoleAssignments

Ghi chú: Parse JSON chỉ để dễ truy xuất d.results và có intellisense. Bạn vẫn có thể dùng body() expressions nếu muốn.

⸻

4) Apply to each — Lặp qua từng RoleAssignment
	•	Action: Apply to each
	•	Value:

body('Parse_RoleAssignments')?['d']?['results']

	•	Tên action: ForEach_RoleAssignment

Bên trong Apply to each sẽ có nhiều action con — mình liệt kê theo thứ tự.

⸻

4.1) Compose — Lấy LoginName, Title
	•	Action: Compose (tên: Compose_LoginName)
	•	Input:

items('ForEach_RoleAssignment')?['Member']?['LoginName']

	•	Action: Compose (tên: Compose_Title)
	•	Input:

items('ForEach_RoleAssignment')?['Member']?['Title']


⸻

4.2) Compose — Lấy RoleNames (optional, để kiểm tra nếu cần)
	•	Action: Compose (tên: Compose_Roles)
	•	Input (lấy tất cả role names nối thành chuỗi):

join(items('ForEach_RoleAssignment')?['RoleDefinitionBindings']?['results']?['Name'],',')

Nếu expression trên lỗi (vì kết cấu mảng), bạn có thể dùng first() hoặc kiểm tra từng phần; mục đích là để log role dễ nhìn.

⸻

4.3) Get items — Kiểm tra LogList đã có login này chưa
	•	Action: Get items (SharePoint)
	•	Site Address: https://TENANT.sharepoint.com/sites/YourSite
	•	List Name: UserPermissionsLog (tạo sẵn; trường chính LoginName kiểu Single line text)
	•	Filter Query (OData):

LoginName eq '@{outputs('Compose_LoginName')}'

	•	Tên action: SP_Get_Log

Nếu internal column name khác, dùng internal name thay LoginName.

⸻

4.4) Condition — Nếu chưa có trong Log (mới)
	•	Action: Condition
	•	Expression left:

length(body('SP_Get_Log')?['value'])

	•	Operator: is equal to
	•	Expression right: 0

If yes (mới) → thực hiện remove + ghi log + notify
If no → bỏ qua (đã xử lý)

⸻

5) Branch If yes (chưa có) — Remove flow

5.1) Compose — URL-encode LoginName (để dùng EnsureUser)
	•	Action: Compose (tên: Compose_EncodedLoginName)
	•	Input:

encodeUriComponent(outputs('Compose_LoginName'))


⸻

5.2) EnsureUser — Lấy PrincipalId
	•	Action: Send an HTTP request to SharePoint
	•	Site Address: như trước
	•	Method: GET
	•	URI:

_api/web/EnsureUser()?loginName='@{outputs('Compose_EncodedLoginName')}'

	•	Headers:

Accept: application/json;odata=verbose

	•	Tên action: HTTP_EnsureUser

⸻

5.3) Parse JSON — EnsureUser (lấy d.Id)
	•	Action: Parse JSON
	•	Content: body('HTTP_EnsureUser')
	•	Schema:

{
  "type": "object",
  "properties": {
    "d": {
      "type": "object",
      "properties": {
        "Id": { "type": "integer" },
        "LoginName": { "type": "string" },
        "Title": { "type": "string" }
      }
    }
  }
}

	•	Tên action: Parse_EnsureUser

⸻

5.4) Initialize variable — principalId (optional)
	•	Action: Initialize variable
	•	Name: principalId
	•	Type: Integer
	•	Value:

body('Parse_EnsureUser')?['d']?['Id']


⸻

5.5) (Optional) Kiểm tra user có nằm trong Site Group “Members”
	•	Action: Send an HTTP request to SharePoint
	•	Method: GET
	•	URI:

_api/web/sitegroups/getbyname('Members')/users?$filter=LoginName eq '@{outputs('Compose_LoginName')}'

	•	Headers:

Accept: application/json;odata=verbose

	•	Tên action: HTTP_Check_InGroup

Bạn có thể kiểm tra length(body('HTTP_Check_InGroup')?['d']?['results']) để biết có >0 (có trong group).

⸻

5.6) Nếu có trong Group → Remove khỏi Group (optional)
	•	Action: Send an HTTP request to SharePoint
	•	Method: POST
	•	URI:

_api/web/sitegroups/getByName('Members')/users/removeByLoginName(@v)?@v='@{outputs('Compose_LoginName')}'

	•	Headers:

Accept: application/json;odata=verbose
Content-Type: application/json;odata=verbose

	•	Body: leave empty
	•	Tên action: HTTP_RemoveFromGroup

Lưu ý: một số tenant cần escape/format khác — nếu lỗi thử removeByLoginName(@v)?@v='i%3A0%23.f%7Cmembership%7Cuser%40domain.com' (URL encoded inside) hoặc thử dùng removeById với user id.

⸻

5.7) Remove RoleAssignment (xóa mọi quyền) — bắt buộc
	•	Action: Send an HTTP request to SharePoint
	•	Method: POST
	•	URI:

_api/web/RoleAssignments/RemoveByPrincipalId(@{body('Parse_EnsureUser')?['d']?['Id']})

	•	Headers:

Accept: application/json;odata=verbose
Content-Type: application/json;odata=verbose

	•	Body: leave empty
	•	Tên action: HTTP_RemoveRoleAssignment

Nếu trả 403: kiểm tra quyền account chạy flow.

⸻

6) Sau khi Remove — Ghi log vào SharePoint List
	•	Action: Create item (SharePoint)
	•	Site Address: https://TENANT.sharepoint.com/sites/YourSite
	•	List Name: UserPermissionsLog
	•	Fields:
	•	Title: Removed @{outputs('Compose_LoginName')}
	•	LoginName: @{outputs('Compose_LoginName')}
	•	PrincipalId: @{body('Parse_EnsureUser')?['d']?['Id']}
	•	Roles: @{outputs('Compose_Roles')}
	•	DetectedTime: @{utcNow()}
	•	ActionTaken: Removed RoleAssignment (and removed from Members group if existed)
	•	Tên action: SP_Create_Log

⸻

7) Gửi thông báo (gom nhiều user hoặc từng user)

Bạn có 2 cách:
	•	Gửi riêng cho mỗi user (trong Apply to each) — nhanh triển khai.
	•	Gom vào array và gửi 1 email/tin nhắn sau Apply to each.

Cách gom (ví dụ)
	•	Trước Apply to each, Initialize variable RemovedUsers type Array, giá trị [].
	•	Trong branch If yes sau khi Remove thành công: Append to array variable RemovedUsers giá trị:

{
  "LoginName": "@{outputs('Compose_LoginName')}",
  "PrincipalId": "@{body('Parse_EnsureUser')?['d']?['Id']}",
  "Time": "@{utcNow()}"
}

	•	Sau Apply to each, Condition: length(variables('RemovedUsers')) > 0 → nếu true, Send an email hoặc Post message to Teams liệt kê array (dùng join() hoặc convert to string).

⸻

8) Error handling & Retry
	•	Với các HTTP actions (HTTP_EnsureUser, HTTP_RemoveRoleAssignment, HTTP_RemoveFromGroup) bạn có thể Configure run after để retry khi fail or timeouts.
	•	Nếu HTTP_EnsureUser fail → Create item log lỗi (Status = Failed) và notify admin.

⸻

9) Tổng hợp biểu diễn flow (pseudo-steps)
	1.	Recurrence trigger
	2.	HTTP_Get_RoleAssignments (GET)
	3.	Parse_RoleAssignments
	4.	ForEach_RoleAssignment over d.results:
	•	Compose_LoginName, Compose_Title, Compose_Roles
	•	SP_Get_Log (Filter LoginName)
	•	Condition (length == 0)? If yes:
	•	Compose_EncodedLoginName
	•	HTTP_EnsureUser (GET)
	•	Parse_EnsureUser
	•	Initialize/Set principalId
	•	HTTP_Check_InGroup (optional)
	•	HTTP_RemoveFromGroup (if in group)
	•	HTTP_RemoveRoleAssignment (POST RemoveByPrincipalId)
	•	SP_Create_Log
	•	Append to RemovedUsers array
	5.	After loop: If RemovedUsers not empty → Send mail/Teams summary

⸻

10) Một số chú ý thực tế và troubleshooting
	•	LoginName: thường có dạng i:0#.f|membership|user@domain.com. Luôn dùng fullname loginName khi EnsureUser / Remove.
	•	Encode: dùng encodeUriComponent() cho query string trong EnsureUser(). Khi dùng trong removeByLoginName(@v) thường truyền nguyên chuỗi trong phần @v (thử 2 cách nếu lỗi).
	•	Quyền: Flow account phải có quyền ManagePermissions. Nếu không, RemoveByPrincipalId trả 403.
	•	Throttle: nếu site có nhiều RoleAssignments, tăng interval recurrence hoặc xử lý batch.
	•	Backup: luôn log trước khi xóa để có thể phục hồi nếu xóa nhầm.
	•	Test: test trên site dev trước, thử với 1 user external.

⸻

11) Sample expressions hữu ích (copy/paste)
	•	Lấy mảng results:

body('Parse_RoleAssignments')?['d']?['results']

	•	Lấy login trong foreach:

items('ForEach_RoleAssignment')?['Member']?['LoginName']

	•	Kiểm tra length của Get items:

length(body('SP_Get_Log')?['value'])

	•	Lấy principalId sau Parse_EnsureUser:

body('Parse_EnsureUser')?['d']?['Id']

	•	Encode loginName:

encodeUriComponent(outputs('Compose_LoginName'))


⸻

12) Muốn mình export flow (package) để bạn import không?

Mình có thể chuẩn bị flow JSON export với:
	•	giá trị site URL đã điền (bạn cung cấp)
	•	tên list UserPermissionsLog (mình sẽ tạo template fields)
Nếu bạn muốn, gửi Site URL và mình trả lại file importable.

⸻

Nếu bạn muốn, mình cũng có thể gởi bản tóm tắt ngắn (checklist) để bạn copy từng action vào Power Automate — hoặc mình build file export luôn (bạn chỉ cần import + cung cấp connections). Bạn chọn nhé — mình làm tiếp luôn.
