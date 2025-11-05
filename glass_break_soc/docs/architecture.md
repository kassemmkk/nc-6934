# Glass Break Sensing SoC - Detailed Architecture

## System Overview

The Glass Break Sensing SoC is designed to detect acoustic signatures characteristic of breaking glass. The system uses a hybrid approach with off-chip analog signal conditioning and on-chip digital processing.

## Architecture Philosophy

### Design Principles
1. **Modularity**: Clean separation between data acquisition, processing, and communication
2. **Configurability**: Runtime configuration of thresholds, sampling rates, and detection parameters
3. **Low Power**: Efficient sampling and processing with optional power-down modes
4. **Simplicity**: Prioritize reliable detection over complex algorithms
5. **Caravel-Compatible**: Fit within ~3000x3600µm user project area

### Processing Strategy: Minimal On-Chip
- **Rationale**: Balance between functionality and area/complexity
- **Approach**: 
  - Acquire samples via SPI from external ADC
  - Buffer samples in SRAM
  - Apply configurable threshold detection
  - Alert via GPIO and stream data via UART
  - Optionally forward all samples to external processor

## Block-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           Off-Chip (PCB Level)                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌──────────┐    ┌───────────┐    ┌────────────┐    ┌──────────────┐     │
│  │Electret  │───>│Microphone │───>│ Bandpass   │───>│   MCP3202    │     │
│  │Microphone│    │ Amplifier │    │  Filter    │    │  SPI ADC     │     │
│  │          │    │(MAX4466)  │    │(100Hz-15kHz│    │  (12-bit)    │     │
│  └──────────┘    └───────────┘    └────────────┘    └──────┬───────┘     │
│                                                              │ SPI          │
└──────────────────────────────────────────────────────────────┼─────────────┘
                                                               │
┌──────────────────────────────────────────────────────────────┼─────────────┐
│                        Caravel User Project                  │             │
│  ┌──────────────────────────────────────────────────────────┼──────────┐  │
│  │            Glass Break Detection Subsystem               │          │  │
│  │                                                           ↓          │  │
│  │  ┌────────────────────────────────────────────────────────────┐    │  │
│  │  │              Data Acquisition Path                         │    │  │
│  │  │                                                             │    │  │
│  │  │  ┌──────────────┐   ┌──────────────┐   ┌───────────────┐ │    │  │
│  │  │  │  SPI Master  │──>│Sample Buffer │──>│  Threshold    │ │    │  │
│  │  │  │  Controller  │   │  Controller  │   │  Detector     │ │    │  │
│  │  │  │              │   │              │   │               │ │    │  │
│  │  │  │- Configurable│   │- Circ buffer │   │- Peak detect  │ │    │  │
│  │  │  │  rate        │   │- SRAM IF     │   │- Threshold    │ │    │  │
│  │  │  │- ADC control │   │- Read/Write  │   │- Validation   │ │    │  │
│  │  │  └──────┬───────┘   └──────┬───────┘   └───────┬───────┘ │    │  │
│  │  │         │                   │                   │         │    │  │
│  │  └─────────┼───────────────────┼───────────────────┼─────────┘    │  │
│  │            │                   │                   │              │  │
│  │            ↓                   ↓                   ↓              │  │
│  │  ┌────────────────────────────────────────────────────────────┐  │  │
│  │  │              Control & Status Path                         │  │  │
│  │  │                                                             │  │  │
│  │  │  ┌──────────────┐   ┌──────────────┐   ┌───────────────┐ │  │  │
│  │  │  │Configuration │   │  Detection   │   │  Alert/Status │ │  │  │
│  │  │  │  Registers   │   │   Engine     │   │    Logic      │ │  │  │
│  │  │  │              │   │              │   │               │ │  │  │
│  │  │  │- Thresholds  │──>│- FSM         │──>│- Alert GPIO   │ │  │  │
│  │  │  │- Sample rate │   │- Event count │   │- Status GPIO  │ │  │  │
│  │  │  │- Enable/Mode │   │- Validation  │   │- Interrupt    │ │  │  │
│  │  │  └──────────────┘   └──────────────┘   └───────────────┘ │  │  │
│  │  │                                                             │  │  │
│  │  └─────────────────────────────────────────────────────────────┘  │  │
│  │                                                                     │  │
│  └───────────────────────────┬─────────────────────────────────────────┘  │
│                               │ Wishbone Bus                               │
│  ┌────────────────────────────┴──────────────────────────────────────┐    │
│  │                 Wishbone Interconnect                              │    │
│  │                                                                     │    │
│  │  ┌─────────────────────────────────────────────────────────────┐  │    │
│  │  │  Address Decoder & Arbiter                                  │  │    │
│  │  └─────────────────────────────────────────────────────────────┘  │    │
│  │         │              │              │             │               │    │
│  └─────────┼──────────────┼──────────────┼─────────────┼───────────────┘    │
│            ↓              ↓              ↓             ↓                    │
│  ┌──────────────┐  ┌─────────────┐  ┌──────────┐  ┌──────────────┐       │
│  │   CF_UART    │  │  EF_GPIO8   │  │ EF_TMR32 │  │ CF_SRAM      │       │
│  │   v2.0.1     │  │   v1.1.0    │  │  v1.1.0  │  │ _1024x32     │       │
│  │              │  │             │  │          │  │  v1.2.0      │       │
│  │- 115200 baud │  │- Alert      │  │- Sampling│  │              │       │
│  │- 8N1         │  │- Status LED │  │  timer   │  │- 4KB buffer  │       │
│  │- Debug/Data  │  │- Control    │  │- Timeout │  │- Circular    │       │
│  └──────┬───────┘  └──────┬──────┘  └────┬─────┘  └──────────────┘       │
│         │                 │               │                                │
└─────────┼─────────────────┼───────────────┼────────────────────────────────┘
          ↓                 ↓               ↓
    UART TX/RX         GPIO Pins        Timer IRQ
```

## Module Descriptions

### 1. SPI Master Controller (Custom RTL)
**Purpose**: Interface with external SPI ADC (MCP3202)

**Responsibilities**:
- Generate SPI clock (configurable frequency)
- Execute SPI transactions to read ADC samples
- Control chip select
- Continuous or triggered sampling modes
- Rate control via timer or software trigger

**Interfaces**:
- External: SPI pins (CLK, MOSI, MISO, CS)
- Internal: Wishbone slave for configuration, data output to buffer controller

**Registers**:
- `CONTROL`: Enable, mode, trigger
- `CLOCK_DIV`: SPI clock divider
- `STATUS`: Busy, error flags
- `DATA`: Last sample read (read-only)

### 2. Sample Buffer Controller (Custom RTL)
**Purpose**: Manage circular buffer in SRAM for audio samples

**Responsibilities**:
- Write samples from SPI controller to SRAM
- Implement circular buffer (wrap-around)
- Provide read access for UART streaming
- Trigger capture mode (pre/post trigger)
- Buffer status management

**Configuration**:
- Buffer size: 1024 samples (configurable subset)
- Format: 12-bit samples packed in 32-bit words
- Multiple samples per word for efficiency

**States**:
- IDLE: Waiting for enable
- STREAMING: Continuous buffer update
- TRIGGERED: Capture mode (freeze buffer on event)
- READOUT: Allow external readout

### 3. Threshold Detector (Custom RTL)
**Purpose**: Detect glass break events using threshold analysis

**Detection Algorithm**:
```
1. Monitor incoming sample stream
2. Calculate absolute value |sample|
3. Compare against HIGH_THRESHOLD
4. If exceeded:
   a. Increment event counter
   b. Start validation window timer
5. In validation window:
   - Count additional threshold crossings
   - If count >= MIN_EVENTS within WINDOW_TIME:
     * Assert ALERT
     * Set detection flag
     * Trigger buffer capture
6. Reset after timeout or acknowledgment
```

**Configurable Parameters**:
- `HIGH_THRESHOLD`: Main detection level (e.g., 75% of full scale)
- `MIN_EVENTS`: Number of crossings needed (e.g., 3-5)
- `WINDOW_TIME`: Validation window (e.g., 10ms)
- `COOLDOWN_TIME`: Rearm delay after detection (e.g., 1s)

**Outputs**:
- Alert GPIO (active high)
- Interrupt to Caravel management SoC
- Status flag in register

### 4. Detection Engine FSM (Custom RTL)
**Purpose**: Overall state management and coordination

**States**:
```
DISABLED     → (enable=1) → IDLE
IDLE         → (sample ready) → MONITORING
MONITORING   → (threshold exceeded) → VALIDATING
VALIDATING   → (criteria met) → ALERT
VALIDATING   → (timeout) → MONITORING
ALERT        → (ack or timeout) → COOLDOWN
COOLDOWN     → (timer expires) → MONITORING
ANY          → (disable=1) → DISABLED
```

**Functions**:
- Coordinate between acquisition, detection, and alert
- Manage timing sequences
- Handle enable/disable
- Provide status to software

### 5. Configuration Registers (Custom RTL)
**Purpose**: Software-accessible configuration and status

**Register Map** (detailed in address_map.md):
- Global control and status
- SPI/sampling configuration
- Detection thresholds and parameters
- Buffer control
- Interrupt management

**Access**:
- Read/Write via Wishbone bus
- 32-bit registers
- Byte-addressable

### 6. Wishbone Interconnect (Custom RTL)
**Purpose**: Connect all peripherals to Caravel management SoC

**Topology**: Shared bus with address decoder

**Slaves**:
- Glass break detection core
- CF_UART
- EF_GPIO8
- EF_TMR32
- CF_SRAM_1024x32

**Features**:
- Single master (Caravel management SoC)
- Priority arbiter (if needed for DMA)
- Address decoding
- Error response for invalid addresses

## IP Integration

### Using Pre-Installed IPs

#### CF_UART (v2.0.1)
**Usage**: Configuration, debug, data streaming
**Configuration**:
- Baud rate: 115200 (configurable)
- Data: 8 bits, No parity, 1 stop bit (8N1)
**Integration**: Wishbone slave interface

#### CF_SPI (v2.0.1)
**Note**: This IP may be used instead of custom SPI controller
**Evaluation needed**: Check if supports continuous streaming mode

#### EF_GPIO8 (v1.1.0)
**Usage**: 
- Output[0]: ALERT signal
- Output[1]: STATUS/activity LED
- Input[0]: ENABLE from external switch
**Configuration**: Direction register, output/input enable

#### EF_TMR32 (v1.1.0)
**Usage**: 
- Sampling rate generation
- Timeout/validation windows
- Cooldown timing
**Configuration**: 32-bit timer with prescaler and auto-reload

#### CF_SRAM_1024x32 (v1.2.0)
**Usage**: Sample buffer storage
**Capacity**: 1024 words × 32 bits = 4 KB
**Organization**: 
- Pack 2× 12-bit samples + metadata per word
- Up to 2048 samples (but use ~1000 for simplicity)

### IP Linking via ipm_linker
All IPs will be linked using the ipm_linker tool (see ip/link_IPs.json)

## Data Flow

### Normal Operation Flow
```
1. System Initialization:
   - Management SoC configures detection core via WB bus
   - Set thresholds, sampling rate, enable peripherals
   
2. Continuous Sampling:
   - Timer triggers SPI controller
   - SPI reads ADC sample
   - Sample written to circular buffer in SRAM
   - Threshold detector analyzes sample
   
3. Detection Event:
   - Threshold exceeded with validation
   - Alert GPIO asserted
   - Buffer capture frozen (optional)
   - Interrupt to management SoC
   
4. Response:
   - Management SoC reads status registers
   - Reads captured buffer via WB bus
   - Streams data via UART to external system
   - Acknowledges alert
   
5. Rearm:
   - Cooldown period expires
   - System returns to monitoring
```

### Data Streaming Mode
```
1. Enable streaming mode (via config register)
2. All samples continuously sent via UART
3. External processor performs analysis
4. Alert can be software-driven (write to GPIO)
```

## Timing and Performance

### Sample Rate
- **Target**: 20 ksps (20,000 samples/second)
- **Period**: 50 µs per sample
- **Nyquist**: Supports frequencies up to 10 kHz
- **Actual detection range**: 100 Hz - 10 kHz (off-chip filter limits to ~7 kHz peak)

### Latency
- **ADC conversion**: ~10 µs (MCP3202)
- **SPI transfer**: ~5 µs (depends on clock rate)
- **Processing**: < 1 µs (threshold comparison)
- **Total per sample**: ~20 µs
- **Detection latency**: 50-200 µs (depends on validation window)

### Throughput
- **Raw sample data**: 20 ksps × 12 bits = 240 kbps
- **UART capacity**: 115200 bps → ~11.5 kBps → 96 kbps
- **Conclusion**: Cannot stream all samples in real-time via UART
- **Solution**: Stream on-demand or use buffered approach

### Buffer Management
- **Buffer size**: 1024 words = 2048 samples
- **At 20 ksps**: ~100 ms of data
- **Use case**: Capture event + context (pre-trigger samples)

## Power Considerations

### Active Mode
- **SPI transactions**: 20k per second
- **SRAM accesses**: 20k writes/second + occasional reads
- **Logic**: Continuous threshold comparison
- **Estimated**: ~1-2 mW (digital core only)

### Idle Mode (Future)
- **Reduced sampling rate**: 1 ksps
- **Sleep between samples**: Clock gating
- **Estimated**: < 100 µW

## Scalability and Future Enhancements

### Phase 2 Enhancements (if area permits)
1. **Simple FIR Filter**: 8-tap bandpass filter
2. **Dual Threshold**: Separate thresholds for low/high frequency components
3. **Energy Detection**: Sum of squared samples over window
4. **Pattern Matching**: Template correlation

### Phase 3 (Advanced)
1. **FFT Core**: 64-point FFT for frequency analysis
2. **ML Inference**: Tiny classifier for pattern recognition
3. **Multi-channel**: Stereo ADC for directionality

## Design Trade-offs

| Aspect | Option A | Option B | Selected | Rationale |
|--------|----------|----------|----------|-----------|
| ADC Interface | Custom SPI | Use CF_SPI IP | Custom (TBD) | Need to verify CF_SPI streaming support |
| Processing | On-chip DSP | External processor | Hybrid | Balance area vs. flexibility |
| Buffer Size | 4 KB | 16 KB | 4 KB | Sufficient for 100ms @ 20ksps |
| Detection | Threshold only | FFT-based | Threshold | Fits in area, reliable |
| Communication | UART only | UART + SPI | UART + GPIO | Simple, adequate bandwidth |

## References
- MCP3202 Datasheet: 12-bit ADC with SPI
- Caravel User Project Documentation
- Wishbone B4 Specification
- Glass Break Detection: Theory and Practice
