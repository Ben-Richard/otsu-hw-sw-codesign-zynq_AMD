# Otsu Thresholding HW/SW Co-Design on Zynq FPGA

![Platform](https://img.shields.io/badge/Platform-AMD%2FXilinx%20Zynq-blue)
![Hardware](https://img.shields.io/badge/Hardware-Verilog%20%7C%20RTL-orange)
![Software](https://img.shields.io/badge/Software-C%20%7C%20Vitis-green)
![Interfaces](https://img.shields.io/badge/Interfaces-AXI4--Stream%20%7C%20AXI--Lite%20%7C%20AXI%20DMA-yellow)

## Overview

This project implements **Otsu’s automatic image thresholding algorithm** using a hardware/software co-design architecture on a Xilinx/AMD Zynq platform. 

The purpose of the project is to demonstrate how an image-processing algorithm can be partitioned between the **FPGA Programmable Logic (PL)** and the **ARM Processing System (PS)** according to the characteristics of each computation. Otsu’s method automatically determines an optimum threshold for converting an 8-bit grayscale image (0–255) into a binary image (0 = Black, 255 = White).

### Project Quick Summary

| Feature | Specification |
|---------|---------------|
| **Input** | 8-bit grayscale image |
| **Output** | 8-bit binary image (0 = Black, 255 = White) |
| **Hardware** | Custom FPGA image-processing accelerator |
| **Software** | ARM application developed using Vitis |
| **Interfaces** | AXI4-Stream, AXI-Lite, AXI DMA |
| **Throughput** | 4 pixels per clock cycle |
| **Histogram Levels**| 256 |
