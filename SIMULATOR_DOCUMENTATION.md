# Sleep-Based Low-Latency Access Simulator Documentation

## Overview

This simulator implements a discrete-event simulation framework for studying sleep-based low-latency access mechanisms in Machine-to-Machine (M2M) communications. The simulator models battery-powered devices (MTDs) that use on-demand sleep scheduling with slotted Aloha for channel access, enabling analysis of the fundamental trade-off between energy efficiency (battery lifetime) and communication latency.

### Purpose

The simulator addresses a critical challenge in 5G and beyond networks: how to extend battery lifetime of IoT sensors to 10+ years while maintaining low latency for mission-critical applications. By implementing an on-demand sleep mechanism (similar to 5G's MICO mode), devices can sleep when idle and wake up only when packets arrive, achieving significant energy savings compared to traditional duty-cycling approaches.

---

## Architecture Overview

The simulator is built using **SimPy**, a process-based discrete-event simulation framework. The architecture consists of three main components:

1. **SimulationConfig**: Centralized configuration for all simulation parameters
2. **SlottedChannel**: Manages channel access and collision resolution
3. **MTD (Machine-Type Device)**: Individual device with state machine and energy tracking

### Design Philosophy

- **Modularity**: Each component is self-contained and can be modified independently
- **Extensibility**: Easy to add new features (e.g., different MAC protocols, power models)
- **Clarity**: Code is structured for live demonstrations and educational purposes
- **Performance**: Efficient enough to simulate thousands of nodes over millions of slots

---

## Component Details

### 1. SimulationConfig

**Purpose**: Centralizes all simulation parameters in a single, easily modifiable dataclass.

**Key Parameters**:

- `num_nodes`: Number of MTDs in the simulation (default: 500)
- `lambda_arrival`: Poisson arrival rate per slot (default: 1e-3)
- `q_tx`: Slotted-Aloha transmission probability (default: 0.05)
- `idle_timer_slots` (t_s): Duration in slots before transitioning from IDLE to SLEEP (default: 15)
- `wake_time_slots` (t_w): Duration in slots for wake-up transition (default: 5)
- `slot_duration_ms`: Duration of one time slot in milliseconds (default: 1.0 ms)
- `max_slots`: Maximum number of slots to simulate (default: 50,000)
- `initial_energy_mj`: Initial battery energy in millijoules (default: 10,000 mJ)
- `power_profile_mw`: Power consumption for each state (ACTIVE: 180 mW, IDLE: 8 mW, SLEEP: 0.02 mW, WAKEUP: 40 mW)

**Reasoning**: 
- Power values are inspired by real 5G specifications (e.g., P_Tx ≈ 23 dBm ≈ 200 mW)
- The idle timer (t_s) represents the T3324 timer in 3GPP MICO mode
- Configurable parameters allow easy parameter sweeps for sensitivity analysis

---

### 2. SlottedChannel

**Purpose**: Implements a shared channel with slotted Aloha collision detection.

**Key Methods**:

- `request_tx(node, slot_index)`: Registers a transmission attempt for a specific slot
- `_run()`: Background process that resolves slots at the end of each slot period
- `_resolve_slot(slot_index)`: Determines success or collision for a slot

**Collision Model**:
- **Empty slot**: No transmission attempts → no action
- **Single transmission**: Exactly one node transmits → success
- **Collision**: Multiple nodes transmit → all fail (collision)

**Timing Logic**:
```
At time t:
  - Nodes process slot t and may request transmissions
  - Channel resolves slot (t-1) that just completed
```

This ensures that all transmission requests for a slot are collected before resolution.

**Statistics Tracked**:
- `slots`: Total number of slots processed
- `successes`: Number of successful transmissions
- `collisions`: Number of slots with collisions

**Reasoning**:
- Simple collision model matches theoretical slotted Aloha analysis
- Slot-based resolution ensures deterministic behavior
- Statistics enable calculation of throughput and collision probability

---

### 3. MTD (Machine-Type Device)

**Purpose**: Represents an individual battery-powered device with a state machine, packet queue, and energy tracking.

#### State Machine

The MTD implements a 4-state machine:

```
ACTIVE → IDLE → SLEEP → WAKEUP → ACTIVE
  ↑                                    ↓
  └────────────────────────────────────┘
```

**State Transitions**:

1. **ACTIVE → IDLE**: When packet queue becomes empty, start idle timer (t_s)
2. **IDLE → SLEEP**: After idle timer expires (t_s slots), transition to sleep
3. **SLEEP → WAKEUP**: When packet arrives during sleep, begin wake-up process
4. **WAKEUP → ACTIVE**: After wake-up time (t_w slots), become active
5. **IDLE → ACTIVE**: When packet arrives during idle, immediately become active
6. **SLEEP → WAKEUP**: When packet arrives during sleep, begin wake-up

**Key Methods**:

##### `generate_packets()`
- Generates packet arrivals using Poisson process
- Inter-arrival times follow exponential distribution: `expovariate(lambda_arrival)`
- When packet arrives:
  - Adds packet to queue with arrival timestamp
  - If in SLEEP, triggers wake-up
  - If in IDLE, immediately becomes ACTIVE

**Reasoning**: Poisson arrivals model realistic traffic patterns for IoT sensors (e.g., periodic health monitoring, event-driven alarms).

##### `run()`
- Main simulation loop that processes each slot
- At each slot boundary:
  - Gets current slot index: `int(env.now)`
  - Processes slot logic via `_handle_slot()`
  - Waits for next slot: `yield env.timeout(1)`

##### `_handle_slot(slot_index)`
Core slot processing logic:

1. **Energy consumption**: Always consume energy based on current state
2. **State-specific logic**:
   - **WAKEUP**: Decrement wake-up counter, transition to ACTIVE when complete
   - **SLEEP**: Do nothing (waiting for packet arrival)
   - **Empty queue**: Manage idle timer and transition to SLEEP if needed
   - **Has packets**: Attempt transmission with probability `q_tx`

**Transmission Logic**:
- Only transmit if in ACTIVE state and has packets
- Transmission probability: `random.random() < q_tx`
- If transmitting, register with channel for current slot

**Reasoning**: 
- Slotted Aloha with probability `q_tx` prevents all nodes from transmitting simultaneously
- State-based logic ensures devices only transmit when ready (not in SLEEP/WAKEUP)

##### `_consume_energy(slot_index)`
- Calculates energy consumed: `power_mw × slot_duration_s`
- Updates remaining energy: `energy_mj -= consumption`
- If energy depleted, mark device as DEAD

**Power Model**:
- Different power levels for each state reflect real hardware behavior
- ACTIVE (180 mW): Radio transmitting/receiving
- IDLE (8 mW): Radio on but not transmitting
- SLEEP (0.02 mW): Deep sleep mode (minimal power)
- WAKEUP (40 mW): Transitioning from sleep to active

##### `on_tx_result(success, slot_start)`
- Called by channel when slot resolution completes
- **Success**: 
  - Remove packet from queue (FIFO)
  - Calculate delay: `(slot_start + 1) - arrival_time`
  - Record delay sample
- **Collision**: Increment collision counter

**Delay Calculation**:
- Delay measured in slots from arrival to completion
- Completion time: `slot_start + 1.0` (end of slot)
- Converts to milliseconds: `delay_slots × slot_duration_ms`

##### `mean_delay_ms` (property)
- Calculates average delay across all successfully transmitted packets
- Returns 0.0 if no packets transmitted

##### `estimate_lifetime_years(elapsed_slots)`
- Estimates battery lifetime based on current energy consumption rate
- Formula:
  ```
  elapsed_seconds = elapsed_slots × slot_duration_s
  energy_drawn = initial_energy - current_energy
  avg_power_mw = energy_drawn / elapsed_seconds
  lifetime_seconds = initial_energy / avg_power_mw
  lifetime_years = lifetime_seconds / (365 × 24 × 3600)
  ```

**Reasoning**: 
- Extrapolates current consumption rate to estimate total lifetime
- Assumes constant traffic pattern (may not hold for very long simulations)
- Returns `inf` if no energy consumed yet (division by zero protection)

---

## Simulation Flow

### Initialization Phase

1. Create SimPy environment
2. Create SlottedChannel (starts background slot resolution process)
3. Create N MTD nodes (each starts packet generation and slot processing processes)
4. All processes run concurrently

### Execution Phase

For each slot `t`:

1. **Slot Start (time = t)**:
   - All MTD nodes process slot `t`:
     - Consume energy
     - Update state machine
     - If has packets and decides to transmit, register with channel
   - Channel collects all transmission requests for slot `t`

2. **Slot End (time = t + 1)**:
   - Channel resolves slot `t`:
     - Check number of contenders
     - If 1 contender → success, notify node
     - If >1 contenders → collision, notify all nodes
   - Nodes receive results and update queues/stats

3. **Repeat** until `max_slots` reached

### Termination Phase

1. Simulation stops at `max_slots`
2. Collect statistics from all nodes and channel
3. Calculate aggregate metrics
4. Generate plots

---

## Utility Functions

### `summarize(nodes, channel, config)`

**Purpose**: Aggregates statistics from all nodes and channel into a structured dictionary.

**Output Structure**:
```python
{
    "config": SimulationConfig object,
    "channel": {
        "slots": int,
        "successes": int,
        "collisions": int
    },
    "node_metrics": [
        {
            "id": int,
            "state": str,
            "queue_len": int,
            "tx_success": int,
            "tx_attempts": int,
            "tx_collisions": int,
            "mean_delay_ms": float,
            "estimated_lifetime_years": float,
            "residual_energy_mj": float
        },
        ...
    ],
    "summary": {
        "mean_delay_ms": float,
        "median_delay_ms": float,
        "median_lifetime_years": float,
        "throughput_packets_per_slot": float
    }
}
```

**Aggregate Metrics**:
- **Mean/Median Delay**: Average delay across all nodes with successful transmissions
- **Median Lifetime**: Median estimated lifetime across all nodes
- **Throughput**: Successful packets per slot (channel capacity utilization)

### `run_simulation(config, seed)`

**Purpose**: Convenience function to run a complete simulation.

**Steps**:
1. Set random seed for reproducibility
2. Create SimPy environment
3. Create channel and nodes
4. Run simulation until `max_slots`
5. Return summarized results

**Reasoning**: Encapsulates simulation setup for easy parameter sweeps and batch runs.

### `plot_delay_lifetime(results, max_points)`

**Purpose**: Visualizes the delay-lifetime trade-off.

**Plot Details**:
- X-axis: Estimated lifetime (years)
- Y-axis: Mean delay per node (ms)
- Scatter plot with up to `max_points` samples (randomly sampled if more nodes)
- Grid for readability

**Reasoning**: 
- The delay-lifetime trade-off is the core research question
- Scatter plot shows distribution of node performance
- Sampling prevents overcrowded plots with thousands of nodes

---

## Key Algorithms and Design Decisions

### 1. On-Demand Sleep Mechanism

**Algorithm**:
```
IF packet_queue.empty():
    IF state == ACTIVE:
        state = IDLE
        idle_timer = t_s
    ELIF state == IDLE:
        idle_timer--
        IF idle_timer == 0:
            state = SLEEP
ELSE:  # Has packets
    IF state == SLEEP:
        state = WAKEUP
        wakeup_timer = t_w
    state = ACTIVE
    IF random() < q_tx:
        attempt_transmission()
```

**Benefits**:
- Devices sleep when idle, saving energy
- Wake up immediately when packets arrive
- Configurable idle timer (t_s) balances latency vs. energy

**Trade-offs**:
- Longer t_s → more energy (stays in IDLE longer) but lower latency (faster wake-up)
- Shorter t_s → less energy but higher latency (must wake from SLEEP)

### 2. Slotted Aloha with Probability q_tx

**Algorithm**:
```
FOR each slot:
    FOR each node with packets:
        IF random() < q_tx:
            request_transmission()
    
    IF num_transmissions == 1:
        success()
    ELIF num_transmissions > 1:
        collision()
```

**Benefits**:
- Simple, distributed protocol (no coordination needed)
- Probability q_tx controls channel load
- Matches theoretical analysis

**Trade-offs**:
- Low q_tx → fewer collisions but higher delay (packets wait longer)
- High q_tx → more collisions, wasted energy

### 3. Energy Consumption Model

**Algorithm**:
```
FOR each slot:
    power = power_profile[state]
    energy_consumed = power × slot_duration
    remaining_energy -= energy_consumed
    IF remaining_energy <= 0:
        state = DEAD
```

**Modeling Assumptions**:
- Constant power per state (simplified model)
- No energy for state transitions (negligible compared to slot duration)
- Battery capacity fixed (no recharging)

**Real-World Considerations**:
- Actual power varies with transmission power, distance, etc.
- State transitions may consume additional energy
- Battery self-discharge not modeled

### 4. Delay Calculation

**Algorithm**:
```
packet_arrival_time = queue.append(env.now)
...
transmission_success_time = slot_start + 1.0
delay = transmission_success_time - packet_arrival_time
```

**Components of Delay**:
1. **Queueing delay**: Time waiting in queue before transmission attempt
2. **Backoff delay**: Time until successful transmission (due to collisions)
3. **Wake-up delay**: If device was sleeping when packet arrived

**Reasoning**: Total delay captures all sources of latency, enabling comprehensive analysis.

---

## Usage Examples

### Basic Simulation

```python
config = SimulationConfig(
    num_nodes=1000,
    lambda_arrival=2e-3,
    q_tx=0.05,
    idle_timer_slots=20,
    wake_time_slots=5,
    max_slots=20000
)

results = run_simulation(config, seed=2025)
print(f"Mean delay: {results['summary']['mean_delay_ms']:.2f} ms")
print(f"Median lifetime: {results['summary']['median_lifetime_years']:.2f} years")
```

### Parameter Sweep

```python
idle_timers = [10, 15, 20, 25, 30]
for t_s in idle_timers:
    config = SimulationConfig(idle_timer_slots=t_s, ...)
    results = run_simulation(config)
    print(f"t_s={t_s}: delay={results['summary']['mean_delay_ms']:.2f}ms, "
          f"lifetime={results['summary']['median_lifetime_years']:.2f}yr")
```

### Custom Power Profile

```python
config = SimulationConfig(
    power_profile_mw={
        "ACTIVE": 200.0,  # Higher transmission power
        "IDLE": 10.0,
        "SLEEP": 0.01,    # Ultra-low power sleep
        "WAKEUP": 50.0,
        "DEAD": 0.0
    }
)
```

---

## Performance Considerations

### Scalability

- **Time Complexity**: O(N × S) where N = nodes, S = slots
- **Space Complexity**: O(N) for node storage
- **Typical Performance**: 
  - 1000 nodes × 20,000 slots ≈ 10 seconds
  - 10,000 nodes × 1,000,000 slots ≈ 5-10 minutes

### Optimization Opportunities

1. **Vectorization**: Use NumPy arrays for bulk operations
2. **Parallelization**: Run multiple simulations in parallel
3. **Early Termination**: Stop if all nodes die or energy depleted
4. **Sampling**: Only track statistics for subset of nodes

---

## Limitations and Future Extensions

### Current Limitations

1. **Simplified Channel Model**: 
   - No fading, interference, or path loss
   - Binary collision model (perfect or collision)

2. **Fixed Power Profile**:
   - Constant power per state
   - No transmission power control

3. **Single Channel**:
   - No frequency diversity or multi-channel access

4. **No Retransmissions**:
   - Collided packets are lost (no retry mechanism)

5. **Homogeneous Nodes**:
   - All nodes have same parameters (could vary)

### Potential Extensions

1. **Advanced MAC Protocols**:
   - CSMA/CA, TDMA, or hybrid schemes
   - Adaptive transmission probability

2. **Realistic Channel Models**:
   - Path loss, shadowing, fading
   - SINR-based success probability

3. **Heterogeneous Networks**:
   - Different node types (sensors, actuators)
   - Varying traffic patterns

4. **Energy Harvesting**:
   - Solar, vibration, or RF energy harvesting
   - Dynamic battery capacity

5. **Network Topology**:
   - Multi-hop communications
   - Relay nodes and routing

---

## References and Standards

- **3GPP MICO Mode**: 3GPP TS 23.501 - Mobile Initiated Connection Only
- **RA-SDT**: Random Access - Small Data Transmission
- **T3324 Timer**: Idle mode timer in 3GPP specifications
- **Wang et al. 2024**: Reference paper on on-demand sleep mechanisms

---

## Conclusion

This simulator provides a flexible framework for studying sleep-based low-latency access in M2M communications. By modeling the fundamental trade-off between energy efficiency and latency, it enables researchers and engineers to:

1. **Understand** the impact of key parameters (t_s, q_tx, λ, N)
2. **Optimize** system parameters for specific use cases
3. **Compare** different sleep strategies and MAC protocols
4. **Validate** analytical models with simulation results

The modular design makes it easy to extend and adapt for specific research questions or deployment scenarios.

