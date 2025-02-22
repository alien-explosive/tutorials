### **Actual Throughput for L2CAP CoC Based on the Article**  

L2CAP CoC provides **higher throughput** than GATT notifications due to **larger packet sizes (MPS)** and **credit-based flow control**. Based on the article’s **theoretical calculations for unacknowledged data transfer**, we can estimate the **actual L2CAP CoC throughput**.

---

## **1. Key Factors Affecting L2CAP CoC Throughput**
To achieve maximum throughput, we consider:
- **Physical Layer (PHY) Bitrate**:  
  - **1 Mbps (Bluetooth 4.2)**
  - **2 Mbps (Bluetooth 5)**
- **Maximum Protocol Data Unit (PDU) Size**:  
  - Up to **251 bytes** (BLE 5.0+ standard)
  - **L2CAP CoC typically supports MPS = 512 bytes**
- **Connection Interval**:  
  - Typically **7.5 ms to 12.5 ms** (lower = better)
- **Packet Overhead**:  
  - **L2CAP Header (4 bytes)**
  - **No GATT header (L2CAP CoC does not use ATT)**
- **Number of Packets per Connection Interval**:  
  - A function of PHY bitrate and connection interval.

---

## **2. Maximum Theoretical Throughput for L2CAP CoC**
From the article, BLE 5.0 at **2 Mbps PHY** achieves:
- **251-byte PDU** takes **1400 µs** to transmit.
- **8 packets fit into a 12.5 ms connection interval**.

Using this logic, for **L2CAP CoC**, we apply a similar formula with **larger MPS (e.g., 512 bytes)**:

### **🔹 Case 1: L2CAP CoC with 1 Mbps PHY (Bluetooth 4.2)**
- **MPS (PDU) = 512 bytes**
- **Each packet takes ≈ 2500 µs** to send (based on article timing)
- **4 packets fit in a 12.5 ms interval**
- **Throughput Calculation**:

\[
4 \times (512 - 4) \text{ bytes} / 12,500 \text{ µs}
\]

\[
= 162,048 \text{ bytes/sec} = 158.3 \text{ KB/sec}
\]

---

### **🔹 Case 2: L2CAP CoC with 2 Mbps PHY (Bluetooth 5.0)**
- **MPS (PDU) = 512 bytes**
- **Each packet takes ≈ 1400 µs** to send
- **8 packets fit in a 12.5 ms interval**
- **Throughput Calculation**:

\[
8 \times (512 - 4) \text{ bytes} / 12,500 \text{ µs}
\]

\[
= 324,096 \text{ bytes/sec} = 316.7 \text{ KB/sec}
\]

---

## **3. Comparison with GATT Notifications**
| **Method** | **1 Mbps PHY Throughput (KB/sec)** | **2 Mbps PHY Throughput (KB/sec)** |
|------------|----------------------------------|----------------------------------|
| **GATT Notifications (MTU 251)** | **95.3 KB/sec** | **173.8 KB/sec** |
| **L2CAP CoC (MPS 512)** | **158.3 KB/sec** | **316.7 KB/sec** |

✔️ **L2CAP CoC is ~1.7x to 3.3x faster than GATT notifications!** 🚀  

---

## **4. Real-World Considerations**
Although theoretical speeds are **158 KB/sec (BLE 4.2) and 317 KB/sec (BLE 5.0)**, real-world throughput is affected by:
- **Interference and retransmissions** (lower PHY speeds in noisy environments)
- **Smartphone BLE stack limitations** (not all phones support 2 Mbps PHY)
- **CPU and buffer processing delays** on both devices
- **Connection interval negotiation** (phones may use **30 ms+ intervals**, reducing throughput)

🔹 **Realistic Expected Throughput:**
- **BLE 4.2 (1 Mbps PHY):** **~120 KB/sec**
- **BLE 5.0 (2 Mbps PHY):** **~250 KB/sec**

---

## **5. Conclusion: How Fast is L2CAP CoC?**
- **Theoretical max (2 Mbps PHY, 512-byte MPS):** **317 KB/sec**
- **Real-world expected:** **120–250 KB/sec**
- **Compared to GATT Notifications:** **2–3× faster**
- **Ideal for:** **Audio streaming, firmware updates, bulk data transfer**.

Would you like help **benchmarking this on your device**? 🔍🚀
