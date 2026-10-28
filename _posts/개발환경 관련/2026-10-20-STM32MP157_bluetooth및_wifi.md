---
title: "한 칩, 두 얼굴 — AP6212 WiFi·블루투스 브링업 기록"
date: 2026-10-20
categories: [embedded, linux]
tags: [stm32mp157, wifi, bluetooth, ap6212, brcmfmac, connman, bluez, uart, rfcomm, board-bringup]
excerpt: "WiFi는 접속까지 됐는데 ping이 안 나갔고, 블루투스는 첫 명령부터 대답이 없었다. 같은 모듈인데 문제는 전혀 달랐다."
---

SC6_0 보드의 무선 모듈 AP6212는 WiFi와 블루투스가 한 칩에 들어 있다. 같은 모듈이니 같이 될 거라 생각했는데, 막힌 곳은 완전히 달랐다.

- **WiFi**: 칩은 바로 동작했다. 막힌 건 리눅스 네트워크 설정이었다
- **블루투스**: 칩이 첫 명령부터 대답하지 않았다. 소프트웨어 원인을 다 지운 뒤에야 하드웨어로 좁혀졌다

이 글은 두 가지를 차례로 정리한 기록이다.

<!--more-->

---

## 0. 구조 — 한 모듈, 두 개의 길

```text
STM32MP157                                    AP6212 (BCM43430A1)
  SDMMC3 (SDIO) ◀──── 데이터 ────▶  WiFi 부분
  PF3  ──── WL_REG_ON (WiFi 전원) ──▶
  UART7: PF7 TX / PF6 RX / PF8 RTS / PF9 CTS ◀─▶  블루투스 부분
  PF15 ──── BT_REG_ON (블루투스 전원) ──▶
                                     공통: +3.3V_W 전원, Y3 32.768kHz 클럭
```

| | WiFi | 블루투스 |
|---|---|---|
| 연결 | SDIO (SDMMC3) | UART (UART7) |
| 전원 핀 | PF3 (WL_REG_ON) | PF15 (BT_REG_ON) |
| 리눅스 드라이버 | `brcmfmac` | `hci_uart_bcm` |
| 칩 펌웨어 | `brcmfmac43430-sdio.bin` + 설정 `.txt` | `BCM43430A1.hcd` |
| 사용자 도구 | connman (`connmanctl`) | BlueZ (`bluetoothctl`, `rfcomm`) |

같은 칩이지만 MPU와 연결되는 길, 전원 핀, 드라이버가 모두 따로다. 그래서 한쪽이 된다고 다른 쪽이 되는 건 아니다. 다만 **전원(+3.3V_W)과 32kHz 클럭(Y3)은 공통**이라, WiFi가 동작한다는 건 공통 부분이 살아 있다는 근거가 된다. 이 점은 나중에 블루투스 원인을 좁힐 때 쓰였다.

---

# Part 1. WiFi

## 1. 칩은 처음부터 동작했다

부팅 로그에서 칩 인식과 펌웨어 적재가 다 보였다.

```text
brcmfmac: brcmf_fw_alloc_request: using brcm/brcmfmac43430-sdio for chip BCM43430/1
brcmfmac: brcmf_c_preinit_dcmds: Firmware: BCM43430/1 wl0: Sep 11 2018 09:22:09 version 7.45.98.65 (r707797 CY) FWID 01-b54727f
```

전원 제어는 벤더 DTS가 이미 정확히 하고 있었다. WiFi 전원 핀을 MMC 전원 순서 장치(`mmc-pwrseq`)에 넘겨서, SDIO 드라이버가 칩을 켜고 끄는 시점을 직접 관리한다.

```c
wifi_pwrseq: wifi-pwrseq {
	compatible = "mmc-pwrseq-simple";
	reset-gpios = <&gpiof 3 GPIO_ACTIVE_LOW>;   /* PF3 = WL_REG_ON */
};
```

회로도에서도 PF3이 모듈의 WL_REG_ON에 연결돼 있었다. 이 차이는 블루투스 편에서 다시 등장한다.

스캔도 바로 됐다.

```console
root@myir:~# iw dev wlan0 scan | grep -E "SSID|signal"
        signal: -67.00 dBm
        SSID: KNVISION
```

### 보드 전용 설정 파일

로그에 이런 줄도 있었다.

