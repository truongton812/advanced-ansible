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
