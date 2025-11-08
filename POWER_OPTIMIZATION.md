# Power Optimization Strategy

## Battery Specifications
- **Type**: LiPo rechargeable
- **Voltage**: 3.7V nominal
- **Capacity**: 7.4Wh = 2000mAh
- **Form Factor**: Swappable

## Power Consumption Analysis

### ESP32 Power Modes

| Mode | Current Draw | Battery Life (2000mAh) | Use Case |
|------|-------------|------------------------|----------|
| WiFi Active | 160-260mA | 8-12 hours | Transmitting data |
| WiFi Idle | 80-100mA | 20-25 hours | Connected, listening |
| BT Classic | 100-130mA | 15-20 hours | Legacy Bluetooth |
| BLE Advertising | 20-40mA | 2-4 days | Broadcasting presence |
| BLE Connected | 15-25mA | 3-5 days | Active BLE connection |
| Modem Sleep | 20-30mA | 3-4 days | CPU on, radios off |
| Light Sleep | 0.8-2mA | 40-100 days | Quick wake, RAM retained |
| Deep Sleep | 10-150µA | 1-5 years | Slowest wake, minimal power |

### Motor Power Consumption
- **Peak Current**: TBD (measure during teardown)
- **Duration**: ~2-3 seconds per operation
- **Frequency**: ~10-20 operations per day
- **Estimated Daily Impact**: 5-20mAh (negligible compared to radio)

### Sensor Power
- **Lock Position Sensor**: <1mA (Hall effect or reed switch)
- **Always On**: 24mAh/day
- **Polled Mode**: <1mAh/day (check only when needed)

## Target: 2-4 Week Battery Life

**Daily Budget**: 71-143mAh per day (for 14-28 day life)

This requires aggressive power management. WiFi cannot be on continuously.

## Recommended Strategy: Hybrid BLE + Deep Sleep

### Architecture

```
┌─────────────────────────────────────────────┐
│          Deep Sleep (23h 59m)               │
│          Power: 150µA average               │
│          Daily: 3.6mAh                      │
└─────────────────────────────────────────────┘
                    ▼
        Wake Events (triggered by):
          • GPIO (keypad press)
          • Timer (every 4 hours)
          • External interrupt (future)
                    ▼
┌─────────────────────────────────────────────┐
│         BLE Active (1 minute)               │
│         Power: 30mA                         │
│         Daily: 180mAh (6x per day)          │
└─────────────────────────────────────────────┘
                    ▼
          Command Received?
                 /    \
               No      Yes
                |       |
             Sleep    Execute
                      & Log
                        |
┌───────────────────────────────────────────┐
│      WiFi Sync (user-triggered)           │
│      Power: 200mA for 2-5 minutes         │
│      Frequency: 1-2x per week             │
│      Daily average: ~10mAh                │
└───────────────────────────────────────────┘
```

### Expected Daily Consumption
- Deep sleep: 3.6mAh
- BLE wake cycles (6x/day, 1min each): 3.0mAh
- Motor operations (15x/day, 3sec): 10mAh
- WiFi sync (amortized): 10mAh
- Sensors & misc: 5mAh
- **Total: ~32mAh/day**
- **Battery Life: 62 days** 🎉

### Wake-Up Strategy

**Timer Wake (Every 4 Hours)**
```cpp
esp_sleep_enable_timer_wakeup(4 * 3600 * 1000000ULL); // 4 hours in µs
```

**GPIO Wake (Keypad/Button Press)**
```cpp
esp_sleep_enable_ext0_wakeup(GPIO_KEYPAD, 1); // Wake on HIGH
// or for multiple pins:
esp_sleep_enable_ext1_wakeup(BUTTON_BITMASK, ESP_EXT1_WAKEUP_ANY_HIGH);
```

**Wake Handler**
```cpp
void handleWake() {
  esp_sleep_wakeup_cause_t wakeup_reason;
  wakeup_reason = esp_sleep_get_wakeup_cause();

  switch(wakeup_reason) {
    case ESP_SLEEP_WAKEUP_EXT0:
      // GPIO wakeup - keypad pressed
      handleKeypadInput();
      break;

    case ESP_SLEEP_WAKEUP_TIMER:
      // Scheduled wakeup - check for BLE commands
      startBLE();
      listenForCommands(60000); // 60 second window
      stopBLE();
      break;

    case ESP_SLEEP_WAKEUP_EXT1:
      // Multi-GPIO wakeup
      handleMultipleInputs();
      break;

    default:
      // First boot
      initializeSystem();
      break;
  }
}
```

## Alternative Strategies

### Option A: BLE-Only Always-On
**Power**: 30mA continuous BLE
**Battery Life**: 2.7 days
**Pros**: Instant response to phone
**Cons**: Frequent charging required

**Verdict**: ❌ Not practical for daily use

### Option B: WiFi Sleep Mode
**Power**: 15-30mA with DTIM optimization
**Battery Life**: 3-5 days
**Pros**: Always connected, remote access
**Cons**: Still requires frequent charging

**Verdict**: ⚠️ Only if wall power backup available

### Option C: Dual MCU (BLE Always + WiFi On-Demand)
**Architecture**:
- Nordic nRF52 handles BLE (5-10mA)
- ESP32 powers on only for WiFi
- nRF52 wakes ESP32 via GPIO

**Power**: 8-12mA average
**Battery Life**: 7-10 days
**Pros**: Great UX, good battery life
**Cons**: More complex hardware

**Verdict**: ✅ Consider for v2.0 hardware

### Option D: Matter/Thread with Hub
**Power**: 10-20mA average
**Battery Life**: 4-8 days
**Requires**: Matter-compatible hub (HomePod, Echo 4th gen, etc.)
**Pros**: Standard protocol, efficient mesh
**Cons**: Requires hub purchase, ESP32-H2 needed

