OTSU THRESHOLDING HW/SW CO-DESIGN ON ZYNQ FPGA

OVERVIEW

This project implements Otsu’s automatic image thresholding algorithm using a hardware/software co-design architecture on a Xilinx/AMD Zynq platform.

The purpose of the project is to demonstrate how an image-processing algorithm can be partitioned between the FPGA Programmable Logic (PL) and the ARM Processing System (PS) according to the characteristics of each computation.

Otsu’s method automatically determines an optimum threshold for converting an 8-bit grayscale image into a binary image.

For an 8-bit grayscale image, pixel values range from:

0 to 255

The final binary image contains:

0 = Black
255 = White

The design uses the FPGA for highly parallel and streaming operations, while the ARM processor handles the control-oriented and division-heavy threshold calculation.


HARDWARE / SOFTWARE PARTITIONING

FPGA PROGRAMMABLE LOGIC

The FPGA handles:

Histogram generation
Four-pixel-per-clock processing
Cumulative histogram generation
Weighted cumulative histogram generation
Pixel threshold comparison
Binary image generation
AXI4-Stream data processing
FSM-based accelerator control
FIFO-based backpressure handling


ARM PROCESSING SYSTEM

The ARM processor handles:

Accelerator configuration
DMA control
Histogram data retrieval
Class mean calculation
Between-class variance calculation
Search for the optimum Otsu threshold
Writing the calculated threshold into the FPGA through AXI-Lite

The main idea of the co-design is to place parallel streaming operations in hardware and keep control-heavy and division-heavy calculations in software.


SYSTEM ARCHITECTURE

ARM Processor
Otsu Threshold Calculation

        |
        |
     AXI-Lite
        |
        v

DDR Memory <----> AXI DMA <----> Custom Otsu FPGA IP

The custom FPGA IP contains:

Histogram Calculation
Cumulative Histogram
Weighted Histogram
Thresholding
FSM
Output FIFO

The input grayscale image is stored in external DDR memory.

The image is transferred between DDR and the custom FPGA accelerator using AXI DMA.

The ARM processor does not have to manually transfer every pixel. Instead, it configures the DMA and allows the DMA engine to move large blocks of data.


OVERALL PROCESSING FLOW

1. Store grayscale image in DDR

2. Configure image size and accelerator registers

3. Reset histogram logic

4. Configure AXI DMA

5. Stream image from DDR to FPGA

6. FPGA calculates histogram

7. FPGA generates cumulative histogram

8. FPGA generates weighted cumulative histogram

9. Histogram information is transferred back to memory

10. ARM processor calculates the optimum Otsu threshold

11. ARM writes threshold into an AXI-Lite register

12. Original image is streamed through FPGA again

13. FPGA compares every pixel against the threshold

14. Binary image is generated

15. Binary image is transferred back to DDR


WHY THE IMAGE IS PROCESSED TWICE

FIRST PASS

The first pass is used to analyse the image.

Image
|
v
Histogram Calculation
|
v
Cumulative Histogram
|
v
Weighted Cumulative Histogram
|
v
ARM Threshold Calculation

The threshold cannot be applied during this pass because the optimum threshold is not known until the histogram of the complete image has been analysed.


SECOND PASS

After the ARM processor calculates the threshold, the original image is streamed through the FPGA again.

Original Image
|
v
FPGA Thresholder
|
v
Binary Image


32-BIT STREAMING DATAPATH

Each grayscale pixel is represented using 8 bits.

The accelerator uses a 32-bit AXI4-Stream datapath.

Therefore, one AXI word contains four grayscale pixels.

32-bit AXI Stream Word

Pixel 3 | Pixel 2 | Pixel 1 | Pixel 0

8 bits | 8 bits | 8 bits | 8 bits

Therefore:

32 bits / 8 bits per pixel = 4 pixels per clock cycle

This allows the accelerator to process four pixels every clock cycle.

The 32-bit interface is also convenient because histogram-related values are transferred as 32-bit words.