```text
Direct firmware load for brcm/brcmfmac43430-sdio.knv,stm32mp157a-sc6.txt failed with error -2
```

`brcmfmac`은 보드의 `compatible` 문자열로 **보드 전용 설정 파일** 이름을 만들어 먼저 찾고, 없으면 일반 파일로 넘어간다. DTS에서 `compatible`을 우리 보드 이름으로 바꿨기 때문에 그런 파일이 없었고, 일반 파일로 대신 동작하고 있다. 벤더 보드용 파일을 우리 이름으로 연결해두면 보정값까지 맞출 수 있다.

```bash
cd /lib/firmware/brcm
ln -sf "brcmfmac43430-sdio.myir,stm32mp157c-ya157c-v2.txt" "brcmfmac43430-sdio.knv,stm32mp157a-sc6.txt"
```

## 2. 접속 — connman의 서비스 ID

보드 이미지에는 connman이 설치돼 있지만 자동 시작이 꺼져 있었다.

```bash
systemctl enable --now connman
connmanctl
```

```text
connmanctl> enable wifi
connmanctl> scan wifi
connmanctl> services
connmanctl> agent on
connmanctl> connect KNVISION
Error /net/connman/service/KNVISION: Method "Connect" with signature "" on interface "net.connman.Service" doesn't exist
```

`connect` 뒤에는 SSID 이름이 아니라 **connman이 붙인 서비스 ID**를 써야 한다. `services` 목록의 오른쪽 열에 나온다.

```text
wifi_40fdf3ecc578_4b4e564953494f4e_managed_psk
     └ wlan0 MAC   └ SSID를 16진수로   └ 보안 방식
                     K N V I S I O N
```

`connect wifi_`까지 치고 Tab을 누르면 자동완성된다. 비밀번호를 넣자 IP를 받았다.

```text
4: wlan0: <BROADCAST,MULTICAST,DYNAMIC,UP,LOWER_UP> ...
    link/ether 40:fd:f3:ec:c5:78
    inet 192.168.45.223/24 brd 192.168.45.255 scope global wlan0
```

DHCP로 IP를 받았다는 건 공유기와 **양방향으로** 주고받았다는 뜻이다. 무선 송수신은 여기서 이미 확인된 셈이다.

## 3. 그런데 인터넷 ping이 안 나간다

```console
root@myir:~# ping -I wlan0 -c 4 8.8.8.8
4 packets transmitted, 0 received, 100% packet loss
```

그 사이 콘솔에는 이런 로그가 계속 찍혔다.

```text
IPv4: martian source 192.168.45.255 from 192.168.45.223, on dev eth0
ll header: 00000000: ff ff ff ff ff ff 40 fd f3 ec c5 78 08 00
```

`40 fd f3 ec c5 78`은 `wlan0`의 MAC이다. **보드가 WiFi로 보낸 브로드캐스트가 공유기를 돌아 유선(`eth0`)으로 다시 들어왔다**는 기록이다. 그제야 구성이 보였다.

```text
eth0   192.168.45.12    (유선)
wlan0  192.168.45.223   (WiFi)
```

같은 공유기에 유선과 무선이 동시에, **같은 대역**으로 붙어 있었다.

리눅스는 목적지마다 나갈 인터페이스를 하나로 정한다. 같은 대역에 출구가 둘이면 한쪽(여기선 `eth0`)이 기본 출구가 된다. `-I wlan0`으로 WiFi로 보내도 응답이 돌아올 때 커널이 "이 주소로 가는 길은 eth0인데 왜 wlan0으로 들어왔지?"라며 버리는 경우(역방향 경로 검사, rp_filter)가 생긴다. 가장 유력한 원인이 이것이었다.

확인은 간단했다. 유선을 잠깐 끄고 WiFi만 남겼다.

```bash
ip link set eth0 down ; sleep 2 ; ping -c 4 8.8.8.8 ; ip link set eth0 up
```

WiFi가 유일한 출구가 되자 정상 동작했다. **WiFi 자체는 문제가 없었다.** SSH가 유선으로 붙어 있으면 `eth0`을 끄는 순간 접속이 끊기니, 이 시험은 시리얼 콘솔에서 했다.

## 4. 둘 다 쓰려면

동시 사용이 불가능한 게 아니라, 같은 대역에 두는 게 문제다. 용도에 따라 세 가지 방법이 있다.

