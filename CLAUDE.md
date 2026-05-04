# signalk-ecowitt-gw2000 — Development Guide

A Signal K plugin for the **Ecowitt GW2000B** Wi-Fi gateway paired with a **WS90** all-in-one weather station.

## Project Overview

- **Type**: Signal K Node Server Plugin
- **Language**: Node.js (JavaScript)
- **Scope**: Polls GW2000B local HTTP API, publishes weather data to Signal K
- **Key Features**:
  - Temperature, humidity, wind speed/direction, solar radiation, UV index
  - Piezo rain sensor (event, hourly, daily, weekly, monthly, yearly totals)
  - Barometric pressure (absolute and sea-level)
  - WS90 battery/capacitor voltage monitoring
  - No cloud account required; local network polling only

## File Structure

```
signalk-ecowitt-gw2000/
├── index.js              # Main plugin code (all-in-one module)
├── package.json          # Dependencies and metadata
├── README.md             # User-facing documentation
├── INSTALL.md            # Installation instructions
├── CLAUDE.md             # This file — development guide
├── test/                 # Jest test suite
│   ├── unit-conversions.test.js      # Unit conversion functions (22 tests)
│   ├── parse-val-and-unit.test.js    # Value/unit parsing (23 tests)
│   ├── data-mapping.test.js          # Data structure mapping (23 tests)
│   ├── fetch-livedata.test.js        # HTTP error handling (6 tests)
│   └── README.md                     # Test guide
└── .gitignore
```

## Quick Start

### Development Setup

```bash
# Install dependencies
npm install

# Run tests
npm test              # Run all tests once
npm run test:watch   # Watch mode (re-run on file changes)
npm run test:coverage # Generate coverage report
```

### Test Suite (74 tests, all passing)

| Test File | Tests | Coverage |
|---|---|---|
| **unit-conversions.test.js** | 22 | `tempToK`, `windToMs`, `pressureToPa`, `rainToM`, `rainRateToMs` |
| **parse-val-and-unit.test.js** | 23 | Value extraction, unit parsing, edge cases, malformed input |
| **data-mapping.test.js** | 23 | `buildCommonMap()`, `buildPiezoMap()`, NaN filtering, extras fields |
| **fetch-livedata.test.js** | 6 | HTTP mocking, timeouts, error scenarios (setup for export) |

## Architecture

### Plugin Structure

The plugin exports two things:

```javascript
module.exports = function (app) { /* plugin factory */ };
module.exports.utils = { /* utility functions for testing */ };
```

### Core Functions

**Utility Functions** (exported for testing):
- `tempToK(val, unit)` — Convert °C/°F to Kelvin
- `windToMs(val, unit)` — Convert knots/km/h/mph to m/s
- `pressureToPa(val, unit)` — Convert hPa/kPa/inHg/mmHg to Pa
- `rainToM(val, unit)` — Convert mm/inches to meters
- `rainRateToMs(val, unit)` — Convert mm/hr/in/hr to m/s
- `parseValAndUnit(item)` — Extract numeric value and unit from API response
- `buildCommonMap(data)` — Map `common_list` array to ID→value lookup
- `buildPiezoMap(data)` — Map `piezoRain` array to ID→value lookup

**Internal Functions**:
- `fetchLiveData(options)` — HTTP GET to `/get_livedata_info`, handles timeouts/errors
- `parseAndPublish(data, options)` — Parse API response, convert units, publish to Signal K
- `poll(options)` — Main polling loop; calls `fetchLiveData()` then `parseAndPublish()`

### Data Flow

```
GW2000B API (/get_livedata_info)
  ↓
fetchLiveData() [HTTP GET, timeout handling, error checking]
  ↓
parseAndPublish() [data parsing]
  ↓
buildCommonMap() / buildPiezoMap() [structure mapping]
  ↓
Unit conversion functions [tempToK, windToMs, etc.]
  ↓
Signal K updates via app.handleMessage()
```

## Configuration

Plugin settings (exposed in Signal K admin UI):

| Setting | Type | Default | Range |
|---|---|---|---|
| `host` | string | `192.168.0.35` | GW2000B LAN IP |
| `port` | number | `80` | Gateway HTTP port |
| `pollInterval` | number | `60` | seconds, 10–300 |
| `windAsTrue` | boolean | `false` | true=true wind, false=apparent |

## API Response Format

The GW2000B `/get_livedata_info` endpoint returns JSON with three main sections:

### `common_list` (array)
Sensor readings as `{id, val, [unit]}`:
- `0x02`: outdoor temperature
- `0x03`: dew point
- `0x07`: outdoor humidity
- `0x0A`: wind direction
- `0x0B`: wind speed
- `0x0C`: wind gust
- `0x19`: max daily gust
- `0x15`: solar radiation
- `0x16`: UV index
- `0x17`: lightning strike count
- `0x6D`: lightning distance (km, converted to m)

### `piezoRain` (array or object)
Rain sensor data as `{id, val}` plus extra scalar fields:
- `srain_piezo`: event total
- `0x0D`: daily total
- `0x0E`: rain rate
- `0x7C`: hourly total
- `0x10`: weekly total
- `0x11`: monthly total
- `0x12`: yearly total
- `ws90cap_volt`: WS90 solar capacitor voltage
- `voltage`: WS90 backup battery voltage
- `battery`: battery status (numeric)

### `wh25` (array or object)
GW2000B built-in sensor:
- `intemp`: indoor temperature (°C)
- `inhumi`: indoor humidity (%)
- `abs`: absolute pressure (hPa)
- `rel`: relative pressure (hPa)

## Code Quality Improvements

### Recent Fixes (v0.1.1)

1. **Test Suite**: Added 74 Jest tests covering all utility functions and edge cases
   - Unit conversions: different input units, edge cases, malformed input
   - Value/unit parsing: separate fields, embedded units, NaN handling
   - Data mapping: array/object formats, extra fields, incomplete entries

2. **Error Handling**: Fixed NaN propagation in WH25 pressure parsing
   - Added `isNaN()` guards before publishing absolute and sea-level pressure
   - Prevents invalid values from reaching Signal K

3. **Configuration**: Aligned poll interval defaults
   - Schema default: 60s (was 60s)
   - Runtime fallback: 60s (was 16s)
   - Now consistent across all configuration paths

4. **Testability**: Exported utility functions
   - Functions now available in `module.exports.utils` for unit testing
   - Enables future test coverage expansion (HTTP layer tests pending `fetchLiveData()` export)

5. **Documentation**: Clarified lightning distance conversion
   - Added explicit comment explaining km→m conversion
   - Makes Signal K unit standard compliance clear

## Testing Guidelines

### Running Tests

```bash
npm test                 # All tests
npm run test:watch      # Watch mode
npm run test:coverage   # Coverage (shows missing coverage)
```

### Writing New Tests

Tests are located in `test/` with pattern `*.test.js`.

**Add to existing file** when testing:
- New unit conversion (add to `unit-conversions.test.js`)
- New sensor format (add to `data-mapping.test.js`)
- Edge case bug fix (add regression test)

**Create new file** when testing:
- New major feature (e.g., new data source)
- Complex integration scenario

### Test Structure

Each test file covers one core concept and uses descriptive `describe` blocks:

```javascript
describe('Module or function name', () => {
  describe('specific behavior', () => {
    test('specific assertion', () => {
      // arrange
      // act
      // assert
    });
  });
});
```

## Known Limitations

1. **HTTP Error Handling**: `fetchLiveData()` is not exported; full unit testing requires refactoring
2. **No Retry Logic**: Failed polls set error status; next poll must succeed to recover
3. **No Caching**: All data is fetched on interval; no delta updates
4. **Single File**: Entire plugin is in `index.js`; consider splitting if it grows

## Future Enhancements

- [ ] Export `fetchLiveData()` for full HTTP error testing
- [ ] Add exponential backoff retry logic for failed polls
- [ ] Support for additional WH sensors (WH57 lightning, WH40 rain gauge)
- [ ] Optional Redis caching for high-frequency consumers
- [ ] ConfigUI schema (Signal K admin panel integration)

## Troubleshooting

### Tests Fail

```bash
npm install                # Reinstall dependencies
npm test -- --verbose     # See detailed output
```

### Plugin Doesn't Connect

Check logs in Signal K:
```bash
# From Signal K server
tail -f /home/pi/.signalk/appdata/signalk-ecowitt-gw2000/logs
```

Verify GW2000B is reachable:
```bash
curl http://192.168.0.35/get_livedata_info | jq .
```

### Coverage Gaps

Generate and review coverage:
```bash
npm run test:coverage
open coverage/lcov-report/index.html
```

## Resources

- **Signal K Docs**: [signalk.org](https://signalk.org)
- **GW2000B API**: Documented in code comments (lines 9–38)
- **Jest Docs**: [jestjs.io](https://jestjs.io)
- **Node.js HTTP**: [nodejs.org/api/http](https://nodejs.org/api/http.html)

## License

MIT
