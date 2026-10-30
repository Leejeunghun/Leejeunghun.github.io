---
title: "PMIC 없는 STM32MP157 보드에 ST 최신 BSP 올리기 — TF-A의 PANIC 두 줄에서 Linux 6.6 로그인까지 하루"
date: 2026-10-30
categories: [embedded, linux]
tags: [stm32mp157, tf-a, op-tee, u-boot, linux-6.6, yocto, scarthgap, device-tree, ddr, bring-up]
excerpt: "MYiR 모듈을 쓴 우리 보드(SC6_0)를 ST OpenSTLinux v6(Scarthgap)로 옮기는 작업을 시작했다. 아침에는 시리얼에 PANIC 두 줄만 나왔고, 저녁에는 커널 6.6에 로그인했다. 그 사이에 패치 전 소스, 사라진 dd, HSE 바이패스, GPT 6칸 제한, DDR 주소 매핑, reserved-memory 셀 정의를 차례로 만났다."
---

어제 chroot 실측으로 "650MHz Cortex-A7 두 개로 EVerest를 돌릴 수 있다"는 답을 얻었다. 그래서 오늘부터 본 작업인 **보드 이식**을 시작했다.

> **목표: 우리 보드(SC6_0)를 MYiR의 2021년 BSP(커널 5.4, Yocto Dunfell)에서 ST 최신 BSP(커널 6.6, Yocto Scarthgap)로 옮긴다.**

부팅은 네 단계를 차례로 거친다. 각 단계를 이정표로 잡았다.

```text
BootROM → TF-A(BL2) → OP-TEE → U-Boot → Linux
           M1: DDR     M2      M3        M4
```

결과부터 말하면 **M1~M4를 하루에 모두 통과했다.** 하지만 대부분의 시간은 에러 메시지를 하나씩 해석하는 데 썼다. 오늘 만난 문제와 그 원인을 정리한다.

<!--more-->

---

## 0. 작업 방식: 레이어 하나와 SSH 하나

### ST의 "CubeMX 머신"을 빌렸다

ST BSP는 자기 평가보드(DK1, DK2, EV1)용 설정만 들어 있다. 새 보드를 붙이는 방법으로 ST가 준비해 둔 것이 **CubeMX 머신**(`stm32mp15-mx.conf`)이다. 보드별 디바이스 트리 네 벌(TF-A, OP-TEE, U-Boot, 커널)을 정해진 폴더에 넣으면 나머지는 ST 레시피가 알아서 빌드한다.

```text
meta-sc6/
├── conf/machine/stm32mp15-sc6.conf     ← require stm32mp15-mx.conf + 보드 이름, DDR 512MB
└── mx/sc6/CA7/DeviceTree/sc6/
    ├── tf-a/        stm32mp157a-sc6.dts, DDR 설정 dtsi
    ├── optee-os/    stm32mp157a-sc6.dts, conf.mk
    ├── u-boot/      stm32mp157a-sc6.dts, -u-boot.dtsi, Makefile
    └── kernel/      stm32mp157a-sc6.dts, Makefile
```

모든 DTS는 **같은 칩(STM32MP157AAC)을 쓰는 ST DK1**을 기준으로 삼았다. 보드마다 다른 값(DDR, 핀, PHY)은 MYiR 5.4 BSP에서 가져왔다.

### 원본은 WSL, 빌드는 빌드 PC

레이어 원본은 WSL의 git 저장소에 두고, 32스레드 빌드 PC에는 SSH로 복사해서 bitbake를 돌렸다. sudo가 필요한 SD 카드 기록과 보드 조작은 사람이 직접 했다.

```text
WSL  ~/work/meta-sc6  (git)  --tar|ssh-->  빌드 PC ~/stm32mp/layers/meta-sc6
                                          MACHINE=stm32mp15-sc6 bitbake ...
```

### 시험은 여분 SD 카드로

전체 이미지를 만들기 전에는 **TF-A만 넣은 SD 카드**로 시험했다. TF-A는 다음 단계를 읽기 전에 DDR부터 초기화하므로, 나머지가 비어 있어도 DDR 설정이 맞는지는 판단할 수 있다.

---

## 1. M1 — TF-A와 DDR

### 첫 부팅: PANIC 두 줄

시리얼에 나온 것은 이게 전부였다.

```text
PANIC at PC : 0x2ffebd2d
Exception mode=0x00000016 at: 0x2ffebd2d
```

콘솔이 켜지기도 전에 죽어서 다른 정보가 없다. 빌드 PC에 남은 디버그용 ELF에서 주소를 찾아봤다.

```bash
addr2line -f -e tf-a-bl2-...-sdcard.elf 0x2ffebd2c
# stm32mp1_clk_init   drivers/st/clk/stm32mp1_clk.c:2377
```

클럭 초기화에서 `panic()`이 호출됐다. 원인은 **오전에 참고한 TF-A 소스가 ST 패치를 적용하기 전의 원본**이었다는 것이다.

- `bitbake -c unpack`으로 풀기만 한 소스에는 ST 패치가 없다.
- ST 패치를 적용하면 **클럭 DT 형식이 바뀐다.** 예를 들어 `st,pll_vco` 노드와 `DIV()` 매크로가 생긴다.
- 나는 옛 형식으로 썼고, 실제 빌드는 새 형식을 기대했다.

`tmp-glibc/work-shared/<machine>/tfa-source`(패치 적용판)를 기준으로 DTS를 다시 썼다.

### 그런데 같은 PANIC — 카드에 쓴 적이 없었다

다시 빌드해서 기록했는데 **PANIC 주소가 바이트 하나까지 같았다.** 빌드 PC의 `/dev`를 보니 이렇게 되어 있었다.

```text
-rw-r--r-- 1 root root 229860 /dev/sdc1    ← 블록 장치(b)가 아니라 일반 파일(-)
-rw-r--r-- 1 root root 229860 /dev/sdc2
brw-rw---- 1 root disk  8, 35 /dev/sdc3    ← 이건 진짜 장치
```

카드 리더가 카드를 인식하지 못한 순간에 `dd of=/dev/sdc1`을 실행하자, dd가 **`/dev` 아래에 그냥 파일을 만들었다.** 그 뒤로는 그 파일이 이름을 차지하고 있어서, 카드를 다시 꽂아도 장치가 생기지 않았다(`not a block device`).

이후 SD 기록용 스크립트를 만들면서 안전장치를 넣었다.

- 대상이 블록 장치인지, USB인지, 파티션 이름이 `fsbl1`/`fsbl2`인지 확인한 뒤에만 쓴다.
- 쓰고 나면 원본 파일과 바이트 단위로 비교한다.
- `check` 명령으로 카드 내용이 최신 빌드와 같은지 언제든 확인할 수 있다.

이 `check`는 나중에 한 번 더 쓸모가 있었다. "기록 완료"라고 생각했던 카드가 실제로는 `DIFFERENT`였던 것을 잡아냈다.

### 콘솔보다 먼저 죽을 때: early console

다음에는 PANIC 주소가 바뀌었는데, 이 함수는 실패 경로 10여 곳이 PANIC 처리 코드 하나를 같이 쓰고 있었다. ST TF-A에는 **클럭 초기화 전에 콘솔부터 켜는 옵션**(`STM32MP_EARLY_CONSOLE=1`)이 있어서 bbappend로 켰다.

여기서 함정이 하나 더 있었다. **TF-A의 make는 `-D` 옵션만 바뀌면 오브젝트를 다시 컴파일하지 않는다.** 그래서 옵션이 들어간 바이너리에 정작 그 기능이 없었다. 역어셈블로 확인한 뒤 `cleansstate`로 다시 빌드했다.

### 진짜 원인: HSE 바이패스

early console이 켜지자 드디어 문장이 나왔다.

```text
ERROR:   OSC 100 @ 50000808 timeout for enable=1 : 0x1800005
ERROR:   stm32mp1_hse_enable: failed
ERROR:   PLL0 start failed @ 0x50000080: 0x1
```

24MHz 외부 클럭(HSE)이 2초가 지나도 준비 상태가 되지 않았다. 나는 HSE를 **바이패스 모드**(외부 발진기가 신호를 넣어 주는 방식)로 설정해 두었다. 근거는 MYiR가 배포한 TF-A 바이너리에서 꺼낸 DTB에 `st,bypass`가 있었기 때문이다.

설정을 **크리스털 모드**(속성 없음)로 바꾸자 바로 통과했다. 크리스털 모드는 OSC_IN에 발진기가 물려 있어도 대개 동작하므로 안전한 쪽이다.

> 배운 점: **남이 만든 바이너리의 값도 우리 보드에서는 틀릴 수 있다.** MYiR U-Boot 소스 주석에는 "크리스털"이라고 적혀 있었다.

### 마지막 관문: GPT 앞의 여섯 칸

```text
ERROR:   Unable to find metadata partition "metadata1"
```

TF-A v2.10은 **DDR을 초기화하기 전에** 펌웨어 업데이트용 정보 파일(FWU metadata)부터 읽는다. 그래서 SD 카드에 `metadata1`/`metadata2`, `fip-a`/`fip-b` 파티션을 추가했다. 그런데도 같은 에러가 났다.

소스를 보니 **TF-A는 GPT 파티션 표의 앞 여섯 칸만 읽는다.**

```makefile
PLAT_PARTITION_MAX_ENTRIES := $(STM32_TF_A_COPIES) + $(STM32_EXTRA_PARTS)   # 2 + 4 = 6
```

기존 MYiR 파티션 일곱 개 뒤(8~11번)에 붙인 새 파티션은 보이지 않았다. 파티션의 위치와 데이터는 그대로 두고 **표의 번호만** 3~6번으로 옮겼다(`sfdisk -d`로 덤프하고 순서를 바꿔 다시 쓰기).

```text
INFO:    RAM: DDR3-DDR3L 32bits 533000Khz
INFO:    Memory size = 0x20000000 (512 MB)
WARNING: Firmware Image Package header check failed.   ← fip-a 가 비어 있음. 여기까지가 M1
```

**M1 통과.** MYiR의 DDR 설정으로 512MB가 올라왔다.

---

## 2. M2 — OP-TEE

ST v6에서는 OP-TEE가 클럭과 리셋을 관리하고, U-Boot와 리눅스는 **SCMI**라는 규약으로 OP-TEE에 요청한다. 그래서 OP-TEE용 DTS가 하나 더 필요하다.

PMIC가 없는 것이 문제였다. OP-TEE 소스를 보니 다음 두 드라이버가 전원 정보를 반드시 요구한다.

| 드라이버 | 요구하는 것 | 없으면 |
|---|---|---|
| CPU OPP | `cpu-supply` (CPU 전압) | probe 실패 |
| PWR 레귤레이터 | `vdd-supply` (3.3V) | probe 실패 |

DK1은 이 값을 PMIC에서 받는다. 우리 모듈은 discrete 전원이라 항상 켜져 있으므로 **`regulator-fixed`로 표현했다**(vddcore 1.2V, vdd 3.3V). 보드 등록과 PMIC 드라이버 끄기는 외부 DT 폴더의 `conf.mk`에서 했다.

```makefile
flavorlist-MP15         += stm32mp157a-sc6.dts
flavorlist-no_cryp-512M += stm32mp157a-sc6.dts   # 157A 는 CRYP 없음, DDR 512MB
CFG_STPMIC1 ?= n
```

빌드는 한 번에 통과했다. 보드 시험은 U-Boot와 함께 했다.

---

## 3. M3 — U-Boot

U-Boot DTS는 콘솔과 SD만 넣는 최소 구성으로 시작했다. 한 가지 원칙도 정해 두었다. **패널, 백라이트, LTDC는 넣지 않는다.** MYiR 원본 U-Boot는 PE11을 패널 리셋으로, PB8을 백라이트로 쓰는데, 우리 보드에서는 둘 다 **릴레이 구동 핀**이다.

