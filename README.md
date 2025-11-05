# Glass Break Sensing SoC - Caravel Integration

## Initial User Prompt
**"Create custom soc based on caravel for glass break sensing and identify ip gaps and off chip options"**

## Project Overview
This project implements a custom System-on-Chip (SoC) for glass break detection, integrated with the Efabless Caravel harness for the Skywater 130nm process. The system detects acoustic signatures characteristic of breaking glass and provides configurable alert mechanisms.

## Glass Break Detection Background

### Detection Principles
Glass break sensors detect the unique acoustic signature of breaking glass:
- **Frequency Range**: 100Hz to 15kHz
- **Characteristic Peak**: Sharp high-frequency component (5-7kHz) followed by lower frequency resonance
- **Duration**: Typical event lasts 2-10ms
- **Pattern**: Initial impact followed by cascading fragments

### Detection Approaches
1. **Threshold Detection**: Simple amplitude threshold on filtered signal
2. **Frequency Analysis**: FFT/DFT to identify characteristic frequency components
3. **Pattern Matching**: Time-domain pattern recognition with templates
4. **Dual-Technology**: Combines acoustic with shock/vibration sensing

## System Architecture

### High-Level Block Diagram
```
┌─────────────────────────────────────────────────────────────────┐
│                    Off-Chip Components                           │
├─────────────────────────────────────────────────────────────────┤
│  Microphone → Amplifier → Bandpass Filter → ADC (SPI/I2C)      │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────┴────────────────────────────────────┐
│                    Caravel SoC (On-Chip)                         │
├─────────────────────────────────────────────────────────────────┤
│  ┌─────────────────────────────────────────────────────────┐   │
│  │           Glass Break Detection Core                     │   │
│  │  ┌────────────┐  ┌──────────────┐  ┌────────────────┐  │   │
│  │  │ SPI Master │→ │ Sample Buffer│→ │ Signal         │  │   │
│  │  │ (ADC IF)   │  │ (SRAM)       │  │ Processing     │  │   │
│  │  └────────────┘  └──────────────┘  └────────┬───────┘  │   │
│  │                                              ↓           │   │
│  │  ┌────────────┐  ┌──────────────┐  ┌────────────────┐  │   │
│  │  │ Detection  │← │ Config Regs  │  │ Alert/Status   │  │   │
│  │  │ Engine     │  │              │  │ Logic          │  │   │
│  │  └────────────┘  └──────────────┘  └────────┬───────┘  │   │
│  └─────────────────────────────────────────────┼──────────┘   │
│                                                 │               │
│  ┌──────────────────────────────────────────────┼──────────┐   │
│  │           Wishbone Interconnect             │          │   │
│  └──────────────────────────────────────────────┼──────────┘   │
│                                                 ↓               │
│  ┌────────────┐  ┌──────────┐  ┌──────────┐  ┌─────────┐     │
│  │ UART       │  │ GPIO     │  │ Timer    │  │ Status  │     │
│  │ (Debug/    │  │ (Alert)  │  │ (Timing) │  │ LEDs    │     │
│  │  Config)   │  │          │  │          │  │         │     │
│  └────────────┘  └──────────┘  └──────────┘  └─────────┘     │
└─────────────────────────────────────────────────────────────────┘
```

## IP Analysis

### Available Pre-Installed IPs
Based on the NativeChips IP library, the following verified IPs are available:

#### Communication Interfaces
- **CF_UART v2.0.1** ✅ - For debug, configuration, and data output
- **CF_SPI v2.0.1** ✅ - For ADC interface (SPI-based ADC preferred)
- **EF_I2C v1.1.0** ✅ - Alternative ADC interface or I2C-based configuration

#### Memory
- **CF_SRAM_1024x32 v1.2.0** ✅ - 4KB SRAM for audio sample buffering
- **CF_SRAM_4096x32 v1.0.2** ✅ - 16KB SRAM for larger buffers (if needed)

#### Timers and Control
- **EF_TMR32 v1.1.0** ✅ - 32-bit timer for sampling control and timeouts
- **EF_GPIO8 v1.1.0** ✅ - GPIO for alert outputs, status LEDs, control signals

#### Optional
- **EF_PWM32 v1.0.0** - Could be used for audio output (testing) or LED dimming
- **EF_WDT32 v1.1.0** - Watchdog timer for system reliability

