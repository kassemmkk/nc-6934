# Glass Break Sensing SoC - IP Gap Analysis & Solutions

## Executive Summary

This document provides a comprehensive analysis of IP requirements for the Glass Break Sensing SoC, identifies gaps in the available NativeChips IP library, and proposes solutions for each gap.

**Key Findings**:
- ✅ **5 Pre-Installed IPs** available and suitable for use
- ❌ **3 Critical Hardware Gaps** requiring off-chip components
- 🔧 **6 Custom RTL Modules** required for application-specific logic
- 💰 **Estimated Off-Chip Cost**: ~$13 per system

---

## Available Pre-Installed IPs

### Communication Interfaces

#### 1. CF_UART (v2.0.1) ✅ **VERIFIED - READY TO USE**
**Purpose**: Debug interface, configuration, and data streaming

**Key Features**:
- Configurable baud rate (target: 115200 bps)
- Standard 8N1 format
- Wishbone slave interface
- TX/RX with status flags

**Usage in Design**:
- Upload configuration parameters
- Stream captured samples to external processor
- Debug and diagnostics
- Firmware updates (if management SoC supports)

**Integration**: Use ipm_linker, connect via Wishbone bus

**Documentation**: Available at `/nc/ip/CF_UART/`

**Status**: ✅ No gaps, ready for integration

---

#### 2. CF_SPI (v2.0.1) ⚠️ **REQUIRES EVALUATION**
**Purpose**: Potential interface for ADC (alternative to custom SPI master)

**Key Features**:
- Configurable SPI modes
- Wishbone slave interface
- Master/slave capable

**Usage Consideration**:
- **Primary Use Case**: Interface to MCP3202 SPI ADC
- **Critical Requirement**: Must support continuous streaming mode at 20 ksps
- **Decision Needed**: Verify if CF_SPI can handle continuous transactions without software intervention

**Action Required**:
```bash
# Query documentation for CF_SPI streaming capability
query_docs_db("CF_SPI continuous streaming mode autonomous")
```

**Alternatives**:
- If CF_SPI supports autonomous mode → Use CF_SPI IP (preferred, saves dev time)
- If CF_SPI requires per-transaction software control → Develop custom SPI master controller

**Status**: ⚠️ Needs evaluation before deciding

---

### Memory

#### 3. CF_SRAM_1024x32 (v1.2.0) ✅ **VERIFIED - READY TO USE**
**Purpose**: Audio sample buffer storage

**Specifications**:
- Size: 1024 words × 32 bits = 4 KB = 32,768 bits
- Access: Synchronous read/write
- Wishbone interface: Available (check documentation)
- Direct interface: Clock, address, data in/out, control signals

**Capacity Analysis**:
```
Storage Format: Dual-sample per word
  - Word structure: [27:16] Sample1 | [11:0] Sample0
  - Effective capacity: 2048 samples
  
At 20 ksps:
  - Buffer duration: 2048 / 20000 = 102.4 ms
  
Use Cases:
  ✓ Pre/post trigger capture (50ms before + 50ms after event)
  ✓ Continuous circular buffer for monitoring
  ✓ Snapshot capture on detection
```

**Alternative**: CF_SRAM_4096x32 (v1.0.2) - 16 KB if larger buffer needed

**Usage in Design**:
- Circular buffer for incoming samples
- Freeze on detection for analysis
- Read via Wishbone for UART transmission

**Integration**: Custom buffer controller interfaces with SRAM

**Status**: ✅ No gaps, sufficient capacity

---

### Timers and Control

#### 4. EF_TMR32 (v1.1.0) ✅ **VERIFIED - READY TO USE**
**Purpose**: Sampling rate control, timeouts, validation windows

**Key Features**:
- 32-bit timer/counter
- Configurable prescaler
- Auto-reload mode
- Interrupt generation
- Wishbone slave interface

