# Hướng dẫn project NX-8-alarm và dữ liệu NX-8

**Bản khảo sát mã gốc trước khi bổ sung API.** Để cài bản firmware mới và REST API
trên Orange Pi, dùng [hướng dẫn triển khai hiện tại](../pi-gateway/README.md).
Các mô tả getter PIN, điều kiện gửi lệnh và lỗi ở đây ghi lại baseline commit bên
dưới; một số đã được sửa trong bản tích hợp API mới.

Project này là firmware C chạy trên ESP8266 để kết nối tủ báo động NetworX NX-8 với HomeKit. ESP đọc bus bàn phím của tủ, chuyển trạng thái sang HomeKit và gửi một số lệnh tương tự thao tác bàn phím. Tủ NX-8 vẫn xử lý cảm biến, chế độ bảo vệ và báo động; ESP đóng vai trò thiết bị tích hợp.

Phạm vi mã nguồn được đối chiếu: commit `3d7aa2021191717e01990b141d0e5a2e8af93ade`, thông điệp phiên bản `0.6.7`. Các giới hạn dưới đây thuộc bản code này, không phải giới hạn phần cứng của toàn bộ dòng NX.

## 1. Mối liên hệ với NX-8

```text
Cảm biến cửa, PIR, các đầu vào zone
                 |
                 v
        Tủ báo động NX-8 <------> Bàn phím NX hiện có
                 |
        Bus POS / COM / DATA
                 |
    Mạch giao tiếp điện áp và driver 4512
                 |
       ESP8266 chạy NX-8-alarm
                 |
            Wi-Fi / HomeKit
                 |
      Apple Home / Eve / Home Assistant
```

Luồng đọc: tủ phát status → GPIO5 → driver bus → kiểm tra checksum → giải mã → HomeKit notification.

Luồng điều khiển: HomeKit ghi target → firmware kiểm tra trạng thái → đợi cửa sổ gửi → GPIO4 và mạch 4512 phát lệnh → đợi ACK → đọc status của tủ để cập nhật kết quả.

Đây là giao tiếp trực tiếp với bus bàn phím. Không có đường gọi REST tới NX-8, không dùng RTSP và không có pipeline camera/AI. `ota-api.c` là mã hỗ trợ cập nhật firmware, không phải API HTTP điều khiển tủ.

Trong `main.c:263` có nhắc tài liệu NX-584 để tham khảo ý nghĩa trạng thái partition. Không vì vậy mà có thể dùng nguyên framing RS-232 của NX-584 cho GPIO của project này; framing thực tế nằm trong `nx8bus.c`.

### Chức năng hiện có

| Chức năng | Code hiện tại |
| --- | --- |
| Xem disarmed, away, night và alarm triggered | Có |
| Gửi Away, Night/Sleep, Disarm | Có, với các điều kiện ở mục 5 |
| HomeKit Security System | 1 accessory |
| HomeKit Motion Sensor | 6 accessory, ID 2–7 |
| Trạng thái partition | Chỉ xử lý `command[2] == 0` |
| Trạng thái zone | Chỉ ánh xạ bit 0–5 của byte `command[2]` trong message `0x04` |
| Zone bypass/blocked | Nhận message `0x07`, chỉ kiểm tra và ghi debug |
| Điều khiển bypass từng zone | Chưa có |
| Lịch sử sự kiện, database, export CSV | Chưa có |
| REST/MQTT/WebSocket cho ứng dụng riêng | Chưa có |
| Wi-Fi provisioning và OTA | Dựa vào Life-Cycle-Manager bên ngoài repo |

Tên NX-8 không đồng nghĩa code này đã xuất đủ 8 zone. Zone 7 trở lên chưa được ánh xạ ra accessory. Sáu sensor đều dùng kiểu Motion Sensor, kể cả khi cảm biến thực tế là công tắc cửa.

Repo có báo cáo người dùng chạy với NX-6v2 trong [tài liệu tương thích](../NX6v2_Compatibility_and_Lessons_Learned.md). Đây là kinh nghiệm được ghi lại, không phải chứng nhận tương thích với mọi NX-6/NX-8/NX-8E và mọi firmware tủ.

## 2. Bản đồ mã nguồn

