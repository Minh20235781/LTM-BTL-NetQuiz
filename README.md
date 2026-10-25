# NetQuiz – Ứng dụng thi trắc nghiệm trực tuyến

Bài tập lớn môn **Lập trình mạng (IT4062)**. NetQuiz là hệ thống thi trắc nghiệm client–server: nhiều người dùng kết nối tới một server qua **socket TCP**, tạo và tham gia phòng thi, làm bài trong thời gian giới hạn, nộp bài và nhận kết quả. Ngoài ra còn có chế độ luyện tập, lưu lịch sử và thống kê bằng đồ thị.

- **Ngôn ngữ:** C (gcc) trên Ubuntu 22.04, cả server và client
- **Mạng:** TCP, server một tiến trình dùng `select()`
- **Giao diện:** GTK3 (viết bằng C), đồ thị vẽ bằng Cairo
- **Lưu trữ:** file text

> Trạng thái: đang phát triển. Tên file thực thi và một số chi tiết dưới đây là thiết kế dự kiến, sẽ được cập nhật khi code hoàn thiện.

---

## 1. Thành viên

| Thành viên | Phụ trách |
|---|---|
| Hoàng | Khung mạng, tài khoản, phiên, kiểm soát quyền, ghi log; giao diện đăng nhập và cửa sổ chính |
| Trọng | Phòng thi và luồng thi (tạo, tham gia, bắt đầu, làm bài, nộp, chấm, kết quả); giao diện phòng và màn hình thi |
| Minh | Ngân hàng câu hỏi, luyện tập, lịch sử, thống kê đồ thị, quản trị câu hỏi; giao diện luyện tập, lịch sử, thống kê |

Cả ba thành viên đều viết phần server lẫn giao diện cho module của mình.

---

## 2. Chức năng

### Chế độ thi
- Tạo phòng thi: đặt số câu hỏi, thời gian thi, số câu dễ/vừa/khó, chủ đề
- Xem danh sách phòng và trạng thái: chưa bắt đầu, đang diễn ra, đã kết thúc
- Tham gia phòng khi phòng chưa bắt đầu
- Chủ phòng quyết định bắt đầu thi và có thể kết thúc sớm
- Khi bắt đầu, server gửi đề thi cho mọi người trong phòng
- Đổi đáp án đã chọn khi còn thời gian
- Nộp bài sớm; hết giờ server tự chấm và thông báo số câu đúng cho từng người
- Xem kết quả và bảng xếp hạng của phòng đã kết thúc

### Chế độ luyện tập
- Tự chọn số câu, thời gian, độ khó và chủ đề
- Nộp bài và nhận số câu đúng

### Chức năng khác
- Đăng ký, đăng nhập, quản lý phiên, phân quyền (khách, thí sinh, chủ phòng, quản trị viên)
- Phân loại câu hỏi theo độ khó (1–3) và chủ đề; sinh đề theo cấu hình
- Lưu lịch sử các bài đã làm, thống kê bằng đồ thị
- Ghi log hoạt động của server
- Giao diện đồ họa

### Chức năng nâng cao (dự kiến)
- Vào lại bài đang làm sau khi mất kết nối (RESUME)
- Băm mật khẩu có salt, khóa tạm khi đăng nhập sai nhiều lần, ngắt kết nối nhàn rỗi
- Bảng xếp hạng tức thời, chat trong phòng chờ
- Quản trị và import ngân hàng câu hỏi, xáo trộn đề cho từng thí sinh
- Tự tìm server bằng UDP broadcast

---

## 3. Kiến trúc

```
 Client 1 (GTK) ──┐
 Client 2 (GTK) ──┼── TCP :8888 ──►  SERVER (1 tiến trình, vòng lặp select)
 Client N (GTK) ──┘                   ├─ net: accept / recv / send, bộ đệm nhận mỗi kết nối
                                      ├─ dispatcher: kiểm tra trạng thái + quyền, gọi handler
                                      ├─ handler: auth | room | exam | practice | history
                                      ├─ timer: timeout của select (1 giây) kiểm tra phòng hết giờ
                                      └─ storage: users.txt, questions.txt, history.txt, server.log
```

**Vì sao dùng `select()` một tiến trình?** Trạng thái phòng thi là dữ liệu chung của nhiều client. Dùng một tiến trình thì không cần mutex hay IPC như khi dùng `fork` hoặc thread, việc gửi thông báo tới cả phòng chỉ là duyệt mảng client, và bộ đếm giờ tận dụng luôn timeout của `select()`. Giới hạn: handler phải ngắn, không được chặn; số kết nối bị giới hạn bởi `FD_SETSIZE` (1024), dư cho quy mô một lớp học.

