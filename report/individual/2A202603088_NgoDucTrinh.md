# Báo cáo Day 13 — Ngọ Đức Trinh, 2A202603088

## 1. Kết luận trước
- Mã số sinh viên: **2A202603088**
- Task CVAT: **`D13-2A202603088`**
- 10 frame của tôi: **`s01f21, s01f57, s02f21, s02f40, s03f02, s03f25, s04f11, s04f37, s05f01, s05f16`**
- Môi trường thực hiện: Hệ điều hành Windows 11, trình duyệt Google Chrome, phiên bản CVAT v2.83.0. MSSV bạn ngồi cạnh đổi kiểm: **2A202603089**.
- Đánh giá tổng quan:
  - Tổng số cuboid đã vẽ và lưu: **71 cuboid** trên 10 frame (đạt trung bình 7.1 cuboid/frame, vượt mức tối thiểu 53 cuboid theo yêu cầu).
  - Frame có ít cuboid nhất: `s03f25` (3 cuboid) do trong cảnh này xe di chuyển trên đoạn đường vắng, các xe khác ở cự ly rất xa (>50 m) hoặc bị khuất tầm nhìn.
  - Tỉ lệ cờ cảnh báo của `selfcheck`: Có **12/71 cuboid** bị gắn cờ cảnh báo (chiếm ~17%), chủ yếu là cảnh báo kích thước xe lớn hơn khoảng tham chiếu (do xe SUV cỡ lớn, có gương chiếu hậu xòe hai bên) hoặc xe ở xa.
  - Phần tự tin nhất: Xác định đúng class, căn chỉnh góc xoay yaw theo hướng di chuyển của đầu xe và đặt đáy cuboid tiếp xúc sát mặt đường ($z_{\text{bottom}} \approx -1.7\text{ m}$).
  - Phần còn yếu nhất: Ước lượng chiều dài và mép sau của các phương tiện ở cự ly xa (>30 m) khi điểm LiDAR bị thưa thớt và bị các xe khác che khuất một phần.
- Cách sử dụng CVAT 3D:
  - Đổi góc nhìn: Kéo chuột trái để xoay không gian 3D (xoay thẳng từ trên xuống để nhìn toàn cảnh Top-down view bao quát 360°), kéo chuột phải để dời cảnh, lăn con lăn chuột để phóng to/thu nhỏ.
  - Vẽ và chỉnh cuboid: Chọn công cụ Cuboid ở thanh bên trái $\rightarrow$ chọn nhãn $\rightarrow$ chọn Shape $\rightarrow$ nháy đúp chuột để đặt hộp; dùng khung Top để chỉnh dài, rộng và xoay yaw; dùng Side/Front để nâng hạ cho đáy chạm đường và chỉnh chiều cao.
  - Bật hướng và xem preview: Vào menu Appearance $\rightarrow$ bật Cuboid orientation (vạch đỏ ngắn chỉ về đầu xe); chạy lệnh `python scripts/lab.py preview` để kiểm tra trực quan các cuboid được chiếu lên 7 ảnh camera.

## 2. Đọc frame (Bước 2)
Hai frame đọc kỹ: `s01f21` và `s01f57`.
Ảnh: `t1-s01f21-3d.png`, `t1-s01f21-top.png`, `t1-s01f57-3d.png`, `t1-s01f57-top.png`.

| Frame | Phía trước xe ở cạnh nào của ảnh nhìn từ trên xuống; +x, +y chỉ về đâu | Mặt đường (phẳng/dốc; thấp hơn LiDAR ~ m) | Vật xa nhất đọc được (~ m) | Loại vật thấy | Vật phía sau / hai bên | Cụm chưa đọc được |
|---|---|---|---|---|---|---|
| `s01f21` | Cạnh dưới (hướng -y); +x chỉ sang bên trái, +y chỉ ra phía sau | Mặt đường phẳng; thấp hơn LiDAR khoảng 1.7 m ($z \approx -1.7\text{ m}$) | Xe con cùng làn phía sau cách ~36.3 m ($y = +36.3\text{ m}$) | Car, Bus | Xe buýt lớn bên trái đang vượt; xe con phía sau cách ~7.2 m | Cụm điểm phản xạ tán xạ ven vỉa hè bên phải và rào chắn ở xa |
| `s01f57` | Cạnh dưới (hướng -y); +x chỉ sang bên trái, +y chỉ ra phía sau | Mặt đường khá phẳng, dốc nhẹ về trước; thấp hơn LiDAR khoảng 1.75 m | Xe con đi trước cùng làn cách ~15.9 m ($y = -15.7\text{ m}$) | Car | Xe con ngay sau đuôi cách ~7.0 m; xe con song song bên trái cách ~3.4 m | Cụm điểm thưa dạng dải dài dọc mép đường phía sau bên phải |