| File/thư mục | Vai trò |
| --- | --- |
| [main.c](../main.c) | Khởi động, parser, ánh xạ trạng thái, commands, HomeKit accessories, watchdog mạng |
| [nx8bus.c](../nx8bus.c), [nx8bus.h](../nx8bus.h) | Giao tiếp bus bằng GPIO, đọc/gửi symbol 9 bit, checksum |
| [ota-api.c](../ota-api.c), [ota-api.h](../ota-api.h) | Đọc metadata từ sysparam; chuyển sang boot slot OTA |
| [ping.c](../ping.c), [ping.h](../ping.h) | ICMP ping phục vụ kiểm tra kết nối |
| [Makefile](../Makefile) | Build ESP-OPEN-RTOS, flash layout, version và components |
| [esp-homekit](../esp-homekit/README.md) | HomeKit protocol, pairing, lưu khóa và pairing vào flash |
| [esp-wolfssl](../esp-wolfssl/README.md) | Thư viện mật mã phục vụ HomeKit |
| [esp-cjson](../esp-cjson/README.md) | Xử lý JSON trong thư viện, không phải database nghiệp vụ |
| [UDPlogger](../UDPlogger/README.md) | Gửi log ra máy thu trong mạng |
| [Sơ đồ điện](../NX8-to-ESP8266-schema.png) | Sơ đồ GPIO, transistor, 4512 và bộ chia áp |

## 3. Phần cứng và dây nối

Các giá trị mặc định trong `main.c:38–42`:

| Tham số | Giá trị | Ý nghĩa |
| --- | --- | --- |
| `RX_PIN` | GPIO5 | Đọc DATA qua bộ chia áp |
| `TX_PIN` | GPIO4 | Điều khiển NPN và 4512 để kéo DATA xuống thấp |
| `MY_ID` | `0xD8` | Địa chỉ thiết bị trên bus |
| `PIN_RTC_ADDR` | `0x600013A8` | Vùng RTC lưu dấu nhận diện và PIN |

GPIO là số GPIO của chip, cần đối chiếu nhãn chân trên board cụ thể.

### Mạch đọc

Theo sơ đồ repo: DATA → điện trở 33 kΩ → nút RX/GPIO5 → điện trở 10 kΩ → COM/GND. ESP và tủ cần chung COM theo sơ đồ.

Điện áp tại RX theo bộ chia lý tưởng:

```text
Vrx = Vdata × 10 / (33 + 10)
12.0 V → 2.791 V
13.5 V → 3.140 V
13.8 V → 3.209 V
14.4 V → 3.349 V
```

Cần đo mức DATA thực tế và đối chiếu giới hạn điện của board/chip, kể cả dung sai và xung nhiễu. Không nối DATA hoặc POS của tủ trực tiếp vào GPIO hay chân nguồn 3.3 V của ESP. Sơ đồ giao tiếp không thay thế mạch ổn áp cấp nguồn cho ESP.

### Mạch gửi

Sơ đồ dùng 4512 và NPN 2N3904: chân 16 của 4512 nối POS, chân 14 nối DATA; chân inhibit 15 được kéo lên POS qua 10 kΩ và điều khiển bằng NPN; GPIO4 đi vào base qua 180 kΩ. Các chân còn lại phải nối đúng sơ đồ.

- GPIO4 = 0: driver ở trạng thái trở kháng cao, nhả bus.
- GPIO4 = 1: driver kéo DATA xuống 0.

Nên xác minh đường đọc trước bằng mạch chia áp, chưa nối mạch TX vào DATA. Khi đó firmware có thể gọi hàm gửi nhưng không điều khiển được bus vật lý; repo không có tùy chọn phần mềm `receive_only` hoàn chỉnh. Muốn nhận status, vẫn phải hoàn tất HomeKit pairing vì `alarm_init()` chỉ mở bus sau khi đã pair.

## 4. Giao thức và data trên bus

### Định dạng truyền

Nguồn: `nx8bus.c:21–34`, `38–63`, `83–112`.

| Thuộc tính | Giá trị trong driver |
| --- | --- |
| Tốc độ | 4.000 baud |
| Thời gian 1 bit | 250 µs |
| 1 symbol | 1 start bit + 9 data bit + 1 stop bit |
| Thứ tự bit | LSB trước |
| Bit thứ 9 | Đánh dấu symbol địa chỉ bắt đầu message |
| Checksum | Fletcher-16, dù tên hàm là `nx8bus_CRC` |
| Thứ tự checksum | Byte thấp trước, byte cao sau |
| Bộ đệm RX | 128 phần tử `uint16_t`, dung lượng hàng đợi hữu dụng 127 symbol |
| Gửi/nhận | Half-duplex; tạm tắt ngắt RX trong lúc gửi |