Server giữ trạng thái (stateful): nó theo dõi phiên của từng client và trạng thái của từng phòng.

---

## 4. Giao thức tầng ứng dụng

### 4.1 Đóng khung message

TCP là dòng byte, không có ranh giới message, nên mọi message được đóng khung:

```
+-------------------+----------------------------------------+
| LEN (4 byte)      | PAYLOAD (LEN byte, text)               |
| uint32, htonl     | OPCODE|tham số 1|tham số 2|...         |
+-------------------+----------------------------------------+
```

- Payload là text, các trường cách nhau bởi ký tự `|`, trường đầu là mã lệnh (opcode)
- Payload tối đa 64 KB; `LEN` vượt giới hạn thì server đóng kết nối
- Dữ liệu người dùng không được chứa `|` và ký tự xuống dòng
- Mỗi kết nối có một bộ đệm nhận cộng dồn dữ liệu; hàm tách khung chỉ trả message khi đã đủ `LEN` byte. Cách này xử lý đúng cả trường hợp hai message dính nhau và một message bị tách qua nhiều lần `recv`
- Đề thi gửi thành nhiều message nhỏ: `EXAM_BEGIN`, các `QUESTION`, rồi `EXAM_READY`
- Server không bao giờ gửi đáp án đúng xuống client trong lúc thi

### 4.2 Các message chính

**Client gửi lên server**

| Opcode | Tham số | Trạng thái yêu cầu | Quyền |
|---|---|---|---|
| `REGISTER` | user, pass | CONNECTED | Khách |
| `LOGIN` | user, pass | CONNECTED | Khách |
| `RESUME` | token | CONNECTED | Khách |
| `LOGOUT` | | AUTH | Thí sinh |
| `LIST_ROOMS` | | AUTH | Thí sinh |
| `LIST_TOPICS` | | AUTH | Thí sinh |
| `CREATE_ROOM` | name, num_q, minutes, n_easy, n_med, n_hard, topic | AUTH | Thí sinh |
| `JOIN_ROOM` | room_id | AUTH | Thí sinh |
| `LEAVE_ROOM` | | IN_ROOM | Thí sinh |
| `START_ROOM` | | IN_ROOM | Chủ phòng |
| `END_ROOM` | | IN_EXAM | Chủ phòng |
| `ANSWER` | q_no, choice (A–D) | IN_EXAM, IN_PRACTICE | Thí sinh |
| `SUBMIT` | | IN_EXAM, IN_PRACTICE | Thí sinh |
| `GET_ROOM_RESULT` | room_id | AUTH | Thí sinh |
| `PRACTICE_START` | num_q, minutes, n_easy, n_med, n_hard, topic | AUTH | Thí sinh |
| `GET_HISTORY` | | AUTH | Thí sinh |
| `GET_STATS` | | AUTH | Thí sinh |
| `ADD_QUESTION` | topic, diff, text, A, B, C, D, correct | AUTH | Quản trị |

**Server trả về**

| Message | Ý nghĩa |
|---|---|
| `OK` | Thành công (kèm thông tin; `LOGIN` trả thêm token và vai trò) |
| `ERR` | Thất bại, kèm mã lỗi và mô tả |
| `ROOM_ITEM` ... `ROOM_LIST_END` | Danh sách phòng |
| `EXAM_BEGIN`, `QUESTION`, `EXAM_READY` | Gửi đề thi |
| `RESULT_ITEM` ... `RESULT_END` | Bảng điểm của phòng |
| `HISTORY_ITEM` ... `HISTORY_END` | Lịch sử bài làm |

**Sự kiện server chủ động đẩy xuống**

| Sự kiện | Khi nào |
|---|---|
| `EVT_PLAYER_JOINED`, `EVT_PLAYER_LEFT` | Có người vào hoặc rời phòng |
| `EVT_ROOM_STARTED` | Chủ phòng bắt đầu thi |
| `EVT_TIME_SYNC` | Định kỳ, đồng bộ thời gian còn lại (server là nguồn thời gian chuẩn) |
| `EVT_ROOM_FINISHED` | Hết giờ hoặc chủ phòng kết thúc, kèm số câu đúng |

### 4.3 Mã lỗi