| 방법 | 구성 | 적합한 경우 |
|---|---|---|
| 대역 분리 | 유선은 현장 장비용 고정 IP, WiFi는 인터넷용 | 가장 깔끔. 산업 장비에서 흔함 |
| 주/예비 | 평소 유선, 끊기면 WiFi로 전환 | 서버 통신이 필요한 충전기 |
| 같은 대역 유지 | rp_filter 완화(`=2`) 또는 출발 주소별 정책 라우팅 | 특별한 이유가 있을 때만 |

주/예비 방식은 connman 설정으로 된다.

```ini
# /etc/connman/main.conf
[General]
PreferredTechnologies = ethernet,wifi
```

PLC 모뎀(`eth1`)은 IPv4 없이 IPv6 링크 로컬 주소만 쓰므로 이 문제와 상관없다.

---

# Part 2. 블루투스

WiFi가 되니 블루투스도 될 거라 생각했는데, 부팅 로그부터 달랐다.

```text
hci_uart_bcm serial0-0: No reset resource, using default baud rate
Bluetooth: hci0: command 0x0c03 tx timeout
Bluetooth: hci0: BCM: Reset failed (-110)
```

`0x0c03`은 `HCI_Reset`이다. 블루투스 드라이버가 칩에게 보내는 **첫 번째 명령**이고, 여기에 대답이 없었다.

## 5. 배선은 맞는가

블루투스 노드는 벤더 DTS 그대로였다.

```c
&uart7 {
	bluetooth {
		pinctrl-0 = <&btreg0>;          /* PF15 를 출력 HIGH 로 고정 */
		compatible = "brcm,bcm43438-bt";
		max-speed = <3000000>;
	};
};
```

회로도 넷리스트와 대조했다.

| 신호 | 칩 핀 | MPU 핀 | 회로상 처리 |
|---|---|---|---|
| BT_REG_ON | U5 34 (BT_RST) | PF15 | R8 10K로 +3.3V 풀업 |
| MPU TX → 칩 RX | U5 43 | PF7 | 직결 |
| 칩 TX → MPU RX | U5 42 | PF6 | 직결 |
| 칩 RTS → MPU CTS | U5 41 | PF9 | R7 10K로 **GND 풀다운** |
| MPU RTS → 칩 CTS | U5 44 | PF8 | R6 10K로 +3.3V 풀업 |
| 32kHz 클럭 (LPO) | U5 24 | — | 발진기 Y3(32.768kHz) → R30 |

전부 맞았다. 칩의 RTS 쪽이 GND로 풀다운돼 있어서, 칩이 신호를 안 내도 MPU는 "보내도 된다"로 본다. 보드의 핀 상태도 기대한 그대로였다.

```console
pin 86 (PF6): alternate 7 (UART7_RX)
pin 87 (PF7): alternate 7 (UART7_TX)
pin 88 (PF8): alternate 7 (UART7_RTS UART7_DE)
pin 89 (PF9): alternate 7 (UART7_CTS)
pin 95 (PF15): output - high - push pull - pull up
```

## 6. "No reset resource"의 뜻

로그 첫 줄의 의미는 드라이버 소스에 있었다.

```c
/* drivers/bluetooth/hci_bcm.c */
if (!bcmdev->shutdown) {
	dev_warn(&serdev->dev,
		 "No reset resource, using default baud rate\n");
	bcmdev->oper_speed = bcmdev->init_speed;
}
```

DTS에 `shutdown-gpios`(칩 전원 핀)가 없으면 나오는 경고다. 드라이버는 칩 전원을 껐다 켤 수 없고, `max-speed`도 무시하고 115200으로 동작한다.

WiFi와 비교하면 차이가 보인다. WiFi는 전원 핀을 드라이버에 넘겨서(`mmc-pwrseq`) 드라이버가 전원을 관리하는데, 블루투스는 핀 설정으로 HIGH만 고정해두고 드라이버에는 알려주지 않는다. 처음엔 이게 원인이라고 짐작했다.

## 7. 소프트웨어 원인 지우기

### 드라이버만 다시 붙이기

```bash
echo serial0-0 > /sys/bus/serial/drivers/hci_uart_bcm/unbind
echo serial0-0 > /sys/bus/serial/drivers/hci_uart_bcm/bind
```