**Usage in Design**:
1. **Sample Rate Generation**:
   - Configure for 50 µs period (20 ksps)
   - Interrupt/flag triggers SPI controller
   
2. **Validation Window Timing**:
   - Count samples in validation window (e.g., 200 samples = 10ms)
   
3. **Cooldown Timer**:
   - Delay before re-arming after detection (e.g., 1 second)
   
4. **Timeout Detection**:
   - SPI transaction timeout
   - System watchdog

**Configuration Example**:
```
System Clock: 50 MHz
Target Sample Rate: 20 ksps (50 µs period)

Timer Config:
  - Prescaler: 1 (or minimal)
  - Auto-reload value: 2500 (50MHz / 20kHz)
  - Mode: Auto-reload with interrupt
```

**Integration**: Configure via Wishbone, use interrupt/flag for timing events

**Status**: ✅ No gaps, meets all timing requirements

---

#### 5. EF_GPIO8 (v1.1.0) ✅ **VERIFIED - READY TO USE**
**Purpose**: Alert output, status indicators, external control

**Key Features**:
- 8 programmable I/O pins
- Configurable direction (input/output)
- Wishbone slave interface
- Direct register access

**Pin Assignment**:
```
GPIO[0] - ALERT (Output)      : Detection event signal
GPIO[1] - STATUS (Output)     : Activity/status LED
GPIO[2] - Reserved (Output)   : Future use
GPIO[3] - Reserved (Output)   : Future use
GPIO[4] - Reserved (I/O)      : Future use
GPIO[5] - Reserved (I/O)      : Future use
GPIO[6] - ENABLE (Input)      : Hardware enable switch
GPIO[7] - Reserved (Input)    : Future use
```

**ALERT Signal Behavior**:
- Asserted when glass break detected
- Remains high until acknowledged
- Can drive LED (with series resistor) or logic input
- Configurable polarity and duration

**STATUS Signal Behavior** (Configurable Modes):
- Mode 1: Toggle on each sample (activity indicator)
- Mode 2: Slow blink (idle), fast blink (active), solid (alert)
- Mode 3: Heartbeat (1 Hz pulse to show system alive)

**Integration**: Connect to Caravel GPIO pins, configure via Wishbone

**Status**: ✅ No gaps, sufficient I/O

---

### Optional/Future Use

#### 6. EF_PWM32 (v1.0.0) 🔵 **OPTIONAL**
**Potential Uses**:
- LED brightness control (status indicator dimming)
- Audio output for test tone generation
- Future: Acoustic feedback or alarm

**Status**: Not required for MVP, available if needed

---

#### 7. EF_WDT32 (v1.1.0) 🔵 **OPTIONAL**
**Potential Uses**:
- System watchdog for reliability
- Automatic recovery from hung states

**Status**: Not required for MVP, recommended for production

---

## IP Gaps - Off-Chip Solutions Required

### Critical Hardware Gaps

#### GAP 1: Analog-to-Digital Converter ❌ **NOT AVAILABLE**

**Requirement**:
- Convert analog audio signal to digital samples
- Sample rate: 10-20 ksps minimum (20 ksps target)
- Resolution: 10-12 bits adequate for acoustic detection
- Interface: Digital (SPI or I2C preferred)

**Why Not Available**:
- Skywater 130nm process used by Caravel is digital-only
- ADC requires analog design (comparators, charge pumps, etc.)
- Analog design outside scope of digital open-source flow

**Off-Chip Solution**: External SPI ADC

**Recommended Component**: **MCP3202**

| Specification | Value | Notes |
|--------------|-------|-------|
| Resolution | 12-bit | 4096 levels, 0.8 mV per step @ 3.3V ref |
| Sample Rate | 100 ksps max | 20 ksps typical for this application |
| Channels | 2 (differential or single-ended) | Use channel 0 for microphone |
| Interface | SPI | Compatible with custom SPI master |
| SPI Clock | Up to 5 MHz | Conservative: 1-2 MHz |
| Supply Voltage | 2.7V - 5.5V | 3.3V nominal |
| Power Consumption | 900 µA active, 0.5 µA standby | Low power |
| Package | DIP-8, SOIC-8 | Easy to prototype |
| Cost | ~$2.50 | Widely available |