| Mã | Ý nghĩa |
|---|---|
| 200 | Thành công |
| 400 | Sai định dạng message |
| 401 | Chưa đăng nhập |
| 403 | Không đủ quyền (không phải chủ phòng, không phải quản trị) |
| 404 | Không tìm thấy (phòng, câu hỏi) |
| 409 | Xung đột (tên đã tồn tại, tài khoản đang đăng nhập nơi khác, phòng đã bắt đầu) |
| 410 | Hết hạn (phòng đã kết thúc, hết giờ làm bài) |
| 422 | Tham số không hợp lệ (ví dụ số câu vượt ngân hàng câu hỏi) |
| 500 | Lỗi server |

### 4.4 Máy trạng thái

**Kết nối của mỗi client:**

```
CONNECTED --LOGIN--> AUTH --CREATE_ROOM/JOIN_ROOM--> IN_ROOM --START_ROOM--> IN_EXAM
                      ^  |                              |                       |
                      |  +--PRACTICE_START--> IN_PRACTICE                       |
                      +-------- LEAVE_ROOM / SUBMIT / hết giờ / END_ROOM -------+
```

**Phòng thi:** `WAITING` (cho phép vào) → `RUNNING` (đang thi) → `FINISHED` (chỉ xem kết quả).

Việc kiểm tra "lệnh này có được gọi ở trạng thái và vai trò hiện tại không" được thực hiện tại một chỗ duy nhất (bộ phân phối lệnh), nên các handler không phải tự kiểm tra quyền.

### 4.5 Luồng thi tiêu biểu

```
 Host              Server                        Thí sinh B
  |--CREATE_ROOM--->|  tạo phòng WAITING             |
  |<--OK------------|                                |
  |                 |<----------- JOIN_ROOM ---------|
  |<-EVT_PLAYER_JOINED--------------OK-------------->|
  |--START_ROOM---->|  RUNNING, đặt giờ kết thúc     |
  |<-EVT_ROOM_STARTED, EXAM_BEGIN, QUESTION x N, EXAM_READY->|
  |--ANSWER-------->|  ghi / đổi đáp án              |
  |                 |<------------ SUBMIT -----------|
  |                 |  chấm bài B, lưu lịch sử       |
  |<------------ EVT_ROOM_FINISHED (score,total) --->|  (hết giờ: chấm người còn lại)
```

---

## 5. Dữ liệu

| File | Định dạng một dòng |
|---|---|
| `data/users.txt` | `username:salt:hash:role:ngày_tạo` (mật khẩu được băm, không lưu thô) |
| `data/questions.txt` | `id\|topic\|diff(1-3)\|nội dung\|A\|B\|C\|D\|đáp án` |
| `data/history.txt` | `username\|mode\|room_id\|thời gian\|topic\|số_câu\|số_đúng\|giây_đã_dùng` |
| `data/rooms.txt` | thông tin các phòng đã kết thúc |
| `data/results_room_N.txt` | bảng điểm của phòng N |
| `logs/server.log` | log hoạt động, ví dụ: `2026-11-02 14:05:01 [INFO] fd=5 user=hoang CMD=JOIN_ROOM room=3 -> 200` |

Server nạp người dùng và ngân hàng câu hỏi vào bộ nhớ khi khởi động và ghi thêm vào file khi có thay đổi. Câu hỏi không được chứa ký tự `|`.

**Sinh đề:** lọc theo chủ đề và độ khó, xáo trộn (Fisher–Yates), lấy đúng số câu mỗi mức; trả lỗi 422 nếu không đủ câu.

---

## 6. Cấu trúc thư mục (dự kiến)

```
netquiz/
├── common/        protocol.h, protocol.c        (đóng khung, dùng chung server và client)
├── server/        net, session, auth, room, exam, practice, history, logger
├── client/        net_client, các file gui_* (GTK3)
├── data/          users.txt, questions.txt, history.txt, ...
├── logs/          server.log
├── test/          công cụ gửi khung thô và script kiểm thử
├── docs/          protocol.md, báo cáo, sơ đồ UML
└── Makefile
```

---

## 7. Cài đặt và chạy

**Yêu cầu:** Ubuntu 22.04, gcc, make, GTK3, libcrypt.

```bash
sudo apt update
sudo apt install build-essential libgtk-3-dev valgrind
```

**Biên dịch:**

```bash
make            # build server và client
make clean
```

**Chạy server** (cổng mặc định 8888):

```bash
./server/netquiz_server 8888
```

**Chạy client** (mỗi người dùng một máy hoặc một cửa sổ):

```bash
./client/netquiz_client
```

