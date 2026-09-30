📸 🖥️ OTSU THRESHOLDING HW/SW CO-DESIGN ON ZYNQ FPGA 🖥️ 📸

========================================================================

📖 OVERVIEW

---

This project implements Otsu’s automatic image thresholding algorithm using a hardware/software co-design architecture on a Xilinx/AMD Zynq platform.

The goal? To demonstrate how an image-processing algorithm can be perfectly partitioned between the FPGA Programmable Logic (PL) and the ARM Processing System (PS) based on what each does best.

Otsu’s method automatically determines an optimum threshold to convert an 8-bit grayscale image into a binary image:
⬛ 0 = Black
⬜ 255 = White

The secret sauce: The design uses the FPGA for highly parallel, streaming operations, while the ARM processor handles the control-oriented, division-heavy math.

⚖️ HARDWARE / SOFTWARE PARTITIONING

---

⚙️ FPGA PROGRAMMABLE LOGIC (HARDWARE)
The FPGA is built for speed. It handles:
🔹 Histogram generation
🔹 Four-pixel-per-clock processing
🔹 Cumulative & weighted cumulative histogram generation
🔹 Pixel threshold comparison & binary image generation
🔹 AXI4-Stream data processing
🔹 FSM-based accelerator control
🔹 FIFO-based backpressure handling

🧠 ARM PROCESSING SYSTEM (SOFTWARE)
The ARM processor acts as the brains. It handles:
🔸 Accelerator & DMA configuration
🔸 Histogram data retrieval
🔸 Class mean & between-class variance calculation
🔸 Search for the optimum Otsu threshold
🔸 Writing the calculated threshold back to the FPGA via AXI-Lite

🏛️ SYSTEM ARCHITECTURE

---

```
   [ ARM Processor ]
   [ Otsu Threshold Calculation ]
           |
       (AXI-Lite)
           |
           v

```

[ DDR Memory ] <==== (AXI DMA) ====> [ Custom Otsu FPGA IP ]
- Histogram Calc
- Cumulative Histogram
- Weighted Histogram
- Thresholding
- FSM
- Output FIFO

💾 The original image lives in external DDR memory. The ARM processor configures the AXI DMA, allowing the DMA engine to move massive blocks of data directly to the FPGA without the ARM lifting a finger.

🔄 OVERALL PROCESSING FLOW

---

Here is the complete journey of a single image:

1️⃣  Store grayscale image in DDR
2️⃣  Configure image size and accelerator registers
3️⃣  Reset histogram logic
4️⃣  Configure AXI DMA
5️⃣  Stream image from DDR to FPGA
6️⃣  FPGA calculates histogram
7️⃣  FPGA generates cumulative histogram
8️⃣  FPGA generates weighted cumulative histogram
9️⃣  Histogram information is transferred back to memory
🔟  ARM processor calculates the optimum Otsu threshold
1️⃣1️⃣ ARM writes threshold into an AXI-Lite register
1️⃣2️⃣ Original image is streamed through FPGA again
1️⃣3️⃣ FPGA compares every pixel against the threshold
1️⃣4️⃣ Binary image is generated
1️⃣5️⃣ Binary image is transferred back to DDR

✌️ WHY THE IMAGE IS PROCESSED TWICE

---

🔍 FIRST PASS (Analysis):
Image -> Histogram -> Cumulative Histogram -> Weighted Cumulative -> ARM Calc
We can't apply the threshold yet because we don't know the optimum value until the entire image is analyzed!

🎬 SECOND PASS (Thresholding):
After the ARM does its math, the image goes back in:
Original Image -> FPGA Thresholder -> Binary Image

🏎️ 32-BIT STREAMING DATAPATH

---

Each grayscale pixel is 8 bits. The accelerator uses a massive 32-bit AXI4-Stream datapath.

[ Pixel 3 (8b) | Pixel 2 (8b) | Pixel 1 (8b) | Pixel 0 (8b) ]

Result? 32 bits / 8 bits = 4 pixels per clock cycle!
This allows four parallel pixels to be processed simultaneously.

📊 PARALLEL HISTOGRAM CALCULATION

---

An 8-bit image has 256 possible intensity levels (0 to 255).
The design uses parallel histogram logic for EVERY grayscale value.

Example: Input pixels are [ 20, 40, 20, 20 ]
In a SINGLE processing cycle, the logic does:
📈 histogram[20] += 3
📈 histogram[40] += 1

This intentionally trades FPGA resources for massive throughput.

➕ CUMULATIVE HISTOGRAM:
Calculated as C(t) = Sum of H(i) for i = 0 to t.
This lets the software quickly determine the number of pixels in a class.

⚖️ WEIGHTED CUMULATIVE HISTOGRAM:
Calculated as W(t) = Sum of (i × H(i)) from i = 0 to t.
Used to calculate the mean intensity of each class.

⚠️ Why isn't the Weighted Histogram fully parallel?
A fully parallel version would need 256 multipliers working at once! Since data goes to software sequentially anyway, we reuse hardware via a multiply-accumulate structure.
💡 Golden Rule: Parallelize when it improves throughput; reuse hardware when parallelism just wastes space.

🧮 OTSU THRESHOLD CALCULATION (SOFTWARE)

---

The FPGA sends 512 total values (256 cumulative + 256 weighted) to the ARM processor.

The ARM evaluates thresholds (0 to 255):
🔹 Class 0 = Pixels below/around threshold
🔹 Class 1 = Pixels above threshold
🔹 Calculates probability/size and mean intensity for both classes.
🔹 Calculates between-class variance: sigma_b^2 = w0 × w1 × (mu0 - mu1)^2

The threshold producing the MAX variance wins. 🏆
Why software? Because this involves division, iterative searching, and conditional logic—tasks the ARM loves and the FPGA hates.

🔌 AXI INTERFACES & HARDWARE MECHANICS

---

🌊 AXI4-STREAM (The Data Plane)
Used for massive data movement (Image & Histograms).

* TVALID: Sender says "My data is valid."
* TREADY: Receiver says "I am ready."
* Transfer only happens when BOTH are 1!
* TLAST: Signals the final transfer of a packet.
* TKEEP: Indicates which bytes are valid (handy if image size isn't divisible by 4).

🛑 AXI BACKPRESSURE & OUTPUT FIFO
If the DMA temporarily says TREADY = 0 (not ready), the FPGA must hold its data. To prevent stalling, an AXI-Stream FIFO buffers the data, decoupling the processing pipeline from the AXI interface.

🕹️ AXI-LITE (The Control Plane)
Used by the ARM for small configurations (Soft Reset, Image Size, sending the Final Threshold).

💾 EXTERNAL DDR MEMORY
A 1080p image is ~2MB. Too big for FPGA Block RAM! Keeping it in DDR and using DMA to stream it is the smartest architectural choice.

⏱️ TIMING & THROUGHPUT

---

Processing 4 pixels per clock at 100 MHz means:
🚀 400,000,000 pixels per second (400 Megapixels/sec!)
*(Theoretical datapath throughput, excluding system overhead).*

Timing constraints? The weighted multiply-accumulate path can create long combinational delays. We solve this using Pipelining (adding registers between Multiply and Add stages) to keep clock speeds high.

🥊 RTL VS VITIS HLS (HIGH-LEVEL SYNTHESIS)

---

This project explored both traditional coding (RTL) and C/C++ hardware generation (HLS).

🛠️ RTL (Verilog)
✅ Exact cycle-level control & explicit architecture.
✅ Highly predictable datapath.
❌ Longer development time & massive verification effort.

🪄 HLS (C/C++)
✅ Fast algorithm development & prototyping.
✅ Easy to use directives (PIPELINE, UNROLL).
❌ Can produce inefficient hardware if coded poorly.
❌ Still requires deep hardware understanding to use well.

🧪 VERIFICATION STRATEGY

---

Tested via RTL simulation against software reference models. Boundary conditions tested:
✔️ All black / All white images
✔️ Identical vs mixed pixels in a single 32-bit word
✔️ Edge cases: pixel < threshold vs pixel <= threshold
✔️ Image sizes not perfectly divisible by four
✔️ Intense AXI backpressure scenarios

🚀 POSSIBLE IMPROVEMENTS

---

Looking ahead, this project could feature:
▪️ Full hardware benchmarking (ARM-only vs HW/SW comparison)
▪️ Resource utilization analysis (LUTs, DSPs, BRAM)
▪️ Wider 8-pixel-per-clock AXI datapath
▪️ RGB-to-Grayscale FPGA preprocessing
▪️ Linux driver & interrupt-driven completion integration

🎓 MAIN ENGINEERING LESSONS

---

FPGA acceleration is NOT just copy-pasting software into hardware. A brilliant HW/SW co-design requires asking:
❓ What benefits from parallel hardware?
❓ What is easier in software?
❓ How do we move data efficiently without bottlenecks?

By splitting Parallel Histogramming (FPGA) and Variance Math (ARM), this project acts as a perfect real-world example of heterogeneous computing on Zynq!

📝 PROJECT SUMMARY

---

🎯 Goal: Otsu Image Thresholding
🖥️ Platform: AMD/Xilinx Zynq SoC
⚙️ Hardware: Custom FPGA accelerator (Verilog)
🧠 Software: ARM application (Vitis / C)
🔌 Interfaces: AXI4-Stream, AXI-Lite, AXI DMA
🏎️ Speed: 4 pixels per clock
📊 Math: 256 Histogram Levels
🖼️ Output: 8-bit Binary Image (0=Black, 255=White)
