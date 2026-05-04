# Test Suite for signalk-ecowitt-gw2000

This directory contains Jest unit tests for the Ecowitt GW2000B Signal K plugin.

## Running Tests

```bash
# Run all tests
npm test

# Run tests in watch mode (re-run on file changes)
npm run test:watch

# Run tests with coverage report
npm run test:coverage
```

## Test Files

### `unit-conversions.test.js`
Tests for unit conversion functions:
- `tempToK()` — Celsius/Fahrenheit to Kelvin
- `windToMs()` — knots/km/h/mph to m/s
- `pressureToPa()` — hPa/kPa/inHg/mmHg to Pa
- `rainToM()` — mm/inches to meters
- `rainRateToMs()` — mm/hr/in/hr to m/s

**Coverage**:
- Correct conversions across different units
- Default unit handling (when unit is missing/empty)
- Edge cases (negative values, zero, scientific notation)
- Case-insensitive unit matching
- Whitespace trimming

### `parse-val-and-unit.test.js`
Tests for `parseValAndUnit()` — the core parser for extracting values and units from API responses.

**Coverage**:
- Separate unit field parsing (`{val: "16.4", unit: "C"}`)
- Embedded unit parsing (`{val: "6.22 knots"}`)
- Mixed formats and edge cases
- Malformed input handling (NaN, missing values, "--")
- Preference for separate unit field over embedded unit

### `data-mapping.test.js`
Tests for `buildCommonMap()` and `buildPiezoMap()` — the functions that build lookup maps from API response arrays.

**Coverage**:
- Array-based API responses
- Object-based API responses (backward compatibility)
- NaN filtering (missing sensors signaled with "--")
- Extra scalar fields (ws90cap_volt, voltage, battery)
- Handling of incomplete entries (missing id/val)
- Empty and missing data structures

### `fetch-livedata.test.js`
Documentation and mocking setup for HTTP layer testing.

**Note**: `fetchLiveData()` is currently internal to the plugin closure. For true unit testing, consider exporting it alongside `utils`:

```javascript
module.exports.fetchLiveData = fetchLiveData;
module.exports.utils = utils;
```

**Test structure prepared for**:
- Valid JSON response parsing
- Non-2xx status code rejection (404, 500, etc.)
- Invalid/truncated JSON handling
- Request timeout handling (5s limit)
- Response size limits (100 KB max)
- Connection errors
- Mid-transfer socket errors

## Coverage Goals

Current focus:
- ✅ Unit conversions (100% coverage)
- ✅ Value/unit parsing (100% coverage)
- ✅ Data mapping (100% coverage)
- 📝 HTTP error handling (mocked, awaiting `fetchLiveData()` export)

## Adding New Tests

1. **New sensor format**: If the API response changes, add test cases to `data-mapping.test.js`
2. **New unit**: Add conversion test to `unit-conversions.test.js`
3. **Edge case bug fix**: Add regression test alongside the fix

## Troubleshooting

### Tests fail with "Cannot find module 'jest'"
Install dependencies:
```bash
npm install
```

### Tests timeout
Increase Jest timeout for integration tests (if added):
```javascript
jest.setTimeout(10000); // 10 seconds
```

### Coverage gaps
Run with coverage report:
```bash
npm run test:coverage
```

Open `coverage/lcov-report/index.html` in a browser to see which lines need tests.
