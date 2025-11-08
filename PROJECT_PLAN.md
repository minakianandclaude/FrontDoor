# Smart Deadbolt Controller Project Plan

## Project Overview
Repurpose an existing electric deadbolt with numerical keypad by replacing the controller board with a custom ESP32-based solution. The original lock mechanism cannot be reprogrammed, so we'll create a new controller that adds WiFi, Bluetooth, and additional unlock methods while reusing existing mechanical components where possible.

## Hardware Components

### Power System
- **Primary Power**: Rechargeable LiPo 3.7V 7.4Wh battery (swappable)
- **Motor Power**: Potentially AA batteries (TBD based on existing motor requirements)
- **Power Management**: Buck/boost converter if needed for ESP32 (runs on 3.3V)

### Microcontroller
- **ESP32-WROOM or ESP32-S3**
  - Built-in WiFi + Bluetooth
  - Dual-core processor
  - Deep sleep capability for battery conservation
  - Multiple GPIO pins for sensors/controls

### Motor Control
- **Motor Driver**: Reuse existing if not integrated into old board, otherwise acquire H-bridge (L298N or DRV8871)
- **Motor**: Reuse existing deadbolt motor

### Sensors & Feedback
- **Lock Position Detection**: Reuse existing mechanism from original lock
- **Status LED**: Prototype phase only
- **Pushbutton**: Prototype phase only (for testing lock/unlock)

### User Interaction (Production)
- **Manual Operation**: Existing deadbolt knob (inside)
- **Existing Keypad**: May or may not be reusable (TBD after teardown)

## Phase 1: MVP Features

### Core Functionality
- [x] WiFi connectivity with mobile app control
- [x] Bluetooth unlock capability
- [x] Activity logging (who unlocked, when)

### Prototype-Only Features
- [ ] Status LED for debugging
- [ ] Pushbutton for manual lock/unlock testing
- [ ] Serial logging for development

### Bluetooth & iPhone Compatibility
**Question: Can we do Bluetooth on iPhone without an official App Store app?**

**Answer: Yes, several options:**
1. **Web Bluetooth** - Limited iOS support, works in Bluefy browser
2. **Home Assistant Companion App** - Use ESPHome integration
3. **BLE GATT Services** - Generic BLE apps like "nRF Connect" or "LightBlue"
4. **Progressive Web App (PWA)** - Install web app to home screen
5. **Shortcuts + BLE** - iOS Shortcuts can trigger BLE actions

**Recommended Approach**: Build a web app that works on all platforms, optionally integrate with Home Assistant for power users.

## Phase 2: Enhanced Features (Future)
*Priorities will be revisited after Phase 1 completion*

- [ ] NFC/RFID unlock
- [ ] Auto-lock timer
- [ ] Door position sensor (is door actually closed?)
- [ ] Push notifications
- [ ] Home Assistant integration
- [ ] Temporary access codes for guests
- [ ] Battery level monitoring & alerts

## Phase 3: Advanced Features (Future)
*Priorities will be revisited after Phase 2 completion*

- [ ] Geofencing (auto-unlock based on location)
- [ ] Voice control (Alexa/Google Home)
- [ ] Biometric unlock (fingerprint)
- [ ] Camera integration (photo on unlock)
- [ ] Tamper detection
- [ ] Vacation/away mode
- [ ] Multi-user management system

## Technical Considerations

### Power Management
- **Battery Life Goal**: Minimize WiFi usage, use deep sleep between operations
- **Swappable Design**: Quick battery replacement without tools
- **Low Battery Warning**: Alert before complete drain
- **Graceful Shutdown**: Save state before battery dies

### Security
- **Encrypted Communications**: TLS for WiFi, encrypted BLE
- **Local-First Design**: Core unlock functions work without internet
- **Activity Logging**: Local storage with optional cloud backup
- **Physical Security**: Controller accessible only from inside
- **Fail-Safe Mode**: Determine locked vs unlocked behavior on power loss

### Motor Control
- **Position Feedback**: Use existing lock/unlock detection
- **Current Sensing**: Detect if motor stalls (door misaligned)
- **Bidirectional Control**: Lock and unlock commands
- **Speed Control**: PWM if needed for smooth operation

### Network Architecture
- **WiFi**: Direct connection to home network
- **Local API**: RESTful or MQTT for commands
- **Web Interface**: Responsive design for phone/tablet/desktop
- **OTA Updates**: Firmware updates without physical access
- **Fallback Mode**: Continue operation if WiFi unavailable

## Development Phases