```text
[10569.677] No reset resource, using default baud rate
[10571.991] Bluetooth: hci0: command 0x0c03 tx timeout
[10580.232] Bluetooth: hci0: BCM: Reset failed (-110)
```

칩 전원은 이미 **3시간 가까이(10,500초)** 켜져 있었다. 그래도 대답이 없었으니 타이밍 문제는 아니다.

### 칩 전원 껐다 켜기

드라이버를 떼면 PF15를 직접 제어할 수 있다.

```bash
echo serial0-0 > /sys/bus/serial/drivers/hci_uart_bcm/unbind
gpioset gpiochip5 15=0
sleep 1
echo serial0-0 > /sys/bus/serial/drivers/hci_uart_bcm/bind
```

결과는 같았다. 칩이 이상한 상태로 멈춰 있던 것도 아니다.

## 8. 바이트를 세다

리눅스 시리얼 드라이버는 블루투스 드라이버가 포트를 쓰고 있어도 포트마다 송수신 바이트 수를 기록한다.

```bash
grep 40018000 /proc/tty/driver/stm32-usart
```

`0x40018000`은 UART7의 주소다. 드라이버를 다시 붙이기 전과 후를 비교했다.

```text
전:  4: uart:stm32-usart mmio:0x40018000 irq:99 tx:4 rx:0 RTS|CTS|DTR|DSR|CD
후:  4: uart:stm32-usart mmio:0x40018000 irq:99 tx:8 rx:0 RTS|CTS|DTR|DSR|CD
```

| 항목 | 의미 |
|---|---|
| `tx` 4 → 8 | 정확히 4바이트 = `HCI_Reset` 한 번 |
| `rx` 0 → 0 | **칩에서 들어온 바이트가 하나도 없다** |
| 수신 오류 없음 | 깨진 데이터조차 오지 않았다 |
| `CTS` 표시 | 흐름제어로 막힌 것도 아니다 |

통신 속도가 틀렸다면 깨진 바이트라도 들어와 `rx`나 오류에 잡혔을 것이다. **명령은 정확히 나갔고, 대답은 0바이트였다.**

> 로그의 "timeout"은 결과일 뿐이다. 바이트 카운터는 "보냈는가"와 "받았는가"를 따로 알려준다.

## 9. 드라이버를 빼고 직접 말 걸기

마지막으로 소프트웨어 계층을 전부 걷어냈다. 블루투스 노드만 뺀 시험용 DTB를 따로 만들었다.

```c
/* stm32mp157a-sc6-v1-bttest.dts */
/dts-v1/;
#include "stm32mp157a-sc6-v1.dts"

/ {
	model = "KNV SC6_0 (MYC-YA157C-V2) BT-TEST";
};

&uart7 {
	/delete-node/ bluetooth;
};
```

블루투스 노드가 없으면 드라이버가 UART7을 잡지 않아서 `/dev/ttySTM4`가 일반 시리얼 포트로 나타난다. 원본 DTB는 그대로 두고 부팅 메뉴(extlinux)에 항목을 하나 추가해서 시험할 때만 골랐다.

```bash
gpioset gpiochip5 15=0 ; sleep 1 ; gpioset gpiochip5 15=1 ; sleep 1
exec 3<>/dev/ttySTM4
stty -F /dev/ttySTM4 115200 raw -echo -crtscts
printf '\001\003\014\000' >&3
timeout 2 od -An -tx1 <&3
exec 3>&-
```

`01 03 0c 00`은 `HCI_Reset` 명령이다. 정상이면 `04 0e 04 01 03 0c 00`(Command Complete, 성공)이 와야 한다.

```text
전:  tx:0 rx:0
후:  tx:4 rx:0
od 출력: 없음
```

흐름제어를 끄고, 드라이버 없이, 전원까지 직접 재시작했는데도 0바이트였다. 다음 날 전원을 완전히 끊었다 켠 뒤에도 똑같이 재현됐다.

| 원인 후보 | 판단 |
|---|---|
| 드라이버 설정, 리셋 순서 | 배제 |
| 흐름제어 | 배제 |
| DTS 설정 | 배제 |
| 통신 속도 | 배제 |
| 칩에 블루투스가 없는 모델 | 배제 — WiFi 쪽이 칩을 `BCM43430/1`(블루투스 포함)로 인식 |
| 공통 전원·클럭 | 가능성 낮음 — 같은 전원과 클럭을 쓰는 WiFi가 동작 |
| **블루투스 쪽 하드웨어** | **남은 원인** |

