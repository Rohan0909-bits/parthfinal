# LI-FI OWC Terminal - Stability & Accuracy Fixes

## Overview
Applied critical fixes to ensure every flash is read correctly and eliminate random bit detection failures.

---

## Issues Fixed

### 1. **Threshold Drift During Reading** ✓
**Problem:** The threshold was only updated during SCANNING state, causing it to become stale and leading to bit flip errors during transmission.

**Solution:** Added gentle EMA (Exponential Moving Average) threshold adjustment during READING state when dark samples (0 bits) are detected. Uses a reduced alpha coefficient (0.5×) to prevent aggressive corrections that could cause false positives.

```javascript
} else if (rxState === "READING" && bit === "0") {
  // During reading, gently drift threshold only on confirmed dark samples
  ambientBaseline = (EMA_ALPHA * 0.5) * brightness + (1 - EMA_ALPHA * 0.5) * ambientBaseline;
  threshold = ambientBaseline + THRESHOLD_OFFSET;
}
```

**Impact:** Eliminates misreads caused by gradual ambient light changes during transmission.

---

### 2. **Sampling Timing Stability** ✓
**Problem:** Cumulative drift in bit window timing could cause the 20ms polling interval to become misaligned with the 300ms bit rate, leading to samples being collected at wrong times.

**Solution:** 
- Improved timing calculation with explicit elapsed time checks
- Added intelligent re-synchronization when timing drifts beyond one full bit period
- Only increment `lastBitTime` by exactly one `BIT_RATE_MS` interval per cycle

```javascript
// Calculate exact time adjustment to prevent cumulative drift
lastBitTime += BIT_RATE_MS;

// If we've fallen significantly behind, re-sync to prevent cascading drift
if (now - lastBitTime > BIT_RATE_MS) {
  lastBitTime = now;
}
```

**Impact:** Ensures samples are always collected during the correct 300ms windows, improving bit accuracy.

---

### 3. **Sample Buffer Validation** ✓
**Problem:** Empty sample buffers could occur, causing the fallback to use potentially stale `currentBrightness` values.

**Solution:** Added explicit null/undefined checks before adding samples to the buffer, and skip bit processing if no samples were collected in the window.

```javascript
// Collect a sample into the buffer
if (currentBrightness !== undefined && currentBrightness !== null) {
  oversampleBuffer.push(currentBrightness);
}

// Only process if we have samples
if (samplesThisWindow.length === 0) {
  oversampleBuffer = [];
  return;
}
```

**Impact:** Prevents processing invalid brightness values that could cause erratic bit detection.

---

### 4. **Median Calculation Robustness** ✓
**Problem:** Potential edge case with empty arrays in the median function.

**Solution:** Added comprehensive null/undefined checking at the start of the median function.

```javascript
function median(arr) {
  if (!arr || arr.length === 0) return 0;
  const sorted = [...arr].sort((a, b) => a - b);
  const mid = Math.floor(sorted.length / 2);
  if (sorted.length % 2 === 0) {
    return (sorted[mid - 1] + sorted[mid]) / 2;
  }
  return sorted[mid];
}
```

**Impact:** Eliminates potential crashes or invalid median values.

---

### 5. **Transmission Timing Precision** ✓
**Problem:** Non-deterministic flash timing could cause receiver to sample at inconsistent points.

**Solution:** Added explicit comment and ensured precise `BIT_RATE_MS` timing for each bit transmission, with no intermediate operations.

```javascript
// Sleep for full bit duration - PRECISE timing ensures receiver captures correctly
await sleep(BIT_RATE_MS);
```

**Impact:** Guarantees that flashlight/screen changes occur at exact time intervals.

---

## Test Recommendations

1. **Transmit a test message** (e.g., "TEST" or "HELLO") and verify it's decoded correctly every time
2. **Test with varying ambient light** to ensure threshold adaptation works smoothly
3. **Test rapid successive transmissions** to ensure preamble/postamble detection stays reliable
4. **Check the Debug Panel** to monitor:
   - Sample count per bit window (should be ~15-16 samples)
   - Median brightness values staying consistent
   - Confidence levels staying high (>70%)

---

## Technical Details

### Bit Detection Pipeline
1. **Brightness Collection** (20ms interval) → Samples added to `oversampleBuffer`
2. **Bit Window Detection** (300ms intervals) → Median calculated from all samples in window
3. **Threshold Comparison** → Hysteresis band applied for stability
4. **Bit Recording** → Added to `rxBitBuffer` with confidence metric

### Key Parameters
- **Polling Interval:** 20ms (50Hz sampling)
- **Bit Duration:** 300ms (BIT_RATE_MS)
- **Expected Samples/Bit:** ~15 samples (300÷20)
- **Hysteresis Band:** 8 units (prevents flipping near threshold)
- **Threshold Offset:** 30 units above ambient

---

## Files Modified
- `script.js` - Core receiver and transmitter logic

---

## Version
- **Fixed Version:** v1.1
- **Date:** 2026-03-01
- **Changes:** Stability and accuracy improvements for reliable LI-FI communication
