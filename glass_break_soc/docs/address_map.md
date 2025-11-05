# Glass Break Sensing SoC - Address Map

## Wishbone Address Space

The Glass Break SoC uses a 32-bit Wishbone address space. The Caravel management SoC acts as the bus master.

### Address Space Overview

| Base Address | Size    | Peripheral               | Description                    |
|--------------|---------|--------------------------|--------------------------------|
| 0x3000_0000  | 256 B   | Detection Core           | Glass break detection control  |
| 0x3000_0100  | 256 B   | CF_UART                  | UART for debug/data           |
| 0x3000_0200  | 256 B   | EF_GPIO8                 | GPIO control                  |
| 0x3000_0300  | 256 B   | EF_TMR32                 | Timer for sampling/timing     |
| 0x3000_1000  | 4 KB    | CF_SRAM_1024x32          | Sample buffer                 |
| 0x3800_0000  | -       | Reserved                 | Future expansion              |

**Note**: Base addresses are configurable in the Wishbone interconnect. Values shown are recommendations.

## Detection Core Register Map

**Base Address**: 0x3000_0000

| Offset | Register Name       | Access | Reset Value | Description                           |
|--------|---------------------|--------|-------------|---------------------------------------|
| 0x00   | CORE_ID             | RO     | 0x47425301  | Core identification ("GBS" + version) |
| 0x04   | VERSION             | RO     | 0x00010000  | Version (major.minor: 1.0)            |
| 0x08   | CONTROL             | RW     | 0x00000000  | Global control register               |
| 0x0C   | STATUS              | RO     | 0x00000000  | Status register                       |
| 0x10   | INTERRUPT_EN        | RW     | 0x00000000  | Interrupt enable                      |
| 0x14   | INTERRUPT_STATUS    | RW1C   | 0x00000000  | Interrupt status (write 1 to clear)   |
| 0x18   | SPI_CONTROL         | RW     | 0x00000000  | SPI controller configuration          |
| 0x1C   | SPI_STATUS          | RO     | 0x00000000  | SPI controller status                 |
| 0x20   | SPI_CLOCK_DIV       | RW     | 0x00000010  | SPI clock divider                     |
| 0x24   | SPI_DATA            | RO     | 0x00000000  | Last SPI sample read                  |
| 0x28   | SAMPLE_RATE_DIV     | RW     | 0x00000C80  | Sample rate divider (default: 50us)   |
| 0x2C   | THRESHOLD_HIGH      | RW     | 0x00000C00  | High threshold (0-4095, def: 3072)    |
| 0x30   | THRESHOLD_LOW       | RW     | 0x00000400  | Low threshold (future use)            |
| 0x34   | DETECT_MIN_EVENTS   | RW     | 0x00000003  | Min events for detection              |
| 0x38   | DETECT_WINDOW       | RW     | 0x000000C8  | Validation window (in samples)        |
| 0x3C   | DETECT_COOLDOWN     | RW     | 0x00004E20  | Cooldown period (in samples)          |
| 0x40   | BUFFER_CONTROL      | RW     | 0x00000000  | Buffer control                        |
| 0x44   | BUFFER_STATUS       | RO     | 0x00000000  | Buffer status                         |
| 0x48   | BUFFER_WRITE_PTR    | RO     | 0x00000000  | Current write pointer                 |
| 0x4C   | BUFFER_READ_PTR     | RW     | 0x00000000  | Read pointer (for software readout)   |
| 0x50   | BUFFER_SIZE         | RW     | 0x00000400  | Buffer size in samples (default: 1024)|
| 0x54   | EVENT_COUNTER       | RO     | 0x00000000  | Total detections since reset          |
| 0x58   | SAMPLE_COUNTER      | RO     | 0x00000000  | Total samples acquired                |
| 0x5C   | ERROR_FLAGS         | RW1C   | 0x00000000  | Error flags                           |

### Register Bit Definitions