PARALLEL HISTOGRAM CALCULATION

An 8-bit grayscale image contains 256 possible intensity levels.

0, 1, 2, 3, ... , 255

The design uses histogram logic corresponding to each grayscale value.

Each histogram unit receives the same four incoming pixels.

Each unit is configured for a different grayscale level.

For example, assume the incoming pixels are:

20, 40, 20, 20

The histogram logic performs:

histogram[20] += 3
histogram[40] += 1

during the same processing cycle.

This architecture intentionally trades FPGA resources for higher throughput.


HISTOGRAM MODULE OPERATION

For one grayscale level, the four incoming pixels are compared against that value.

Conceptually:

match0 = pixel0 == gray_level
match1 = pixel1 == gray_level
match2 = pixel2 == gray_level
match3 = pixel3 == gray_level

The increment value is then calculated:

increment = match0 + match1 + match2 + match3

Therefore the histogram counter can increase by:

0
1
2
3
or
4

during a single clock cycle.

Example:

Gray Level = 100

Input Pixels:

100, 20, 100, 100

Comparison result:

Pixel 0 -> Match
Pixel 1 -> No Match
Pixel 2 -> Match
Pixel 3 -> Match

Therefore:

histogram[100] += 3


CUMULATIVE HISTOGRAM

Once the complete image has been received, the accelerator generates the cumulative histogram.

For histogram H(i), the cumulative histogram is:

C(t) = H(0) + H(1) + ... + H(t)

or:

C(t) = Sum of H(i) for i = 0 to t

Example:

H[0] = 3
H[1] = 5
H[2] = 2
H[3] = 4

The cumulative histogram becomes:

C[0] = 3

C[1] = 3 + 5
     = 8

C[2] = 3 + 5 + 2
     = 10

C[3] = 3 + 5 + 2 + 4
     = 14

The cumulative histogram allows the software to quickly determine the number of pixels belonging to a class for a candidate threshold.


WEIGHTED CUMULATIVE HISTOGRAM

Otsu’s algorithm also requires information about the intensity distribution of the pixels.

The weighted cumulative histogram is calculated as:

W(t) = Sum of i × H(i), from i = 0 to t

Example:

Intensity    Histogram    Weighted Value

0            3            0 × 3 = 0
1            5            1 × 5 = 5
2            2            2 × 2 = 4

Therefore:

W(2) = 0 + 5 + 4
     = 9

This weighted value is used to calculate the mean intensity of each class.


WHY WEIGHTED HISTOGRAM CALCULATION IS NOT FULLY PARALLEL

A completely parallel weighted histogram architecture could require a multiplier for almost every grayscale level.

Conceptually:

0 × H[0]
1 × H[1]
2 × H[2]
...
255 × H[255]

This could require a very large number of multipliers.

However, the histogram values are ultimately transmitted sequentially through the streaming interface.

Therefore, implementing all multiplications simultaneously would increase FPGA resource usage without providing an equivalent improvement in system-level throughput.

For this reason, the weighted cumulative histogram is calculated sequentially using a multiply-accumulate structure.

This demonstrates an important FPGA design principle:

Parallelize operations when parallelism improves throughput.

Reuse hardware when additional parallelism only increases resource usage.


HISTOGRAM DATA SENT TO SOFTWARE

After the first image pass, the FPGA sends:

256 cumulative histogram values

plus

256 weighted cumulative histogram values

which gives:

512 total values

Each value is transferred as a 32-bit word.

Therefore:

512 × 32 bits

of histogram information is transferred to the software.

The ARM processor then uses these values to determine the optimum Otsu threshold.


OTSU THRESHOLD CALCULATION

The ARM processor evaluates possible threshold values from:

0 to 255

For each candidate threshold T, the image is separated into two classes.

Class 0 = pixels below or around the threshold

Class 1 = pixels above the threshold

The algorithm calculates:

w0 = size/probability of class 0

w1 = size/probability of class 1

mu0 = mean intensity of class 0

mu1 = mean intensity of class 1

The between-class variance is then calculated using:

sigma_b^2 = w0 × w1 × (mu0 - mu1)^2

The software repeats this calculation for all possible threshold values.

The threshold producing the maximum between-class variance is selected as the optimum Otsu threshold.

Conceptually:

best_variance = 0
best_threshold = 0

for threshold = 0 to 255:

    calculate class 0

    calculate class 1

    calculate mu0

    calculate mu1

    calculate between-class variance

    if variance > best_variance:

        best_variance = variance
        best_threshold = threshold


WHY THRESHOLD CALCULATION RUNS IN SOFTWARE

The threshold calculation includes operations such as:

Division
Iterative threshold evaluation
Conditional logic
Maximum-value searching
Control-heavy processing

These operations are convenient to perform using the ARM processor.

The FPGA is more useful for the parts that contain:

Parallel comparisons
Repetitive calculations
Continuous data streaming
Fixed datapaths
Deterministic processing

Therefore, the project uses both hardware and software according to their strengths.


BINARY THRESHOLDING

Once the optimum threshold has been calculated, it is written into an FPGA register.

The original image is then streamed through the accelerator again.

Each pixel is compared against the threshold.

Conceptually:

if pixel < threshold:

    output_pixel = 0

else:

    output_pixel = 255

Where:

0 = Black
255 = White

Since four pixels arrive in every 32-bit AXI word, four comparisons can be performed in parallel.

Pixel 0 ---> Comparator ---> 0 / 255
Pixel 1 ---> Comparator ---> 0 / 255
Pixel 2 ---> Comparator ---> 0 / 255
Pixel 3 ---> Comparator ---> 0 / 255

Therefore, the thresholding hardware can process:

4 pixels per clock cycle


AXI4-STREAM INTERFACE

AXI4-Stream is used for high-throughput data movement.

It is used for transferring:

Input image data
Histogram information
Binary output image data

Important AXI4-Stream signals include:

TDATA
TVALID
TREADY
TLAST
TKEEP

A successful transfer occurs when:

TVALID = 1

AND

TREADY = 1

on the active clock edge.


TVALID

TVALID is asserted by the sender when valid data is available on the AXI stream.

TVALID = 1

means:

The value currently present on TDATA is valid.


TREADY

TREADY is asserted by the receiver when it is ready to accept data.

TREADY = 1

means:

The receiver can accept the current transfer.

Actual transfer occurs when:

TVALID && TREADY


AXI BACKPRESSURE

Backpressure occurs when the receiving component temporarily cannot accept more data.

For example:

FPGA Output:

TVALID = 1

DMA:

TREADY = 0

The transfer cannot happen yet.

The data must remain valid until the receiver becomes ready.

This is why correct backpressure handling is important in streaming hardware.


OUTPUT FIFO

An AXI-Stream FIFO is placed between the accelerator processing logic and the DMA.

FPGA Processing Logic
|
v
AXI Stream FIFO
|
v
AXI DMA

The FIFO provides temporary buffering.

If the DMA temporarily becomes unavailable, the FPGA can continue writing data into the FIFO until the FIFO becomes full.

This decouples the internal processing pipeline from the downstream AXI interface.

The FIFO therefore simplifies backpressure handling.


TLAST

TLAST indicates the final transfer of an AXI packet or transaction.

The DMA can use TLAST to identify the end of a transfer.

For example:

Histogram Value 0
Histogram Value 1
...
Histogram Value 511

TLAST = 1

on the final data beat indicates that the histogram transfer has completed.


TKEEP

TKEEP indicates which bytes of a stream word contain valid data.

The AXI datapath is 32 bits wide:

4 bytes per AXI word

If the total image size is not divisible by four, the final AXI word may contain fewer than four valid bytes.

For example:

TDATA:

Byte 3 | Byte 2 | Byte 1 | Byte 0

If only the first two bytes are valid:

TKEEP = 0011

This allows the system to correctly handle images whose sizes are not aligned to four bytes.


AXI-LITE INTERFACE