**Verdict**: ✅ Good option if user has compatible hub

## Power Optimization Techniques

### 1. WiFi Optimizations
```cpp
// Use WiFi power save mode
WiFi.setSleep(WIFI_PS_MAX_MODEM);

// Disconnect when not needed
WiFi.disconnect(true);
WiFi.mode(WIFI_OFF);

// Reduce TX power if router is close
WiFi.setTxPower(WIFI_POWER_11dBm); // vs default 20dBm
```

### 2. BLE Optimizations
```cpp
// Increase advertising interval
BLEAdvertising *pAdvertising = BLEDevice::getAdvertising();
pAdvertising->setMinInterval(800); // 500ms (default 100ms)
pAdvertising->setMaxInterval(1600); // 1000ms

// Reduce TX power
esp_ble_tx_power_set(ESP_BLE_PWR_TYPE_ADV, ESP_PWR_LVL_N12); // -12dBm
```

### 3. CPU Frequency Scaling
```cpp
// Reduce CPU speed when not busy
setCpuFrequencyMhz(80); // vs default 240MHz
// Saves ~30% power for non-radio operations
```

### 4. Peripheral Management
```cpp
// Disable unused peripherals before sleep
adc_power_off();
esp_wifi_stop();
esp_bt_controller_disable();
```

### 5. RTC Memory for Fast Wake
```cpp
// Store state in RTC memory (survives deep sleep)
RTC_DATA_ATTR int bootCount = 0;
RTC_DATA_ATTR bool doorLocked = true;

// Faster than reading from flash on wake
```

## Battery Monitoring

### Voltage Monitoring
```cpp
// Monitor battery voltage via ADC
const int batteryPin = 34; // ADC pin
const float voltageDivider = 2.0; // If using voltage divider

float getBatteryVoltage() {
  int raw = analogRead(batteryPin);
  return (raw / 4095.0) * 3.3 * voltageDivider;
}

float getBatteryPercent() {
  float voltage = getBatteryVoltage();
  // LiPo: 4.2V = 100%, 3.7V = 50%, 3.0V = 0%
  if (voltage >= 4.2) return 100;
  if (voltage <= 3.0) return 0;
  return (voltage - 3.0) / 1.2 * 100;
}
```

### Low Battery Warnings
- **Alert at 20%**: Send notification via BLE/WiFi
- **Critical at 10%**: Reduce wake frequency
- **Shutdown at 5%**: Prevent deep discharge damage

## Testing & Measurement

### Power Profiling Tools
1. **USB Power Meter**: Measure actual current draw
2. **Logic Analyzer**: Verify sleep/wake cycles
3. **Serial Logging**: Track power state changes
4. **Battery Life Test**: Full discharge cycle measurement

### Test Scenarios
1. **Idle Test**: No activity for 24 hours
2. **Normal Use**: 15 unlock operations per day
3. **Heavy Use**: 50 unlock operations per day
4. **WiFi Sync Test**: Measure sync operation cost
5. **Wake Frequency Test**: 1hr vs 4hr vs 8hr intervals

### Success Metrics
- [ ] Deep sleep current < 200µA
- [ ] BLE active current < 40mA
- [ ] WiFi connect time < 5 seconds
- [ ] Full battery lasts minimum 14 days (normal use)
- [ ] Battery warning at >48 hours remaining
- [ ] Wake from deep sleep < 500ms

## Implementation Phases

### Phase 1: Baseline Measurement
- [ ] Measure current draw in all modes
- [ ] Test deep sleep current
- [ ] Verify wake sources work correctly
- [ ] Profile typical operation sequence

### Phase 2: Optimization
- [ ] Implement deep sleep mode
- [ ] Add battery voltage monitoring
- [ ] Optimize BLE advertising parameters
- [ ] Minimize wake time

### Phase 3: Field Testing
- [ ] 7-day battery life test
- [ ] Measure actual vs predicted usage
- [ ] Optimize wake intervals based on usage patterns
- [ ] Tune power parameters

### Phase 4: Adaptive Power Management
- [ ] Learn usage patterns (future enhancement)
- [ ] Adjust wake frequency based on time of day
- [ ] Predictive wake before typical unlock times
- [ ] Vacation mode (reduce wake frequency when away)

## Fallback Strategy

If battery life is insufficient with current approach:

1. **Add AA battery pack for motor** - Separate power domains
2. **Increase LiPo capacity** - Use 5000mAh battery instead
3. **Add solar panel** - Small 5V panel + charging circuit
4. **Switch to wall power primary** - Battery as backup only
5. **Implement dual-MCU architecture** - Hardware revision

## Estimated Costs

| Optimization | Battery Life Gain | Cost | Complexity |
|--------------|-------------------|------|------------|
| Deep sleep mode | +3000% | $0 | Low |
| BLE instead of WiFi | +200% | $0 | Low |
| Optimized wake intervals | +100% | $0 | Low |
| Dual MCU | +300% | $15 | High |
| Larger battery (5000mAh) | +150% | $10 | Low |
| Solar panel | Infinite* | $20 | Medium |

*Assuming adequate sunlight

## Recommended Configuration

**For Phase 1 MVP:**
- Deep sleep with 4-hour wake intervals
- GPIO wake on keypad press
- BLE-only for phone unlock
- WiFi on-demand for sync/updates
- Target: 2-4 week battery life

**For Production:**
- Monitor real-world usage patterns
- Adjust wake intervals dynamically
- Consider dual-MCU for v2.0 if needed
- Evaluate Matter/Thread for standardization

---

**Last Updated**: 2025-11-08
**Status**: Strategy defined, pending hardware testing
