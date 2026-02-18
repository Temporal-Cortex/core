# @temporal-cortex/truth-engine

WASM-powered RRULE expansion, conflict detection, free/busy computation, and multi-calendar availability merging for Node.js.

This package wraps the Rust `truth-engine` library via WebAssembly, providing near-native performance for deterministic calendar operations in JavaScript/TypeScript environments.

## Installation

```bash
npm install @temporal-cortex/truth-engine
# or
pnpm add @temporal-cortex/truth-engine
```

## Usage

### Expand Recurrence Rules

```typescript
import { expandRRule } from "@temporal-cortex/truth-engine";

// "Every Tuesday at 2pm Pacific" for 4 weeks
const events = expandRRule(
  "FREQ=WEEKLY;COUNT=4;BYDAY=TU",
  "2026-02-17T14:00:00",
  60,                    // 60-minute duration
  "America/Los_Angeles", // IANA timezone (DST-aware)
);
// events = [{ start: "2026-02-17T22:00:00+00:00", end: "2026-02-17T23:00:00+00:00" }, ...]
```

### Merge Multi-Calendar Availability

```typescript
import { mergeAvailability } from "@temporal-cortex/truth-engine";

const availability = mergeAvailability(
  [
    { stream_id: "google", events: [{ start: "2026-03-16T09:00:00Z", end: "2026-03-16T10:00:00Z" }] },
    { stream_id: "outlook", events: [{ start: "2026-03-16T14:00:00Z", end: "2026-03-16T15:00:00Z" }] },
  ],
  "2026-03-16T08:00:00Z",
  "2026-03-16T17:00:00Z",
  true, // opaque mode — hides which calendar each block came from
);
// availability.busy = [{ start, end, source_count: 0 }, ...]
// availability.free = [{ start, end, duration_minutes }, ...]
```

### Find Conflicts

```typescript
import { findConflicts } from "@temporal-cortex/truth-engine";

const teamA = [{ start: "2026-02-17T14:00:00Z", end: "2026-02-17T15:00:00Z" }];
const teamB = [{ start: "2026-02-17T14:30:00Z", end: "2026-02-17T15:30:00Z" }];

const conflicts = findConflicts(teamA, teamB);
// [{ event_a: {...}, event_b: {...}, overlap_minutes: 30 }]
```

### Find Free Slots

```typescript
import { findFreeSlots } from "@temporal-cortex/truth-engine";

const busyEvents = [
  { start: "2026-02-17T09:00:00Z", end: "2026-02-17T10:00:00Z" },
  { start: "2026-02-17T11:00:00Z", end: "2026-02-17T12:00:00Z" },
];

const slots = findFreeSlots(busyEvents, "2026-02-17T08:00:00", "2026-02-17T13:00:00");
// [{ start, end, duration_minutes: 60 }, { ... }, { ... }]
```

### Find First Available Slot Across Calendars

```typescript
import { findFirstFreeAcross } from "@temporal-cortex/truth-engine";

const slot = findFirstFreeAcross(
  [
    { stream_id: "google", events: [{ start: "2026-03-16T09:00:00Z", end: "2026-03-16T10:00:00Z" }] },
    { stream_id: "outlook", events: [{ start: "2026-03-16T10:00:00Z", end: "2026-03-16T11:00:00Z" }] },
  ],
  "2026-03-16T08:00:00Z",
  "2026-03-16T17:00:00Z",
  30, // minimum 30-minute slot
);
// { start, end, duration_minutes }
```

## API

### `expandRRule(rrule, dtstart, durationMinutes, timezone, until?, maxCount?): TimeRange[]`

Expand an RFC 5545 RRULE into concrete event instances. Supports FREQ, BYDAY, BYSETPOS, BYMONTHDAY, COUNT, UNTIL, EXDATE. DST-aware — events at 14:00 Pacific stay at 14:00 Pacific across transitions.

### `findConflicts(eventsA, eventsB): Conflict[]`

Find all pairwise overlaps between two event lists. Adjacent events (end === start) are not conflicts.

### `findFreeSlots(events, windowStart, windowEnd): FreeSlot[]`

Find free time slots within a window, given a list of busy events.

### `mergeAvailability(streams, windowStart, windowEnd, opaque?): UnifiedAvailability`

Merge N event streams into a unified busy/free view. In opaque mode (default), source counts are hidden for privacy.

### `findFirstFreeAcross(streams, windowStart, windowEnd, minDurationMinutes): FreeSlot | null`

Find the first free slot of at least `minDurationMinutes` across all merged streams. Returns `null` if no qualifying slot exists.

## Types

```typescript
interface TimeRange { start: string; end: string }
interface Conflict { event_a: TimeRange; event_b: TimeRange; overlap_minutes: number }
interface FreeSlot { start: string; end: string; duration_minutes: number }
interface EventStream { stream_id: string; events: TimeRange[] }
interface BusyBlock { start: string; end: string; source_count: number }
interface UnifiedAvailability { busy: BusyBlock[]; free: FreeSlot[]; window_start: string; window_end: string; privacy: string }
```

## Build from Source

This package requires the WASM artifacts to be built from the Rust crate first:

```bash
# From the monorepo root:

# 1. Build the WASM binary
cargo build -p truth-engine-wasm --target wasm32-unknown-unknown --release

# 2. Generate Node.js bindings
wasm-bindgen --target nodejs \
  --out-dir packages/truth-engine-js/wasm/ \
  target/wasm32-unknown-unknown/release/truth_engine_wasm.wasm

# 3. Rename for ESM/CJS compatibility
mv packages/truth-engine-js/wasm/truth_engine_wasm.js packages/truth-engine-js/wasm/truth_engine_wasm.cjs

# 4. Build TypeScript
pnpm --filter @temporal-cortex/truth-engine build

# 5. Run tests
pnpm --filter @temporal-cortex/truth-engine test
```

## Architecture

```
src/index.ts                     ← Public API, loads WASM via createRequire
wasm/truth_engine_wasm.cjs       ← wasm-bindgen generated CommonJS bindings
wasm/truth_engine_wasm.wasm      ← Compiled WASM binary from truth-engine (Rust)
wasm/truth_engine_wasm.d.ts      ← TypeScript type declarations for WASM exports
```

The package uses `createRequire(import.meta.url)` to load the CommonJS WASM bindings from an ESM context. This bridges the module system mismatch since `wasm-bindgen --target nodejs` generates CommonJS but the package uses `"type": "module"`.

## Testing

13 tests covering RRULE expansion (8), conflict detection (3), and free/busy computation (2):

```bash
pnpm --filter @temporal-cortex/truth-engine test
```

## License

MIT