WiFi가 동작한다는 사실이 여기서 쓰였다. 공통 부분이 살아 있으니, 남는 건 블루투스 전용 경로(BT_REG_ON, UART 배선, 칩의 블루투스부)다.

## 10. 그리고, 됐다

그런데 같은 날 다시 확인했을 때 로그가 달라져 있었다.

```text
[14.445] hci_uart_bcm serial0-0: No reset resource, using default baud rate
[14.872] Bluetooth: hci0: BCM: chip id 94
[14.877] Bluetooth: hci0: BCM: features 0x2e
[14.904] Bluetooth: hci0: BCM43430A1
[14.906] Bluetooth: hci0: BCM43430A1 (001.002.009) build 0000
[18.840] Bluetooth: hci0: BCM43430A1 (001.002.009) build 0264
```

| 줄 | 의미 |
|---|---|
| `chip id 94` | 칩이 첫 명령에 대답했다 |
| `build 0000` | 칩 내장 ROM 상태 |
| `build 0264` | 펌웨어 패치(`.hcd`) 적용 완료 |

```text
4: uart:stm32-usart mmio:0x40018000 irq:99 tx:37239 rx:1974 ...
```

약 37KB는 펌웨어 패치를 내려보낸 양이다. `tx:4 rx:0`과는 완전히 다른 숫자다.

**DTS와 소프트웨어는 실패할 때와 같았다.** `No reset resource` 경고도 그대로다. 바뀐 건 하드웨어 쪽 무언가인데, 그게 무엇인지는 아직 확인하지 못했다.

## 11. 켜기 — RF-kill

칩은 살아났지만 장치가 꺼져 있었다.

```console
root@myir:~# hciconfig hci0 up
Can't init device hci0: Operation not possible due to RF-kill (132)

root@myir:~# rfkill list
0: hci0: Bluetooth
        Soft blocked: yes
        Hard blocked: no
1: phy0: Wireless LAN
        Soft blocked: no
        Hard blocked: no
```

블루투스만 소프트웨어로 막혀 있었다. WiFi 때문에 켠 connman이 블루투스 차단도 관리하는데, 그쪽 설정이 블루투스를 꺼두고 있었을 가능성이 크다. **WiFi 작업이 블루투스에 영향을 준 지점**이다.

```bash
rfkill unblock bluetooth
hciconfig hci0 up
connmanctl enable bluetooth      # 재부팅 후에도 유지
```

```text
hci0:   Type: Primary  Bus: UART
        BD Address: 40:FD:F3:EC:C5:73
        RX bytes:1974 ... events:184 errors:0
        TX bytes:37239 ... commands:184 errors:0
```

명령 184개에 응답 184개, 오류 0개. 블루투스 주소 `40:FD:F3:EC:C5:73`과 WiFi MAC `40:FD:F3:EC:C5:78`은 앞자리가 같다. 같은 모듈이라는 게 주소로도 보인다.

## 12. 검색과 페어링

주변 장치 검색은 바로 됐다.

```text
[CHG] Controller 40:FD:F3:EC:C5:73 Discovering: yes
[NEW] Device 6A:55:D4:7F:B0:A7 L6290 Series
[NEW] Device 18:2C:23:3C:F4:8D 18-2C-23-3C-F4-8D
...
```

이름 없이 주소만 보이는 건 저전력 블루투스(BLE) 기기들이다. 휴대폰이나 이어폰은 개인정보 보호 때문에 이름을 알리지 않는 경우가 많다.

반대로 휴대폰에서는 보드가 보이지 않았다.

```text
Discoverable: no
Pairable: no
DiscoverableTimeout: 0x000000b4
```

`0xb4`는 180초다. 검색 가능 상태를 켜도 3분 뒤에 자동으로 꺼진다.

```text
[bluetooth]# pairable on
[bluetooth]# discoverable-timeout 0
[bluetooth]# discoverable on
[CHG] Controller 40:FD:F3:EC:C5:73 Discoverable: yes
```

그 뒤로 휴대폰에서 보드가 보였고 페어링도 됐다.

## 13. 데이터 주고받기 — 시리얼(SPP)