### 1 GiB라고?

```text
DRAM:  1 GiB
E/TC:0 tzc_it_handler:80 TZC permission failure ... Violation @0xfdedefb0
```

U-Boot가 512MB 모듈을 1GB로 알고 메모리 꼭대기로 자신을 옮기려다, OP-TEE가 막아 둔 영역에 걸렸다. 소스를 따라가 보니 ST U-Boot는 DRAM 크기를 **DT가 아니라 DDR 컨트롤러의 주소 매핑 레지스터(ADDRMAP)에서 계산**한다.

```text
뱅크 3비트 + 열 10비트 + 행 15비트 = 28비트 × 4바이트(32bit 버스) = 1GB
```

MYiR DDR 설정의 `DDR_ADDRMAP6 = 0x0F070707`이 **행 주소 15비트**(4Gb 칩 기준)를 쓰고 있었다. 그런데 같은 파일 헤더에는 기준 칩이 2Gb ×16(행 14비트)이라고 적혀 있다. 쓰지 않는 행 비트 하나를 끄도록 `0x0F0F0707`로 고쳤다. 512MB 아래 주소 매핑은 그대로다. MYiR 원래 U-Boot는 DT에 적힌 크기를 썼기 때문에 이 문제가 드러나지 않았던 것으로 보인다.

### 이번에는 512 MiB인데 또 TZC 위반

```text
DRAM:  512 MiB
E/TC:0 ... Violation @0xdfee1000, non-secure privileged write
```

이번 주소는 OP-TEE 영역(0xDE000000~0xE0000000) **안**이었다. DK1의 SCMI 설정 파일이 `reserved-memory`에 OP-TEE 예약을 넣어 주기는 한다. 하지만 `#address-cells`/`#size-cells`/`ranges` 정의는 DK1의 다른 파일(`stm32mp15xx-dkx.dtsi`)에 있었고, 우리는 그 파일을 쓰지 않았다. 셀 정의가 없으면 `reg`를 잘못 해석해서 예약이 무시된다. 세 줄을 추가해서 해결했다.

여기서도 빌드 쪽 함정이 있었다. **bitbake는 U-Boot의 외부 DT 변경을 감지하지 못한다.** U-Boot와 FIP를 모두 `cleansstate`한 뒤 FIP 크기가 40바이트 늘어난 것까지 확인하고 나서야 반영됐다.

```text
DRAM:  512 MiB
Clocks: MPU 650 MHz / AXI 266.5 MHz / DDR 533 MHz
WDT:   Started watchdog with servicing every 1000ms (32s timeout)
MMC:   STM32 SD/MMC: 0
STM32MP> bdinfo
 reserved[2]    [0xde000000-0xdfffffff], 0x02000000 bytes flags: 4   ← OP-TEE 예약
```

**M2·M3 통과.**

---

## 4. M4 — Linux 6.6

### MYiR의 숨은 드라이버

이더넷 PHY(AR8035) 설정을 MYiR DTS에서 가져오려다 이상한 노드를 발견했다.

```dts
myir_driver {
    compatible = "myir,ar8035";
    reset-gpios = <&gpiog 0 GPIO_ACTIVE_HIGH>;
};
```

커널 소스를 찾아보니 `drivers/char/phy.c`에 MYiR가 직접 만든 드라이버가 있었다. 하는 일은 네 가지다.

1. PG0을 High로 올려 PHY 리셋 해제
2. SmartEEE 끄기 (링크가 끊겼다 붙는 문제 회피)
3. PHY의 CLK_25M 핀으로 125MHz 출력 (MAC의 RGMII 클럭으로 쓰임)
4. TX 클럭 지연 켜기

다행히 6.6의 표준 `at803x` 드라이버는 같은 일을 **DT 속성**으로 지원한다.

```dts
phy0: ethernet-phy@6 {
    reg = <6>;
    reset-gpios = <&gpiog 0 GPIO_ACTIVE_LOW>;
    qca,clk-out-frequency = <125000000>;
    qca,disable-smarteee;
};
/* phy-mode = "rgmii-id" : TX 지연(MYiR fixup) + RX 지연(AR8035 기본값) */
```

