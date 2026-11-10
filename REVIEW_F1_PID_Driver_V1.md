# Design Review — F1_PID_Driver_V1

Ngày review: 2026-11-04 · Công cụ: KiCad 10 CLI (ERC, DRC + refill zone, netlist XML, render 3D), phân tích hình học bằng Python/shapely.
File gốc: `F1_PID_Driver_V1.kicad_sch` / `.kicad_pcb` (sửa lần cuối 04/08/2026).

> Kết luận nhanh: kiến trúc đúng hướng (H-bridge 4×N-MOSFET + 2×IR2184 + opto cách ly + STM32F103), nhưng có **6 lỗi mạch cần sửa trước khi đặt mạch lần tới**, PCB có **vòng gate/commutation quá dài** và **thiếu tụ ceramic sát cầu H**, và bộ Gerber trong `gerber_to_order/` (18/06) **cũ hơn file PCB** (04/08).

---

## 1. Chức năng mạch (đọc từ netlist)

| Khối | Linh kiện | Chức năng |
|---|---|---|
| Nguồn động lực | J1 → F1 20 A → `+VM`; D6 SS36 song song ngược; C11 470 µF/50 V + C3 10 µF | Cấp nguồn cầu H, bảo vệ ngược cực kiểu "crowbar" |
| Nguồn driver | U4 L7815 (`+VM`→`+15VP`), U5 AMS1117-5.0 (`+15VP`→`+5VP`) | 15 V cho IR2184, 5 V cho phía ra opto |
| Cầu H | Q1/Q3 high-side, Q2/Q4 low-side IRFS4321 (150 V, ~15 mΩ, Qg ≈ 71 nC) | Đảo chiều + PWM động cơ DC |
| Gate driver | U1, U2 IR2184 (half-bridge, 1 input, dead-time nội ~0.5 µs); bootstrap D16/D5 LL4148 + C1/C2 1 µF; R 4R7 + diode 1N4148W tắt nhanh; R 47 k kéo gate–source | Điều khiển mỗi nhánh bằng 1 PWM (PWMA/PWMB) |
| Cách ly | U3 HCPL-0630 (2 kênh), LED qua 330 Ω từ PB6/PB7; ngõ ra kéo lên `+5VP` | Cách ly MCU (GND) ↔ công suất (COM) |
| MCU | STM32F103C8, thạch anh Y2, SW3 reset, SW4 BOOT0, SWD J9 | PID vị trí/tốc độ |
| Ngõ vào lệnh | 74x14 (Schmitt đảo) cho EA/EB (encoder → TIM2 PA0/PA1), PULSE/DIR (PA6/PA7), ADIR (PA4); ADC biến trở J5 → PA3 | Encoder, Step/Dir, analog |
| Truyền thông | J4 UART1 (PA9/PA10), J8 I2C2 (PB10/PB11, pull-up 4k7 lên 3V3) | Lệnh từ master |
| Địa chỉ | JP1–JP4 hàn tắt xuống GND trên PB12–PB15 | ID node 0–15 |
| Logic | U6 AMS1117-3.3 từ `+5V` **lấy từ các cổng JST J4–J8** (không có nguồn 5 V on-board) | |

Mapping chân MCU ↔ ngoại vi là **hợp lý**: TIM2 encoder mode (PA0/PA1), TIM3_CH1 cho xung step, TIM4_CH1/CH2 (PB6/PB7) cho PWM, ADC1_IN3, USART1, I2C2, SWD đều đúng chân chức năng.

Lưu ý firmware: 74x14 **đảo** tín hiệu → DIR bị đảo, xung step đếm sườn ngược lại; IN của IR2184 cùng pha HO → PWMA mức cao = high-side A dẫn; do bootstrap, duty high-side không được giữ 100 % lâu (giới hạn ~95–98 %).

---

## 2. Lỗi schematic (xếp theo mức độ)

### 🔴 Nghiêm trọng — sửa bắt buộc