### IP Gaps (Require Off-Chip or Custom RTL)

#### Critical Gaps - Require Off-Chip Components

1. **Analog-to-Digital Converter (ADC)**
   - **Gap**: No ADC IP available in library
   - **Off-Chip Solution**: External ADC with digital interface
   - **Recommended Options**:
     - **SPI ADCs**: MCP3201/3202 (12-bit, 100ksps), TLV1572 (10-bit, 1.25Msps)
     - **I2C ADCs**: ADS1115 (16-bit, 860sps), MCP3421 (18-bit, 240sps)
     - **Selection Criteria**: 
       - Sample rate: 10-20 ksps minimum for glass break detection
       - Resolution: 10-12 bits adequate for acoustic detection
       - SPI preferred for higher throughput

2. **Analog Front-End (AFE)**
   - **Gap**: No analog signal conditioning
   - **Off-Chip Solution**: External analog circuit
   - **Required Components**:
     - Electret microphone with bias circuit
     - Pre-amplifier (gain: 40-60dB)
     - Bandpass filter (100Hz - 15kHz)
     - Anti-aliasing filter (before ADC)
   - **Recommended ICs**: 
     - MAX4466 (microphone amplifier with AGC)
     - MCP6022 (dual op-amp for filtering)
     - Discrete RC filters

3. **Microphone/Acoustic Sensor**
   - **Gap**: No on-chip acoustic sensor
   - **Off-Chip Solution**: External microphone
   - **Options**:
     - Electret microphone (analog output) - Cost effective
     - MEMS microphone with analog output (better SNR)
     - MEMS I2S digital microphone (requires custom I2S interface)

#### Design Gaps - Require Custom RTL

1. **Digital Signal Processing Core**
   - **Gap**: No FFT, FIR, or DSP IP available
   - **Custom RTL Required**: 
     - Simple threshold detector (magnitude comparison)
     - Optional: Basic IIR/FIR digital filters
     - Optional: Simplified frequency analysis (DFT at specific bins)
   - **Alternative**: Send raw samples via UART to external processor

2. **Sample Buffer Controller**
   - **Gap**: Need custom logic to interface SRAM with data acquisition
   - **Custom RTL Required**:
     - Circular buffer management
     - DMA-like transfer from ADC to SRAM
     - Trigger/capture logic

3. **Detection Engine**
   - **Gap**: Application-specific detection logic
   - **Custom RTL Required**:
     - Threshold comparator
     - Event counter/validator (reduce false alarms)
     - State machine for detection sequence
     - Configurable sensitivity

4. **Wishbone Interconnect**
   - **Gap**: Multi-slave WB interconnect for IP integration
   - **Custom RTL Required**:
     - WB crossbar or shared bus arbiter
     - Address decoder
     - Peripheral integration

## System Design Approach

### Option 1: Minimal On-Chip Processing (Recommended for Caravel)
**Rationale**: Leverages available IPs, minimal custom RTL, fits within Caravel area constraints

**On-Chip Components**:
- SPI master to interface external ADC
- SRAM for sample buffering (1024x32)
- Simple threshold detection logic
- UART for configuration and data streaming
- GPIO for alert output
- Timer for sampling control

**Processing Strategy**:
- Continuous sampling via SPI ADC
- Store samples in circular buffer
- Apply simple threshold detection (configurable)
- On detection: Assert GPIO alert, send samples via UART
- Optional: Stream continuous data to external processor

**Pros**:
- Small area footprint
- Lower design complexity
- Easier verification
- Flexible (processing can be updated in external firmware)

**Cons**:
- Requires external MCU/processor for advanced processing
- Higher system power (external processor)

### Option 2: Advanced On-Chip Processing
**Rationale**: More autonomous detection, reduced external dependencies

**Additional On-Chip Components**:
- Custom FFT/DFT core (resource intensive)
- Multi-stage digital filters
- Pattern matching engine
- Larger SRAM (4096x32)

**Processing Strategy**:
- Windowed FFT analysis
- Frequency band energy detection
- Multi-criteria detection algorithm
- Full on-chip decision making

**Pros**:
- Autonomous operation
- Reduced external component count
- Lower system latency

**Cons**:
- Significant area increase
- Higher design complexity
- May not fit in Caravel user project area
- Longer design/verification cycle