Ba điều CHỈ CAMERA cho biết:
1. Màu sắc xe (trắng, đen, đỏ, bạc) và nhận diện thương hiệu, kiểu dáng xe chi tiết.
2. Thông tin biển báo giao thông, biển số xe và các vạch kẻ phân làn trên mặt đường.
3. Trạng thái đèn tín hiệu giao thông, đèn pha và đèn hậu của các phương tiện.

Ba điều CHỈ LIDAR cho biết:
1. Khoảng cách vật lý 3D chính xác tuyệt đối (tính bằng mét) từ cảm biến đến từng bề mặt của vật thể mà không bị biến dạng phối cảnh.
2. Kích thước hình học thực tế (dài, rộng, cao) theo tỉ lệ 1:1 trong không gian thực.
3. Tầm quan sát toàn cảnh 360° đồng nhất, không bị ảnh hưởng bởi góc mù quang học giữa các camera và không bị phụ thuộc vào điều kiện ánh sáng môi trường.

Tám frame lướt:

| Frame | Phía trước ở cạnh nào của ảnh nhìn từ trên xuống | Điều lạ (nếu có) |
|---|---|---|
| `s02f21` | Cạnh dưới (hướng -y) | Có người đi bộ và người đi xe máy di chuyển ở làn bên trái |
| `s02f40` | Cạnh dưới (hướng -y) | Xe con rẽ ngang ở ngã rẽ bên phải (yaw lệch góc vuông so với trục đường) |
| `s03f02` | Cạnh dưới (hướng -y) | Mật độ xe rất đông đúc ở cả trước và sau (10 xe trong 30 m), có xe tải nhỏ đi trước |
| `s03f25` | Cạnh dưới (hướng -y) | Đường rất thoáng, có xe buýt ở cự ly rất xa (>60 m) phía trước |
| `s04f11` | Cạnh dưới (hướng -y) | Xe buýt lớn ở làn ngược chiều bên trái (yaw ngược chiều, phản xạ điểm mạnh) |
| `s04f37` | Cạnh dưới (hướng -y) | Xe tải lớn và người đi xe máy di chuyển song song bên phải làn đường |
| `s05f01` | Cạnh dưới (hướng -y) | Có xe tải bám sau và nhiều xe con đỗ dọc mép đường bên phải |
| `s05f16` | Cạnh dưới (hướng -y) | Xe tải thùng dài ở phía trước bên phải cách ~30.9 m, mặt đường dốc nhẹ |

## 3. Ba cuboid có hướng dẫn (Bước 3)
Ảnh: `t2-a-car.png`, `t2-b-bus-truck.png`, `t2-c-ped-cyclist.png`.

| Vật | Frame, idx | Class | Mũi tên yaw đúng hướng? | Đáy chạm đường? | Đã kiểm Top/Side/Front + ảnh? | Kích thước trong bảng 4.3? |
|---|---|---|---|---|---|---|
| (a) xe con gần | `s01f21`, idx 7 | Car | Có (yaw = 271.5°) | Có (z_bottom = -1.96 m) | Có | Có ($4.48 \times 1.84 \times 1.78\text{ m}$, chuẩn SUV/sedan) |
| (b) xe buýt hoặc xe tải | `s01f21`, idx 1 | Bus | Có (yaw = 269.6°) | Có (z_bottom = -1.96 m) | Có | Có ($12.08 \times 2.60 \times 3.81\text{ m}$, chuẩn xe buýt thường) |
| (c) người đi bộ hoặc xe hai bánh | `s02f21`, idx 2 | Cyclist | Có (yaw = 90.7°) | Có (z_bottom = -1.62 m) | Có | Có ($1.88 \times 0.91 \times 1.62\text{ m}$, chuẩn xe máy có người) |