| # | Vấn đề | Bằng chứng | Hậu quả | Cách sửa |
|---|---|---|---|---|
| S1 | **D10 (zener 3.6 V) lắp sai**: anode ở `+3V3`, cathode ở `ADC_IN` | net `+3V3: D10.2(A)`, `ADC_IN: D10.1(K)` | Khi ADC < ~2.6 V zener **dẫn thuận** từ 3V3 vào ADC → ADC không đọc được dải thấp, biến trở vô dụng | Đổi zener: K→ADC_IN, A→GND; tốt hơn: dùng BAT54S kẹp về 3V3/GND + **cầu chia 10 k/20 k** (biến trở J5 cấp 5 V → 0–5 V phải chia xuống 0–3.3 V) + tụ 100 nF tại PA3 |
| S2 | **AMS1117-5.0 nhận 15 V** (U5 VI = `+15VP`) | AMS1117 Vin abs-max = 15 V; L7815 dung sai ±4 % → tới 15.6 V | Vượt abs-max, dễ chết LDO → mất 5VP → mất PWM | Dùng 78L05/LM317L/HT7550 (Vin ≥ 30 V), hoặc hạ gate drive xuống **12 V (L7812)** — 12 V vẫn đủ cho MOSFET và giảm tổn hao driver |
| S3 | **LDO không có tụ ra đủ lớn**: `+5VP` chỉ có C7+C10 = 2×100 nF; `+3V3` chỉ có C15–C17 = 3×100 nF | netlist | AMS1117 cần ≥ 10 µF (tantal/ESR phù hợp) ở ngõ ra để ổn định → dao động, nhiễu MCU/opto | Thêm 10–22 µF ở mỗi ngõ ra U5, U6 |
| S4 | **Dòng LED opto quá nhỏ**: R8/R9 = 330 Ω từ 3.3 V | I_F ≈ (3.3−1.5−0.2)/330 ≈ 4.1–4.8 mA; HCPL-0630 I_FT max 5 mA, khuyến nghị ≥ 6.3 mA | PWM chập chờn theo nhiệt độ/lô linh kiện | R8/R9 = 150 Ω (≈ 10 mA) |
| S5 | **nSD luôn kéo lên** (R7 → `+5VP`), MCU không thể tắt cầu H | `nSD: R7.1 U1.2 U2.2` | Khi MCU reset/treo: PB6/PB7 thả nổi → LED opto tắt → INA=INB=1 → **cả hai high-side bật** (phanh qua high-side), không có đường tắt khẩn | Đưa nSD qua kênh cách ly thứ 3, **mặc định tắt** (kéo xuống); dùng digital isolator có output mặc định thấp (ISO772xF / Si86xx "default low") thay HCPL-0630 |
| S6 | **VBAT (pin 1) và BOOT1/PB2 thả nổi** | ERC: `U7 Pin 1 [VBAT] not connected`, PB2 unconnected | VBAT nổi trái khuyến cáo ST (nối VDD + 100 nF); BOOT1 nổi → vào bootloader UART không ổn định | VBAT → 3V3; PB2 → 10 k xuống GND |

### 🟠 Quan trọng