### Recommended Approach: **Option 1 with Enhancement Path**
Start with minimal processing, design modularity to allow future enhancement.

## Off-Chip Component Recommendations

### Minimum System Configuration

1. **Microphone Circuit**
   - Electret microphone (e.g., CUI CMC-4015-42316)
   - Bias resistor: 2.2kΩ
   - DC blocking capacitor: 10µF

2. **Pre-Amplifier & Filter**
   - IC: MAX4466 or MCP6022
   - Gain: 46dB (configurable via resistor)
   - Bandpass: 100Hz - 15kHz (RC passive or active)

3. **ADC**
   - IC: MCP3202 (12-bit, dual channel, SPI)
   - Sample rate: 20-50 ksps (configurable)
   - Reference voltage: 3.3V or internal

4. **Power Supply**
   - 3.3V regulator (LDO) for analog section
   - Separate analog and digital grounds

5. **Optional: External MCU (for Option 1)**
   - ESP32 or STM32 for advanced processing
   - Connects via UART for samples/configuration

### System Interface Pinout

```
Caravel GPIO Allocation (38 available):
- SPI_ADC_CLK   : GPIO[0]
- SPI_ADC_MOSI  : GPIO[1]
- SPI_ADC_MISO  : GPIO[2]
- SPI_ADC_CS    : GPIO[3]
- UART_TX       : GPIO[4]
- UART_RX       : GPIO[5]
- GPIO_ALERT    : GPIO[6]  (Detection output)
- GPIO_STATUS   : GPIO[7]  (Activity LED)
- GPIO_ENABLE   : GPIO[8]  (Enable detection)
- Reserved      : GPIO[9-37] (Future expansion)
```

## Project Structure
```
nc-6934/
├── glass_break_soc/              # Main project directory
│   ├── docs/                     # Documentation
│   │   ├── architecture.md       # Detailed architecture
│   │   ├── address_map.md        # Memory map
│   │   └── interface_specs.md    # Interface specifications
│   ├── ip/                       # IP integration
│   │   └── link_IPs.json         # IPM linker configuration
│   ├── rtl/                      # RTL source files
│   │   ├── core/                 # Core detection logic
│   │   ├── peripherals/          # Peripheral wrappers
│   │   ├── interconnect/         # Wishbone bus
│   │   └── top/                  # Top-level integration
│   ├── tb/                       # Testbenches (cocotb)
│   ├── openlane/                 # OpenLane configurations
│   ├── verilog/                  # Caravel integration
│   │   ├── rtl/                  # RTL for Caravel
│   │   └── gl/                   # Gate-level netlists
│   ├── gds/                      # Final GDS files
│   ├── lef/                      # LEF files
│   └── lib/                      # Liberty files
└── README.md                     # This file
```

## Development Roadmap

### Phase 1: Architecture & Planning ✓
- [x] Requirements analysis
- [x] IP gap identification
- [x] Off-chip component selection
- [ ] Detailed architecture documentation
- [ ] Address map definition

### Phase 2: RTL Development
- [ ] Custom module design (buffer controller, detector)
- [ ] IP integration using ipm_linker
- [ ] Wishbone interconnect
- [ ] Top-level integration
- [ ] Lint and review

### Phase 3: Verification
- [ ] Module-level testbenches (cocotb)
- [ ] Integration testing
- [ ] Caravel-cocotb verification
- [ ] Coverage analysis

### Phase 4: Caravel Integration
- [ ] Wishbone wrapper creation
- [ ] user_project_wrapper integration
- [ ] OpenLane hardening (macro)
- [ ] OpenLane hardening (wrapper)
- [ ] Final verification

### Phase 5: Documentation & Delivery
- [ ] Complete documentation
- [ ] Firmware examples
- [ ] Hardware integration guide
- [ ] Test results and metrics

## Next Steps
1. Review and approve architecture
2. Create detailed design specifications
3. Begin RTL development for custom modules
4. Setup IP linking and integration

## Key Design Decisions Pending
- [ ] Sample rate and buffer size
- [ ] Detection algorithm specifics (thresholds, timing)
- [ ] UART baud rate and protocol
- [ ] Power management requirements
- [ ] Test and debug features

---

**Status**: Architecture and planning phase
**Last Updated**: 2025-11-05