- Vòng hiệu chuẩn:
  - Khi so sánh với kết quả của các bạn trong lớp, điểm lệch chủ yếu nằm ở việc xác định độ cao đáy hộp ở view Side/Front (dễ bị chìm sâu xuống mặt đường) và hướng yaw của xe ở các khúc cua.
  - Cả lớp đã thống nhất cách hiểu: Luôn căn vạch đỏ ngắn dọc thân xe chỉ về đầu xe theo ảnh camera; đáy hộp phải vừa chạm tới cụm điểm mặt đường; xe bán tải phải gán nhãn Truck; người điều khiển xe hai bánh gán nhãn Cyclist bao trùm cả xe và người.
- Vật bỏ qua vì không đủ cơ sở: Tại frame `s03f25`, một cụm điểm thưa ở cách xa khoảng 55 m phía trước bên trái chỉ có 3 điểm rời rạc, không đủ căn cứ nhận diện $\ge 15\%$ thể tích và camera không soi rõ $\rightarrow$ Bỏ qua và ghi nhận loại `LOW_VISIBILITY`.

## 4. Bốn hiện tượng chất lượng (Bước 4)
Ảnh: `t4-1-bong-che.png`, `t4-2-thua-o-xa.png`, `t4-3-phan-xa-thap.png`, `t4-4-diem-la.png`.

| Hiện tượng | Frame / tên ảnh / nguồn | Nguyên nhân (giả thuyết) | Thấy gì trên point cloud | Hệ quả khi gán (theo guideline) |
|---|---|---|---|---|
| 1. Bóng che | `s01f21` / `t4-1-bong-che.png` | Thân xe buýt lớn bên trái che chắn chùm tia LiDAR chiếu về phía sau | Xuất hiện vùng bóng tối hình quạt hoàn toàn không có điểm phía sau xe buýt, dù xung quanh vẫn dày điểm | Vật thể phía sau bị che một phần thì bật thuộc tính `occluded` và ước lượng cuboid theo logic; vùng trống là "chưa quan sát được", không phải đường trống |
| 2. Thưa ở xa | `s01f21` / `t4-2-thua-o-xa.png` | Góc mở giữa 40 tia LiDAR dãn rộng theo khoảng cách, mật độ điểm giảm tỉ lệ nghịch với bình phương cự ly | Xe con ở cự ly 36.3 m chỉ còn 104 điểm, các lát cắt quét thưa thớt, mất chi tiết cản va và bánh xe | Áp dụng quy tắc: nếu thấy $\ge 15\text{–}20\%$ thể tích thì vẽ và dựa vào bảng 4.3 để khóa kích thước; nếu quá thưa dưới 15% thì bỏ qua, ghi `LOW_VISIBILITY` |
| 3. Phản xạ thấp / ít điểm bất thường | `s01f57` / `t4-3-phan-xa-thap.png` | Kính lái gây khúc xạ/phản xạ gương tia LiDAR ra hướng khác; bề mặt sơn xe màu tối hấp thụ ánh sáng hồng ngoại | Phần kính chắn gió và mui xe tối màu bị rỗng điểm hoặc intensity thấp hơn hẳn so với bề mặt kim loại | Không co hẹp cuboid theo phần bị khuyết điểm mà phải căn theo khung gầm kim loại để bao trọn thể tích thực của xe |
| 4. Điểm lạ | `s02f35` / `t4-4-diem-la.png` (Task chung `D13-HC485087`) | Hiện tượng phản xạ đa đường (multipath) từ bề mặt kim loại/mặt đường phản chiếu hoặc tán xạ cảm biến | Xuất hiện cụm khoảng 46 điểm nằm sâu dưới lòng đường ở độ cao $z \approx -13.5\text{ m}$ (thấp hơn mặt đường khoảng 11 m) | Không gán nhãn cụm điểm này thành bất kỳ vật thể nào; tuyệt đối không vẽ cuboid vào điểm lạ dưới lòng đất |

