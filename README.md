## DMA-FIR CORE: Tích hợp lọc FIR vào hệ thống SoC
 
| | |
|---|---|
| **SVTH** | Trần Hồng Sơn |
| **Môn học** | Thiết kế SoC |
| **Năm** | 2024 |
 
---
 
## Mục lục
 
1. [Yêu cầu](#i-yêu-cầu)
2. [Mô tả hệ thống](#ii-mô-tả-hệ-thống)
3. [Thiết kế](#iii-thiết-kế)
4. [Xây dựng phần mềm và mô phỏng](#iv-xây-dựng-phần-mềm-và-mô-phỏng)
5. [Đánh giá](#v-đánh-giá)
---
 
## I. Yêu cầu
 
Bài tập tích hợp lọc FIR vào hệ thống SoC với các yêu cầu:
 
- **Phát triển/tích hợp** Master Write và Master Read (DMA) vào lọc FIR trong hệ thống SoC.
- **Viết file C code** trên Eclipse để chạy phần mềm điều khiển DMA và FIR.
- **Mô phỏng** toàn hệ thống, phân tích và giải thích dạng sóng.
**Tệp nộp bài:**
- Toàn bộ code Verilog/SystemVerilog: `DMA_FIR_core.sv`, `FIR_Core.v`, `FIR_Csr.v`, `FIR_Wrapper.v`, `FIFO.v`, `RisiEdgeDetector.sv`
- File báo cáo dạng PDF/Word
- File C code chạy trên Nios II Eclipse
---
 
## II. Mô tả hệ thống
 
### 2.1 Yêu cầu kỹ thuật chung
 
- Cấu hình bộ DMA để đọc và ghi giá trị từ/vào On-Chip Memory qua Avalon Master.
- Bộ lọc FIR **8 hệ số** (H0–H7), dữ liệu và hệ số đều **8-bit**.
- Dùng **hai bộ FIFO** (FIFOi và FIFOo) để kiểm soát luồng dữ liệu giữa Master Read/Write và FIR.
- Giao tiếp dữ liệu vào/ra **32 bit** theo chuẩn Avalon Memory-Mapped.
### 2.2 Cấu trúc tổng quan
 
#### DMA FIR Core (`DMA_FIR_core.sv`)
- **Master Read**: đọc dữ liệu từ On-Chip Memory vào FIFOi.
- **Master Write**: lấy dữ liệu từ FIFOo ghi vào On-Chip Memory kết quả.
- Đường dữ liệu 32 bit, địa chỉ 32 bit.
#### Hai bộ FIFO (Intel Megafunction `dcfifo` — Cyclone V)
- **FIFOi**: nhận dữ liệu từ Master Read, cung cấp cho FIR. Độ sâu 256 từ × 32 bit.
- **FIFOo**: nhận kết quả từ FIR_Wrapper (Yn), cung cấp cho Master Write. Độ sâu 256 từ × 32 bit.
#### FIR_Wrapper (`FIR_Wrapper.v`)
- **FIR_Core**: thực hiện phép lọc FIR 8 tap; tín hiệu `Wait` để đợi hệ số nạp xong; `Write` để dịch thanh ghi trễ khi có mẫu mới.
- **FIR_Csr**: lưu hệ số H0–H7, mẫu X, kết quả Yn; giao tiếp qua `ChipSelect`, `Write`, `Read`, `Address`, `WriteData`, `ReadData`.
### 2.3 Sơ đồ hệ thống
 
```
┌───────────────────────────────────────────────────────────────────┐
│                         DMA_FIR_core                              │
│  ┌──────────────────────────────────────────────────────────────┐ │
│  │  Control/Status Registers                                    │ │
│  │  control | status | initial | read_addr | write_addr | len   │ │
│  └──────────────────────────────────────────────────────────────┘ │
│          ↑↓  Avalon Slave (iChipSelect / iWrite / iRead)          │
│                                                                   │
│  ┌──────────┐  wrreq/data  ┌─────────────┐  fifoo_wr  ┌────────┐  │
│  │ Read FSM │─────────────→│ FIR_Wrapper │←──────────-│WriteFSM│  │
│  │ r_idle   │              │  FIR_Csr    │            │ w_idle │  │
│  │ r_fw     │←─-rdempty─── │  FIR_Core   │──ReadData→ │ w_frd  │  │
│  │ r_req    │              │  Yn (24 bit)│            │ w_req  │  │
│  │ r_incr   │              └─────────────┘            │ w_incr │  │
│  └──────────┘    FIFOi          FIFOo                 └────────┘  │
│       ↕ Avalon Master Read              ↕ Avalon Master Write     │
└───────────────────────────────────────────────────────────────────┘
         ↕                                        ↕
   On-Chip Memory 1                       On-Chip Memory 2
   (dữ liệu đầu vào)                      (kết quả lọc FIR)
```
 
---
 
## III. Thiết kế
 
### 3.1 Thanh ghi điều khiển (Control/Status Registers)
 
Giao tiếp Avalon Slave dùng **3-bit address**, hỗ trợ 6 thanh ghi (Offset 0–5):
 
| Offset | Tên | Truy cập | Mô tả chi tiết |
|--------|-----|----------|----------------|
| `0` | **Control** | R/W | **Bit 0**: Start DMA — ghi 1 để bắt đầu, tự xóa khi `done_trigger`. **Bit 1**: `fifoi_clear` — xóa FIFOi. **Bit 4**: `fifoo_clear` — xóa FIFOo. **Bits [9:8]**: `fir_add` — địa chỉ FIR (00=H0-H3, 01=H4-H7, 10=X). |
| `1` | **Initial** | R/W | Giá trị hệ số lọc hoặc mẫu X. Khi `fir_add=00` ghi H0-H3; `fir_add=01` ghi H4-H7 (4 hệ số × 8 bit đóng gói trong 32 bit). |
| `2` | **Read Address** | R/W | Địa chỉ bắt đầu đọc dữ liệu từ On-Chip Memory (byte address). |
| `3` | **Write Address** | R/W | Địa chỉ bắt đầu ghi kết quả vào On-Chip Memory (byte address). |
| `4` | **Length** | R/W | Số byte cần truyền (độ dài dữ liệu DMA). |
| `5` | **Status** | R | Xem bảng chi tiết bên dưới. |
 
#### Chi tiết thanh ghi Control (Offset 0)
 
| Bit | Tên | Mô tả |
|-----|-----|-------|
| `0` | `start` | Ghi 1 để khởi động DMA. Tự xóa khi DMA hoàn thành (`done_trigger`). |
| `1` | `fifoi_clear` | Ghi 1 để xóa toàn bộ dữ liệu trong FIFOi. |
| `4` | `fifoo_clear` | Ghi 1 để xóa toàn bộ dữ liệu trong FIFOo. |
| `[9:8]` | `fir_add` | `2'b00` = ghi H0-H3; `2'b01` = ghi H4-H7; `2'b10` = ghi X vào FIR_Csr. |
 
#### Chi tiết thanh ghi Status (Offset 5)
 
| Bit | Tên | Mô tả |
|-----|-----|-------|
| `0` | `start` | 1 = DMA đang chạy; 0 = IDLE. |
| `1` | `fifoi_clear` | Mirror của `control[1]`. |
| `2` | `fifoi_empty` | FIFOi rỗng. |
| `3` | `fifoi_full` | FIFOi đầy (256 word). |
| `4` | `fifoo_clear` | Mirror của `control[4]`. |
| `5` | `fifoo_empty` | FIFOo rỗng. |
| `6` | `fifoo_full` | FIFOo đầy. |
| `7` | `dma_done` | Lên 1 khi DMA hoàn thành. Tự xóa khi `start_trigger`. |
| `[9:8]` | `fir_add` | Phản ánh `control[9:8]`. |
 
> **Biểu thức SystemVerilog:**
> ```systemverilog
> assign status = {22'd0, fir_add, dma_done,
>                  fifoo_full, fifoo_empty, fifoo_clear,
>                  fifoi_full, fifoi_empty, fifoi_clear, start};
> ```
 
---
 
### 3.2 FIFO Connection
 
Hai bộ FIFO Intel `dcfifo` với thông số:
 
| Thông số | FIFOi (Input FIFO) | FIFOo (Output FIFO) |
|----------|--------------------|---------------------|
| Megafunction | `dcfifo` (Intel) | `dcfifo` (Intel) |
| Độ rộng dữ liệu | 32 bit | 32 bit |
| Độ sâu | 256 từ | 256 từ |
| `wrreq` / `data` | `iDataValid_Master_Read` / `iReadData_Master_Read` | `fifoo_write` (Read FSM) / `fifoo_in_data` (FIR ReadData) |
| `rdreq` | `fifoi_read` (Write FSM) | `fifoo_read = oWrite_Master_Write` |
| `q` (dữ liệu ra) | `fifoi_out_data` → `FIR_Wrapper.WriteData` | `oData_Master_Write` → Avalon Master Write |
| `aclr` | `fifoi_clear = control[1]` | `fifoo_clear = control[4]` |
 
**Sơ đồ kết nối FIFO:**
 
```
 iReadData_Master_Read  ──wrreq/data──→ [ FIFOi ] ──rdreq/q──→ FIR_Wrapper.WriteData
 iDataValid_Master_Read ─────────────→ wrreq                fifoi_out_data (fir_data)
                                           │
                                     rdempty / wrfull → Write FSM / Read FSM
 
 FIR_Wrapper.ReadData ──→ fifoo_in_data
 fifoo_write (Read FSM) → wrreq [ FIFOo ] ──q──→ oData_Master_Write (→ Memory)
 oWrite_Master_Write   → rdreq
                              │
                        rdempty / wrfull → Read FSM / Write FSM
```
 
---
 
### 3.3 Máy trạng thái đọc (Read FSM)
 
Read FSM điều khiển Master Read: lấy dữ liệu từ On-Chip Memory, ghi vào FIFOi.
 
**Các trạng thái:**
 
| Trạng thái | Điều kiện vào | Hành động | Trạng thái kế |
|------------|---------------|-----------|---------------|
| `r_idle` | Reset hoặc `master_read_done` | `address_read_fetch = 1` (lấy địa chỉ đầu) | `r_fifo_write` khi `start_trigger` |
| `r_fifo_write` | `!fifoo_empty` | `fifoo_write = 1` (ghi kết quả FIR ra FIFOo) | `r_request` |
| `r_request` | Từ `r_fifo_write` hoặc `r_incr` | `oAddress_Master_Read`, `oRead_Master_Read=1`; chờ nếu `iWait \| fifoo_full` | `r_incr` khi không chờ |
| `r_incr` | Đọc thành công | `address_read_incr = 1` (tăng 4) | `r_idle` nếu done; `r_request` nếu chưa xong |
 
**Sơ đồ trạng thái:**
 
```
           start_trigger
  ┌──────────────────────────────────────────────────────┐
  │                                                      ↓
  │       ┌──────────┐  start_trigger  ┌──────────────┐ │
  │ done  │  r_idle  │────────────────→│ r_fifo_write │ │
  └───────│          │                 └──────────────┘ │
          └──────────┘                       │ !fifoo_empty
                ↑                            ↓
          done  │        ┌───────────────────────────┐
                │        │        r_request           │←─┐
          ┌──────────┐   │  oRead=1, addr=addr_read   │  │ iWait |
          │  r_incr  │←──│  → r_incr when !iWait      │  │ fifoo_full
          └──────────┘   └───────────────────────────┘──┘
```
 
> **Ghi chú:** `start_trigger` là xung 1 chu kỳ clock phát hiện rising-edge của `start` (qua `RisiEdgeDectector`).
 
---
 
### 3.4 Máy trạng thái ghi (Write FSM)
 
Write FSM điều khiển Master Write: lấy dữ liệu từ FIFOi, đưa qua FIR, ghi kết quả ra On-Chip Memory.
 
**Các trạng thái:**
 
| Trạng thái | Điều kiện vào | Hành động | Trạng thái kế |
|------------|---------------|-----------|---------------|
| `w_idle` | Reset hoặc `done_trigger` | `address_write_fetch=1` khi `!fifoi_empty` | `w_fifo_read` |
| `w_fifo_read` | Từ `w_idle` | `fifoi_read=1` nếu `!fifoi_empty` (đọc FIFOi → FIR) | `w_request` |
| `w_request` | Từ `w_fifo_read` | `oWrite_Master_Write=1`, `oAddress_Master_Write`; chờ `iWait \| fifoi_full` | `w_incr` khi không chờ |
| `w_incr` | Ghi thành công | `address_write_incr=1`; `done_trigger=1` nếu `master_write_done` | `w_idle` nếu done; `w_fifo_read` nếu chưa |
 
**Sơ đồ trạng thái:**
 
```
  ┌──────────────────────────────────────────────────────────┐
  │ done_trigger                                             │
  ↓                                                          │
  ┌──────────┐  !fifoi_empty  ┌──────────────┐               │
  │  w_idle  │───────────────→│ w_fifo_read  │─────┐         │
  └──────────┘                └──────────────┘     │ fifoi_empty (loop)
                                    ↑              ↓
                              !done │   ┌─────────────────────┐
                                    │   │     w_request       │
                              ┌──────────┐  oWrite=1, addr=.. │
                              │  w_incr  │←── → w_incr !iWait │
                              └──────────┘  └────────────────-┘
                                  │
                             done_trigger
```
 
---
 
### 3.5 FIR Wrapper
 
#### FIR_Core — Lõi lọc FIR 8 tap
 
Thực hiện phép tính lọc FIR:
 
```
Yn = Xn[0]*H7 + Xn[1]*H6 + Xn[2]*H5 + Xn[3]*H4
   + Xn[4]*H3 + Xn[5]*H2 + Xn[6]*H1 + Xn[7]*H0
```
 
| Cổng | Hướng | Rộng | Mô tả |
|------|-------|------|-------|
| `clk` | input | 1 | Xung clock hệ thống |
| `RstN` | input | 1 | Reset tích cực thấp |
| `Wait` | input | 1 | `1` = chờ hệ số nạp xong (Yn=0); `0` = tính toán |
| `Write` | input | 1 | `1` = dịch thanh ghi trễ `Xn[1..7] ← Xn[0..6]` sau mỗi mẫu mới |
| `X` | input | 8 | Mẫu đầu vào 8 bit |
| `H0–H7` | input | 8×8 | Hệ số lọc 8 bit (8 hệ số) |
| `Yn` | output | 24 | Kết quả lọc 24 bit (tránh tràn số) |
 
#### FIR_Csr — Thanh ghi điều khiển FIR
 
Quản lý hệ số H0–H7, mẫu X, tín hiệu Wait, kết quả Yn. Địa chỉ 2-bit:
 
| Address | Write (`ChipSelect & Write=1`) | Read (`ChipSelect & Read=1`) |
|---------|-------------------------------|------------------------------|
| `2'b00` | Ghi H0=[7:0], H1=[15:8], H2=[23:16], H3=[31:24]; `Wait ← 1` | — |
| `2'b01` | Ghi H4=[7:0], H5=[15:8], H6=[23:16], H7=[31:24]; `Wait ← 1` | — |
| `2'b10` | Ghi `X ← WriteData[7:0]`; `Wait ← 0` (bắt đầu tính) | `ReadData ← {8'b0, Yn[23:0]}` |
 
#### Tín hiệu điều khiển nội bộ
 
```systemverilog
// fir_data: chọn nguồn dữ liệu cho FIR_Wrapper
assign fir_data   = (fir_add == 2'b10) ? fifoi_out_data : inital;
 
// fir_select: ChipSelect cho FIR_Wrapper
assign fir_select = iChipSelect_Control | !dma_done;
 
// fir_wr: Write cho FIR_Wrapper
assign fir_wr     = fifoi_read | (iWrite_Control & (fir_add == 2'b00 | fir_add == 2'b01));
 
// FIFOi write: khi Master Read trả về dữ liệu hợp lệ
assign fifoi_write = iDataValid_Master_Read;
 
// FIFOo read: khi Master Write phát yêu cầu ghi
assign fifoo_read  = oWrite_Master_Write;
```
 
#### RisiEdgeDectector — Phát hiện cạnh lên
 
```systemverilog
assign trigger = ~sign_dl & sign;  // xung 1 chu kỳ khi sign 0→1
```
 
---
 
### 3.6 Port Interface của DMA_FIR_core
 
| Cổng | Hướng | Rộng | Interface | Mô tả |
|------|-------|------|-----------|-------|
| `iClk` | input | 1 | Clock | Xung clock hệ thống |
| `iRstn` | input | 1 | Reset | Active-low reset không đồng bộ |
| `iChipSelect_Control` | input | 1 | Avalon Slave | Chip select |
| `iWrite_Control` | input | 1 | Avalon Slave | Write strobe |
| `iRead_Control` | input | 1 | Avalon Slave | Read strobe |
| `iAddress_Control` | input | 3 | Avalon Slave | Địa chỉ thanh ghi (0–5) |
| `iData_Control` | input | 32 | Avalon Slave | Dữ liệu ghi từ CPU |
| `oData_Control` | output | 32 | Avalon Slave | Dữ liệu đọc trả về CPU |
| `oAddress_Master_Read` | output | 32 | Avalon Master Read | Địa chỉ đọc từ memory |
| `oRead_Master_Read` | output | 1 | Avalon Master Read | Read request |
| `iDataValid_Master_Read` | input | 1 | Avalon Master Read | Dữ liệu đọc hợp lệ |
| `iWait_Master_Read` | input | 1 | Avalon Master Read | Waitrequest từ slave |
| `iReadData_Master_Read` | input | 32 | Avalon Master Read | Dữ liệu đọc được |
| `oAddress_Master_Write` | output | 32 | Avalon Master Write | Địa chỉ ghi vào memory |
| `oData_Master_Write` | output | 32 | Avalon Master Write | Dữ liệu ghi (từ FIFOo.q) |
| `oWrite_Master_Write` | output | 1 | Avalon Master Write | Write request |
| `iWait_Master_Write` | input | 1 | Avalon Master Write | Waitrequest từ slave |
 
---
 
## IV. Xây dựng phần mềm và mô phỏng
 
### 4.1 Mô hình hệ thống trên Platform Designer
 
| Thành phần | Loại | Địa chỉ cơ sở | Vai trò |
|------------|------|---------------|---------|
| `clk_0` | Clock Source | — | Nguồn clock 50 MHz |
| `nios2_gen2_0` | Nios II/e Processor | `0x0002_0800` | CPU điều khiển toàn hệ thống |
| `onchip_memory2_0` | On-Chip RAM 64KB | `0x0001_8000` | Lưu chương trình Nios II |
| `jtag_uart_0` | JTAG UART | `0x0002_1038` | In kết quả qua `printf` |
| `onchip_memory2_1` | On-Chip RAM 32KB | `0x0000_8000` | Bộ nhớ dữ liệu đầu vào FIR |
| `onchip_memory2_2` | On-Chip RAM 32KB | `0x0001_0000` | Bộ nhớ kết quả đầu ra FIR |
| `dma_fir_0` | DMA for FIR (custom) | `0x0000_0000` | DMA_FIR_core: Slave + 2 Masters |
 
**Kết nối:**
- `dma_fir_0.Master_Read` → `onchip_memory2_1` (đọc input)
- `dma_fir_0.Master_Write` → `onchip_memory2_2` (ghi output)
- `Nios II` → `dma_fir_0.Slave` (cấu hình DMA và FIR)
---
 
### 4.2 Mã C kiểm tra (Nios II Eclipse)
 
#### Hàm `init_mem1()` — Khởi tạo dữ liệu đầu vào
 
```c
int init_mem1() {
    volatile int* mem_ptr = (int*) ONCHIP_MEMORY2_1_BASE;
    unsigned int inputs[] = {
        0x11, 0x13, 0x0E, 0x0B, 0x0F, 0x11, 0x0E, 0x17,
        0x0E, 0x16, 0x14, 0x11, 0x19, 0x10, 0x0D, 0x04, 0x02
    };
    int numInputs = sizeof(inputs) / sizeof(inputs[0]);
    for (int i = 0; i < numInputs; i++)
        *(mem_ptr + i) = inputs[i];
    return numInputs;  // trả về số mẫu (length)
}
```
 
#### Hàm `main()` — Cấu hình và khởi động DMA
 
```c
int main() {
    volatile int* mem_ptr    = (int*) ONCHIP_MEMORY2_1_BASE;
    volatile int* dmafir_ptr = (int*) DMA_FIR_0_BASE;
    volatile int* result_ptr = (int*) ONCHIP_MEMORY2_2_BASE;
    int length = init_mem1();
 
    // Bước 1–3: Ghi địa chỉ đọc, ghi và length vào CSR
    *(dmafir_ptr + 2) = (volatile int) ONCHIP_MEMORY2_1_BASE; // Read Address
    *(dmafir_ptr + 3) = ONCHIP_MEMORY2_2_BASE;                // Write Address
    *(dmafir_ptr + 4) = length;                               // Length
 
    // Bước 4–5: Cấu hình FIR — ghi H0-H3 (fir_add = 2'b00)
    *(dmafir_ptr + 0) = 0;           // Control: chưa start, fir_add=00
    *(dmafir_ptr + 1) = 0x040E0C08;  // H3=0x04, H2=0x0E, H1=0x0C, H0=0x08
 
    // Bước 6: Cấu hình FIR — ghi H4-H7 (fir_add = 2'b01)
    *(dmafir_ptr + 0) = 0x100;       // Control: fir_add=01
    *(dmafir_ptr + 1) = 0x0B11D06;   // H7-H4
 
    // Bước 7: Khởi động DMA (fir_add = 2'b10, start = 1)
    *(dmafir_ptr + 0) = 0x201;       // Control: fir_add=10, start=1
 
    // Bước 8: Polling dma_done (Status bit 7)
    while (1) {
        int status = *(dmafir_ptr + 5);    // Đọc Status (Offset 5)
        if ((status & 0x800) == 0x800) {   // Bit 7 (dma_done) = 1
            printf("Dma Done\n");
            for (int i = 0; i < length; i++) {
                printf("mem1[%d] = %d\n", i, *(mem_ptr + i));
                printf("mem2[%d] = %d\n", i, *(result_ptr + i));
            }
            break;
        }
    }
    printf("Hello from System");
    return 0;
}
```
 
### 4.3 Trình tự khởi động phần mềm
 
| Bước | Hành động | Ghi vào thanh ghi |
|------|-----------|-------------------|
| 1 | Ghi dữ liệu đầu vào vào On-Chip Memory 1 | Qua con trỏ `mem_ptr` |
| 2 | Đặt Read Address | `*(dmafir_ptr+2) = ONCHIP_MEMORY2_1_BASE` |
| 3 | Đặt Write Address | `*(dmafir_ptr+3) = ONCHIP_MEMORY2_2_BASE` |
| 4 | Đặt Length | `*(dmafir_ptr+4) = length` |
| 5 | Ghi hệ số H0-H3 (`fir_add=2'b00`) | `Control=0x000`, `Initial=0x040E0C08` |
| 6 | Ghi hệ số H4-H7 (`fir_add=2'b01`) | `Control=0x100`, `Initial=0x0B11D06` |
| 7 | Khởi động DMA (`fir_add=2'b10`, `start=1`) | `Control=0x201` |
| 8 | Polling bit `dma_done` (Status[7]) | Đọc `*(dmafir_ptr+5) & 0x800` |
| 9 | Đọc kết quả từ On-Chip Memory 2 | Qua con trỏ `result_ptr` |
 
---
 
### 4.4 Dạng sóng mô phỏng
 
#### Giai đoạn 1: Cấu hình hệ số H0-H3 (`fir_add = 2'b00`)
 
- `iChipSelect_Control=1`, `iWrite_Control=1`, `iAddress_Control=3'b001` (offset 1 — Initial).
- `iData_Control = 32'h040E0C08`: H0=0x08, H1=0x0C, H2=0x0E, H3=0x04.
- FIR_Csr ghi các giá trị vào H0–H3 và đặt `Wait = 1`.
- `fir_wr` được bật (`iWrite_Control & fir_add == 2'b00`).
#### Giai đoạn 2: Cấu hình hệ số H4-H7 (`fir_add = 2'b01`)
 
- Tương tự giai đoạn 1 nhưng `Control[9:8] = 2'b01`.
- `iData_Control = 32'h0B11D06` ghi vào H4–H7, `Wait` giữ = 1.
#### Giai đoạn 3: Khởi động DMA và xử lý mẫu (`fir_add = 2'b10`)
 
- `Control = 0x201`: `start=1`, `fir_add=2'b10` → `start_trigger` kích hoạt Read FSM.
- Read FSM: `r_idle` → `r_fifo_write` → `r_request`: phát `oRead_Master_Read`.
- `iDataValid_Master_Read` kích hoạt `fifoi_write` → dữ liệu vào FIFOi.
- Write FSM: `w_idle` → `w_fifo_read` → `w_request`: `fifoi_read=1` → FIR nhận X.
- FIR_Csr: `Address=2'b10`, `Wait←0`, FIR_Core tính Yn.
- `ReadData` từ FIR vào FIFOo → Master Write ghi ra On-Chip Memory 2.
- Kết thúc: `master_write_done=1`, `done_trigger=1`, `dma_done=1`, `start←0`.
#### Bảng tín hiệu theo giai đoạn
 
| Tín hiệu | Giai đoạn 1 | Giai đoạn 2 | Giai đoạn 3 (DMA chạy) |
|----------|-------------|-------------|------------------------|
| `fir_add` | `2'b00` | `2'b01` | `2'b10` |
| `fir_select` | `1` | `1` | `1` (`iChipSelect \| !dma_done`) |
| `fir_wr` | `1` | `1` | `= fifoi_read` |
| `Wait` (FIR_Csr) | `1` (set) | `1` | `0` (khi ghi X) |
| H0-H3 | Được ghi | Giữ nguyên | Giữ nguyên |
| H4-H7 | `0` | Được ghi | Giữ nguyên |
| `fifoi_empty` | `1` | `1` | `0` → `1` khi đọc xong |
| `fifoo_empty` | `1` | `1` | `0` → `1` khi ghi xong |
| `dma_done` | `0` | `0` | `1` khi `master_write_done` |
| `status[7]` | `0` | `0` | `1` (= `dma_done`) |
 
---
 
## V. Đánh giá
 
### 5.1 Kết quả đạt được
 
- ✅ Thiết kế `DMA_FIR_core` hoàn chỉnh: Read FSM, Write FSM, CSR, tích hợp `FIR_Wrapper`.
- ✅ Hai bộ FIFO kiểm soát luồng dữ liệu hiệu quả giữa Master Read/Write và FIR.
- ✅ Cấu hình hệ số lọc qua Avalon Slave: H0-H3 và H4-H7 mỗi lần 4 hệ số (đóng gói 32 bit).
- ✅ `FIR_Core` thực hiện đúng phép lọc 8 tap, kết quả 24 bit tránh tràn số.
- ✅ Tín hiệu `Wait` đảm bảo FIR không tính khi hệ số chưa nạp xong.
- ✅ Mô phỏng xác nhận đúng trình tự: cấu hình → khởi động DMA → xử lý → done.
### 5.2 Hạn chế
 
- ⚠️ `FIR_Core` dùng **blocking assignment** (`=`) trong `always @(posedge clk)`, có thể gây vấn đề synthesis.
- ⚠️ Chưa có **interrupt**: CPU phải polling `dma_done` liên tục.
- ⚠️ Thiếu cơ chế xử lý lỗi khi FIFO tràn/rỗng bất thường.

### 5.3 Hướng cải thiện
 
- 🔧 Thêm **Avalon Interrupt Sender** để CPU nhận ngắt khi DMA hoàn thành.
- 🔧 Chuyển `FIR_Core` sang **non-blocking assignment** (`<=`) để đảm bảo synthesis chính xác.
- 🔧 Mở rộng số hệ số FIR (16/32 tap) qua tham số hóa module.
- 🔧 Thêm cơ chế **back-pressure** hoàn chỉnh giữa FIFOi và `FIR_Core`.
---
 
*— HẾT —*
