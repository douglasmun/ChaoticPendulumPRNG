# Architecture Documentation
# Chaotic Pendulum PRNG - Production Build v2.0

**Version:** 2.0  
**Last Updated:** 2025-11-15  
**Target Audience:** Developers, Code Reviewers, Contributors

---

## Table of Contents

1. [System Overview](#system-overview)
2. [Architecture Diagram](#architecture-diagram)
3. [Core Components](#core-components)
4. [Data Flow](#data-flow)
5. [Class Design](#class-design)
6. [Security Architecture](#security-architecture)
7. [Performance Optimizations](#performance-optimizations)
8. [Error Handling Strategy](#error-handling-strategy)
9. [Testing Strategy](#testing-strategy)

---

## System Overview

### Purpose

Chaotic Pendulum PRNG v2.0 is an educational demonstration of pseudo-random number generation using the inherently chaotic behavior of double pendulum systems. It combines:

- **Physics Simulation:** Accurate double pendulum dynamics
- **Randomness Extraction:** Bit extraction from chaotic motion
- **Production-Grade Engineering:** Health monitoring, error handling, resource management
- **Educational Value:** Visual demonstration of chaos theory

### Technology Stack

- **Language:** Pure JavaScript (ES6+)
- **Rendering:** SVG with DOM manipulation
- **Styling:** CSS3 with Grid and Flexbox
- **Security:** Web Crypto API
- **Architecture:** Event-driven, single-threaded

### Design Principles

1. **Fail-Safe:** System degrades gracefully, never silently corrupts
2. **Observable:** All state visible for educational purposes
3. **Validated:** All inputs and outputs rigorously checked
4. **Resource-Bounded:** Hard limits on memory and CPU usage
5. **Self-Healing:** Automatic recovery from transient failures

---

## Architecture Diagram

```
┌─────────────────────────────────────────────────────────────┐
│                         User Interface                       │
│  ┌─────────────┐  ┌──────────┐  ┌─────────────────────────┐│
│  │Security     │  │Controls  │  │  SVG Canvas            ││
│  │Warning      │  │(Sliders) │  │  (8 Pendulums)        ││
│  │Banner       │  │          │  │  + Trails             ││
│  └─────────────┘  └──────────┘  └─────────────────────────┘│
│  ┌─────────────┐  ┌──────────────────────────────────────┐ │
│  │Status Bar   │  │  Output Grid + Heatmap              │ │
│  │(Health)     │  │  (Generated Values)                 │ │
│  └─────────────┘  └──────────────────────────────────────┘ │
└────────────┬────────────────────────────────────────────────┘
             │
             ▼
┌─────────────────────────────────────────────────────────────┐
│                      Application State                       │
│  ┌──────────────────────────────────────────────────────────┐│
│  │ AppState (Global State Manager)                          ││
│  │  - PRNG Instance                                         ││
│  │  - Animation Control                                     ││
│  │  - Output Arrays                                         ││
│  │  - Event Handlers                                        ││
│  │  - Locks & Flags                                         ││
│  └──────────────────────────────────────────────────────────┘│
└────────────┬────────────────────────────────────────────────┘
             │
             ▼
┌─────────────────────────────────────────────────────────────┐
│                    Core Engine Layer                         │
│  ┌────────────────┐  ┌─────────────────┐  ┌──────────────┐ │
│  │ChaoticPRNG     │  │HealthMonitor    │  │ValidatedSlider│ │
│  │                │  │                 │  │              │ │
│  │- 8 Pendulums   │  │- Metrics        │  │- Validation  │ │
│  │- Byte Gen      │  │- Circuit Breaker│  │- Clamping    │ │
│  │- Sampling      │  │- Recovery       │  │              │ │
│  └────────────────┘  └─────────────────┘  └──────────────┘ │
│           │                    │                             │
│           ▼                    ▼                             │
│  ┌──────────────────────────────────────────────────────┐  │
│  │         ChaoticPendulum (×8 instances)               │  │
│  │  - Double Pendulum Physics                            │  │
│  │  - State Validation                                   │  │
│  │  - Bit Extraction                                     │  │
│  │  - Health Monitoring                                  │  │
│  │  - Circular Buffer Trail                              │  │
│  └──────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
             │
             ▼
┌─────────────────────────────────────────────────────────────┐
│                   Utility & Support Layer                    │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐  │
│  │Validation│  │ Math     │  │ Rendering│  │ Modal    │  │
│  │Functions │  │ Utils    │  │ Functions│  │ System   │  │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘  │
└─────────────────────────────────────────────────────────────┘
```

---

## Core Components

### 1. Application State (`AppState`)

**Purpose:** Central state manager, implements Singleton pattern

**Responsibilities:**
- Manages global application state
- Controls animation lifecycle
- Prevents resource leaks
- Coordinates between components

**Key Properties:**
```javascript
const AppState = {
  // Core instances
  prng: ChaoticPRNG instance,
  healthMonitor: HealthMonitor instance,
  svg: SVG element reference,

  // State flags
  isRunning: boolean,
  updateLock: boolean,
  initializationLock: boolean,
  resizePending: boolean,

  // Data storage
  outputList: Array<number>,
  valueCounts: Array<number>,
  valueCountsSnapshot: Array<number>,

  // Resource tracking
  animationFrameId: number,
  eventHandlers: Map<string, EventHandler>,
  sliders: Object<ValidatedSlider>,

  // Lifecycle methods
  startAnimation(),
  stopAnimation(),
  reset()
};
```

**Design Patterns:**
- Singleton: Only one AppState exists
- Facade: Simplifies complex subsystem interactions
- Observer: Event handlers stored and managed centrally

---

### 2. Chaotic Pendulum (`ChaoticPendulum`)

**Purpose:** Simulates a single double pendulum with health monitoring

**Responsibilities:**
- Accurate physics simulation
- State validation
- Bit extraction
- Self-healing on failures

**Class Structure:**
```javascript
class ChaoticPendulum {
  // Physics state
  theta1, theta2: number    // Angles
  omega1, omega2: number    // Angular velocities
  length1, length2: number  // Rod lengths
  mass1, mass2: number      // Bob masses

  // Configuration
  origin: {x, y}
  externalForce: {x, y}
  damping: number

  // Trail (circular buffer)
  trail: Array<{x,y}>
  trailIndex: number
  trailFilled: boolean

  // Health tracking
  failureCount: number
  isHealthy: boolean

  // Methods
  step(dt)              // Advance physics
  extractBit()          // Generate random bit
  validateState()       // Check validity
  handleFailure(reason) // Error recovery
  resetToSafeState()    // Recovery mechanism
}
```

**Physics Implementation:**

Uses Lagrangian mechanics for double pendulum:

```javascript
// Angular accelerations
domega1 = [
  -g(m1+m2)sin(θ1) - m2·g·sin(θ1-2θ2)
  - 2sin(θ2-θ1)·m2·(ω2²·l2 + ω1²·l1·cos(θ2-θ1))
] / [l1·((m1+m2) - m2·cos²(θ2-θ1))]

domega2 = [
  2sin(θ2-θ1) · (
    ω1²·l1·(m1+m2) + g·(m1+m2)·cos(θ1)
    + ω2²·l2·m2·cos(θ2-θ1)
  )
] / [l2·((m1+m2) - m2·cos²(θ2-θ1))]
```

**Bit Extraction Algorithm:**

```javascript
extractBit() {
  // 1. Get current position and velocity
  const {x2, y2} = this.getTipPosition();

  // 2. Normalize to prevent overflow
  const x2_mod = ((abs(x2) % 1000) + 1000) % 1000;
  const y2_mod = ((abs(y2) % 1000) + 1000) % 1000;
  const omega1_mod = ((abs(omega1) % 1000) + 1000) % 1000;
  const omega2_mod = ((abs(omega2) % 1000) + 1000) % 1000;

  // 3. Combine with prime multipliers
  const combined = (x2_mod * 1.1 + y2_mod * 1.3 +
                   omega1_mod * 1.7 + omega2_mod * 1.9) % 10000;

  // 4. Extract LSB
  const asInt = floor(abs(combined)) & 0xFFFFFFFF;
  return asInt & 1;
}
```

---

### 3. Chaotic PRNG (`ChaoticPRNG`)

**Purpose:** Manages 8 pendulums and generates bytes

**Responsibilities:**
- Initialize pendulum array
- Coordinate physics updates
- Sample at controlled rate
- Generate validated bytes
- Handle mouse forces

**Byte Generation:**

```javascript
generateByte() {
  // Sample at reduced rate (every 8 frames)
  this.sampleCounter++;
  if (this.sampleCounter < 8) return null;
  this.sampleCounter = 0;

  // Extract 1 bit from each of 8 pendulums
  const bits = [];
  for (let i = 0; i < 8; i++) {
    try {
      bits.push(this.pendulums[i].extractBit());
    } catch (error) {
      // Fallback to crypto, not Math.random()
      const arr = new Uint8Array(1);
      crypto.getRandomValues(arr);
      bits.push(arr[0] & 1);
    }
  }

  // Combine into byte
  const byte = parseInt(bits.join(''), 2);

  // Validate before returning
  if (!Number.isInteger(byte) || byte < 0 || byte > 255) {
    throw new Error(`Invalid byte: ${byte}`);
  }

  return byte;
}
```

**Sampling Strategy:**

- Physics runs at 60 FPS
- Byte generation at ~7.5 Hz (every 8 frames)
- Decorrelates sequential samples
- Prevents aliasing with pendulum period

---

### 4. Health Monitor (`HealthMonitor`)

**Purpose:** Circuit breaker pattern implementation

**Responsibilities:**
- Monitor system health metrics
- Detect anomalies
- Trigger circuit breaker
- Attempt automatic recovery

**Monitored Metrics:**

| Metric | Threshold | Action |
|--------|-----------|--------|
| Time since last byte | 5 seconds | Circuit breaker |
| Frame rate | < 30 FPS | Warning |
| Memory usage | > 500 MB | Circuit breaker |
| Unhealthy pendulums | > 4 of 8 | Circuit breaker |
| Error count | > 100 | Circuit breaker |

**Circuit Breaker State Machine:**

```
┌─────────┐    health OK     ┌─────────┐
│ HEALTHY │ ───────────────> │ HEALTHY │
└────┬────┘                  └─────────┘
     │
     │ threshold exceeded
     ▼
┌─────────┐    trigger()     ┌──────────┐
│ HEALTHY │ ───────────────> │TRIGGERED │
└─────────┘                  └────┬─────┘
                                  │
                                  │ after 2s delay
                                  ▼
                             ┌──────────┐
                             │RECOVERING│
                             └────┬─────┘
                                  │
                    ┌─────────────┼─────────────┐
                    │ success     │     failure │
                    ▼             ▼             │
              ┌─────────┐   ┌──────────┐       │
              │ HEALTHY │   │  FAILED  │───────┘
              └─────────┘   └──────────┘
                                  │
                                  │ manual refresh required
                                  ▼
                             [User Action]
```

**Recovery Procedure:**

```javascript
attemptRecovery() {
  try {
    // 1. Reset error counters
    this.metrics.errorCount = 0;
    this.metrics.lastByteTime = Date.now();

    // 2. Reset application state
    AppState.reset();

    // 3. Mark as healthy
    this.isHealthy = true;

    // 4. Notify user
    Modal.show('Recovery Successful', ...);
  } catch (error) {
    // Recovery failed - require manual intervention
    Modal.show('Recovery Failed', 'Please refresh page', true);
  }
}
```

---

### 5. Validated Slider (`ValidatedSlider`)

**Purpose:** Prevent slider exploitation attacks

**Security Features:**
- Input validation
- Range clamping
- Type checking
- Event handler cleanup

**Implementation:**

```javascript
class ValidatedSlider {
  handleInput(e) {
    let value = parseFloat(e.target.value);

    // 1. Validate type
    if (!isFinite(value)) value = this.defaultValue;

    // 2. Clamp to range
    value = clamp(value, this.min, this.max);

    // 3. Enforce integer if needed
    if (Number.isInteger(this.min) && Number.isInteger(this.max)) {
      value = Math.round(value);
    }

    // 4. Update DOM (prevent drift)
    this.element.value = value;
    this.currentValue = value;

    // 5. Trigger callback
    if (this.onChange) this.onChange(value);
  }
}
```

---

## Data Flow

### Initialization Flow

```
User loads page
      │
      ▼
DOMContentLoaded event
      │
      ▼
initApplication()
      │
      ├─> Check crypto.getRandomValues() support
      ├─> Stop existing animations
      ├─> Cleanup event listeners
      ├─> Create HealthMonitor
      ├─> Get SVG element
      ├─> Create ChaoticPRNG
      │    └─> Initialize 8 pendulums
      ├─> Init heatmap UI
      ├─> Setup validated sliders
      └─> Start animation loop
```

### Animation Loop Flow

```
requestAnimationFrame(update)
      │
      ▼
Check isRunning flag
      │
      ▼
Acquire updateLock
      │
      ├─> Record frame (health monitor)
      ├─> Check health (circuit breaker)
      ├─> Get validated speed
      │
      ├─> Step physics (speed × iterations)
      │    └─> For each pendulum:
      │         ├─> Apply external forces
      │         ├─> Calculate accelerations
      │         ├─> Update angles & velocities
      │         ├─> Normalize angles
      │         ├─> Clamp velocities
      │         ├─> Validate state
      │         └─> Update trail (circular buffer)
      │
      ├─> Generate byte (at reduced rate)
      │    └─> Extract 8 bits (1 from each pendulum)
      │         ├─> Validate byte
      │         ├─> Add to outputList
      │         ├─> Update valueCounts
      │         └─> Enforce size limits
      │
      ├─> Render output grid
      ├─> Update heatmap (every 3rd frame)
      ├─> Update statistics
      ├─> Render pendulums (SVG)
      └─> Update health display
      │
      ▼
Release updateLock
      │
      ▼
Schedule next frame: requestAnimationFrame(update)
```

### Error Handling Flow

```
Error occurs
      │
      ▼
Caught by try-catch
      │
      ├─> Log to console
      ├─> Record in HealthMonitor
      │
      ├─> Is error count > threshold?
      │   │
      │   └─> YES: Trigger circuit breaker
      │        │
      │        ├─> Stop animation
      │        ├─> Show modal
      │        └─> Schedule recovery
      │             │
      │             └─> attemptRecovery()
      │                  ├─> Reset state
      │                  ├─> Reinitialize
      │                  └─> Resume or fail
      │
      └─> NO: Continue with degraded mode
           └─> Use crypto.getRandomValues() fallback
```

---

## Security Architecture

### Defense in Depth

**Layer 1: Input Validation**
- Validated sliders (min/max enforcement)
- Type checking (Number.isInteger, isFinite)
- Bounds checking (array access)

**Layer 2: State Protection**
- Initialization lock
- Update lock
- Immutable snapshots

**Layer 3: Resource Limits**
- Max output size: 2,000
- Max total samples: 100,000
- Max memory: 500 MB
- Max trail length: 20

**Layer 4: Error Handling**
- Try-catch blocks
- State validation
- Health monitoring

**Layer 5: Circuit Breakers**
- Automatic failure detection
- Graceful degradation
- Auto-recovery

**Layer 6: User Warnings**
- Security banner
- Modal alerts
- Export warnings

### Secure Random Number Generation

**Primary:** `crypto.getRandomValues()`
```javascript
function secureRandom() {
  const array = new Uint32Array(1);
  crypto.getRandomValues(array);
  return array[0] / 0x100000000;
}
```

**Used in:**
- Initial pendulum velocities
- Chaos injection
- Fallback when pendulum fails

**Never used:** `Math.random()` (completely eliminated in v2.0)

---

## Performance Optimizations

### Memory Optimizations

1. **Circular Buffers:**
   ```javascript
   // Trail uses fixed-size array, not push/shift
   this.trail[this.trailIndex] = { x, y };
   this.trailIndex = (this.trailIndex + 1) % MAX_TRAIL_LENGTH;
   ```

2. **Bounded Arrays:**
   ```javascript
   // outputList capped at 2,000 elements
   if (outputList.length > MAX_OUTPUT_SIZE) {
     const removeCount = Math.floor(MAX_OUTPUT_SIZE * 0.1);
     outputList.splice(0, removeCount);
   }
   ```

3. **Snapshots for Race-Free Access:**
   ```javascript
   // Avoid locks by using immutable snapshots
   if (snapshotDirty) {
     valueCountsSnapshot = [...valueCounts];
     snapshotDirty = false;
   }
   ```

### Rendering Optimizations

1. **Reduced Update Frequency:**
   ```javascript
   // Heatmap updates every 3rd frame
   if (frameCount % 3 === 0) {
     updateHeatmap();
   }
   ```

2. **Batch DOM Updates:**
   ```javascript
   // Clear once, then append all
   while (outputGrid.firstChild) {
     outputGrid.removeChild(outputGrid.firstChild);
   }
   displayBytes.forEach(byte => {
     outputGrid.appendChild(createByteElement(byte));
   });
   ```

3. **Event Debouncing:**
   ```javascript
   // Resize debounced to 250ms
   clearTimeout(resizeTimeout);
   resizeTimeout = setTimeout(handleResize, 250);
   ```

### CPU Optimizations

1. **Controlled Sampling Rate:**
   - Physics: 60 FPS
   - Byte generation: ~7.5 Hz
   - Reduces CPU load by 8×

2. **Efficient Math:**
   ```javascript
   // Pre-calculate constants
   const cosDelta = Math.cos(theta2 - theta1);
   const sinDelta = Math.sin(theta2 - theta1);

   // Reuse in both equations
   ```

3. **Early Returns:**
   ```javascript
   // Skip frame if locked
   if (updateLock) {
     requestAnimationFrame(update);
     return;
   }
   ```

---

## Error Handling Strategy

### Error Categories

**1. Transient Errors (Recoverable)**
- Singularity in physics calculation
- Single pendulum failure
- Temporary NaN/Infinity

**Recovery:** Retry, use fallback, reset to safe state

**2. Persistent Errors (Circuit Breaker)**
- Multiple pendulum failures
- Continuous NaN generation
- Memory exhaustion

**Recovery:** Stop system, attempt auto-recovery

**3. Fatal Errors (User Action Required)**
- Crypto API unavailable
- DOM elements not found
- Initialization failure

**Recovery:** Show error, require page refresh

### Error Handling Pattern

```javascript
try {
  // Attempt operation
  result = riskyOperation();

  // Validate result
  if (!isValid(result)) {
    throw new Error('Invalid result');
  }

  // Reset failure counters on success
  failureCount = 0;

} catch (error) {
  // Log error
  console.error('Operation failed:', error);

  // Record for monitoring
  healthMonitor.recordError(error);

  // Attempt recovery
  if (failureCount < MAX_FAILURES) {
    failureCount++;
    useFailback();
  } else {
    // Give up, trigger circuit breaker
    circuitBreaker.trigger('Too many failures');
  }
}
```

---

## Testing Strategy

### Unit Testing (Recommended)

**Test Coverage Areas:**
1. Utility functions (clamp, validateInteger, normalizeAngle)
2. Entropy calculation
3. Bit extraction
4. Byte validation
5. State validation

**Example Tests:**
```javascript
describe('clamp', () => {
  it('should clamp value to range', () => {
    expect(clamp(5, 0, 10)).toBe(5);
    expect(clamp(-5, 0, 10)).toBe(0);
    expect(clamp(15, 0, 10)).toBe(10);
  });

  it('should handle non-finite values', () => {
    expect(clamp(NaN, 0, 10)).toBe(0);
    expect(clamp(Infinity, 0, 10)).toBe(10);
  });
});
```

### Integration Testing

**Scenarios to Test:**
1. Full initialization cycle
2. Byte generation over time
3. Error recovery
4. Circuit breaker triggering
5. Resource limit enforcement

### Stress Testing

**Load Tests:**
1. Run for 24 hours continuously
2. Rapidly toggle speed slider
3. Spam resize events
4. Generate 100,000 samples
5. Trigger all error paths

**Expected Behavior:**
- No memory growth beyond limits
- No crashes or freezes
- Circuit breaker activates correctly
- Auto-recovery succeeds

---

## Future Enhancements

### Potential Improvements

1. **WebGL Rendering:**
   - Hardware-accelerated graphics
   - Support for 100+ pendulums
   - Better performance

2. **Web Workers:**
   - Move physics to background thread
   - Keep UI responsive
   - Parallel pendulum updates

3. **Statistical Testing:**
   - Built-in NIST test suite
   - Chi-square testing
   - Autocorrelation analysis

4. **Advanced Features:**
   - Phase space plots
   - Lyapunov exponent calculation
   - Parameter sweeps

---

## Conclusion

The v2.0 architecture prioritizes **stability**, **security**, and **educational value** while maintaining the chaotic dynamics that make this PRNG interesting.

Key architectural decisions:
- ✅ Comprehensive health monitoring
- ✅ Fail-safe error handling
- ✅ Resource-bounded execution
- ✅ Input validation everywhere
- ✅ Secure random number generation
- ✅ Observable for educational purposes

**The system is production-ready for educational use, but remains fundamentally unsuitable for cryptographic applications due to its deterministic, observable nature.**

---

**Document Version:** 2.0  
**Maintained By:** Development Team  
**Last Review:** 2025-11-15
