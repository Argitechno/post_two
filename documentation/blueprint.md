# Station System Implementation Blueprint

## 1. Parcel
- **Purpose:** Represents the payload passed through the system.
- **Fields / Members:**
  - `msg_id`: UUID/string, unique per parcel
  - `creation_time`: float, time the parcel was generated
  - `src`: string, last publisher/station
  - `hop_count`: int, increments on each forwarding
  - `payload`: bytes or any type
  - `meta`: dict, optional metadata for testing purposes
- **Responsibilities:**
  - Provide a snapshot of the header for logging
  - Update `src` and `hop_count` when forwarded

---

## 2. RoutingEvent
- **Purpose:** Captures full evaluation and routing process for each parcel
- **Fields / Members:**
  - `event_id`: UUID/string, unique per routing event
  - `msg_snapshot`: snapshot of parcel header at ingress
  - `ingress_time`: float, time received by station
  - `egress_times`: dict, mapping each publisher to send time
  - `src_station`: string, station handling the parcel
  - `dest`: list of strings, actual publishers selected
  - `mode`: string, routing mode applied
  - `rule_id`: string, which rule was applied
  - `status`: string, e.g., forwarded, dropped, error
  - `notes`: string, optional extra info (RR index, weights, etc.)
- **Responsibilities:**
  - Capture all routing decisions for logging and analysis
  - Be serializable to console, file, or database

---

## 3. SubscriberState
- **Purpose:** Holds routing state needed per subscriber
- **Fields / Members (depend on mode):**
  - Round-robin: `next_index`
  - Random: optional PRNG seed
  - First-response-wins: ephemeral `pending_publishers` per parcel
  - Weighted: `weights`, `cumulative_weights`
- **Responsibilities:**
  - Maintain mode-specific state across parcels
  - Update state after each routing decision
  - Allow deterministic replay if needed

---

## 4. RoutingTable
- **Purpose:** Maps each subscriber to its publishers, mode, state, and rules
- **Fields / Members:**
  - Key: subscriber ID
  - Value: dictionary/object containing:
    - `publishers`: list of linked publisher IDs
    - `mode`: routing mode
    - `state`: SubscriberState
    - `rules`: optional list of routing rules
- **Responsibilities:**
  - Provide fast lookup for routing decisions
  - Allow dynamic updates (add/remove subscribers/publishers/rules)

---

## 5. Station
- **Purpose:** Core router handling parcels, applying rules and routing modes
- **Methods / Responsibilities:**
  - `add_subscriber(sub_id)`: register subscriber
  - `add_publisher(pub_id)`: register publisher
  - `add_rule(rule)`: attach rule to a subscriber
  - `receive_parcel(msg)`: 
    - Record ingress timestamp
    - Fetch routing table entry
    - Evaluate rules to filter publishers
    - Apply routing mode using subscriber state
    - Forward parcel to selected publishers
    - Update state as needed
    - Generate and enqueue RoutingEvent

---

## 6. Logging Subsystem
- **Purpose:** Record full routing events asynchronously without slowing stations
- **Components:**
  - Thread-safe queue for routing events
  - Pluggable sinks:
    - Console: prints in-order
    - File: asynchronous logging
    - Database: MongoDB, SQL, etc.
- **Responsibilities:**
  - Consume events from queue
  - Dispatch to configured sinks
  - Optionally sort by timestamp for DB sinks

---

## 7. Generator
- **Purpose:** Produces parcels into the system for testing
- **Methods / Responsibilities:**
  - `generate_parcel(payload, meta)`:
    - Assign unique `msg_id` and `creation_time`
    - Send to designated station subscriber
- **Notes:** Keep generator separate from station to maintain pure router abstraction

---

## 8. Sink
- **Purpose:** Receives parcels at the edge for logging and/or validation
- **Methods / Responsibilities:**
  - Passive sink:
    - Log all incoming parcels
    - Record timestamps for analysis
  - Optional active sink:
    - Validate parcel ordering, routing compliance, or latency
- **Notes:** Design sink to be easily upgradeable from passive to active

---

## 9. Routing Modes
- **Purpose:** Define how a subscriber forwards parcels to multiple publishers
- **Modes and Needed State:**
  - **Broadcast:** forward to all publishers (stateless)
  - **Round-Robin:** forward to next publisher, increment index (state: `next_index`)
  - **Random:** pick one publisher at random (state: optional PRNG seed)
  - **First-Response-Wins:** send to all, track pending responses per parcel (state: `pending_publishers`)
  - **Weighted:** probabilistic selection based on weights (state: `weights`, `cumulative_weights`)
- **Responsibilities:**
  - Determine forwarding decisions
  - Update subscriber state for deterministic replay
  - Support dynamic switching per test

---

## 10. System Capabilities
- **Dynamic configuration:**
  - Add/remove subscribers and publishers at runtime
  - Add/remove routing rules dynamically
- **Flexible testing:**
  - Support #unk→1, 1→#unk, and 1→1 scenarios
  - Capture full routing events for each parcel
  - Allow multiple routing modes per subscriber
- **Logging:**
  - Full ingress/egress timestamps
  - Mode, rule, status, and notes recorded
  - Pluggable sinks (console, file, DB)

---

This blueprint provides a **complete modular architecture** for building the station, generators, and sinks with full logging and stress-testing capabilities.
