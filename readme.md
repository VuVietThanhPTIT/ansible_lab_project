# Sơ đồ kiến trúc: Ansible Multi-VM Docker Web Stack

## Thông tin IP các VM

| VM | Vai trò | IP | Group Ansible |
|---|---|---|---|
| VM1 | Load Balancer (Nginx - native) | 192.168.39.73 | loadbalancer |
| VM2 | Web App backend #1 (Docker) | 192.168.39.71 | webservers |
| VM3 | Web App backend #2 (Docker) | 192.168.39.54 | webservers |
| VM4 | Database (Docker - Postgres) | 192.168.39.15 | dbservers |

## Chú thích luồng dữ liệu

1. **User → VM1 (Nginx)**: request HTTP vào port 80, nginx nhận và đóng vai trò reverse proxy
2. **VM1 → VM2/VM3**: nginx dùng `upstream` block (render động qua Jinja2, loop qua `groups['webservers']`) để load-balance round-robin sang 2 container webapp qua port 5000
3. **VM2/VM3 → VM4**: mỗi container webapp đọc biến `DB_HOST=192.168.39.15` (lấy động qua `hostvars[groups['dbservers'][0]].ansible_host`, không hardcode) để kết nối Postgres qua port 5432
4. **Firewall trên VM4**: chỉ whitelist đúng 2 IP `192.168.39.71` và `192.168.39.54` được phép vào port 5432 — chặn mọi nguồn khác kể cả VM1
5. **Control node**: kết nối SSH riêng biệt tới cả 4 VM để chạy Ansible playbook — đây là kênh quản trị, tách biệt hoàn toàn với luồng traffic ứng dụng thật (đường nét đứt trong sơ đồ)

## Bước 1 : Cấu hình firewall 
- Viểt playbook testfirewall 
![alt text](img/p1.png)

## Bước 2 : Cài đặt firewall trên các VM 
- Viết file site.yml cập nhật thêm các role docker , firewall , 
- Dùng tag docker để chạy mỗi docker 
```
roles/docker/
└── tasks/
    └── main.yml
```
```
ansible-playbook site.yml --tags docker
```
## Bước 3 : Cấu hình database 
- Cấu trúc thư mục database trong ansible 
```
roles/database/
├── tasks/
│   └── main.yml
└── templates/
    └── db_env.j2
```
- Chạy test database 
## Bước 4 : Cấu hình webapp theo 
- Sử dụng images horlar/flask-restapi-crud-app để test thử crud  
```
roles/webapp/
├── tasks/
│   └── main.yml
└── templates/
    └── app_env.j2
```
## Bước 5 : Cấu hình nginx 
```
roles/loadbalancer/
├── tasks/
│   └── main.yml
├── templates/
│   └── nginx.conf.j2
└── handlers/
    └── main.yml
```

- Do tải image trên docker hub nên  vào thử container để đọc source code xem endpoint -> vào src/route.py
- Test thử link trang chủ 
```
HTTP/1.1 200 OK
Server: nginx/1.24.0 (Ubuntu)
Date: Fri, 24 Jul 2026 09:39:46 GMT
Content-Type: text/html; charset=utf-8
Content-Length: 118
Connection: keep-alive

<h1> Flask Crud APP</h1>
                <p> A flask rest api implementation using Docker Container. </p>

```
- Truy cập vào để test thử GET
```

root@thanhvv17:~/ansible_lab_project# curl -i http://192.168.39.73/bookapi/books
HTTP/1.1 500 INTERNAL SERVER ERROR
Server: nginx/1.24.0 (Ubuntu)
Date: Fri, 24 Jul 2026 09:44:02 GMT
Content-Type: application/json
Content-Length: 34
Connection: keep-alive

{"message":"error getting books"}
```
- nginx route thành công , lỗi bên trong bảng book db

