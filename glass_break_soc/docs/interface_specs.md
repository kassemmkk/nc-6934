# Glass Break Sensing SoC - Interface Specifications

## External Interfaces

### 1. SPI Interface to ADC (MCP3202)

#### Signal Definitions
| Signal Name | Direction | Description                          | Caravel GPIO |
|-------------|-----------|--------------------------------------|--------------|
| SPI_CLK     | Output    | SPI clock (max 5 MHz for MCP3202)   | GPIO[0]      |
| SPI_MOSI    | Output    | Master Out Slave In (command)       | GPIO[1]      |
| SPI_MISO    | Input     | Master In Slave Out (data)          | GPIO[2]      |
| SPI_CS_N    | Output    | Chip Select (active low)            | GPIO[3]      |

#### Timing Specifications
- **Clock Frequency**: 100 kHz to 5 MHz (configurable)
- **SPI Mode**: Mode 0 (CPOL=0, CPHA=0) or Mode 3 (CPOL=1, CPHA=1)
- **Data Format**: MSB first
- **Chip Select**: Active low, must be high between transactions

#### MCP3202 Transaction Sequence

**Command Structure** (3 bytes):
```
Byte 0 (Command):
  [7:6] = 01 (Start bit + Single-ended mode)
  [5]   = Channel select (0 or 1)
  [4]   = MSB first (1)
  [3:0] = Don't care

Byte 1-2 (Response): 12-bit ADC result
```

**Timing Diagram**:
```
CS_N   : ‾‾‾\______________________________/‾‾‾‾
CLK    :    _|‾|_|‾|_|‾|_|‾|_|‾|_|‾|_|‾|_|‾|_
MOSI   : ------[cmd_byte]----[don't care]------
MISO   : ------[null]--------[adc_data]--------
         ^     ^             ^                 ^
         t0    t1            t2                t3
```

**Timing Parameters**:
- t0: CS falls, start of transaction
- t1: Command byte clocked out (8 clocks)
- t2: ADC data available (starts after null bit + sign bit)
- t3: CS rises, end of transaction
- Total: 24 clock cycles minimum

#### Sample Acquisition Flow
```c
1. Assert CS_N (low)
2. Send command byte: 0x68 (channel 0) or 0xE8 (channel 1)
3. Clock out 16 more bits while reading response
4. Extract 12-bit result from bits [11:0] of response
5. Deassert CS_N (high)
6. Wait for next sample period
```

### 2. UART Interface

#### Signal Definitions
| Signal Name | Direction | Description              | Caravel GPIO |
|-------------|-----------|--------------------------|--------------|
| UART_TX     | Output    | Transmit data            | GPIO[4]      |
| UART_RX     | Input     | Receive data             | GPIO[5]      |

#### Configuration
- **Baud Rate**: 115200 bps (configurable)
- **Data Format**: 8 bits, No parity, 1 stop bit (8N1)
- **Flow Control**: None (can add RTS/CTS if needed)
- **Protocol**: Custom binary protocol (defined below)

#### UART Protocol

**Command Format** (Host → SoC):
```
[0xAA] [CMD] [LEN] [DATA...] [CHECKSUM]

0xAA      - Sync byte
CMD       - Command code (1 byte)
LEN       - Data length (1 byte, 0-255)
DATA      - Command data (LEN bytes)
CHECKSUM  - XOR of all previous bytes
```

**Response Format** (SoC → Host):
```
[0x55] [STATUS] [LEN] [DATA...] [CHECKSUM]

0x55      - Sync byte
STATUS    - Status code (1 byte)
LEN       - Data length (1 byte)
DATA      - Response data (LEN bytes)
CHECKSUM  - XOR of all previous bytes
```

**Command Codes**:
| Code | Name            | Description                        |
|------|-----------------|------------------------------------|
| 0x01 | READ_REG        | Read register                      |
| 0x02 | WRITE_REG       | Write register                     |
| 0x03 | READ_BUFFER     | Read sample buffer                 |
| 0x04 | START_STREAM    | Start streaming samples            |
| 0x05 | STOP_STREAM     | Stop streaming                     |
| 0x10 | GET_STATUS      | Get system status                  |
| 0x11 | GET_VERSION     | Get firmware version               |
| 0x20 | TRIGGER_SAMPLE  | Software trigger single sample     |
| 0xFF | RESET           | Software reset                     |

