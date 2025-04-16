### 📥 I/O Techniques

1. **Programmed I/O**:  
   - CPU actively waits in a loop to read/write each byte or word  
2. **Interrupt-Driven I/O**:  
   - CPU initiates transfer, then gets interrupted upon completion  
3. **DMA (Direct Memory Access)**:  
   - Dedicated controller handles block transfers with a final interrupt to CPU  

---

### 📚 I/O Software Structure

- **Four-layered I/O architecture**:
  1. **Interrupt Handlers**
  2. **Device Drivers** – Interface with hardware
  3. **Device-Independent I/O Software** – Buffering, naming, error handling
  4. **User-level Libraries & Spoolers** – Abstracted interfaces in user space

---

### 💾 Storage Devices

- **Disk types**: Magnetic disks, SSDs, flash, RAID arrays  
- **Rotational disks**: Performance improved via **disk arm scheduling**
- **RAID**: Offers redundancy & parallel I/O (RAID levels 0–6)
- **Stable storage**: Achieved by duplication & atomic writes

---

### ⏲️ Clocks

Used for:
- Keeping **real-time**
- **Time slicing** (process quantum)
- **Watchdog timers**
- **CPU usage accounting**

---

### ⌨️ Character-Based I/O

- **Input modes**:
  - **Raw Mode**: Delivers every key press
  - **Cooked Mode**: Line-oriented, with editing support
- **Special input characters**: e.g. CTRL-C (SIGINT), CTRL-D (EOF), etc.
- **Output via escape sequences**: Cursor control, clearing screen, etc.

---

### 🪟 Graphical Interfaces

- **X Window System (UNIX)**:
  - Client-server architecture
  - X Server: Local display manager
  - X Clients: Remote/local apps using Xlib, Motif, GTK+, or Qt

- **GUIs follow WIMP model**:
  - **Windows, Icons, Menus, Pointing devices**
  - Event-driven programming model

---

### 💻 Thin Clients

- Lightweight terminals with minimal software  
- Benefit: **Low maintenance**, **centralized control**
- Example: **Chromebooks** (Web-centric, Android app support)

---

### 🔋 Power Management

- Important for:
  - **Mobile devices**: To conserve **battery**
  - **Desktops/Servers**: To reduce **power costs**

- OS strategies:
  - Turn off/sleep hardware
  - Use low-power CPU states (C-states, P-states)
  - Thermal & battery-aware scheduling

- **Apps can adapt**:
  - Lower resolution, grayscale, frame rate
  - Offload to cloud to reduce local computation

---