| # | Vấn đề | Cách sửa |
|---|---|---|
| S7 | U8 ghi **"7414"**: 74LS14/SN7414 là TTL 5 V, không chạy ở 3.3 V. Nếu là 74HC14 @3.3 V thì ngõ vào kéo lên **+5 V** (R17–R19, R23, R24) vượt VCC → dòng qua diode kẹp, cấp ngược nguồn | Ghi rõ **74LVC14A** (ngõ vào chịu 5 V). PA0/PA1/PA4/PA6/PA7 **không** 5 V-tolerant nên bộ đệm này là cần thiết |
| S8 | Cổng thừa U8D: **ngõ vào pin 9 thả nổi** (ERC) | Nối pin 9 xuống GND (CMOS không để nổi) |
| S9 | R10 4k7 **0603** nối M+↔M− (LED chiều D7/D8): P = 24²/4700 ≈ 0.12 W (35 V → 0.26 W) > 0.1 W | 10 k 0805/1206 hoặc 2 điện trở nối tiếp |
| S10 | **Bảo vệ ngược cực bằng SS36 (3 A) song song** với cầu chì 20 A: diode chết trước khi cầu chì đứt | P-MOSFET/N-MOSFET + ideal-diode controller (LM74700…), hoặc tối thiểu diode ≥ dòng cắt cầu chì |
| S11 | **Không có TVS** trên `+VM`; chỉ 1 tụ ceramic 10 µF 1206 (sụt còn ~2–3 µF do DC-bias) | TVS SMBJ/SMCJ phù hợp VM (vd SMCJ33A cho 24 V), 2–4 × 1–2.2 µF/100 V X7R sát cầu H, tụ bulk ≥ 63 V low-ESR |
| S12 | **Không đo dòng, không đo VM, không đo nhiệt** → PID không có vòng dòng, không chống kẹt/quá dòng | Shunt low-side 2–5 mΩ + INA181/INA240 → ADC & comparator → nSD; chia áp VM; NTC gần MOSFET (xem mục 5 về kiến trúc cách ly) |
| S13 | **Dải VM thực tế chỉ ~18–35 V** (L7815 cần ≥ 17.5 V, max 35 V; C11 50 V) — không chạy được motor 12 V | Ghi rõ trên silkscreen; hoặc dùng buck rộng dải (vd 8–60 V) cho rail 12 V |
| S14 | Toàn bộ `+5V` logic lấy từ chân 2 của **cả 5 cổng JST** nối chung, không bảo vệ | Một cổng nguồn chính + Schottky/ideal-diode + polyfuse; hoặc DC-DC cách ly B0505S on-board |
| S15 | Cầu chì 20 A > định mức terminal MKDS 1,5/2-5,08 (~13.5 A) và > khả năng tải đồng 1 oz (mục 3) | Chọn cầu chì ≈ 1.25 × dòng liên tục thiết kế (vd 10 A) |

### 🟡 Nhỏ / tài liệu

- ERC: thiếu **PWR_FLAG** cho `COM`, `+VM`, `+5V`, `GND` (5 lỗi `power_pin_not_driven`); 13 chân MCU chưa có **No-Connect flag**; 4 cảnh báo `lib_symbol_mismatch` SolderJumper.
- J9 symbol "Screw_Terminal_01x03" nhưng footprint JST PH SMD → đặt tên đúng; nên thêm chân **3V3 (VTref) và NRST** cho ST-Link.
- Giá trị ghi mã "C104/C105/C106" → ghi rõ `100nF 50V X7R`, `1uF 25V X7R`, `10uF 50V X5R` (tụ bootstrap/VM cần điện áp ≥ VM). Y2 ghi "Crystal" → `8MHz CL=?pF` và tính lại C20/C21.
- C23 1 µF trên NRST hơi lớn (khuyến nghị ~100 nF); R20 10 k có thể bỏ (NRST có pull-up nội).
- Thiếu lọc VDDA (ferrite + 1 µF), thiếu tụ 4.7 µF chung cho các VDD.
- Không có series resistor/ESD trên UART/I2C/encoder đi ra dây ngoài.

---

## 3. Lỗi PCB

Board 45 × 65 mm, **2 lớp, đồng 1 oz (35 µm)**, nửa trên logic (GND), nửa dưới công suất (COM).

### Kết quả DRC (KiCad 10)