#### CONTROL (0x08)
```
[31:8]  Reserved (write 0, read undefined)
[7]     STREAM_MODE    - Enable continuous UART streaming
[6]     BUFFER_FREEZE  - Freeze buffer on detection
[5]     Reserved
[4]     SOFT_TRIGGER   - Software trigger for single sample
[3]     CONTINUOUS     - Continuous sampling mode (1) vs triggered (0)
[2]     DETECT_EN      - Enable detection engine
[1]     SPI_EN         - Enable SPI controller
[0]     CORE_EN        - Enable detection core
```

**Usage Example**:
- Enable core with detection: `CONTROL = 0x07` (CORE_EN | SPI_EN | DETECT_EN)
- Enable with streaming: `CONTROL = 0x87` (add STREAM_MODE)

#### STATUS (0x0C)
```
[31:8]  Reserved
[7]     STREAMING      - Currently streaming data
[6]     BUFFER_FROZEN  - Buffer is frozen
[5]     ALERT_ACTIVE   - Alert signal asserted
[4]     COOLDOWN       - In cooldown period
[3]     VALIDATING     - In validation window
[2]     MONITORING     - Actively monitoring samples
[1]     SPI_BUSY       - SPI transaction in progress
[0]     CORE_BUSY      - Core is active
```

#### INTERRUPT_EN (0x10)
```
[31:8]  Reserved
[7:4]   Reserved
[3]     ERROR_INT_EN   - Enable error interrupt
[2]     BUFFER_INT_EN  - Enable buffer full interrupt
[1]     SAMPLE_INT_EN  - Enable sample ready interrupt
[0]     DETECT_INT_EN  - Enable detection interrupt
```

#### INTERRUPT_STATUS (0x14) - Write 1 to Clear
```
[31:8]  Reserved
[7:4]   Reserved
[3]     ERROR_INT      - Error occurred
[2]     BUFFER_INT     - Buffer full or threshold reached
[1]     SAMPLE_INT     - New sample available
[0]     DETECT_INT     - Detection event occurred
```

#### SPI_CONTROL (0x18)
```
[31:8]  Reserved
[7:4]   SPI_MODE       - SPI mode (usually 0 for ADC)
[3]     CS_MANUAL      - Manual chip select control
[2]     CS_VALUE       - CS value when manual mode
[1]     Reserved
[0]     SPI_START      - Start SPI transaction (auto-clear)
```

#### SPI_STATUS (0x1C)
```
[31:8]  Reserved
[7:2]   Reserved
[1]     SPI_ERROR      - SPI error occurred
[0]     SPI_DONE       - SPI transaction complete
```

#### SPI_CLOCK_DIV (0x20)
```
[31:16] Reserved
[15:0]  CLOCK_DIV      - SPI clock divider
                         SPI_CLK = SYS_CLK / (2 * (CLOCK_DIV + 1))
                         Default: 0x10 (16) → 1.5625 MHz @ 50 MHz system clock
```

**Example Calculations** (assuming 50 MHz system clock):
- `CLOCK_DIV = 1`: SPI_CLK = 12.5 MHz
- `CLOCK_DIV = 4`: SPI_CLK = 5 MHz (max for MCP3202)
- `CLOCK_DIV = 16`: SPI_CLK = 1.5625 MHz (default, safe)

#### SPI_DATA (0x24)
```
[31:16] Reserved
[15:12] Reserved (or channel info)
[11:0]  ADC_DATA       - 12-bit ADC sample (right-aligned)
```

#### SAMPLE_RATE_DIV (0x28)
```
[31:16] Reserved
[15:0]  SAMPLE_DIV     - Sample period in system clocks
                         Sample_Rate = SYS_CLK / SAMPLE_DIV
                         Default: 0x0C80 (3200) → 15.625 kHz @ 50 MHz
```

**Example Calculations** (50 MHz system clock):
- 10 ksps: `SAMPLE_DIV = 5000` (0x1388)
- 20 ksps: `SAMPLE_DIV = 2500` (0x09C4)
- 50 ksps: `SAMPLE_DIV = 1000` (0x03E8)

#### THRESHOLD_HIGH (0x2C)
```
[31:12] Reserved
[11:0]  THRESHOLD      - Detection threshold (0-4095)
                         Typically 70-80% of full scale (2867-3277)
                         Default: 3072 (75%)
```