`108` trong log là số hex `0x108`, tức địa chỉ `0x08` có bit đánh dấu `0x100`. Nó không phải hai byte `01 08`. Checksum chỉ tính trên các byte thấp, bỏ bit đánh dấu này.

```text
c0 = 0, c1 = 0
với từng byte b:
    c0 += b
    c1 += c0
checksum = ((c1 mod 255) << 8) | (c0 mod 255)
```

Một symbol chiếm khoảng 2,75 ms trên dây. Ví dụ Away gồm 5 byte nội dung + 2 checksum, khoảng 19,25 ms chưa tính các khoảng nghỉ. Đây là thời gian tính từ framing, không phải latency đầu cuối đã đo. Thời gian chờ tủ, Wi-Fi và HomeKit còn cộng thêm.

### Các message mà ứng dụng xử lý

Nguồn: `main.c:339–425`. Độ dài dưới đây tính theo symbol nhận, có địa chỉ và checksum.

| Địa chỉ đầu | Loại | Tổng symbol | Xử lý |
| --- | --- | --- | --- |
| `0x100`…`0x107` | `0x04` | 10 | Trigger bitmap; đưa 6 bit đầu lên HomeKit |
| `0x100`…`0x107` | `0x07` | 10 | Blocked zones; chỉ log |
| `0x100`…`0x107` | `0x18` | 12 | Status partition; chỉ giải mã partition 0 |
| `0x100`…`0x107` | `0x01` | 14 | Kiểm tra checksum, chỉ log |
| `0x100`…`0x107` | `0x00`, `02`, `03`, `05`, `06`, `08`…`0C` | 10 | Kiểm tra checksum, chỉ log |
| `MY_ID + 0x100` | `0x10` | 4 | Poll keepalive; ESP phản hồi `08 44 00` + checksum |
| `MY_ID + 0x100` | `0x40` | 4 | ACK lệnh; nhả semaphore `acked` |
| `MY_ID + 0x100` | `0x70` | 10 | PIN-return theo chú thích code; ESP trả ACK, không xuất nội dung thành entity |

Không suy diễn các byte chưa được parser giải mã thành trường nghiệp vụ đã hỗ trợ.

### Giải mã message 0x18

Chỉ số dưới đây là vị trí trong `command[]` sau khi bỏ bit đánh dấu địa chỉ.

| Vị trí | Mask | Ý nghĩa theo parser |
| --- | --- | --- |
| `command[2]` | Giá trị bằng 0 | Partition được xử lý |
| `command[3]` | `0x40` | Armed |
| `command[4]` | `0x01` | Alarm active |
| `command[5]` | `0x04` | Stay; project ánh xạ thành HomeKit Night = 2 |
| `command[8]` | `0x90` | Khi cả hai bit bằng 0 thì coi trạng thái stable |
| `command[8]` | `0x04` | Gán `r2arm`; dùng làm điều kiện gửi lệnh |

Nguồn: `main.c:263–299`. Biến nội bộ `currentstate` giữ chế độ 1/2/3; characteristic `current` được đổi thành 4 nếu alarm active.

| Giá trị HomeKit | Cách project dùng |
| --- | --- |
| 0 — Stay Arm | Thư viện cho phép giá trị này, nhưng `target_task()` không có lệnh tương ứng |
| 1 — Away Arm | Gửi `away` |
| 2 — Night Arm | Gửi `asleep`; đọc bit stay của tủ thành giá trị này |
| 3 — Disarmed | Gửi `off` khi trạng thái nội bộ đang armed |
| 4 — Alarm Triggered | Chỉ là current state, không phải target |

### Giải mã message 0x04 và retention

`command[2] & 0x01` → Sensor1, `0x02` → Sensor2, `0x04` → Sensor3, `0x08` → Sensor4, `0x10` → Sensor5, `0x20` → Sensor6.

Khi bit bật, HomeKit MotionDetected thành `true` và timer được reset. Khi bit tắt, code không đổi ngay thành `false`; timer hết hạn mới xóa. Thời gian mặc định 60 giây, có thể đặt 1–240 giây qua custom characteristic. Nếu tủ tiếp tục phát bit bật, timer tiếp tục được kéo dài.