휴대폰 쪽은 안드로이드의 Serial Bluetooth Terminal 앱을 썼다. 아이폰은 앱에서 클래식 블루투스 시리얼을 쓸 수 없다.

시리얼 서비스를 등록하는 `sdptool`은 블루투스 서비스가 호환 모드(`-C`)로 떠 있어야 동작한다. 서비스 파일을 직접 고치지 않고 덧붙이는 설정(drop-in)으로 바꿨다.

```ini
# /etc/systemd/system/bluetooth.service.d/compat.conf
[Service]
ExecStart=
ExecStart=/usr/libexec/bluetooth/bluetoothd -C
```

```bash
sdptool add SP
rfcomm listen /dev/rfcomm0 1
```

`rfcomm listen`이 실행 중이어야 휴대폰 연결을 받아준다. 앱에서는 **Bluetooth Classic** 목록에서 보드를 골라야 한다.

### 보낸 글자가 되돌아온다

연결 후 앱에서 글자를 보내자, 보낸 글자가 그대로 돌아왔다.

이상해 보였지만 좋은 신호였다. 휴대폰에서 보낸 데이터가 보드에 도착했고, 보드가 그걸 휴대폰으로 다시 보냈다는 뜻이다. **블루투스 양방향 통신은 이미 되고 있었다.**

되돌아온 원인은 리눅스 시리얼 포트의 기본 설정(echo)이다. 받은 글자를 되돌려 보내는 기능이 켜져 있다.

```bash
stty -F /dev/rfcomm0 raw -echo
cat /dev/rfcomm0                          # 휴대폰 → 보드
echo "hello from SC6_0" > /dev/rfcomm0    # 보드 → 휴대폰
```

---

## 14. 남은 것

| 항목 | 상태 |
|---|---|
| WiFi 접속, DHCP, 인터넷 (단독 사용 시) | 성공 |
| WiFi 보드 전용 설정 파일 연결 | 남음 |
| 유선·WiFi 동시 사용 방식 결정 | 남음 (용도에 따라) |
| 블루투스 칩 초기화, 검색, 페어링 | 성공 |
| 블루투스 시리얼 데이터 양방향 | 성공 (echo로 확인) |
| **블루투스가 실패하다 성공한 이유** | **미확인** |
| 블루투스 재현성 | 미확인 |
| 블루투스 `shutdown-gpios` DTS 수정 | 보류 |

가장 큰 숙제는 블루투스다. 이틀 동안 0바이트였던 칩이 같은 소프트웨어에서 갑자기 대답했다. 보드 교체, 납땜, 전원, 측정 프로브 접촉 중 무엇이 달라졌는지 짚어야 한다. 아무것도 바꾸지 않았다면 간헐적 불량일 수 있고, 충전기에서 이런 불량은 현장에서 "가끔 블루투스가 안 된다"로 나타난다.

다음은 전원 완전 차단과 부팅을 여러 번 반복하며 매번 한 줄로 판정하는 것이다.

```bash
dmesg | grep -E "BCM43430A1 \(|Reset failed"
```

---

## 정리

같은 모듈인데 막힌 층이 달랐다.

| | 막힌 곳 | 결정적 단서 |
|---|---|---|
| WiFi | 리눅스 네트워크 (같은 대역 이중 연결) | `martian source` 로그의 MAC 주소 |
| 블루투스 | 칩 쪽 하드웨어 (원인 미확인) | 바이트 카운터 `tx:4 rx:0` |

WiFi는 칩이 처음부터 정상이었고, IP까지 받은 뒤 경로 설정에서 막혔다. 블루투스는 반대로 칩과의 첫 대화부터 막혔고, 소프트웨어 계층을 하나씩 걷어내 하드웨어만 남겼다.

두 경우 모두 처음 보이는 증상("ping 100% loss", "tx timeout")은 어디가 막혔는지 말해주지 않았다. WiFi에서는 `martian source` 로그에 찍힌 MAC 주소가, 블루투스에서는 `tx`와 `rx` 숫자 두 개가 진짜 위치를 알려줬다.

그리고 블루투스는 원인을 못 찾은 채로 동작하기 시작했다는 것을 그대로 남겨둔다. 동작한다고 끝난 게 아니다. **왜 안 됐는지 모르면, 왜 되는지도 모르는 것**이다.