#### DETECT_MIN_EVENTS (0x34)
```
[31:8]  Reserved
[7:0]   MIN_EVENTS     - Minimum threshold crossings for valid detection
                         Default: 3 events
                         Range: 1-255 (typical: 2-5)
```

#### DETECT_WINDOW (0x38)
```
[31:16] Reserved
[15:0]  WINDOW_SIZE    - Validation window in samples
                         Default: 200 samples (10ms @ 20ksps)
                         Range: 1-65535
```

#### DETECT_COOLDOWN (0x3C)
```
[31:16] Reserved
[15:0]  COOLDOWN       - Cooldown period in samples
                         Default: 20000 samples (1s @ 20ksps)
                         Range: 1-65535
```

#### BUFFER_CONTROL (0x40)
```
[31:8]  Reserved
[7:4]   Reserved
[3]     BUFFER_RESET   - Reset buffer pointers
[2]     FREEZE_EN      - Enable auto-freeze on detection
[1]     CIRCULAR_EN    - Enable circular mode (wrap around)
[0]     BUFFER_EN      - Enable buffer writes
```

#### BUFFER_STATUS (0x44)
```
[31:16] BUFFER_COUNT   - Number of samples in buffer
[15:8]  Reserved
[7:4]   Reserved
[3]     FROZEN         - Buffer is frozen
[2]     FULL           - Buffer is full
[1]     EMPTY          - Buffer is empty
[0]     ACTIVE         - Buffer is actively recording
```

#### BUFFER_WRITE_PTR (0x48)
```
[31:12] Reserved
[11:0]  WRITE_PTR      - Current write address (0-1023)
```

#### BUFFER_READ_PTR (0x4C)
```
[31:12] Reserved
[11:0]  READ_PTR       - Read address for software access (0-1023)
```

#### ERROR_FLAGS (0x5C) - Write 1 to Clear
```
[31:8]  Reserved
[7:4]   Reserved
[3]     BUFFER_OVERRUN - Buffer overrun occurred
[2]     SPI_TIMEOUT    - SPI transaction timeout
[1]     SPI_ERROR      - SPI communication error
[0]     CONFIG_ERROR   - Invalid configuration detected
```

## SRAM Buffer Memory Map

**Base Address**: 0x3000_1000  
**Size**: 4096 bytes (1024 × 32-bit words)

### Buffer Organization

Each 32-bit word can store multiple samples:

#### Option 1: Dual Sample Storage (Recommended)
```
[31:28] Reserved (0)
[27:16] Sample 1 (12-bit)
[15:12] Reserved (0)
[11:0]  Sample 0 (12-bit)
```
- Capacity: 2048 samples
- Address calculation: `word_addr = sample_num / 2`

#### Option 2: Single Sample with Metadata
```
[31:16] Timestamp (lower 16 bits)
[15:12] Flags/Status
[11:0]  Sample (12-bit)
```
- Capacity: 1024 samples with context
- More overhead but includes timing info

### Buffer Access

**Write**: Automatic via buffer controller during acquisition

**Read**: Software access via Wishbone
```
1. Set BUFFER_READ_PTR to starting address
2. Read from SRAM base + (READ_PTR * 4)
3. Increment READ_PTR (or hardware auto-increment)
4. Repeat until desired samples retrieved
```

**Example**: Read 100 samples starting from beginning
```c
write(BUFFER_READ_PTR, 0);
for (int i = 0; i < 50; i++) {  // 50 words = 100 samples
    uint32_t data = read(SRAM_BASE + i * 4);
    uint16_t sample0 = data & 0xFFF;
    uint16_t sample1 = (data >> 16) & 0xFFF;
    // Process samples...
}
```

## CF_UART Register Map

**Base Address**: 0x3000_0100

Refer to CF_UART IP documentation for detailed register map. Common registers:

| Offset | Register    | Description              |
|--------|-------------|--------------------------|
| 0x00   | RX_DATA     | Receive data             |
| 0x04   | TX_DATA     | Transmit data            |
| 0x08   | STATUS      | UART status              |
| 0x0C   | CONTROL     | UART control             |
| 0x10   | PRESCALER   | Baud rate prescaler      |

