# Add automatic buffer reset on large RTP discontinuities

## Overview

This PR improves the resilience of the jitter buffer by automatically detecting and handling large RTP discontinuities in both sequence numbers and timestamps.

## Problem

The jitter buffer could end up in an invalid state when experiencing large gaps in RTP sequence numbers or timestamps, such as:
- Stream source switches or failovers
- Network reconnections
- Long pauses in transmission
- Clock adjustments

This resulted in:
- Choppy playback after recovery
- Packets being incorrectly dropped or delayed
- Buffer stuck waiting for missing packets that would never arrive

## Solution

Added automatic detection and reset mechanism for large discontinuities:

### New Features

1. **Large sequence number jump detection** (threshold: 1000 packets)
   - Handles both forward and backward jumps with wraparound support
   - Prevents false positives near uint16 boundaries

2. **Large timestamp jump detection** (threshold: 30 seconds)
   - Detects clock resets or stream source changes
   - Handles 32-bit timestamp wraparound correctly

3. **Automatic buffer reset**
   - Clears all buffered packets when discontinuity detected
   - Reinitializes buffer state for clean recovery
   - Logs warnings for operational visibility

4. **Previous timestamp tracking**
   - Added `prevTS` field to track last processed timestamp
   - Enables timestamp-based discontinuity detection

### Bug Fixes

- Buffer no longer stays in invalid state after large gaps
- Improved packet expiration handling after discontinuities
- Reduced premature packet drops and delayed emissions

## Implementation Details

### Detection Logic

```go
// Detect jumps accounting for wraparound
func (b *Buffer) isLargeSequenceJump(current, prev uint16) bool {
    const MAX_SEQUENCE_JUMP = 1000
    // Calculate minimum distance considering wraparound
    // ...
}

func (b *Buffer) isLargeTimestampJump(current, prev uint32) bool {
    const MAX_TIMESTAMP_JUMP = 8000 * 30 // 30 seconds at 8kHz
    // Calculate minimum distance considering wraparound
    // ...
}
```

### Reset Mechanism

When a large discontinuity is detected:
1. Log warning with current/previous values for debugging
2. Clear all buffered packets
3. Reset state variables (`initialized`, `prevSN`, `prevTS`)
4. Reset timer to latency duration
5. Continue processing new packets normally

## Testing

Added comprehensive tests:

- **TestLargeSequenceJump**: Verifies buffer recovers after sequence jump >1000
- **TestLargeTimestampJump**: Verifies buffer recovers after timestamp jump >30s
- **TestSequenceWraparound**: Ensures normal wraparound (e.g., 65535 → 0) works correctly

All existing tests continue to pass, ensuring backward compatibility.

## Performance Impact

- Minimal overhead: discontinuity checks only compare integers
- Reset is rare in normal operation
- No impact on hot path for normal packet processing

## Logs

When discontinuity is detected, a warning log is emitted:
```
large RTP discontinuity detected current_ts=X prev_ts=Y current_sn=Z prev_sn=W
resetting jitter buffer due to RTP discontinuity
```

This provides operational visibility without cluttering logs during normal operation.

## Backward Compatibility

✅ Fully backward compatible
- No API changes
- Existing behavior preserved for normal packet streams
- Only affects behavior when large discontinuities occur (which previously caused issues)

## Related Issues

This addresses issues related to:
- Stream recovery after network interruptions
- Multi-source streaming scenarios
- Clock adjustment handling
- Failover situations