Vì vậy `MotionDetected=true` có nghĩa trigger đã xuất hiện gần đây theo retention, không đảm bảo đầu vào vật lý vẫn đang kích hoạt tại đúng thời điểm đọc. Retention không điều chỉnh thời gian còi của NX-8. Nguồn: `main.c:177–207`, `302–326`.

### Ví dụ data

Log đã có trong báo cáo NX-6v2 của repo:

```text
1c0 10 d0 91          Tủ poll địa chỉ c0
108 44 00 4c a0       Phản hồi keepalive
1c8 10 d8 a1          Tủ poll địa chỉ c8
108 44 00 4c a0       Phản hồi keepalive
```

Đây là log trong tài liệu của người đóng góp, không phải capture từ tủ của bạn.

Các frame sau được dựng từ code và tính checksum, chưa được phát hoặc capture trên phần cứng:

```text
1d8 10 e8 c1                         Poll địa chỉ mặc định d8
108 d1 d8 02 01 b5 fe                Lệnh Away
108 d1 d8 00 01 b3 fa                Lệnh Night/Sleep
1d8 40 19 f1                        ACK gửi đến địa chỉ d8
100 04 05 00 00 00 00 00 09 3a      Bitmap sensor 1 và 3 bật
100 18 00 40 00 00 00 00 04 00 5c a2  Fixture Away, stable, r2arm bật
```

Các byte chưa sử dụng trong fixture được đặt 0 để minh họa parser, không khẳng định đó là status hoàn chỉnh một tủ thật sẽ phát. Bộ dữ liệu đọc bằng máy ở [nx8_data_examples.json](nx8_data_examples.json) phân biệt rõ `repository_capture` và `synthetic`.

## 5. Lệnh điều khiển và PIN

Nguồn: `main.c:44–50`, `77–83`, `146–162`, `236–259`.

| Lệnh | Byte nội dung trước checksum |
| --- | --- |
| Away | `08 D1 MY_ID 02 01` |
| Night/Sleep | `08 D1 MY_ID 00 01` |
| Off | `08 D0 MY_ID 00 01 PIN_A PIN_B 00` |
| Prog | `08 D0 MY_ID 01 01 PIN_A PIN_B 00`; mảng có sẵn nhưng không dùng trong luồng bình thường |

Firmware này hỗ trợ PIN tủ gồm 4 chữ số, đóng gói theo công thức `PIN_A = digit1 + digit2 × 16`, `PIN_B = digit3 + digit4 × 16`. Đây là biểu diễn số, không phải mã hóa bảo mật.

Code phân biệt hai mã: mã HomeKit dùng để pair thiết bị, và PIN NX-8 dùng trong lệnh tủ. Mã pair nằm trong `config.password` ở `main.c:562–566`; bản repo dùng một mã mẫu cố định, nên đặt mã riêng khi chuẩn bị triển khai.

Các điều kiện thực tế trước khi gửi:

1. HomeKit đã pair và bus tasks đã chạy.
2. PIN_A/PIN_B không đồng thời bằng 0. Dù Away/Night không chứa PIN, `target_task()` vẫn chờ bước này.
3. Setter chỉ nhận chuyển Disarmed → Armed hoặc Armed → Disarmed; chuyển Away ↔ Night trực tiếp bị bỏ qua.
4. `r2arm` phải khác 0 cho mọi lệnh trong nhánh gửi, kể cả Off.
5. Parser phải cấp `send_ok` sau xử lý một status có địa chỉ byte thấp bằng 0.
6. Code chờ ACK tối đa 150 ms. ACK chỉ xác nhận ở tầng giao tiếp; phải đối chiếu status tiếp theo và trạng thái tủ để kết luận arm/disarm thành công.

`off` không nên được hiểu là lệnh disarm bất biến: chú thích code cảnh báo gửi thao tác PIN khi tủ đang off có thể làm tủ arm. Code dùng `currentstate<3` để chặn trường hợp đó; độ mới của trạng thái bus vì vậy rất quan trọng.

