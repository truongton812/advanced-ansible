#### Asynchronous Action:

By default, để run task trên target server, ansible server cần mở một SSH connection đến target server và duy trì nó trong suốt thời gian chạy task (gọi là synchronous action)
Ta có thể thay đổi thành asynchronous action, tức server chỉ gửi lệnh đến target và đóng luôn SSH connection

VD: chạy 1 script health check, gather metric cần thời gian 5 phút để xong, ta nên dùng asynchronous để không phải quan tâm đến script đấy, chỉ quay lại check sau 5'
Các option của asynchronos: async - là khoảng thời gian tối đa mà 1 task có thể chạy, nếu quá thời gian ansible sẽ chuyển qua task tiếp theo và poll - là khoảng thời gian ansible sẽ quay lại để check kết quả (default là 10s)

Example 1
```
# Ansible Playbook
- name: Deploy Web Application
  hosts: db_and_web_server
  tasks:
    - command: /opt/monitor_webapp.py
      async: 360
      poll: 60

    - command: /opt/monitor_database.py
      async: 360
      poll: 60
```
-> Behavior của ansible: chạy 1st task, 60s quay lại check 1 lần, chờ cho hết 6 phút sau đó chuyển qua 2nd task
 Ansible sẽ kiểm tra trạng thái tác vụ không đồng bộ này mỗi 60 giây. Nếu sau 360 giây mà tác vụ vẫn chưa kết thúc, Ansible sẽ kết thúc việc theo dõi.
 Như vậy, mỗi lệnh sẽ được kích hoạt trên remote host rồi chạy ở chế độ nền (background) trong tối đa 6 phút, và Ansible theo dõi bằng cách hỏi kết quả mỗi phút một lần nhưng không chặn hoặc đợi lệnh hoàn tất ngay lập tức

Example 2
```
# Ansible Playbook
- name: Deploy Web Application
  hosts: db_and_web_server
  tasks:
    - command: /opt/monitor_webapp.py
      async: 360
      poll: 0

    - command: /opt/monitor_database.py
      async: 360
      poll: 0
```
-> Behavior của ansible: sau khi gửi lệnh chạy task 1 sẽ kick off luôn task 2, ansible cũng không quay lại để check result task 1 nữa. Để muốn chạy song song 2 task mà vẫn handle việc quay lại check thì ta cần register result của task 1 thành variable

Example 3
```
# Ansible Playbook
-
  name: Deploy Web Application
  hosts: db_and_web_server
  tasks:
    - command: /opt/monitor_webapp.py
      async: 360
      poll: 0
      register: webapp_result

    - command: /opt/monitor_database.py
      async: 360
      poll: 0
      register: database_result

    - name: Check status of tasks
      async_status: jid={{ webapp_result.ansible_job_id }}
      register: job_result
      until: job_result.finished
      retries: 30
```
async_status module dùng để check status của 1 async task, và module này cần đầu vào là job id. Lấy job id bằng cách register result của task thành variable và lấy ra job id từ đấy
     

#### strategy: định nghĩa cách playbook được thực thi
khi ansible chạy trên nhiều servers thì default strategy là linear : chạy task 1 trên tất cả các servers, nếu tất cả sv xong task 1 thì mới chuyển sang task 2
strategy free: mỗi server tự chạy 1 lifecycle, server nào xong task 1 trước tự chuyển sang task 2 ko cần đợi ai
strategy batch (chỉ định bằng serial: number hoặc percentage): chạy theo batch (VD 3 trong 5 sv). Và strategy khi chạy trong 1 batch là linear. 

Mặc định ansible có thể chạy trên 5 sv cùng lúc (cấu hình ở ansible.cfg, trường forks). Ta có thể tăng lên bao nhiêu forks cũng được, miễn đủ CPU và bandwidth


#### error handling
Khi ansibble chạy trên nhiều servers với thì by default, nếu có 1 server bị fail ở 1 bước nào đấy thì ansible sẽ loại bỏ server đấy ra khỏi list và tiếp tục chạy trên các server còn lại
Ta có thể thay đổi thành any_error_fatal: true -> nếu có 1 server fail, ansible dừng thực thi ở tất cả các server và thoát