- tcpdump đọc thử gói tin trên lb1
```

root@thanhvm-lab-1:~# sudo tcpdump -i any 'port 80' -nnv
tcpdump: data link type LINUX_SLL2
tcpdump: listening on any, link-type LINUX_SLL2 (Linux cooked v2), snapshot length 262144 bytes
22:37:50.234503 eth0  In  IP (tos 0x0, ttl 64, id 49079, offset 0, flags [DF], proto TCP (6), length 60)
    192.168.39.82.49284 > 192.168.39.73.80: Flags [S], cksum 0xaa86 (correct), seq 2099545936, win 64240, options [mss 1460,sackOK,TS val 315685506 ecr 0,nop,wscale 7], length 0
22:37:50.234587 eth0  Out IP (tos 0x0, ttl 64, id 0, offset 0, flags [DF], proto TCP (6), length 60)
    192.168.39.73.80 > 192.168.39.82.49284: Flags [S.], cksum 0xd01a (incorrect -> 0x1e15), seq 2656811015, ack 2099545937, win 65160, options [mss 1460,sackOK,TS val 1934081821 ecr 315685506,nop,wscale 7], length 0
22:37:50.235480 eth0  In  IP (tos 0x0, ttl 64, id 49080, offset 0, flags [DF], proto TCP (6), length 52)
    192.168.39.82.49284 > 192.168.39.73.80: Flags [.], cksum 0x4972 (correct), ack 1, win 502, options [nop,nop,TS val 315685508 ecr 1934081821], length 0
22:37:50.235514 eth0  In  IP (tos 0x0, ttl 64, id 49081, offset 0, flags [DF], proto TCP (6), length 128)
    192.168.39.82.49284 > 192.168.39.73.80: Flags [P.], cksum 0xc51b (correct), seq 1:77, ack 1, win 502, options [nop,nop,TS val 315685508 ecr 1934081821], length 76: HTTP, length: 76
        GET / HTTP/1.1
        Host: 192.168.39.73
        User-Agent: curl/8.5.0
        Accept: */*

22:37:50.235523 eth0  Out IP (tos 0x0, ttl 64, id 50919, offset 0, flags [DF], proto TCP (6), length 52)
    192.168.39.73.80 > 192.168.39.82.49284: Flags [.], cksum 0xd012 (incorrect -> 0x491e), ack 77, win 509, options [nop,nop,TS val 1934081822 ecr 315685508], length 0
22:37:50.239067 eth0  Out IP (tos 0x0, ttl 64, id 50920, offset 0, flags [DF], proto TCP (6), length 342)
    192.168.39.73.80 > 192.168.39.82.49284: Flags [P.], cksum 0xd134 (incorrect -> 0x7e95), seq 1:291, ack 77, win 509, options [nop,nop,TS val 1934081825 ecr 315685508], length 290: HTTP, length: 290
        HTTP/1.1 200 OK
        Server: nginx/1.24.0 (Ubuntu)
        Date: Fri, 24 Jul 2026 15:37:50 GMT
        Content-Type: text/html; charset=utf-8
        Content-Length: 118
        Connection: keep-alive

        <h1> Flask Crud APP</h1>
                        <p> A flask rest api implementation using Docker Container. </p>
                     [|http]
22:37:50.239186 eth0  In  IP (tos 0x0, ttl 64, id 49082, offset 0, flags [DF], proto TCP (6), length 52)
    192.168.39.82.49284 > 192.168.39.73.80: Flags [.], cksum 0xd012 (incorrect -> 0x47fd), ack 291, win 501, options [nop,nop,TS val 315685512 ecr 1934081825], length 0
22:37:50.239296 eth0  In  IP (tos 0x0, ttl 64, id 49083, offset 0, flags [DF], proto TCP (6), length 52)
    192.168.39.82.49284 > 192.168.39.73.80: Flags [F.], cksum 0xd012 (incorrect -> 0x47fc), seq 77, ack 291, win 501, options [nop,nop,TS val 315685512 ecr 1934081825], length 0
22:37:50.239308 eth0  Out IP (tos 0x0, ttl 64, id 50921, offset 0, flags [DF], proto TCP (6), length 52)
    192.168.39.73.80 > 192.168.39.82.49284: Flags [F.], cksum 0xd012 (incorrect -> 0x47f3), seq 291, ack 78, win 509, options [nop,nop,TS val 1934081825 ecr 315685512], length 0
22:37:50.239342 eth0  In  IP (tos 0x0, ttl 64, id 49084, offset 0, flags [DF], proto TCP (6), length 52)
    192.168.39.82.49284 > 192.168.39.73.80: Flags [.], cksum 0xd012 (incorrect -> 0x47fb), ack 292, win 501, options [nop,nop,TS val 315685512 ecr 1934081825], length 0

```