## EF_GPIO8 Register Map

**Base Address**: 0x3000_0200

| Offset | Register    | Description              |
|--------|-------------|--------------------------|
| 0x00   | DATA_IN     | GPIO input values        |
| 0x04   | DATA_OUT    | GPIO output values       |
| 0x08   | DIR         | Direction (0=in, 1=out)  |
| 0x0C   | CONTROL     | GPIO control             |

**GPIO Assignments**:
```
GPIO[0] - ALERT output (detection signal)
GPIO[1] - STATUS output (activity indicator)
GPIO[2] - Reserved
GPIO[3] - Reserved
GPIO[4] - Reserved
GPIO[5] - Reserved
GPIO[6] - ENABLE input (external enable switch)
GPIO[7] - Reserved
```

## EF_TMR32 Register Map

**Base Address**: 0x3000_0300

Refer to EF_TMR32 IP documentation. Commonly used for:
- Generating sampling rate clock
- Timeout detection
- Validation window timing

## Software Access Examples

### Initialize Detection Core
```c
// Enable core and SPI
write(CONTROL, 0x03);  // CORE_EN | SPI_EN

// Configure SPI clock (5 MHz for MCP3202)
write(SPI_CLOCK_DIV, 4);

// Set sample rate (20 ksps @ 50 MHz system clock)
write(SAMPLE_RATE_DIV, 2500);

// Configure detection parameters
write(THRESHOLD_HIGH, 3072);      // 75% of 4095
write(DETECT_MIN_EVENTS, 3);      // 3 crossings
write(DETECT_WINDOW, 200);        // 10ms window @ 20ksps
write(DETECT_COOLDOWN, 20000);    // 1s cooldown

// Enable buffer
write(BUFFER_CONTROL, 0x07);  // BUFFER_EN | CIRCULAR_EN | FREEZE_EN

// Enable detection
uint32_t ctrl = read(CONTROL);
write(CONTROL, ctrl | 0x04);  // Set DETECT_EN

// Enable detection interrupt
write(INTERRUPT_EN, 0x01);  // DETECT_INT_EN
```

### Handle Detection Event
```c
// Check interrupt status
uint32_t int_status = read(INTERRUPT_STATUS);
if (int_status & 0x01) {  // DETECT_INT
    // Read buffer
    uint16_t samples[1024];
    read_buffer(samples, 1024);
    
    // Process or stream samples
    uart_send_samples(samples, 1024);
    
    // Clear interrupt
    write(INTERRUPT_STATUS, 0x01);  // Write 1 to clear
    
    // Acknowledge alert (if needed)
    // System will automatically enter cooldown
}
```

### Stream Samples via UART
```c
// Enable streaming mode
uint32_t ctrl = read(CONTROL);
write(CONTROL, ctrl | 0x80);  // Set STREAM_MODE

// In interrupt handler or polling loop:
while (1) {
    uint32_t status = read(STATUS);
    if (status & 0x80) {  // STREAMING
        uint32_t sample = read(SPI_DATA);
        uart_send((sample >> 8) & 0xFF);  // MSB
        uart_send(sample & 0xFF);         // LSB
    }
}
```

## Memory Map Summary Table

| Start Address | End Address  | Size   | Component        | Type          |
|---------------|--------------|--------|------------------|---------------|
| 0x3000_0000   | 0x3000_00FF  | 256 B  | Detection Core   | Registers     |
| 0x3000_0100   | 0x3000_01FF  | 256 B  | CF_UART          | Registers     |
| 0x3000_0200   | 0x3000_02FF  | 256 B  | EF_GPIO8         | Registers     |
| 0x3000_0300   | 0x3000_03FF  | 256 B  | EF_TMR32         | Registers     |
| 0x3000_0400   | 0x3000_0FFF  | 3 KB   | Reserved         | -             |
| 0x3000_1000   | 0x3000_1FFF  | 4 KB   | SRAM Buffer      | Memory        |
| 0x3000_2000   | 0x3FFF_FFFF  | ~255MB | Reserved         | Future        |

---

**Document Version**: 1.0  
**Last Updated**: 2025-11-05  
**Status**: Initial specification