### Phase 0: Research & Planning ✓
- [x] Define activation methods
- [x] Select microcontroller
- [x] Identify features and limitations
- [x] Create project plan document

### Phase 1A: Hardware Assessment
- [ ] Teardown existing lock
- [ ] Document existing wiring
- [ ] Test motor voltage/current requirements
- [ ] Verify position detection mechanism
- [ ] Check if keypad can be reused
- [ ] Assess motor driver reusability

### Phase 1B: Prototype Hardware
- [ ] Create circuit schematic
- [ ] Build breadboard prototype
- [ ] Test motor control
- [ ] Verify position detection
- [ ] Test power consumption
- [ ] Validate battery life estimates

### Phase 1C: Firmware Development
- [ ] Set up development environment (PlatformIO or Arduino IDE)
- [ ] Implement motor control
- [ ] Add position detection
- [ ] Create WiFi AP for initial setup
- [ ] Build basic web interface
- [ ] Implement Bluetooth unlock
- [ ] Add activity logging
- [ ] Create OTA update mechanism

### Phase 1D: Integration & Testing
- [ ] Install in actual door
- [ ] Test all unlock methods
- [ ] Verify battery life
- [ ] Security testing
- [ ] User acceptance testing
- [ ] Documentation

### Phase 1E: Production Hardware
- [ ] Design custom PCB (optional)
- [ ] Create enclosure
- [ ] Finalize wiring
- [ ] Remove prototype-only features (LED, pushbutton)
- [ ] Final installation

## Bill of Materials (Preliminary)

### Confirmed Components
- ESP32-WROOM or ESP32-S3 development board
- 3.7V LiPo battery, 7.4Wh, with JST connector
- Existing deadbolt motor (reuse)
- Existing lock position sensor (reuse)

### To Be Determined
- Motor driver (may reuse existing)
- Voltage regulators/converters
- AA battery holder (if needed for motor)
- Prototype components (breadboard, pushbutton, LED, resistors)
- Enclosure/housing
- Wiring/connectors

### Optional Components
- PN532 NFC/RFID reader (Phase 2)
- Reed switch for door position (Phase 2)
- Piezo buzzer for audio feedback

## Risk Assessment

### High Priority Risks
1. **Battery life insufficient** - Mitigation: Optimize deep sleep, test extensively
2. **Motor driver not reusable** - Mitigation: Budget for H-bridge module
3. **Existing mechanism damaged during removal** - Mitigation: Careful teardown, photo documentation
4. **Lock yourself out during testing** - Mitigation: Always keep traditional key accessible

### Medium Priority Risks
1. **WiFi reliability issues** - Mitigation: Offline mode for core functions
2. **Position sensor incompatible** - Mitigation: Add hall effect or limit switches
3. **3.7V insufficient for motor** - Mitigation: Use separate AA battery pack
4. **Security vulnerabilities** - Mitigation: Code review, penetration testing

### Low Priority Risks
1. **User interface too complex** - Mitigation: User testing, iterative design
2. **Bluetooth range insufficient** - Mitigation: External antenna option
3. **Weather/humidity damage** - Mitigation: Conformal coating, sealed enclosure

## Success Criteria

### Phase 1 Complete When:
- [ ] Lock/unlock reliably via WiFi app
- [ ] Lock/unlock reliably via Bluetooth
- [ ] Activity log captures all unlock events
- [ ] Battery lasts minimum 2 weeks under normal use
- [ ] System recovers gracefully from power loss
- [ ] No traditional key required for daily use
- [ ] Family members can use it without instructions

## Open Questions

1. **What is the existing lock model/brand?** - Affects teardown approach
2. **What voltage/current does the motor require?** - Determines power system design
3. **Is the existing keypad matrix-based or has its own controller?** - Affects reusability
4. **Are you comfortable with SMD soldering for custom PCB?** - Determines if we use modules vs custom board
5. **Do you have a home automation system already?** (Home Assistant, etc.) - Affects integration approach
6. **What's your preferred programming environment?** - Arduino IDE, PlatformIO, ESPHome?

## Next Steps

1. **Hardware Assessment**: Teardown existing lock and document findings
2. **BOM Finalization**: Order missing components based on teardown results
3. **Development Environment**: Set up ESP32 toolchain
4. **Create Circuit Diagram**: Design initial schematic
5. **Start Basic Firmware**: Begin with simple motor control test

---

**Document Version**: 1.0
**Last Updated**: 2025-11-08
**Status**: Planning Phase Complete, Ready for Hardware Assessment