AXI-Lite is used as the control interface between the ARM processor and the FPGA accelerator.

It is used for small configuration values rather than large image transfers.

The processor can write values such as:

Soft Reset
Image Size
Threshold Value
Control Information

The calculated Otsu threshold is written from the ARM processor into an FPGA register using AXI-Lite.

A simple way to describe the interfaces is:

AXI4-Stream = Data Plane

AXI-Lite = Control Plane


AXI DMA

AXI DMA is used to transfer data between DDR memory and the FPGA accelerator.

Without DMA, the ARM processor would have to manually read and write every pixel.

With DMA:

The ARM processor configures the DMA.

The DMA transfers data directly between DDR and the FPGA.

This allows the processor to focus on the software part of the algorithm.


EXTERNAL DDR MEMORY

The input image is stored in external DDR memory.

For a Full-HD grayscale image:

1920 × 1080

the total number of pixels is:

2,073,600 pixels

Since each pixel uses one byte:

approximately 2 MB per grayscale frame

Keeping the complete image in external DDR is more practical than storing the whole image using FPGA Block RAM.

The image can then be streamed from DDR to the accelerator whenever required.


FSM CONTROL

A finite-state machine controls the main accelerator sequence.

A simplified state sequence is:

IDLE

RECEIVE IMAGE

SEND CUMULATIVE HISTOGRAM

SEND WEIGHTED HISTOGRAM

WAIT FOR SOFTWARE THRESHOLD

SECOND IMAGE PASS

THRESHOLD IMAGE

COMPLETE

The FSM coordinates:

Histogram generation
Histogram transmission
Waiting for software
Second-pass image processing
Final completion


PIXEL COUNTER

The software first provides the expected image size.

The FPGA uses a counter to determine how much data has been received.

Because the datapath is 32 bits wide and each pixel is 8 bits:

4 pixels arrive per valid transfer

The counter is used by the FSM to determine when the complete image has been received.


SOFT RESET

The accelerator contains internal state that must be cleared before processing another image.

This includes:

Histogram counters
Pixel counter
FSM state
Cumulative histogram state
Weighted accumulator
Interrupt/status signals

A software-controlled soft reset allows the ARM processor to reset the accelerator without resetting the complete Zynq system.


RESOURCE VS THROUGHPUT TRADE-OFF

One of the main design decisions in this project is the trade-off between:

FPGA Resource Usage

versus

Processing Throughput

The histogram stage uses a highly parallel architecture.

This consumes more FPGA logic, but allows several pixels to be processed simultaneously.

The weighted histogram calculation reuses arithmetic hardware because a fully parallel architecture would use many multipliers without giving an equivalent throughput benefit.

This project therefore demonstrates that FPGA optimisation is not simply about maximizing parallelism.

The important question is:

Does additional parallelism actually improve system throughput enough to justify the hardware cost?


TIMING CONSIDERATIONS

One potential timing-sensitive operation is the weighted multiply-accumulate path.

Conceptually:

Histogram Value
|
v
Multiplier
|
v
Adder
|
v
Accumulator

If the multiplication and accumulation create a long combinational path, timing may fail at higher clock frequencies.

One possible solution is pipelining.

Example:

Cycle N:

Multiply
|
Register

Cycle N+1:

Add
|
Register

Pipelining increases latency but can improve the maximum clock frequency.


THROUGHPUT

The accelerator processes:

4 pixels per clock cycle

For example, if the design runs at:

100 MHz

the theoretical datapath throughput is:

4 × 100,000,000

=

400,000,000 pixels per second

or:

400 megapixels per second

This represents theoretical streaming throughput and does not include DMA stalls, memory bandwidth, software processing, or system overhead.


VIVADO DESIGN FLOW

Vivado is used for the FPGA hardware development.

The hardware development flow is:

Algorithm Analysis

HW/SW Partitioning

RTL Design

Functional Simulation

Custom IP Packaging

Zynq Block Design

AXI DMA Integration

AXI-Lite Integration

AXI-Stream Integration