**Alternative Options**:

| Part Number | Resolution | Sample Rate | Interface | Cost | Notes |
|-------------|-----------|-------------|-----------|------|-------|
| MCP3201 | 12-bit | 100 ksps | SPI | $2.00 | Single channel |
| TLV1572 | 10-bit | 1.25 Msps | SPI | $3.50 | Higher speed |
| ADS1115 | 16-bit | 860 sps | I2C | $4.50 | High res, low speed |
| ADS7816 | 12-bit | 200 ksps | SPI | $5.00 | Higher performance |

**MCP3202 Selection Rationale**:
- ✅ Adequate resolution (12-bit) for acoustic signals
- ✅ Sample rate (100 ksps max) covers 20 ksps requirement with margin
- ✅ SPI interface for fast, continuous acquisition
- ✅ Low cost and widely available
- ✅ Dual channel (can add second microphone later)
- ✅ Simple command structure

**Interface Requirements**:
- **Signals**: CLK, MOSI (DIN), MISO (DOUT), CS
- **SPI Mode**: Mode 0 (CPOL=0, CPHA=0) or Mode 3
- **Clock Frequency**: 100 kHz to 5 MHz (configurable)
- **Transaction**: 24 clock cycles (8-bit command + 16-bit response)

**Integration**:
- Connect to Caravel GPIO[0:3] for SPI interface
- Custom SPI master controller (or CF_SPI IP if suitable)
- Continuous polling at 20 ksps (50 µs period)

**Cost Impact**: +$2.50 per system

---

#### GAP 2: Analog Front-End (Microphone + Amplifier + Filter) ❌ **NOT AVAILABLE**

**Requirement**:
- Capture acoustic signal from environment
- Amplify low-level microphone signal (mV range) to ADC input range (0-3.3V)
- Filter to pass glass break frequencies (100 Hz - 15 kHz)
- Remove noise and DC offset

**Why Not Available**:
- Requires analog components (op-amps, resistors, capacitors)
- Acoustic transducer (microphone) is inherently off-chip
- Analog signal conditioning not available in digital ASIC

**Off-Chip Solution**: External analog circuit

**System Block Diagram**:
```
[Microphone] → [Bias Circuit] → [Pre-Amp] → [Bandpass Filter] → [ADC]
```

---

**Component 2A: Microphone** 🎤

**Recommended**: **Electret Condenser Microphone (ECM)**

**Example Part**: CUI CMC-4015-42316

| Specification | Value | Notes |
|--------------|-------|-------|
| Type | Electret condenser | Standard, cost-effective |
| Sensitivity | -42 dB (7.9 mV/Pa) | Good for acoustic detection |
| Frequency Range | 100 Hz - 10 kHz | Covers glass break spectrum |
| SNR | 58 dB | Adequate |
| Supply Voltage | 1.5V - 10V | 3.3V via bias resistor |
| Output Impedance | 2.2 kΩ | Requires bias resistor |
| Cost | ~$1.50 | Very affordable |

**Bias Circuit**:
```
VCC (3.3V)
    |
   [R: 2.2kΩ] ← Bias resistor
    |
    +----[Microphone]
    |         |
   [C: 10µF] GND
    |
    +--→ To Amplifier (AC coupled)
```

**Alternative**: MEMS Microphone (e.g., Knowles SPH0645)
- Pros: Better SNR, smaller, digital output option (I2S)
- Cons: Higher cost ($2-3), requires I2S interface if digital

**Cost Impact**: +$1.50 per system

---

**Component 2B: Pre-Amplifier** 📢

**Recommended**: **MAX4466** (Microphone Amplifier with AGC)