ignore_errors: yes -> dùng để tiếp tục chạy playbook kể cả khi task fail

failed_when: dùng để đánh fail dựa vào condition
VD
```
- command: cat /var/log/server.log
  register: command_output
  failed_when: "'ERROR' in command_output.stdout"
```
-> fail nếu trong file log có chữ ERROR. Nếu không có failed_when thì task sẽ thành công nếu file server.log tồn tại. Note là failed_when sẽ fail condition = true

#### String manipulation - filter
The name is {{ my_name }} => The name is Bond

The name is {{ my_name | upper }} => The name is BOND

The name is {{ my_name | lower }} => The name is bond

The name is {{ my_name | title }} => The name is Bond

The name is {{ my_name | replace("Bond", "Bourne") }} => The name is Bourne

The name is {{ first_name | default("James") }} {{ my_name }} => The name is James Bond

#### filter - list and set 
{{ [1, 2, 3] | min }}                 => 1

{{ [1, 2, 3] | max }}                 => 3

{{ [1, 2, 3, 2] | unique }}           => 1, 2, 3

{{ [1, 2, 3, 4] | union([4, 5]) }}    => 1, 2, 3, 4, 5

{{ [1, 2, 3, 4] | intersect([4, 5])}} => 4

{{ 100 | random }}                    => Random number

{{ ["The", "name", "is", "Bond"] | join(" ") }} => The name is Bond

#### filter - file

{{ "/etc/hosts" | basename }}                           => hosts

{{ "c:\windows\hosts" | win_basename }}                 => hosts

{{ "c:\windows\hosts" | win_splitdrive }}               => ["c:", "\windows\hosts"]

{{ "c:\windows\hosts" | win_splitdrive | first }}       => "c:"

{{ "c:\windows\hosts" | win_splitdrive | last }}        => "\windows\hosts"

#### lookups plugin
Use case: đọc nội dung file, VD file csv có format là Hostname,Password thì ta có thể lấy ra Password

{{ lookup('csvfile', 'target1 file=/tmp/credentials.csv delimiter=,') }}

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
Example inventory.py
```
#!/usr/bin/env python

import json

# Get inventory data from source - CMDB or any other API
def get_inventory_data():
    return {
        "databases": {
            "hosts": ["db_server"],
            "vars": {
                "ansible_ssh_pass": "Passw0rd",
                "ansible_ssh_host": "192.168.1.1"
            }
        },
        "web": {
            "hosts": ["web_server"],
            "vars": {
                "ansible_ssh_pass": "Passw0rd",
                "ansible_ssh_host": "192.168.1.2"
            }
        }
    }

# Default main function
if __name__ == "__main__":
    inventory_data = get_inventory_data()
    print(json.dumps(inventory_data))
```
ansible-playbook main.yaml -i inventory.py
Mục đích của script inventory.py này là in ra thông tin inventory, sẽ là thông tin đầu vào cho ansible thực thi (lưu ý đây là khai báo static, còn trong thực tế thì inventory.py sẽ là script lấy thông tin từ external source và in ra)
Format của output phải là json và là dạng dictionary of group

Example 2
```
#!/usr/bin/env python

import json

# Get inventory data from source - CMDB or any other API
def get_inventory_data():
    return {
        "databases": {
            "hosts": ["db_server"],
            "vars": {
                "ansible_ssh_pass": "Passw0rd",
                "ansible_ssh_host": "192.168.1.1"
            }
        },
        # . . . . . . // Truncated
    }

def read_cli_args():
    global args
    parser = argparse.ArgumentParser()
    parser.add_argument('--list', action='store_true')
    parser.add_argument('--host', action='store')
    args = parser.parse_args()

# Default main function
if __name__ == "__main__":
    global args
    read_cli_args()
    inventory_data = get_inventory_data()
    if args.list:
        print(json.dumps(inventory_data))
    elif args.host:
        print(json.dumps({'_meta': {'hostvars': {}}}))
```

Đây là script có truyền vào argumement, chạy lệnh 
./inventory.py --list -> để lấy ra list các target
./inventory.py --host web -> để lấy thông tin của target tên web