Nhập địa chỉ IP và cổng của server trên màn hình đăng nhập, sau đó đăng ký tài khoản và đăng nhập.

**Tạo tài khoản quản trị:** đăng ký một tài khoản bình thường, dừng server, đổi trường `role` của tài khoản đó thành `admin` trong `data/users.txt`, rồi chạy lại server.

---

## 8. Hướng dẫn sử dụng nhanh

1. **Thi:** đăng nhập → tab *Phòng thi* → *Tạo phòng* (hoặc chọn một phòng đang chờ để tham gia) → chủ phòng bấm *Bắt đầu* → chọn và đổi đáp án → *Nộp bài* hoặc đợi hết giờ → xem số câu đúng và bảng xếp hạng.
2. **Luyện tập:** tab *Luyện tập* → chọn số câu, thời gian, độ khó, chủ đề → làm bài → nộp.
3. **Lịch sử và thống kê:** tab *Lịch sử* xem các bài đã làm; tab *Thống kê* xem đồ thị điểm theo thời gian và theo chủ đề.

---

## 9. Kiểm thử

Do giao thức dùng tiền tố độ dài nên không thể thử trực tiếp bằng `netcat`; dùng công cụ trong `test/` để dựng và gửi khung.

| Nhóm | Ca kiểm thử | Kết quả mong đợi |
|---|---|---|
| Truyền dòng | Gửi một khung từng byte một | Server ghép đúng, xử lý một lần |
| Truyền dòng | Gửi hai khung trong một lần `send` | Server xử lý đủ hai lệnh theo thứ tự |
| Truyền dòng | `LEN` lớn hơn 64 KB hoặc khung sai | Ghi log và đóng kết nối, server không sập |
| Đồng thời | 30 kết nối cùng đăng nhập và vào phòng | Không treo, không lẫn dữ liệu |
| Mất kết nối | `kill -9` client giữa lúc thi | Server vẫn chạy, đáp án giữ nguyên, hết giờ vẫn chấm |
| Thời gian | Phòng 1 phút, không ai nộp | Đúng giờ server chấm và gửi `EVT_ROOM_FINISHED` |
| Quyền | Người không phải chủ phòng gửi `START_ROOM` | `ERR 403` |
| Trạng thái | `JOIN_ROOM` khi phòng đã chạy; `ANSWER` sau `SUBMIT` | `ERR 409`, `ERR 410` |
| Chất lượng mã | `valgrind`, biên dịch với `-Wall -Wextra` | Không rò rỉ bộ nhớ, không cảnh báo |

---

## 10. Phân công điểm theo chức năng

| Chức năng | Điểm | Người phụ trách |
|---|---|---|
| Xử lý truyền dòng | 1 | Hoàng |
| Socket I/O trên server (select) | 2 | Hoàng |
| Đăng ký và quản lý tài khoản | 2 | Hoàng |
| Đăng nhập và quản lý phiên | 2 | Hoàng |
| Kiểm soát quyền truy cập | 1 | Hoàng |
| Ghi log hoạt động | 1 | Hoàng |
| Tạo phòng thi | 2 | Trọng |
| Xem danh sách phòng | 1 | Trọng |
| Tham gia phòng | 2 | Trọng |
| Bắt đầu bài thi | 1 | Trọng |
| Thay đổi phương án trả lời | 1 | Trọng |
| Nộp bài và chấm điểm | 2 | Trọng |
| Xem kết quả phòng | 1 | Trọng |
| Luyện tập | 1 | Minh |
| Phân loại câu hỏi theo độ khó, chủ đề | 1–3 | Minh |
| Lưu trữ bài thi và đồ thị | 2 | Minh |
| Giao diện đồ họa | 3 | Cả ba |

---

## 11. Quy ước làm việc nhóm

- Một repo Git, mỗi người một nhánh theo chức năng (`feature/auth`, `feature/room`, `feature/practice`), merge vào `main` khi build sạch
- `docs/protocol.md` là hợp đồng giữa ba người: sửa giao thức phải báo cả nhóm
- Mã C viết đơn giản, mỗi hàm một việc, đặt tên rõ ràng, bám nội dung bài giảng của môn học

---

## 12. Tài liệu tham khảo

- Bài giảng Lập trình mạng (IT4062), SoICT, HUST
- W. Richard Stevens, *Unix Network Programming, Vol. 1*, 3rd edition
- Michael Donahoo, Kenneth Calvert, *TCP/IP Sockets in C: Practical Guide for Programmers*