## 5. Số từ cuboid của tôi (Bước 5)
Số liệu trích xuất từ `submission/D13-2A202603088-bangA.csv` và `submission/D13-2A202603088-tukiem.csv`:

| Cuboid (frame, idx, class) | z | h | z đáy = z − h/2 | dist_m | r = √(x²+y²+z²) | yaw_deg | Nhận xét (lơ lửng / chìm / chạm đường) |
|---|---|---|---|---|---|---|---|
| `s01f21`, idx 7, Car | -1.072 m | 1.781 m | -1.963 m | 3.11 m | 3.29 m | 271.5° | Chạm sát mặt đường phẳng ($gap = -0.21\text{ m}$) |
| `s01f57`, idx 1, Car | -1.059 m | 1.528 m | -1.823 m | 9.28 m | 9.34 m | 272.5° | Chạm đường chuẩn xác ($gap = -0.01\text{ m}$) |
| `s02f21`, idx 2, Cyclist | -0.811 m | 1.622 m | -1.622 m | 6.15 m | 6.20 m | 90.7° | Đáy tiếp xúc mặt đường ($gap = 0.14\text{ m}$) |

Phân tích số liệu:
- (a) Khoảng cách cuboid gần nhất và xa nhất:
  - **Gần nhất:** `s01f21`, idx 7 (Car): $\text{dist\_m} = 3.11\text{ m}$, khoảng cách không gian 3D $r = 3.29\text{ m}$.
  - **Xa nhất (trong phạm vi đánh giá $\le 50\text{ m}$):** `s01f21`, idx 4 (Car): $\text{dist\_m} = 36.33\text{ m}$, khoảng cách không gian 3D $r = 36.35\text{ m}$. *(Ngoài ra, tại `s03f25`, idx 3 có xe buýt ở cự ly $\text{dist\_m} = 60.72\text{ m}$, $r = 60.72\text{ m}$ thuộc nhóm ngoài ranh giới 50 m)*.
- (b) Kích thước $l, w, h$ so với Bảng 4.3:
  - Cyclist (`s02f21`, idx 2): $l = 1.88\text{ m}, w = 0.91\text{ m}, h = 1.62\text{ m} \rightarrow$ Nằm hoàn toàn trong khoảng chuẩn của xe máy có người ($1.8\text{–}2.3\text{ m} \times 0.70\text{–}0.95\text{ m} \times 1.4\text{–}1.8\text{ m}$).
  - Car (`s01f21`, idx 8): $l = 5.31\text{ m}, w = 2.33\text{ m}, h = 1.92\text{ m} \rightarrow$ Chiều rộng $w = 2.33\text{ m}$ lớn hơn khoảng chuẩn của sedan ($1.7\text{–}1.9\text{ m}$) và SUV ($1.8\text{–}2.1\text{ m}$) do chiếc xe này là dòng SUV lớn và hộp cuboid bao trọn cả hai gương chiếu hậu chìa ra hai bên.
- (c) Góc yaw:
  - Xe cùng chiều phía trước (`s01f57`, idx 1): $\text{yaw\_deg} = 272.5^\circ$ (xấp xỉ $270^\circ$, đúng hướng -y).
  - Xe ngược chiều (`s02f21`, idx 5): $\text{yaw\_deg} = 86.9^\circ$ (xấp xỉ $90^\circ$, đúng hướng +y).

Mật độ theo khoảng cách (4 cuboid ở các khoảng cách khác nhau):

| Cuboid (frame, idx, class) | dist_m | Số điểm (`points_in`) |
|---|---|---|
| `s01f21`, idx 7, Car | 3.1 m | 3238 điểm |
| `s01f57`, idx 1, Car | 9.3 m | 1484 điểm |
| `s01f21`, idx 3, Car | 19.7 m | 433 điểm |
| `s01f21`, idx 4, Car | 36.3 m | 104 điểm |