| Specification | Value | Notes |
|--------------|-------|-------|
| Type | Low-noise op-amp with AGC | Optimized for microphones |
| Gain | 25x to 125x adjustable | 46 dB typical (set via resistor) |
| Supply Voltage | 2.4V - 5.5V | 3.3V operation |
| Output Swing | Rail-to-rail | Maximizes ADC input range |
| Bandwidth | DC to 600 kHz | Covers audio spectrum |
| Noise | Low | Good for acoustic signals |
| Cost | ~$1.95 | Single IC solution |
| Package | TSSOP-8 | Compact |

**Gain Calculation**:
```
Microphone Output: ~7.9 mV/Pa
Typical acoustic level: 80 dB SPL = 0.2 Pa
Microphone signal: 7.9 mV × 0.2 = 1.58 mV

Target ADC input: 1V (center of 0-3.3V range)
Required gain: 1V / 1.58mV = 633 (56 dB)

MAX4466 gain: Adjustable 25x to 125x (28-42 dB)
Use maximum gain (125x = 42 dB) + additional stage if needed
Or use dual op-amp configuration
```

**Alternative**: Generic op-amp solution with **MCP6022** (Dual Op-Amp)
- Cost: $0.60 (cheaper)
- Requires external gain-setting components
- More design work but lower BOM cost

**Recommendation**: Use MAX4466 for MVP (simpler), consider MCP6022 for cost optimization

**Cost Impact**: +$1.95 per system (MAX4466) or +$0.60 (MCP6022)

---

**Component 2C: Bandpass Filter** 📊

**Purpose**: 
- Remove DC offset (high-pass)
- Limit frequency range to glass break spectrum (100 Hz - 15 kHz)
- Anti-aliasing (low-pass at ~10 kHz for 20 ksps ADC)

**Recommended**: **Active RC Filter (Sallen-Key Topology)**

**High-Pass Section** (Remove DC, pass > 100 Hz):
```
[Input] → [C1: 1µF] → [R1: 1.6kΩ] → [Op-Amp Buffer] → [Output]
                       |
                      [C2: 1µF]
                       |
                      GND

Cutoff Frequency: fc = 1 / (2π × R1 × C1) ≈ 100 Hz
```

**Low-Pass Section** (Anti-aliasing, pass < 15 kHz):
```
[Input] → [R2: 1kΩ] → [C3: 10nF] → [Op-Amp Buffer] → [Output]
                       |
                      GND

Cutoff Frequency: fc = 1 / (2π × R2 × C3) ≈ 16 kHz
```

**Combined Solution**: Second op-amp in MCP6022 used for active filtering

**Component Cost**:
- Op-amp (MCP6022): $0.60 (shared with amplifier stage if designed correctly)
- Resistors (4x): $0.20
- Capacitors (4x): $0.40
- **Total**: ~$0.60 (if using separate filter stage)

**Alternative**: Passive RC filter (simpler, lower cost, but less sharp cutoff)

**Cost Impact**: +$0.60 per system (included in amplifier design)

---

**Total Analog Front-End Cost**: ~$4 (microphone + MAX4466 + passives)  
**Budget Option**: ~$2.50 (microphone + MCP6022 + passives)

---

#### GAP 3: Power Supply & Regulation ⚡ **PARTIALLY AVAILABLE**

**Requirement**:
- Provide clean 3.3V for analog components (ADC, amplifier, microphone)
- Separate analog ground from digital ground (reduce noise)

**Caravel Provides**:
- 1.8V digital core power (VPWR/VGND)
- 3.3V I/O power (VDD3V3/VSS)

**Gap**: May need separate analog 3.3V regulator for cleanest analog performance

**Off-Chip Solution**: **Optional LDO Regulator**

**Recommended**: MCP1700-33 (3.3V LDO)