| Loại | SL | Đánh giá |
|---|---|---|
| `clearance` zone GND ↔ track/via **PULSE** = 0 mm, `hole_clearance` = 0 | 4 | **Zone fill cũ** — sau khi refill thì biến mất. ⇒ Luôn bấm **B (Refill All Zones)** trước khi xuất Gerber |
| `copper_edge_clearance` J9 pad = **0 mm**, mounting hole 0.25 mm, SW3/SW4/R22 ≈ 0.29 mm | 10 | Pad SWD chạm mép board → bị cắt/lộ đồng; dời vào ≥ 0.5 mm |
| `clearance` 0.2 < 0.2032 mm ở LQFP-48 | 44 | Giả: rule Default 8 mil > khe chân 0.5 mm pitch. Đặt clearance Default 0.15–0.2 mm, tạo netclass riêng |
| `courtyards_overlap` | 94 | Mật độ quá dày → khó hàn/AOI; nhiều cụm R/C, J4/J5, Q2, Y2, U7 |
| `starved_thermal` U6 pin1, U8 pin7 (GND) | 2 | Thêm spoke hoặc nối trực tiếp |
| `silk_overlap` / `silk_over_copper` / `text_height` 0.6 mm | 78 | Thẩm mỹ, chữ < 0.8 mm có thể không in được |
| `footprint_symbol_field_mismatch` (Datasheet) / `lib_footprint_mismatch` | 74 / 67 | Vô hại; chạy *Update PCB from Schematic* và *Update footprints from library* |
| Unconnected | 0 | ✔ |

### Lỗi layout không DRC nào bắt được

1. **Gate loop quá dài và bắt chéo**: U1 (x≈194) lái Q1/Q2 (x≈169) và U2 (x≈187) lái Q3/Q4 (x≈158) → đường HO/LO dài ~25–30 mm, U2 nằm gần cặp Q1/Q2 nhưng lại lái cặp xa. → Đặt mỗi IR2184 **ngay cạnh** nửa cầu của nó (≤ 10 mm), đổi chỗ U1/U2, đi HO song song VS, LO song song COM.
2. **Không có tụ ceramic trong vòng commutation**: C3 (10 µF) và C11 nằm ở góc phải dưới, cách drain Q1/Q3 – source Q2/Q4 ~20–30 mm → ringing ở nút M+/M−, gai áp lên MOSFET/IR2184 VS. → Mỗi nửa cầu 2 × 1–2.2 µF/100 V đặt sát giữa drain high-side và source low-side.
3. **Khả năng tải dòng**: đường `+VM` chính 2.54 mm/1 oz ≈ **4.7 A (ΔT 10 °C) / 6.4 A (ΔT 20 °C)** theo IPC-2221; zone rộng hơn và lớp đồng hở mask (tráng thiếc) giúp tăng, nhưng thực tế nên coi board là **~8–10 A liên tục**, không phải 20 A. → Đồng 2 oz, hoặc mở rộng pour, hoặc thanh đồng/busbar hàn thêm.
4. **Cầu chì 5×20 (23.6 mm)** chiếm trung tâm board và kéo dài đường nguồn → thay mini-blade/ATO hoặc fuse SMD (vd 2410/Nano2) để rút ngắn đường VM.
5. **Khoảng cách cách ly** GND (logic) ↔ đồng công suất: tối thiểu đo được **1.33 mm (F.Cu)**, 3.29 mm (B.Cu). Đủ cho cách ly chức năng < 50 V, nhưng nên ≥ 2–3 mm + rãnh/không đổ đồng dưới U3 nếu muốn chống nhiễu tốt.
6. **Lỗ bắt vít M3 có pad đồng không net**, 2 lỗ ở vùng logic, 2 lỗ ở vùng công suất: nếu gắn lên khung kim loại qua cọc đồng, chỉ cần một pad chạm via/đồng là **nối tắt GND ↔ COM**, phá cách ly. → Dùng NPTH, hoặc chủ đích nối một phía.
7. **Gerber cũ**: `gerber_to_order/*.zip` tạo 18/06/2026, PCB sửa 04/08/2026 → **không đặt mạch bằng bộ gerber này**; xuất lại sau khi refill zone + DRC sạch.

Điểm tốt của PCB: chia vùng logic/công suất rõ; mở mask trên đường công suất; nhiều via (64 via `+VM`, ~30 via mỗi M+/M−) dưới tab TO-263; silkscreen nhãn cổng đầy đủ ở mặt dưới (VIN/GND/M+/M−, TX/RX…); có teardrop; lỗ bắt vít 4 góc.

---

## 4. Dư thừa / có thể rút gọn