**Status Codes**:
| Code | Name            | Description                        |
|------|-----------------|------------------------------------|
| 0x00 | OK              | Command successful                 |
| 0x01 | ERROR           | General error                      |
| 0x02 | INVALID_CMD     | Unknown command                    |
| 0x03 | INVALID_PARAM   | Invalid parameter                  |
| 0x04 | BUSY            | System busy                        |
| 0x10 | CHECKSUM_ERROR  | Checksum mismatch                  |

**Streaming Format** (continuous, no command/response framing):
```
[0xAA] [0xAA] [SAMPLE_H] [SAMPLE_L] [SAMPLE_H] [SAMPLE_L] ...

0xAA 0xAA  - Stream sync (2 bytes)
SAMPLE_H   - Sample high byte (bits [11:4])
SAMPLE_L   - Sample low byte (bits [3:0] in upper nibble)
```

### 3. GPIO Interface

#### Signal Definitions
| Signal Name  | Direction | Description                | Caravel GPIO |
|--------------|-----------|----------------------------|--------------|
| GPIO_ALERT   | Output    | Detection alert (active H) | GPIO[6]      |
| GPIO_STATUS  | Output    | Activity/status LED        | GPIO[7]      |
| GPIO_ENABLE  | Input     | External enable switch     | GPIO[8]      |

#### GPIO_ALERT Behavior
- **Default State**: Low (0)
- **Active State**: High (1) when glass break detected
- **Duration**: Remains high until acknowledged or timeout
- **Typical Duration**: 100ms - 10s (configurable)
- **Electrical**: 3.3V CMOS, can drive LED with series resistor or logic input

**Alert Sequence**:
```
1. Detection criteria met
2. GPIO_ALERT asserted (goes high)
3. Optional: Interrupt to management SoC
4. Software reads status and buffer
5. Software acknowledges (writes to register)
6. GPIO_ALERT deasserted (goes low)
7. Cooldown period begins
```

#### GPIO_STATUS Behavior
- **Mode 1 - Activity Indicator**:
  - Toggles on each sample acquired
  - Frequency: Half of sample rate (e.g., 10 Hz @ 20 ksps)
  
- **Mode 2 - Status Indicator**:
  - Slow blink: Idle/monitoring (1 Hz)
  - Fast blink: Active detection (5 Hz)
  - Solid on: Alert active
  - Off: Disabled

- **Mode 3 - Heartbeat**:
  - Regular pulse to indicate system alive
  - 1 Hz on/off

#### GPIO_ENABLE Behavior
- **Purpose**: Hardware enable/disable of detection
- **Default**: Internal pull-up (enabled if not connected)
- **Active**: High (1) = Enable, Low (0) = Disable
- **Effect**: Same as software enable but hardware override
- **Debouncing**: Implemented in hardware (10ms typical)

### 4. Power Interface

#### Power Domains
| Signal Name | Type        | Voltage | Description                    |
|-------------|-------------|---------|--------------------------------|
| VPWR        | Power       | 1.8V    | Digital core power             |
| VGND        | Ground      | 0V      | Digital ground                 |
| VDD3V3      | Power       | 3.3V    | I/O power (Caravel provides)   |
| VSS         | Ground      | 0V      | I/O ground                     |

**Power Consumption** (Estimated):
- **Active Sampling**: ~1-2 mW (digital core @ 1.8V)
- **Idle**: ~100 µW (clock gated, reduced rate)
- **Total System**: Add external ADC, amplifier, etc.

### 5. Clock and Reset

#### Clock Input
- **Source**: Caravel system clock (wb_clk_i)
- **Frequency**: Typically 25-50 MHz
- **Duty Cycle**: 50% ± 10%
- **Jitter**: < 1% of period

**Internal Clock Domains**:
- System clock: Direct from wb_clk_i
- SPI clock: Divided from system clock
- Sample clock: Divided from system clock or timer-based

#### Reset Input
- **Source**: Caravel reset (wb_rst_i)
- **Type**: Synchronous or asynchronous (design decision)
- **Polarity**: Active high (1 = reset)
- **Duration**: Minimum 10 system clocks
- **Effect**: Returns all registers to default values, clears buffers

**Reset Sequence**:
```
1. Assert wb_rst_i (high)
2. All state machines return to IDLE
3. Registers reset to default values
4. Buffer pointers cleared
5. SPI controller disabled
6. Outputs driven to safe state (ALERT=0, STATUS=0)
7. Deassert wb_rst_i (low)
8. System ready for configuration
```

## Internal Interfaces

### 6. Wishbone Bus Interface