| Specification | Value | Notes |
|--------------|-------|-------|
| Output Voltage | 3.3V | Fixed |
| Input Voltage | 3.5V - 16V | Can use 5V USB or battery |
| Output Current | 250 mA | More than enough (<50mA needed) |
| Dropout Voltage | 625 mV | Low dropout |
| Quiescent Current | 1.6 µA | Very low standby |
| Cost | ~$0.40 | Very affordable |

**PCB Design**:
- Separate analog and digital ground planes
- Star ground topology
- Ferrite bead between digital and analog power

**Cost Impact**: +$0.40 per system (optional but recommended)

---

### IP Gaps Summary Table

| Gap | Type | Off-Chip Solution | Cost | Mandatory |
|-----|------|-------------------|------|-----------|
| ADC | Hardware | MCP3202 (12-bit SPI) | $2.50 | ✅ Yes |
| Microphone | Hardware | Electret ECM | $1.50 | ✅ Yes |
| Amplifier | Hardware | MAX4466 or MCP6022 | $1.95 | ✅ Yes |
| Filter | Hardware | Active RC (op-amp + passives) | $0.60 | ✅ Yes |
| Power Reg | Hardware | MCP1700-33 (optional) | $0.40 | ⚠️ Recommended |
| Passives | Hardware | Resistors, capacitors | $1.00 | ✅ Yes |
| PCB | Hardware | Custom 2-layer board | $5.00 | ✅ Yes |
| **Total** | - | - | **~$13** | - |

---

## Custom RTL Gaps

### Modules Required

#### 1. SPI Master Controller 🔧 **CUSTOM RTL REQUIRED**
**Status**: May be replaceable by CF_SPI IP (pending evaluation)

**Functionality**:
- Generate SPI clock (configurable divider)
- Execute SPI transactions to MCP3202
- Continuous or triggered sampling modes
- Wishbone slave interface for configuration

**Complexity**: Medium  
**Estimated LOC**: 300-400  
**Estimated Effort**: 2-3 days

**Decision Point**:
```
IF CF_SPI supports autonomous continuous mode:
    → Use CF_SPI IP (0 days development)
ELSE:
    → Develop custom SPI master (2-3 days)
```

---

#### 2. Sample Buffer Controller 🔧 **CUSTOM RTL REQUIRED**

**Functionality**:
- Interface between SPI controller and SRAM
- Implement circular buffer (wrap-around)
- Manage read/write pointers
- Freeze buffer on detection event (capture mode)
- Provide software read access via Wishbone

**Features**:
- Configurable buffer size
- Pre-trigger capture (keep N samples before event)
- Post-trigger capture (keep M samples after event)
- Buffer full/empty status flags

**Complexity**: Medium  
**Estimated LOC**: 400-500  
**Estimated Effort**: 2-3 days

**Interfaces**:
- Input: Sample data from SPI controller
- Output: SRAM interface (address, data, control)
- Control: Wishbone slave for configuration and readout

---

#### 3. Threshold Detector 🔧 **CUSTOM RTL REQUIRED**

**Functionality**:
- Monitor incoming sample stream
- Compare against configurable threshold
- Count threshold crossings in validation window
- Generate detection flag if criteria met

**Algorithm**:
```verilog
1. Calculate |sample| (absolute value)
2. Compare: |sample| > THRESHOLD_HIGH
3. If exceeded:
   - Increment crossing_count
   - Start validation_window_timer
4. If crossing_count >= MIN_EVENTS within WINDOW_TIME:
   - Assert detection_flag
   - Trigger buffer freeze
   - Assert alert output
5. Enter cooldown period
```

**Complexity**: Low-Medium  
**Estimated LOC**: 200-300  
**Estimated Effort**: 1-2 days

**Configurable Parameters** (via Wishbone registers):
- THRESHOLD_HIGH: Detection threshold (0-4095)
- MIN_EVENTS: Minimum crossings (typically 3-5)
- WINDOW_TIME: Validation window in samples (e.g., 200 = 10ms @ 20ksps)
- COOLDOWN_TIME: Rearm delay in samples (e.g., 20000 = 1s)