```mermaid
flowchart TD
    Start(["ansible-playbook site.yml --tags database"]) --> Cfg["Đọc ansible.cfg<br/>(inventory path, forks, remote_user...)"]
    Cfg --> Inv["Parse Inventory<br/>hosts.yml → dựng cây group/host<br/>all → webservers/dbservers/loadbalancer"]
    Inv --> Vars["Nạp group_vars/, host_vars/<br/>Gộp biến theo precedence cho từng host"]
    Vars --> Parse["Parse site.yml<br/>Từng Play: hosts, roles, tags<br/>import_tasks/import_role ráp nối NGAY (static)"]
    Parse --> Filter["Lọc theo --tags database<br/>Bỏ Play/task không khớp tag"]

    Filter --> PlayLoop{"Còn Play nào<br/>chưa chạy?"}
    PlayLoop -->|Có| Strategy["Strategy Plugin (linear/free)<br/>điều phối host nào chạy gì, khi nào"]
    Strategy --> TQM["Task Queue Manager<br/>Sinh worker process theo forks"]
    TQM --> Facts["Gather Facts (module setup)<br/>Thu thập ansible_distribution, ansible_architecture..."]

    Facts --> TaskLoop{"Còn Task nào<br/>chưa chạy trong Play?"}
    TaskLoop -->|Có| TE["TaskExecutor nhận task<br/>vd: template src=db_env.j2 dest=/opt/database/db.env"]

    TE --> ActionCheck{"Module hay<br/>Action Plugin đặc biệt?"}

    ActionCheck -->|"template/debug/assert..."| SpecialAction["Action Plugin đặc biệt<br/>CHẠY NGAY TRÊN CONTROL NODE"]
    SpecialAction --> Jinja["Jinja2 Engine render file .j2<br/>Đọc context biến của host hiện tại<br/>Thay {{ }} {% %} → giá trị thật"]
    Jinja --> TempFile["Sinh file kết quả tạm<br/>(text thuần, hết Jinja2)"]
    TempFile --> CallCopy["Gọi tiếp module copy<br/>để đẩy file tạm này đi"]
    CallCopy --> NormalAction

    ActionCheck -->|"module thường<br/>(docker_image, apt, ufw...)"| NormalAction["normal Action Plugin<br/>Nhạc trưởng chính"]

    NormalAction --> Conn["Connection Plugin (SSH)<br/>Mở/tái sử dụng kết nối (ControlPersist)<br/>TCP handshake → SSH handshake"]
    Conn --> Pack["module_common.py đóng gói module<br/>Nhận diện loại module<br/>Python → framework Ansiballz<br/>Zip + Base64 + wrapper script"]
    Pack --> Push["Đẩy module đã đóng gói sang Managed Node<br/>Pipelining: pipe qua stdin<br/>Không pipelining: ghi file tạm + exec riêng"]

    Push -.->|SSH| RemoteExec

    subgraph Remote["MANAGED NODE — thực thi thật"]
        RemoteExec["Wrapper giải mã zip<br/>Set PYTHONPATH, import module như __main__"]
        RemoteExec --> ModRun["Module thực thi<br/>vd: ghi file /opt/database/db.env<br/>So sánh checksum → idempotency check"]
        ModRun --> JSON["Sinh JSON kết quả<br/>{changed, failed, msg, rc...}"]
    end

    JSON -.->|SSH gửi ngược| Parse2

    Parse2["normal Action Plugin nhận JSON<br/>Parse thành dict Python<br/>Đánh dấu mọi string là Unsafe"]
    Parse2 --> Directive["TaskExecutor xử lý directive<br/>register → lưu biến<br/>changed_when/failed_when → override kết quả<br/>notify → đánh dấu handler sẽ chạy cuối Play"]
    Directive --> Callback["Callback Plugin in kết quả<br/>ra terminal: TASK [...] *** changed: [db1]"]

    Callback --> TaskLoop
    TaskLoop -->|Hết task| Handlers{"Có handler<br/>được notify?"}
    Handlers -->|Có| RunHandler["Chạy Handler<br/>(đi lại đúng luồng Task ở trên)"]
    RunHandler --> PlayLoop
    Handlers -->|Không| PlayLoop

    PlayLoop -->|Hết Play| Recap["PLAY RECAP<br/>ok= changed= failed=<br/>Trả exit code"]
    Recap --> End(["Kết thúc"])

    style Start fill:#fff3e0
    style End fill:#c8e6c9
    style Remote fill:#ffe0b2
    style SpecialAction fill:#bbdefb
    style Jinja fill:#bbdefb
    style NormalAction fill:#d1c4e9
    style Conn fill:#d1c4e9
    style ModRun fill:#ffccbc
```

---

## Chú thích đọc sơ đồ

- **Vùng màu cam (Remote/Managed Node)**: đây là phần DUY NHẤT thực sự chạy trên VM đích (db1/web1/web2/lb1) — mọi thứ còn lại chạy trên control node.
- **Vùng màu xanh dương (Action Plugin đặc biệt)**: chỉ áp dụng cho `template`, `debug`, `assert`... — các plugin này KHÔNG kết nối SSH, xử lý xong ngay tại chỗ.
- **Vùng màu tím (normal Action Plugin + Connection)**: mọi module thường (kể cả module được `template` gọi ngầm) đều đi qua đường này để tới được managed node.
- **Vòng lặp `TaskLoop`**: lặp lại cho từng task trong 1 role, đúng thứ tự viết trong file `tasks/main.yml`.
- **Vòng lặp `PlayLoop`**: lặp lại cho từng Play trong `site.yml` (Play cho `dbservers`, rồi Play cho `webservers`...).