#### Signal List
| Signal Name    | Width | Direction | Description                          |
|----------------|-------|-----------|--------------------------------------|
| wb_clk_i       | 1     | Input     | Wishbone clock                       |
| wb_rst_i       | 1     | Input     | Wishbone reset (active high)         |
| wbs_stb_i      | 1     | Input     | Strobe (valid transaction)           |
| wbs_cyc_i      | 1     | Input     | Cycle (transaction in progress)      |
| wbs_we_i       | 1     | Input     | Write enable (1=write, 0=read)       |
| wbs_sel_i      | 4     | Input     | Byte select                          |
| wbs_dat_i      | 32    | Input     | Data input (write data)              |
| wbs_adr_i      | 32    | Input     | Address                              |
| wbs_ack_o      | 1     | Output    | Acknowledge                          |
| wbs_dat_o      | 32    | Output    | Data output (read data)              |

#### Timing Diagrams

**Write Transaction**:
```
CLK    : _/‾\_/‾\_/‾\_/‾\_/‾\_/‾\_
CYC    : _/‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾\_
STB    : _/‾‾‾‾‾‾‾‾‾‾‾\_________
WE     : _/‾‾‾‾‾‾‾‾‾‾‾\_________
ADR    : X[==ADDRESS==]XXXXXXXXX
DAT_I  : X[===DATA===]XXXXXXXXX
SEL    : X[==0xF==]XXXXXXXXXXX
ACK    : _______/‾‾‾\_________
```

**Read Transaction**:
```
CLK    : _/‾\_/‾\_/‾\_/‾\_/‾\_/‾\_
CYC    : _/‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾\_
STB    : _/‾‾‾‾‾‾‾‾‾‾‾\_________
WE     : _________________________
ADR    : X[==ADDRESS==]XXXXXXXXX
DAT_O  : XXXXXXXXX[===DATA===]X
SEL    : X[==0xF==]XXXXXXXXXXX
ACK    : _______/‾‾‾\_________
```

**Burst Transaction** (not required, but can optimize):
```
CLK    : _/‾\_/‾\_/‾\_/‾\_/‾\_/‾\_/‾\_
CYC    : _/‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾\_
STB    : _/‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾\_____
WE     : _____________________________
ADR    : X[=A0=][=A1=][=A2=]XXXXXXXXX
DAT_O  : XXXXX[=D0=][=D1=][=D2=]XXXXX
ACK    : _____/‾‾‾\_/‾‾‾\_/‾‾‾\_____
```

#### Wishbone Slave Behavior
- **Latency**: 1 cycle (ACK on next clock after STB)
- **Burst**: Supported for buffer reads (optional)
- **Error Response**: Assert ERR for invalid addresses (optional)
- **Byte Select**: Full 32-bit access (SEL=0xF) recommended

### 7. SRAM Interface (to CF_SRAM_1024x32)

#### Signal List
| Signal Name | Width | Direction | Description              |
|-------------|-------|-----------|--------------------------|
| sram_clk    | 1     | Output    | SRAM clock               |
| sram_csb    | 1     | Output    | Chip select (active low) |
| sram_web    | 1     | Output    | Write enable (active low)|
| sram_addr   | 10    | Output    | Address (1024 words)     |
| sram_din    | 32    | Output    | Write data               |
| sram_dout   | 32    | Input     | Read data                |

#### Usage
- **Write**: Assert CSB=0, WEB=0, provide address and data
- **Read**: Assert CSB=0, WEB=1, provide address, read data next cycle
- **Timing**: Synchronous to system clock

### 8. Timer Interface (to EF_TMR32)

The timer is used for:
- **Sample Rate Generation**: Trigger ADC reads at fixed intervals
- **Timeout Detection**: Validation windows, cooldowns
- **Timestamp**: Optional timestamp for samples

**Integration**: Configure timer via Wishbone, use interrupt/flag for timing events

## Electrical Specifications

### DC Characteristics (1.8V Core)
| Parameter               | Min   | Typ   | Max   | Unit |
|-------------------------|-------|-------|-------|------|
| Supply Voltage (VPWR)   | 1.62  | 1.8   | 1.98  | V    |
| Logic High Input        | 1.17  | -     | 1.98  | V    |
| Logic Low Input         | 0     | -     | 0.45  | V    |
| Logic High Output       | 1.44  | 1.7   | 1.98  | V    |
| Logic Low Output        | 0     | 0.1   | 0.36  | V    |
| Input Leakage           | -     | -     | ±1    | µA   |

### I/O Characteristics (3.3V I/O)
| Parameter               | Min   | Typ   | Max   | Unit |
|-------------------------|-------|-------|-------|------|
| I/O Supply Voltage      | 3.0   | 3.3   | 3.6   | V    |
| Logic High Input        | 2.0   | -     | 3.6   | V    |
| Logic Low Input         | 0     | -     | 0.8   | V    |
| Logic High Output       | 2.4   | 3.2   | -     | V    |
| Logic Low Output        | -     | 0.1   | 0.4   | V    |
| Output Drive (Source)   | -     | 4     | 8     | mA   |
| Output Drive (Sink)     | -     | 4     | 8     | mA   |

### AC Characteristics
| Parameter               | Min   | Typ   | Max   | Unit |
|-------------------------|-------|-------|-------|------|
| System Clock Frequency  | 1     | 25    | 50    | MHz  |
| SPI Clock Frequency     | 0.1   | 1     | 5     | MHz  |
| Setup Time (Input)      | 5     | -     | -     | ns   |
| Hold Time (Input)       | 2     | -     | -     | ns   |
| Output Delay            | -     | -     | 10    | ns   |

## Off-Chip Component Interfaces

### ADC (MCP3202) Connection
```
MCP3202 Pin  | Connection
-------------|-----------------------------------------
VDD          | 3.3V (with 0.1µF decoupling cap)
VREF         | 3.3V or external reference
AGND         | Analog ground
CH0          | Microphone amplifier output
CH1          | Not used (or second channel)
CLK          | SoC SPI_CLK (GPIO[0])
DIN          | SoC SPI_MOSI (GPIO[1])
DOUT         | SoC SPI_MISO (GPIO[2])
CS/SHDN      | SoC SPI_CS_N (GPIO[3])
DGND         | Digital ground
```

**PCB Layout Recommendations**:
- Separate analog and digital ground planes
- Star ground connection
- Keep ADC close to SoC
- Short SPI traces (< 6 inches)
- Series resistors on SPI lines (22-47Ω) for impedance matching

### Microphone Amplifier Circuit

**Electret Microphone Biasing**:
```
VCC (3.3V)
    |
   [R1: 2.2kΩ]
    |
    +-----> [Microphone]
    |            |
   [C1: 10µF]   GND
    |
    +--> To Amplifier
```

**Amplifier (MAX4466 or similar)**:
```
Microphone Output --> [MAX4466] --> [Bandpass Filter] --> ADC CH0
                           |
                          Gain Control: 25x to 125x (46dB typical)
```

**Bandpass Filter (Active)**:
```
[Input] --> [High-Pass: 100Hz] --> [Low-Pass: 15kHz] --> [Output]
            (removes DC offset)     (anti-aliasing)
```

## Firmware/Software Interface

### Register Access Macros (Example)
```c
#define DETECT_BASE     0x30000000

#define REG_CORE_ID     (DETECT_BASE + 0x00)
#define REG_CONTROL     (DETECT_BASE + 0x08)
#define REG_STATUS      (DETECT_BASE + 0x0C)
// ... etc

#define REG_READ(addr)          (*(volatile uint32_t*)(addr))
#define REG_WRITE(addr, val)    (*(volatile uint32_t*)(addr) = (val))

// Example usage
void enable_detection(void) {
    uint32_t ctrl = REG_READ(REG_CONTROL);
    REG_WRITE(REG_CONTROL, ctrl | 0x07);  // EN bits
}
```

### Interrupt Handler (Pseudo-code)
```c
void detection_isr(void) {
    uint32_t int_status = REG_READ(REG_INTERRUPT_STATUS);
    
    if (int_status & DETECT_INT) {
        // Detection event
        handle_detection_event();
        REG_WRITE(REG_INTERRUPT_STATUS, DETECT_INT);  // Clear
    }
    
    if (int_status & BUFFER_INT) {
        // Buffer ready
        handle_buffer_ready();
        REG_WRITE(REG_INTERRUPT_STATUS, BUFFER_INT);
    }
}
```

## Test Points and Debug

### Recommended Test Points on PCB
- TP1: Microphone output (pre-amplifier)
- TP2: Amplifier output (pre-filter)
- TP3: ADC input (post-filter)
- TP4: SPI_CLK
- TP5: SPI_MISO (ADC data)
- TP6: GPIO_ALERT
- TP7: 3.3V analog supply
- TP8: Ground reference

### Debug Features
- **UART Debug**: Stream raw samples for analysis
- **Status LED**: Visual indication of operation
- **Register Readback**: Verify configuration
- **Sample Counter**: Verify sampling rate
- **Event Counter**: Count detections

---

**Document Version**: 1.0  
**Last Updated**: 2025-11-05  
**Status**: Initial specification