Custom PIN gồm `PIN1-digit`…`PIN4-digit`. Getter `pin_get()` thực hiện đóng gói cả bốn chữ số và ghi RTC khi có chữ số >0, sau đó đổi các giá trị hiển thị thành -1. Cơ chế này không có nút Apply hay transaction nguyên tử; client đọc xen giữa các lần ghi có thể chốt một bộ PIN chưa đầy đủ. Cần xác nhận đủ cả bốn trường và thử trên tủ trong điều kiện kiểm soát. PIN `0000` không dùng được với điều kiện hiện tại.

Không nên hardcode PIN thực vào repository. Debug hiện in toàn bộ byte lệnh gửi nên có thể làm lộ PIN dù giao diện trả -1.

## 6. Dữ liệu nằm ở đâu

| Loại dữ liệu | Vị trí | Tồn tại sau khởi động lại |
| --- | --- | --- |
| Symbol bus chưa xử lý | Ring buffer RAM trong `nx8bus.c` | Không |
| Message hiện tại | `command[20]` trong RAM | Không |
| `currentstate`, `new_target`, `acked_target`, `r2arm` | RAM | Không; phải đọc lại status |
| Motion state và timer | RAM | Không |
| Retention, debug | HomeKit characteristic trong RAM | Không; retention có TODO lưu sysparam |
| PIN NX-8 | RAM và RTC user memory | Có thể giữ qua software reset khi RTC còn hợp lệ; không coi là bền vững qua mất nguồn |
| HomeKit accessory ID, khóa và pairing | SPI flash, base `0x8C000` theo Makefile | Có, trừ khi xóa/ghi đè vùng flash |
| `ota_repo`, `ota_version`, `ota_string` | Đọc từ sysparam | Cấu hình flash bên hệ ESP/LCM, không phải file trong repo |
| Wi-Fi | Được cấu hình bởi LCM/SDK | Phụ thuộc trạng thái vùng cấu hình thiết bị |
| Log | Serial và/hoặc UDP đến máy thu | Firmware không lưu lịch sử; máy thu phải lưu nếu cần |

`ota_string` được project lấy token đầu tiên trước dấu `;` làm địa chỉ IPv4 để ping. Ví dụ cấu hình minh họa: `192.168.1.10`, cần thay bằng host thực tế ổn định và có phản hồi ICMP. Các token sau chưa được project sử dụng.

`ota_repo` được tách tại `/` để tạo manufacturer/model; `ota_version` dùng làm revision; MAC của ESP được đưa vào serial metadata. Không có bảng SQL hoặc collection dữ liệu báo động.

README có ghi lịch sử phiên bản từng lưu alarm state trong RTC; ở code hiện tại, đường ghi RTC đang lưu magic và PIN. Trạng thái alarm hiện tại được lấy lại từ bus. Nhánh đồng bộ `after_reset` có thêm điều kiện RTC magic hợp lệ (`main.c:282–285`), nên cần kiểm tra cả cold boot và software reset.

HomeKit dùng cơ chế pairing và kênh mã hóa của thư viện; điều đó không mã hóa log UDP hay biến PIN đóng gói thành dữ liệu bí mật được mã hóa tại chỗ lưu. Nguồn flash: `Makefile:15`, `esp-homekit/src/storage.c:17–29`, `75–119`, `222–250`.

## 7. Chuẩn bị build và cài đặt

### Build firmware

Đây là dự án ESP-OPEN-RTOS, không phải ESPHome YAML. Cần SDK ESP-OPEN-RTOS, toolchain `xtensa-lx106-elf`, GNU Make và các submodule đúng revision. README thư viện có nhắc ESP-IDF nhưng Makefile của ứng dụng này dùng `$(SDK_PATH)/common.mk`.

Lệnh mẫu trong Linux/WSL hoặc môi trường tương đương đã cài toolchain:

```bash
cd /path/to/NX-8-alarm
git submodule update --init --recursive
export SDK_PATH=/path/to/esp-open-rtos
make VERSION=0.6.7
```

Phải truyền `VERSION=x.y.z`; nếu thiếu, `main.c:34–36` dừng compile bằng `#error`. Artifact dự kiến theo build là `firmware/main.bin`. `build/` và `firmware/` đang nằm trong `.gitignore`, không có sẵn binary ứng dụng trong checkout được khảo sát.

