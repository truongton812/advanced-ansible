#### Asynchronous Action:

By default, để run task trên target server, ansible server cần mở một SSH connection đến target server và duy trì nó trong suốt thời gian chạy task (gọi là synchronous action)
Ta có thể thay đổi thành asynchronous action, tức server chỉ gửi lệnh đến target và đóng luôn SSH connection

VD: chạy 1 script health check, gather metric cần thời gian 5 phút để xong, ta nên dùng asynchronous để không phải quan tâm đến script đấy, chỉ quay lại check sau 5'
Các option của asynchronos: async - là khoảng thời gian mà task cần để chạy và poll - là khoảng thời gian ansible sẽ quay lại để check kết quả (default là 10s)

Bổ sung nội dung từ ảnh chụp màn hình


#### strategy: định nghĩa cách playbook được thực thi
khi ansible chạy trên nhiều servers thì default strategy là linear : chạy task 1 trên tất cả các servers, nếu tất cả sv xong task 1 thì mới chuyển sang task 2
strategy free: mỗi server tự chạy 1 lifecycle, server nào xong task 1 trước tự chuyển sang task 2 ko cần đợi ai
strategy batch (chỉ định bằng serial: number hoặc percentage): chạy theo batch (VD 3 trong 5 sv). Và strategy khi chạy trong 1 batch là linear. 

Mặc định ansible có thể chạy trên 5 sv cùng lúc (cấu hình ở ansible.cfg, trường forks). Ta có thể tăng lên bao nhiêu forks cũng được, miễn đủ CPU và bandwidth


#### error handling
Khi ansibble chạy trên nhiều servers với thì by default, nếu có 1 server bị fail ở 1 bước nào đấy thì ansible sẽ loại bỏ server đấy ra khỏi list và tiếp tục chạy trên các server còn lại
Ta có thể thay đổi thành any_error_fatal: true -> nếu có 1 server fail, ansible dừng thực thi ở tất cả các server và thoát

ignore_errors: yes -> dùng để tiếp tục chạy playbook kể cả khi task fail


lookups plugin
Use case: đọc nội dung file, VD file csv có format là Hostname,Password thì ta có thể lấy ra Password
Ngoài csv còn có thể đọc ini, dns, mongodb


#### Vault
Dùng để lưu các sensitive data dưới dạng encrypted . VD mã hóa file inventory : #ansible-vault encrypt inventory.txt (sau đó nhập password để tạo vault)
Lưu ý khi này inventory.txt sẽ bị mã hóa và không thể dùng như bình thường, phải thêm option -ask-vault-pass
VD ansible-playbook main.yaml -i inventory -ask-vault-pass -> sau đó nhập password lúc tạo vault
Ngoài ra có thể lưu password vault ra 1 file và chỉ định file đấy lúc chạy
ansible-playbook main.yaml -i inventory -vault-password-file /path/to/file -> cũng chưa bảo mật lắm.
cách tốt nhất là trỏ -vault-password-file về 1 script dùng để retrieve vault password và pass qua cho ansible playbook thực thi

ansible-vault view inventory.txt -> đọc nội dung file mã hóa
ansible-vault create inventory.txt -> tạo file mã hóa

#### dynamic inventory
dùng để lấy thông tin target từ external source như CMDB database, cloud platform API
