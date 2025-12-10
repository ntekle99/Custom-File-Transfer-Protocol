# Architecture of the Custom File Transfer Protocol

Our file transfer system is built on a **custom UDP-based reliable transport protocol** designed to handle packet loss, high latency, and bandwidth constraints. The architecture follows a **client-server model**, with multi-threaded execution on both sides to achieve efficient parallelism between sending, receiving, and retransmission logic.

---

## High-Level Design Overview

### Design Philosophy
A UDP-based reliable file transfer protocol using **NACK-based retransmission**. The system uses multi-threading to parallelize sending, receiving, and retransmission operations, achieving high throughput even under adverse network conditions.

### System Architecture

```
┌─────────────────┐                    ┌─────────────────┐
│     CLIENT      │                    │     SERVER      │
│   (Sender)      │                    │   (Receiver)    │
└─────────────────┘                    └─────────────────┘
         │                                       │
         │  UDP Packets (fire-and-forget)       │
         ├──────────────────────────────────────>│
         │                                       │
         │  NACK Packets (missing packets)      │
         │<─────────────────────────────────────┤
         │                                       │
         │  Retransmitted Packets               │
         ├──────────────────────────────────────>│
         │                                       │
         │  Completion Signal (NACK -1)         │
         │<─────────────────────────────────────┤
```

### Protocol Flow

**Phase 1: Initial Transmission**
- Client preloads file → splits into packets → loads into stack
- Client sender thread sends all packets rapidly via UDP
- Server receiver thread receives packets, detects gaps, enqueues missing packets

**Phase 2: Retransmission**
- Server sender thread detects expired missing packets (>400ms) → sends NACK
- Client NACK thread receives NACK → pushes packet back onto stack
- Client sender thread retransmits requested packet
- Server receiver thread receives retransmission → removes from queue

**Phase 3: Completion**
- Server: All packets received + queue empty → sends completion signal (NACK -1)
- Client: Receives completion → stops threads → calculates throughput
- Server: Writes file → computes MD5 → verifies integrity

### Key Design Decisions

1. **NACK-Based (Not ACK-Based)**
   - Only requests missing packets, reducing bandwidth overhead
   - More efficient than ACK-based protocols in high-loss scenarios

2. **Multi-Threading**
   - Parallel sending and receiving operations
   - Overlaps transmission with retransmission handling
   - Maximizes throughput

3. **Preloading**
   - All packets loaded into memory before transfer
   - Enables fast retransmission (no disk I/O during transfer)
   - Trade-off: higher memory usage

4. **Out-of-Order Handling**
   - Server accepts packets in any order
   - Reassembly map enables correct ordering
   - Handles network reordering gracefully

5. **Timeout-Based NACK**
   - Waits 2×RTT before requesting retransmission
   - Allows late packets to arrive naturally
   - Reduces unnecessary retransmissions

6. **Thread-Safe Data Structures**
   - Mutexes protect all shared state
   - Prevents race conditions
   - Ensures data integrity

### Performance Characteristics

- **Throughput**: High (UDP bulk transmission)
- **Reliability**: High (NACK-based retransmission)
- **Latency Tolerance**: Good (handles high RTT)
- **Memory Usage**: High (preloads entire file)
- **Bandwidth Efficiency**: Good (only requests missing packets)

### Trade-offs

| Aspect | Benefit | Cost |
|--------|---------|------|
| UDP | Fast, no connection overhead | No built-in reliability |
| Preloading | Fast retransmission | High memory usage |
| NACK-based | Bandwidth efficient | More complex than ACK |
| Multi-threading | Parallel operations | Synchronization overhead |
| Out-of-order | Handles network reordering | Requires reassembly buffer |

---

---

## Client Side (Sender)

### Preload and Segmentation
- Reads the input file.
- Splits it into fixed-size packets (1024 bytes per packet, minus headers).
- Pushes packets into a buffer.

### Sender Thread
- Dedicated thread continuously transmits packets from a stack to the server over UDP.  
- Operates as a “fire-and-forget” worker, ensuring the entire file is initially transmitted as quickly as possible.

### NACK Receiver Thread
- Listens for **NACKs** (Negative Acknowledgments) from the server.  
- Reinserts requested packets into the stack for retransmission.  
- Stops when the server sends a completion signal (`NACK = -1`).

---

## Server Side (Receiver)

### Receiver Thread
- Accepts incoming packets and reassembles them into a **hash map** keyed by sequence number.  
- Detects missing sequence numbers and enqueues them into a **lost packet queue**.  
- Removes entries from the queue when retransmitted packets arrive.

### Sender Thread (NACK Generator)
- Periodically checks the lost packet queue.  
- If a packet hasn’t arrived within **2× RTT threshold**, sends a NACK to request retransmission.  
- Avoids wasting bandwidth with unnecessary acknowledgments.  
- Once all packets are received, sends a **completion signal** (`NACK = -1`).

---

## Data Structures

- **Reassembly Map**  
  `unordered_map<int, vector<char>>` for storing received packets by sequence number, enabling efficient in-order reconstruction.

- **Lost Packet Queue**  
  Custom `PacketQueue` with timestamping for retransmission timers.  
  - Prevents duplicate NACKs.  
  - Ensures fairness in packet recovery.

- **Packet Stack (Client)**  
  `stack<packet>` used to hold packets awaiting transmission or retransmission.

---

## Reliability and Verification

- After transfer completion, the server reassembles the file and computes an **MD5 checksum**.  
- Compares with the original file’s MD5 checksum for correctness.  
- Records **start and end timestamps** to measure throughput and evaluate protocol performance under varying RTT, loss, and MTU conditions.

---

## Summary

This design balances:
- **Speed**: Bulk UDP transmission.  
- **Reliability**: Selective NACK-based retransmission.  

By decoupling sender, receiver, and retransmission logic into separate threads and using efficient data structures, the protocol sustains **high throughput** even under **high RTT** and **packet loss** conditions.