---

#### 4. Detection Engine FSM 🔧 **CUSTOM RTL REQUIRED**

**Functionality**:
- Overall state machine for detection process
- Coordinate between acquisition, detection, and alert
- Manage timing sequences
- Handle enable/disable

**States**:
- DISABLED: System off
- IDLE: Enabled but not sampling
- MONITORING: Continuous sampling and detection active
- VALIDATING: Potential event, counting crossings
- ALERT: Detection confirmed, alert asserted
- COOLDOWN: Rearm delay after detection

**Complexity**: Medium  
**Estimated LOC**: 300-400  
**Estimated Effort**: 2 days

**Interfaces**:
- Inputs: Enable, trigger, detection flags, timer status
- Outputs: State outputs, control signals to other modules
- Control: Wishbone slave for status readback

---

#### 5. Configuration Registers 🔧 **CUSTOM RTL REQUIRED**

**Functionality**:
- Wishbone-accessible register bank
- Store configuration parameters
- Provide status readback
- Interrupt management

**Registers**: ~20 registers (see address_map.md)

**Complexity**: Low  
**Estimated LOC**: 200-300  
**Estimated Effort**: 1-2 days

**Features**:
- Read/write access via Wishbone
- Default values on reset
- Write protection for critical registers (optional)
- Interrupt enable/status/clear

---

#### 6. Wishbone Interconnect 🔧 **CUSTOM RTL REQUIRED**

**Functionality**:
- Connect multiple Wishbone slaves to single master (Caravel mgmt SoC)
- Address decoding
- Transaction routing
- Arbitration (if multiple masters needed)

**Topology**: Shared bus or crossbar

**Slaves**:
1. Detection core registers
2. CF_UART
3. EF_GPIO8
4. EF_TMR32
5. CF_SRAM_1024x32

**Complexity**: Medium  
**Estimated LOC**: 400-600  
**Estimated Effort**: 2-3 days

**Features**:
- Address decoder (5+ slaves)
- Error response for invalid addresses
- Single-cycle or pipelined (design choice)
- Optional: Burst support for buffer reads

---

### Custom RTL Summary Table

| Module | Complexity | LOC | Effort | Priority | Alternative |
|--------|-----------|-----|--------|----------|-------------|
| SPI Master | Medium | 300-400 | 2-3 days | HIGH | CF_SPI IP (TBD) |
| Buffer Controller | Medium | 400-500 | 2-3 days | HIGH | None |
| Threshold Detector | Low-Med | 200-300 | 1-2 days | HIGH | None |
| Detection FSM | Medium | 300-400 | 2 days | HIGH | None |
| Config Registers | Low | 200-300 | 1-2 days | HIGH | None |
| WB Interconnect | Medium | 400-600 | 2-3 days | HIGH | None |
| **Total** | - | **~2000-2500** | **12-16 days** | - | - |

---

## Design Alternatives & Trade-offs

### Alternative 1: Use I2C ADC Instead of SPI
**Pros**:
- Can use EF_I2C IP (no custom controller needed)
- Fewer pins (2 vs 4)

**Cons**:
- I2C ADCs typically slower (860 sps vs 100 ksps)
- May not achieve 20 ksps target
- More complex protocol

**Decision**: SPI preferred for speed

---

### Alternative 2: Digital MEMS Microphone with I2S
**Pros**:
- No external ADC needed (microphone has built-in)
- Better SNR
- Digital output (I2S protocol)

**Cons**:
- Requires custom I2S interface (similar effort to SPI)
- Higher cost ($3-5 vs $1.50 + $2.50)
- I2S continuous streaming needs more logic

**Decision**: Analog microphone + SPI ADC preferred for MVP (lower cost, simpler)

---

### Alternative 3: On-Chip FFT/DSP Processing
**Pros**:
- More sophisticated detection algorithm
- Can identify frequency signatures
- Reduced false alarms

**Cons**:
- Significant area increase (FFT core ~10k gates+)
- Longer development time (+4-6 weeks)
- May not fit in Caravel user project area
- Higher verification effort

**Decision**: Defer to Phase 2 enhancement (keep threshold detection for MVP)

---

### Alternative 4: External MCU for All Processing
**Pros**:
- Minimal on-chip logic (just SPI master + UART)
- Flexible algorithm updates
- Use existing MCU ADC

**Cons**:
- Defeats purpose of custom ASIC
- Higher system power
- More expensive BOM

**Decision**: Not aligned with project goals

---

## Cost-Optimized BOM

### Minimum Viable System

| Component | Part Number | Qty | Unit Cost | Total |
|-----------|-------------|-----|-----------|-------|
| ADC | MCP3202 | 1 | $2.50 | $2.50 |
| Microphone | Electret ECM | 1 | $1.50 | $1.50 |
| Op-Amp | MCP6022 (dual) | 1 | $0.60 | $0.60 |
| Voltage Regulator | MCP1700-33 | 1 | $0.40 | $0.40 |
| Resistors | Various | 10 | $0.02 | $0.20 |
| Capacitors | Various | 10 | $0.05 | $0.50 |
| PCB (2-layer) | Custom | 1 | $5.00 | $5.00 |
| **Total** | - | - | - | **$10.70** |

**Volume Pricing** (1000 units):
- ADC: $1.50
- Microphone: $1.00
- Other: Similar
- PCB: $2.00
- **Total @ Volume**: ~$6.50 per system

---

## Recommendations

### Immediate Actions
1. ✅ **Review CF_SPI IP documentation** to determine if suitable for ADC interface
   ```bash
   query_docs_db("CF_SPI IP continuous streaming autonomous mode")
   ```

2. 🔲 **Procure off-chip components** for prototyping:
   - MCP3202 ADC
   - Electret microphone
   - MAX4466 or MCP6022
   - Passives and PCB

3. 🔲 **Begin custom RTL development**:
   - Start with SPI master (or wait for CF_SPI evaluation)
   - Parallel development: Config registers, buffer controller

### Design Validation
1. 🔲 **Breadboard analog front-end** to validate:
   - Microphone biasing
   - Amplifier gain
   - Filter frequency response
   - ADC interface

2. 🔲 **Capture real glass break audio** for algorithm tuning:
   - Record samples with external ADC
   - Analyze frequency spectrum
   - Determine optimal thresholds
   - Test false alarm rate

### Future Enhancements
1. **Phase 2**: Add simple FIR filter for frequency selectivity
2. **Phase 3**: Implement FFT core for frequency analysis
3. **Phase 4**: ML-based detection (TinyML on managementSoC)

---

## Conclusion

The Glass Break Sensing SoC requires:

**✅ Available Pre-Installed IPs** (5):
- CF_UART, (CF_SPI - TBD), EF_GPIO8, EF_TMR32, CF_SRAM_1024x32
- All verified and ready for integration

**❌ Off-Chip Components Required** (3 critical):
- ADC: MCP3202 ($2.50)
- Microphone + Amplifier: ECM + MAX4466 ($3.50)
- Passives + PCB (~$7)
- **Total off-chip cost**: ~$13 per system (~$7 at volume)

**🔧 Custom RTL Required** (6 modules):
- SPI Master (maybe), Buffer Controller, Threshold Detector, FSM, Registers, Interconnect
- **Total development effort**: 12-16 days

**Overall Assessment**: ✅ **Feasible and Practical**
- All critical gaps have viable solutions
- Off-chip cost is reasonable for a sensing application
- Custom RTL effort is manageable (2-3 weeks)
- Design fits within Caravel constraints

---

**Document Version**: 1.0  
**Last Updated**: 2025-11-05  
**Author**: NativeChips Agent  
**Status**: Complete - Ready for review and implementation