Synthesis

Implementation

Timing Analysis

Bitstream Generation

Vivado is used for:

RTL design
Verilog implementation
Simulation
IP packaging
AXI interface integration
Zynq block design
AXI DMA integration
Synthesis
Implementation
Timing analysis
Bitstream generation


VITIS SOFTWARE DEVELOPMENT

Vitis is used to develop the embedded software running on the ARM Processing System.

The Vitis application performs the following operations:

1. Initialize the platform

2. Configure the custom FPGA accelerator

3. Configure image size

4. Reset histogram logic

5. Configure AXI DMA

6. Transfer the grayscale image from DDR to FPGA

7. Receive cumulative histogram information

8. Receive weighted cumulative histogram information

9. Calculate the optimum Otsu threshold in software

10. Write the threshold to the FPGA using AXI-Lite

11. Configure the second DMA transfer

12. Stream the original image through the FPGA again

13. Receive the binary output image

14. Check processing completion

The Vitis software therefore acts as the control layer of the HW/SW system.


VITIS HLS EXPLORATION

The project also explores how image-processing operations can be expressed using Vitis HLS.

Vitis HLS allows C/C++ algorithms to be converted into synthesizable FPGA hardware.

For streaming image processing, important HLS concepts include:

PIPELINE
UNROLL
ARRAY_PARTITION
AXI4-Stream interfaces
AXI4-Lite interfaces
Loop optimization
Initiation interval
Hardware resource analysis


HLS PIPELINE

The PIPELINE directive allows loop iterations to overlap.

Without pipelining:

Iteration 1 finishes

then

Iteration 2 starts

then

Iteration 3 starts

With pipelining:

Multiple loop iterations can occupy different pipeline stages at the same time.

The aim for a streaming design is typically to achieve:

Initiation Interval = 1

or:

II = 1

This means a new input can be accepted every clock cycle.


HLS UNROLL

The UNROLL directive can be used to replicate hardware and process multiple operations simultaneously.

Because one AXI word contains four pixels, the pixel operations can conceptually be unrolled:

Pixel 0 ---> Hardware Operation 0

Pixel 1 ---> Hardware Operation 1

Pixel 2 ---> Hardware Operation 2

Pixel 3 ---> Hardware Operation 3

This exposes parallelism to the HLS compiler.


HLS ARRAY_PARTITION

Histogram arrays can become a memory bottleneck if several pixels need to update the histogram simultaneously.

ARRAY_PARTITION can split an array into multiple hardware storage structures.

Original Histogram Array:

hist[0 ... 255]

can be partitioned into multiple independently accessible sections.

This can increase memory bandwidth and expose additional parallelism.

However, aggressive array partitioning can significantly increase FPGA resource usage.

Therefore, the partitioning factor must be chosen according to throughput and utilization requirements.


HLS AXI INTERFACES

An HLS accelerator can expose:

AXI4-Stream

for high-throughput input/output data.

For example:

AXI DMA

to

AXI4-Stream

to

HLS Accelerator

AXI-Lite can be used for configuration values such as:

Image Size
Threshold
Control Registers

After HLS synthesis, the generated IP can be exported into Vivado and connected to the Zynq Processing System and AXI DMA.


RTL VS HLS

This project also helped in understanding the trade-off between RTL design and High-Level Synthesis.


RTL ADVANTAGES

Exact cycle-level control
Explicit hardware architecture
Fine control over registers
Fine control over interfaces
Predictable datapath structure
Direct control over FSM implementation


RTL DISADVANTAGES

More development effort
More code
More detailed verification
Longer development time


HLS ADVANTAGES

Faster algorithm development
C/C++ based hardware description
Easier design-space exploration
Convenient pipeline directives
Convenient loop unrolling
Faster prototyping


HLS DISADVANTAGES

The generated architecture must still be analysed
Poor coding style can produce inefficient hardware
Resource utilization must still be checked
Timing must still be verified
Memory access patterns strongly affect performance

The key lesson is:

HLS reduces the amount of RTL coding, but it does not remove the need to understand hardware architecture.


VERIFICATION STRATEGY

The accelerator can be verified using RTL simulation together with a software reference implementation.

Important test cases include:

All pixels equal to zero
All pixels equal to 255
All pixels equal to the same grayscale value
Four identical pixels in one input word
Four different pixels in one input word
Pixels below the threshold
Pixels equal to the threshold
Pixels above the threshold
Threshold equal to zero
Threshold equal to 255
Image size not divisible by four
AXI backpressure
Reset operation
Multiple consecutive images


EXAMPLE HISTOGRAM VERIFICATION

For a small input image:

10   10   20   30
10   20   30   30

the expected histogram is:

hist[10] = 3

hist[20] = 2

hist[30] = 3

All other bins should remain zero.

The RTL output can be compared against this expected result.


THRESHOLD VERIFICATION

Assume:

Threshold = 100

Input pixels:

20   99   100   200

For:

pixel < threshold -> 0

pixel >= threshold -> 255

the output becomes:

0   0   255   255

Boundary conditions should be tested carefully because:

pixel < threshold

and

pixel <= threshold

produce different results when the pixel equals the threshold.


POSSIBLE IMPROVEMENTS

Future improvements for this project include:

Automated comparison against a Python/OpenCV Otsu implementation

ARM-only versus HW/SW performance comparison

End-to-end DMA latency measurement

LUT utilization analysis

Flip-Flop utilization analysis

DSP utilization analysis

BRAM utilization analysis

Post-implementation timing analysis

Critical-path optimisation

Pipelining of weighted multiply-accumulate logic

BRAM-based histogram architecture

Banked histogram memories

Partial histogram architectures

Wider AXI-stream datapath

Eight-pixel-per-clock processing

RGB-to-grayscale preprocessing

Interrupt-driven completion handling

Linux driver integration

HLS versus RTL performance comparison

Power analysis

Full hardware benchmarking


KEY SKILLS DEMONSTRATED

FPGA architecture

Verilog

RTL design

Hardware/software co-design

AMD/Xilinx Zynq SoC

ARM Processing System

Programmable Logic

Vivado

Vitis

Vitis HLS concepts

AXI4-Stream

AXI-Lite

AXI DMA

DDR memory

FIFO buffering

Backpressure

FSM design

Parallel processing

Image processing

Histogram calculation

Cumulative histogram calculation

Weighted histogram calculation

Otsu thresholding

Pipelining

Timing considerations

Resource/performance trade-offs

Embedded software development

FPGA/ARM communication

Hardware verification


MAIN ENGINEERING LESSONS

The most important lesson from this project is that FPGA acceleration is not simply about translating an entire software algorithm into hardware.

A good HW/SW co-design requires understanding:

Which operations benefit from parallel FPGA hardware?

Which operations are more convenient in software?

How much hardware should be replicated?

Where should arithmetic resources be reused?

How will data move between memory, processor, and FPGA?

How will AXI backpressure be handled?

What are the resource and timing trade-offs?

In this project:

Parallel histogram processing

and

Streaming thresholding

are accelerated in FPGA hardware.

The more control-oriented Otsu threshold calculation is performed on the ARM processor.

This creates a practical example of heterogeneous computing using the Zynq Processing System and Programmable Logic together.


PROJECT SUMMARY

Input:
8-bit grayscale image

Platform:
AMD/Xilinx Zynq SoC

Hardware:
Custom FPGA image-processing accelerator

Software:
ARM application developed using Vitis

Hardware Interfaces:
AXI4-Stream
AXI-Lite
AXI DMA

Pixels Processed:
4 pixels per clock

Histogram Levels:
256

Output:
8-bit binary image

Black Pixel:
0

White Pixel:
255

The project demonstrates a complete hardware/software co-design workflow for FPGA-based image processing, including algorithm analysis, HW/SW partitioning, RTL implementation, AXI communication, DMA-based memory transfers, embedded software control, streaming data processing, and system-level performance trade-offs.