| Thành phần | Nhận xét |
|---|---|
| Chuỗi L7815 → AMS1117-5 | Hai tầng tuyến tính chỉ để có 5 V cho opto. Dùng 12 V (L7812) + 78L05, hoặc 1 buck → 12 V + LDO 5 V nhỏ |
| HCPL-0630 + 3 pull-up + LDO 5VP | Một digital isolator 3 kênh (ISO7731F/Si8631, có default-low) làm luôn PWMA, PWMB, nSD với ít linh kiện hơn |
| D12, D13 (LED trên DIR_IN/PULSE_IN) | Tùy chọn; tải thêm lên ngõ ra 74x14 — có thể bỏ, giữ LED nguồn + LED trạng thái |
| D7/D8/R10 (LED chiều motor) | Hữu ích khi debug nhưng phải sửa công suất R10 |
| R20 (pull-up NRST) | Có thể bỏ (pull-up nội ~40 k) |
| Cầu chì 5×20 | Quá to cho board 45×65 mm |
| MOSFET 150 V cho VM ≤ 35 V | Dư điện áp, Rds và Qg lớn hơn cần thiết. 60 V (vd IRFS3206 ~3 mΩ) giảm tổn hao dẫn ~4–5 lần |

---

## 5. Đề xuất bổ sung / cải tiến (V2)

**Kiến trúc** — chọn một trong hai:
- **A. Giữ cách ly MCU** (như hiện tại): thêm DC-DC cách ly 1 W cho phía logic để board tự chạy; đo dòng bằng isolated amplifier (AMC1301/AMC1311) hoặc chỉ dùng comparator quá dòng phía công suất kéo nSD (latch).
- **B. MCU ở phía công suất, cách ly đường truyền thông** (phổ biến hơn cho driver PID): buck từ VM → 12 V/3.3 V, đo dòng shunt + INA240 trực tiếp vào ADC, cách ly UART/CAN bằng ISO772x. Ít linh kiện hơn, có vòng dòng.

**Danh sách bổ sung:**
1. Đo dòng (shunt + amp) → vòng dòng/torque, chống kẹt; comparator quá dòng phần cứng → nSD.
2. Chia áp đo VM (bảo vệ thấp/quá áp, phát hiện năng lượng hãm tái sinh), NTC đo nhiệt MOSFET.
3. TVS + tụ ceramic sát cầu H + bulk 63 V low-ESR; bảo vệ ngược cực bằng MOSFET.
4. nSD điều khiển được từ MCU, mặc định tắt.
5. Bus đa node: PA11/PA12 đang trống → thêm **CAN** (SN65HVD230) — phù hợp với jumper địa chỉ 4 bit hơn UART/I2C dây dài.
6. RC (100 Ω + 1 nF) + ESD (TPD4E05…) trên encoder/step/UART/I2C.
7. SWD 5 chân (3V3, SWDIO, SWCLK, NRST, GND) hoặc Tag-Connect.
8. Netclass: `Power` (≥ 2 mm, clearance 0.5 mm) cho +VM/M±/COM; `Gate` (0.5 mm, ngắn); `Default` 0.2 mm.
9. Đồng 2 oz nếu dòng > 8 A.

---

## 6. Checklist sửa nhanh cho V1.1 (không đổi kiến trúc)

- [ ] Đảo D10 + thêm cầu chia cho ADC, tụ 100 nF PA3
- [ ] U5 → 78L05 (hoặc L7815 → L7812)
- [ ] 10 µF ngõ ra U5 và U6
- [ ] R8/R9 = 150 Ω
- [ ] nSD: thêm đường điều khiển từ MCU, mặc định tắt
- [ ] VBAT → 3V3, PB2 → 10 k GND, U8 pin 9 → GND, U8 = 74LVC14A
- [ ] R10 → 10 k 0805
- [ ] TVS + 2×1 µF/100 V mỗi nửa cầu, sát MOSFET
- [ ] Đặt lại U1/U2 cạnh MOSFET tương ứng
- [ ] Dời J9/SW3/SW4/R22 khỏi mép board; mounting hole NPTH
- [ ] PWR_FLAG, No-Connect flag; Refill zone → DRC sạch → xuất lại Gerber