`FLASH_SIZE ?= 8` là **8 megabit = 1 MiB** theo quy ước [ESP-OPEN-RTOS parameters.mk](https://raw.githubusercontent.com/SuperHouse/esp-open-rtos/master/parameters.mk), không phải 8 MB. Phải đối chiếu dung lượng chip, layout OTA và bootloader trước khi thay giá trị. Bài NX-6v2 dùng board 4 MB, không nên sao chép thông số flash đó cho mọi ESP.

Tài liệu NX-6v2 ghi môi trường Docker `homeaccessorykid/esp-open-rtos:20220108`. Đây là gợi ý môi trường cũ trong repo; chưa xác minh image hay build ứng dụng bằng image đó trong lần đọc này.

### Nạp và cấu hình thiết bị

Luồng cài lần đầu phụ thuộc [Life-Cycle-Manager](https://github.com/HomeACcessoryKid/life-cycle-manager): nạp bộ bootloader/config/`otaboot.bin` tương ứng, cấu hình Wi-Fi, repository `HomeACcessoryKid/NX-8-alarm` và binary `main.bin`. Hãy dùng hướng dẫn flash/layout của bản LCM được chọn; không nhầm `otaboot.bin` với firmware ứng dụng.

Trong cấu hình của ứng dụng này, đặt `ota_string` thành IPv4 của host ping. Giữ `ota_repo` đúng dạng `owner/repository`. Code hiện không xử lý an toàn chuỗi repo thiếu dấu `/`.

Sau khi thiết bị vào Wi-Fi, pair `NX-8-alarm` bằng HomeKit. Chỉ lúc đó `alarm_init()` mới khởi động logger, parser và driver bus (`main.c:470–493`).

### Đăng ký ESP với tủ

1. Kiểm tra đường đọc, mức điện áp, nguồn và COM.
2. Pair HomeKit, mở debug và đọc log bus để xác định địa chỉ các keypad đang dùng.
3. Chọn `MY_ID` chưa bị chiếm; mặc định `0xD8` không đảm bảo luôn trống. Nếu đổi, build và nạp lại firmware.
4. Hoàn thiện mạch TX theo sơ đồ.
5. Thực hiện quy trình enroll module/keypad của đúng model và revision tủ. Báo cáo NX-6v2 mô tả vào rồi thoát programming mode, đợi khoảng 12 giây. Không tự suy ra mã installer hoặc location của tủ khác.
6. Xác nhận poll đến địa chỉ ESP có phản hồi keepalive ổn định; kiểm tra đèn Service trước khi thử điều khiển.

Tủ có thể báo mất module khi ESP đã enroll nhưng không còn trả lời, ví dụ lúc reboot/OTA. Cần xác định nguyên nhân mất phản hồi trước khi thay cấu hình âm báo giám sát.

### Apple Home, Eve và Home Assistant

Apple Home sử dụng các trạng thái bảo mật và motion tiêu chuẩn. Repo mô tả Eve để truy cập các custom characteristic PIN, debug và OTA; việc chúng hiển thị thế nào cần kiểm tra trên phiên bản app đang dùng.

Với Home Assistant, dùng tích hợp [HomeKit Device](https://www.home-assistant.io/integrations/homekit_controller/) để nhận accessory. Tên đường dẫn cũ vẫn là `homekit_controller`. Thiết bị cần vào mạng và sẵn sàng pair. Nếu muốn đưa tiếp sang Apple Home qua Home Assistant, dùng HomeKit Bridge.

Báo cáo NX-6v2 nói các trường custom không được ánh xạ đầy đủ trong Home Assistant. Không giả định PIN/debug/OTA dùng được chỉ vì entity Alarm đã xuất hiện; phải kiểm tra khả năng client hoặc bổ sung cấu hình phù hợp.

## 8. Thu log và dữ liệu thực

Client có sẵn tại `UDPlogger/client/udplog-client.c`, dùng API socket POSIX. Ví dụ trên Linux/macOS có GCC:

```bash
cd UDPlogger/client
gcc udplog-client.c -o udplog-client
./udplog-client 192.168.1.50 | tee nx8-session.log
```

Thay IP bằng ESP thực tế. Trên Windows cần môi trường tương thích cho client này, không chỉ chạy file C trong PowerShell. `nx8-session.log` sẽ nằm ở thư mục chạy lệnh; file đó chưa tồn tại trong repo.

| Cổng UDP | Vai trò theo code |
| --- | --- |
| 45676 | ESP nhận đăng ký/duy trì phiên thu log |
| 45677 | Client nghe thông báo khởi động từ ESP |
| 45678 | Client nghe log LCM |

Log ứng dụng trả về địa chỉ/cổng nguồn của client đã đăng ký; không phải mọi dòng log đều được gửi đến 45677. Client tự duy trì đăng ký. Cần mở đúng chiều mạng/firewall; WSL NAT có thể làm việc nhận broadcast/UDP khác Linux chạy trực tiếp.

Debug raw bus mặc định tắt. `DEBUGP` chỉ in khi `debug.value.bool_value` bật. Việc chỉ mở client không tự bật raw bus debug. Serial console của ứng dụng đặt 115200 baud, khác tốc độ 4000 baud của bus NX.

Các log có thể gồm raw frame, checksum, lệnh SEND, trạng thái arm/stay/current/alarm, heap, block cấp phát lớn nhất, Wi-Fi channel và stack high-water mark. Mốc thời gian là thời gian chạy thiết bị, không phải timestamp UTC đáng tin cậy cho lịch sử sự kiện. Nên thêm thời gian nhận tại máy thu nếu xây hệ thống lưu trữ.

Raw SEND có thể chứa PIN. Cần che các trường PIN trước khi chia sẻ log và tắt debug sau chẩn đoán. UDPlogger hiện không xác thực client và không mã hóa payload.

## 9. Điểm cần xử lý trước triển khai ổn định

Đây là các phát hiện từ luồng code, chưa phải lỗi tái hiện trên tủ của bạn.

| Vấn đề | Vị trí | Hệ quả và hướng xử lý |
| --- | --- | --- |
| Chỉ 6 zone và partition 0 | `main.c:319–325`, `386` | Mở rộng theo cấu hình zone/partition thực tế; đối chiếu từng bit với đầu vào tủ |
| Stay target 0 có trong metadata nhưng không có lệnh | `main.c:84`, `245–252` | Giới hạn giá trị công bố hoặc bổ sung mapping có kiểm chứng |
| Task nhận chạy trước khi tạo motion timers | `main.c:484–492` | Status04 tới sớm có thể gọi `xTimerReset` với timer chưa tạo; tạo và kiểm tra timers trước khi mở RX |
| CRC sai ở partition 0 làm đọc thừa 2 symbol | `main.c:386–387` | `CRC_OK` đã lấy checksum, nhánh else lại đọc tiếp; có thể nuốt đầu frame sau; tách kiểm tra partition và CRC |
| Đọc frame thiếu không có timeout | `main.c:230–234`, `376–415` | Có thể chờ hoặc ghép nhầm dữ liệu frame sau; bổ sung timeout và resync ở symbol địa chỉ |
| Tràn buffer chỉ bật cờ | `nx8bus.c:53–59` | Chưa có báo cáo/recovery cho ứng dụng; cần đếm dropped symbols và tái đồng bộ |
| Chưa đánh dấu trạng thái cũ khi mất bus | `main.c:263–326`, `339–427` | Giao diện có thể giữ trạng thái cuối dù đã ngắt DATA; cần bus heartbeat, timestamp và fault/unavailable |
| `r2arm` chặn cả Off | `main.c:242–251` | Cần thử các tình huống armed, zone mở và alarm trước khi coi disarm đáng tin cậy |
| PIN được commit trong getter | `main.c:152–161` | Có thể chốt PIN chưa đủ; chuyển sang cập nhật có validate và Apply rõ ràng |
| Debug lộ PIN qua UDP | `main.c:226–228`, `UDPlogger/udplogger.c` | Che payload nhạy cảm và giới hạn kênh quản trị |
| Host ping thiếu/sai vẫn được dùng | `main.c:433`, `455–462` | Validate IPv4 và cấu hình; không biến host không hỗ trợ ICMP thành vòng restart |
| `ota_repo` thiếu `/` | `ota-api.c:24–28` | `strchr()` có thể trả NULL; kiểm tra trước khi tách chuỗi |
| Compaction pairing dùng địa chỉ flash làm chỉ số RAM | `esp-homekit/src/storage.c:160–173` | `data[PAIRINGS_ADDR + ...]` vượt vùng heap khi chạy nhánh dồn pairing; cần dùng offset và kiểm thử chu kỳ thêm/xóa pairing |

Watchdog mạng khởi đầu với ngân sách 120 lần lỗi, trừ 1 mỗi ping lỗi; đây không phải đồng hồ đếm chính xác 120 giây. Một lỗi có thể gồm timeout nhận 1 giây cộng delay 1 giây. Host khỏe được ping thưa dần tới 60 giây/lần. Nguồn: `main.c:429–452`, `ping.c`.

Driver lấy mẫu bằng vòng chờ trong ISR, mỗi symbol giữ xử lý cỡ 2,6 ms. Trước khi thêm tính năng cần đo checksum failures, số symbol bị bỏ, độ trễ ACK, heap tối thiểu, stack high-water và số lần reset dưới tải HomeKit/debug. Không có benchmark CPU hoặc latency thực tế được lưu trong repo.

### Kiểm tra nghiệm thu gợi ý

1. Đối chiếu trạng thái HomeKit với keypad khi disarmed, away, night và alarm.
2. Kích hoạt từng zone để xác nhận mapping; kiểm tra cả zone chưa được code xuất ra.
3. So sánh bit raw với motion sau retention; không dùng motion đã giữ 60 giây để kết luận cửa còn mở.
4. Thử cold boot và software reset trong từng chế độ; xác nhận ESP không tạo chuyển trạng thái ngoài ý muốn.
5. Thử mất Wi-Fi, host ping không trả lời, tháo DATA và khôi phục; đo thời gian phát hiện và đồng bộ lại.
6. Thử ACK timeout, frame lỗi và tải log; kiểm tra parser phục hồi và keypad vẫn ổn định.
7. Thử OTA và giám sát module trong điều kiện có người tại tủ; giữ keypad vật lý để xác minh kết quả.

## 10. Nếu cần tích hợp backend hoặc camera

Đường có sẵn là NX-8 → ESP/HomeKit → Home Assistant. Có thể dùng trạng thái đó để kích hoạt automation hoặc lưu lịch sử ở hệ thống ngoài. Muốn server riêng nhận JSON/MQTT trực tiếp cần bổ sung adapter; repo hiện chưa có endpoint/topic và không tự phát file JSON ví dụ đi kèm tài liệu này.

Payload nghiệp vụ nên phân biệt raw zone trigger với motion giữ theo timer, current với target, ACK với kết quả thực tế. Các trường hữu ích khi phát triển tiếp gồm `panel_id`, `partition`, `zone`, `event_type`, `value`, `received_at`, `last_seen_at`, `source` và `quality`. Đây là đề xuất thiết kế, không phải schema hiện có.

Với camera, dữ liệu báo động có thể dùng làm tín hiệu để đánh dấu clip hoặc yêu cầu hệ thống CV xác minh. Firmware này không chứa ảnh, video, model detection, embedding hay dataset huấn luyện.

## 11. Tài liệu tham khảo và mức xác minh

- [README của project](../README.md): mục tiêu, sơ lược phần cứng, lịch sử thay đổi.
- [NX-6v2 Compatibility and Lessons Learned](../NX6v2_Compatibility_and_Lessons_Learned.md): trải nghiệm lắp, enroll, Home Assistant, ảnh thực tế.
- [nx8bus.h](../nx8bus.h): mô tả mạch và API driver; chi tiết thực thi nằm trong `nx8bus.c` và `main.c`.
- [Sơ đồ điện](../NX8-to-ESP8266-schema.png), [ảnh lắp mẫu](../NX8-to-ESP8266.png), [ảnh terminal bus](../bus_wiring.jpg).
- [UDPlogger README](../UDPlogger/README.md): hướng dẫn client log.
- [Life-Cycle-Manager](https://github.com/HomeACcessoryKid/life-cycle-manager): provisioning và cài/OTA.
- [HomeKit Device của Home Assistant](https://www.home-assistant.io/integrations/homekit_controller/): tích hợp accessory hiện hành.
- [NX-8 Installation Manual, bản PDF lưu tại Inner West Security](https://www.innerwestsecurity.com/fileadmin/template/images/pdf/Networx_NX8_Installation_Manual_1_.pdf): tài liệu hãng được lưu trên website đơn vị khác; cần đối chiếu đúng model/revision/region trước khi áp dụng địa chỉ lập trình.

Đã đối chiếu code ứng dụng, driver, storage HomeKit, logger, sơ đồ và các ví dụ checksum. Chưa compile/flash firmware hoặc kết nối NX-8/ESP8266 trong lần khảo sát này; các frame synthetic và nhận xét độ tin cậy không thay thế phép thử phần cứng.