MYiR DTS의 `phy-mode = "rgmii"`를 그대로 쓰면 안 된다. 6.6의 at803x는 이 모드에서 지연을 둘 다 꺼 버리기 때문에 `rgmii-id`로 바꿨다.

### 로그인

ST 스크립트로 만든 5.15GB SD 이미지를 기록했다. 첫 부팅 때 rootfs를 4GB로 늘리는 데 130초가 걸렸고, 그 뒤 로그인 프롬프트가 나왔다.

```text
ST OpenSTLinux - Weston - (A Yocto Project Based Distro) 5.0.17-snapshot stm32mp15-sc6 ttySTM0
root@stm32mp15-sc6:~# uname -a
Linux stm32mp15-sc6 6.6.129 #1 SMP PREEMPT ... armv7l GNU/Linux
root@stm32mp15-sc6:~# cat /proc/device-tree/model
KNV SC6_0 (MYiR MYC-YA157C-V2, STM32MP157AAC)
```

| 항목 | 결과 | 비고 |
|---|---|---|
| 메모리 | 전체 454MB, 사용 가능 396MB | 예전 5.4: 425MB / 330MB. GPU 예약 64MB가 빠짐 |
| SD | 11개 파티션, rootfs 4GB | ST 정식 레이아웃 |
| eMMC | 7.28GB, 8bit DDR, boot0/1, rpmb | 기존 MYiR OS가 그대로 보임 |
| 워치독 | 130초 확장 동안 재부팅 없음 | 커널이 OP-TEE를 통해 IWDG1 갱신 |
| 이더넷 | `end0` 생성, **Generic PHY** | ⚠️ 남은 숙제 |

**M4 통과.**

---

## 5. 남은 것

- **이더넷**: AR8035에 at803x가 아니라 범용 드라이버(Generic PHY)가 붙었다. 그러면 DT의 PHY 설정이 하나도 적용되지 않는다. 커널이 PHY ID를 읽는 시점에 PHY가 아직 리셋 상태였을 가능성이 크다. PHY ID를 DT에 직접 적는 방법(`compatible = "ethernet-phy-id004d.d072"`)으로 고칠 예정이다.
- **M5**: SC6 고유 기능을 5.4 DTS에서 옮겨 온다. 릴레이 PWM, CP PWM/ADC, DIN/DOUT, UART3/5/7/8, I2C6(ATECC608B), QCA7005 SPI, WiFi/BT가 대상이다.
- **정리**: 브링업용 early console 끄기, MAC 주소, u-boot-env, SD 대신 eMMC로 굽기.

---

## 6. 오늘의 교훈

1. **"같은 PANIC"이면 바이너리부터 의심한다.** 빌드는 바뀌었는데 결과가 바이트 하나까지 같다면, 새 바이너리가 실행되지 않았을 가능성이 크다. 이번에는 dd가 카드 대신 `/dev`에 파일을 만들었다.
2. **패치 전 소스를 기준으로 쓰지 않는다.** `-c unpack`과 `-c patch`는 다르다. ST처럼 큰 패치를 적용하는 BSP라면 특히 그렇다.
3. **빌드 도구가 변경을 놓칠 수 있다.** TF-A의 make는 `-D` 옵션 변경을, bitbake는 U-Boot의 외부 DT 변경을 놓쳤다. 바뀌었다고 믿지 말고 결과물의 크기와 시각, 역어셈블로 확인한다.
4. **남의 설정값은 출처와 함께 의심한다.** MYiR 바이너리의 HSE 바이패스와 DDR ADDRMAP6은 그들의 부트로더 조합에서는 문제가 드러나지 않았을 뿐, 우리 조합에서는 틀린 값이었다.
5. **에러 메시지가 없으면 먼저 만든다.** early console 하나로 "PANIC 두 줄"이 "HSE timeout"이 됐다. 원인을 추측하는 것보다 로그를 확보하는 쪽이 빠르다.