- Quy luật mật độ điểm:
  Khi khoảng cách tăng lên gấp đôi thì số lượng điểm phản xạ trong cuboid giảm đi khoảng 3 đến 4 lần (tỉ lệ nghịch xấp xỉ với bình phương khoảng cách $1/r^2$). Số liệu này hoàn toàn khớp với đường cong phân bố lý thuyết trong bài giảng (cự ly 0–10 m có từ 1500–3200 điểm, 10–20 m giảm còn ~400–1000 điểm, và ở cự ly >30 m chỉ còn khoảng ~100 điểm).
- Tự kiểm & sửa cuboid:
  - Sau khi kiểm tra lần đầu bằng lệnh `sync`:
    - Cuboid `s02f40`, idx 5 (Cyclist) bị cảnh báo `length < width` do nhầm hướng góc, đã xoay lại 90° để vạch đỏ dọc theo xe.
    - Cuboid `s04f11`, idx 7 (Car) bị cờ đáy lơ lửng 0.46 m, đã hạ thấp hộp trên khung Side/Front để đáy chạm mặt đường.
  - Các cảnh báo giữ nguyên: Các cờ `width ngoài khoảng` ở xe `s01f21` idx 8 và `s04f11` idx 6 được giữ nguyên vì xe có gương chiếu hậu lớn xòe ra hai bên và thuộc dòng xe thể thao cỡ lớn.
- Đổi kiểm chéo:
  Đã đối chiếu với bạn ngồi cạnh (MSSV: **2A202603089**), chọn ngẫu nhiên 2 số đo ở frame `s01f21`: xe Car idx 2 có $\text{dist\_m} = 7.19\text{ m}$ và xe Bus idx 1 có $\text{dist\_m} = 9.75\text{ m}$. Kết quả hoàn toàn trùng khớp giữa Bảng A và màn hình CVAT của cả hai máy.

## 6. Ý nghĩa của nhãn, cách dùng AI và điều còn chưa chắc
- (a) Ý nghĩa từ phía mô hình học (Model):
  1. Nhãn 3D bắt buộc phải đo bằng mét trong không gian thực thay vì pixel 2D vì hệ thống xe tự hành cần xác định chính xác khoảng cách vật lý thực tế để lập quỹ đạo di chuyển, tính toán khoảng cách phanh an toàn và tránh va chạm; pixel 2D không phản ánh được độ sâu không gian do phụ thuộc vào góc đặt và tiêu cự camera.
  2. Các vật thể thưa điểm cần có quy tắc gán nhãn riêng (như ngưỡng quan sát $\ge 15\text{–}20\%$ thể tích và sử dụng bảng kích thước tham chiếu) vì nếu mô hình học trên dữ liệu đoán mò không căn cứ sẽ sinh ra nhiễu (noise) và dự đoán ảo (false positives), trong khi nếu bỏ qua hoàn toàn các vật ở xa thì hệ thống xe tự hành sẽ mất khả năng cảnh báo sớm chướng ngại vật từ xa.
- (b) Cách dùng trợ lý AI:
  - Công cụ: Sử dụng trợ lý AI Antigravity.
  - Câu hỏi chính: Hướng dẫn cấu hình môi trường lab, giải thích hệ tọa độ VinFast (trục -y là phía trước), quy tắc phân loại class (xe 8 chỗ, xe máy) và hỗ trợ kiểm tra tính hợp lệ bài nộp.
  - Cách dùng kết quả: Đối chiếu với tài liệu guideline để xoay đúng góc yaw, hiểu rõ các cảnh báo trong phiếu tự kiểm và hoàn thiện báo cáo khoa học, trung thực.
- (c) Điều còn chưa chắc chắn:
  - Một số xe ở làn đường xa bị cây xanh hoặc dải phân cách che khuất phần gầm xe nên độ cao đáy hộp chủ yếu được ước lượng dựa trên mặt đường lân cận.
  - Chiếc xe buýt ở frame `s03f25` tại cự ly $>60\text{ m}$ chỉ có 86 điểm, kích thước hộp được dựng ước lượng dựa trên hình ảnh thu phóng từ camera tele chứ chùm tia LiDAR không bao trọn được các góc của xe.
