# Station System Implementation Plan

## What is a station?
A black box that connects various subscribers and publishers of a node. It routes messages according to rules and modes.

## How is it different than a hard-coded node?
Instead of each subscriber explicitly publishing to a fixed destination, the station dynamically selects publishers based on routing rules and state.

---

### 1. How do we handle multiple routing modes for 1 → #unk?
- **Need:** Decide which publisher(s) a single subscriber’s message should go to.
- **Solution:** Implement selectable modes per subscriber:
  - **Broadcast:** send to all linked publishers.
  - **Round-robin:** cycle through publishers using stored index.
  - **Random:** pick one publisher per message.
  - **First-response-wins:** send to all but track first reply (mainly for service-like tests).
  - **Weighted:** pick based on assigned probabilities.
- **State:** maintain per-subscriber routing state (index, pending publishers, PRNG seed, etc.).

---

### 2. How do we handle #unk → 1?
- **Need:** forward messages from multiple subscribers to a single publisher.
- **Solution:** simple: forward every message to the single linked publisher.
- **State:** none required.

---

### 3. How do we handle 1 → 1?
- **Need:** forward messages from a single subscriber to a single publisher.
- **Solution:** direct forward, log as a standard routing event.
- **State:** none required; mode can be “direct” for consistency.

---

### 4. How do we process and route messages efficiently?
- **Need:** minimize overhead while allowing full logging.
- **Solution:**
  - Maintain a per-subscriber routing table with publishers, mode, state, and rules.
  - Apply routing rules to filter eligible publishers.
  - Use the mode and state to pick publisher(s).
  - Forward messages immediately.
  - Update state as necessary (RR index, pending publishers, etc.).

---

### 5. How do we log the entire evaluation process without slowing the hot path?
- **Need:** track ingress/egress, routing decisions, and message metadata.
- **Solution:**
  - Create a `routing_event` object with:
    - `event_id`, `msg_snapshot` (msg_id, creation_time, src, hop_count, meta)
    - `ingress_time`, `egress_times`
    - `src_station`, `dest`, `mode`, `rule_id`
    - `status`, `notes`
  - Push events to a thread-safe queue.
  - Use an asynchronous logging thread with pluggable sinks: console, file, MongoDB, SQL, etc.
  - Console sink preserves order; DB sink can rely on timestamps for sorting.

---

### 6. How do we generate messages for testing?
- **Need:** produce controlled test traffic.
- **Solution:** implement separate **generator** objects:
  - Create messages with `msg_id`, `creation_time`, payload, meta.
  - Send messages to station subscribers.
  - Keep generator separate to maintain station as pure router.

---

### 7. How do we handle message consumption/verification?
- **Need:** measure outcomes and validate routing.
- **Solution:** implement **sinks**:
  - Start as passive loggers (record received messages).
  - Optionally upgrade to active validators (check ordering, rules, latency).

---

### 8. How do we structure the system for multiple tests and modes?
- **Need:** be able to run #unk→1, 1→#unk, and 1→1 tests without changing core code.
- **Solution:**
  - Keep routing table and per-subscriber state dynamic.
  - Expose APIs to add/remove subscribers, publishers, and routing rules.
  - Allow mode selection per subscriber or per test.
  - Log all routing decisions with timestamps for replay or analysis.

---

### 9. What message format should be used between generators, stations, and sinks?
- **Need:** consistent, minimal, and extensible format.
- **Solution:** define message header:
