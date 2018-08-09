XDCFEB PROMless proof of concept firmware
========================

This repository contains sources for common GEM AMC firmware logic as well as board specific implementation for CTP7 (Virtex 7). It is an old branch of GEM AMC firmware which supported PROMless programming of OH boards, and was adapted to program the XDCFEB FPGAs using GBT links. It is a working prototype and is intended to be used as an example for future firmware developments (the firmware was adapted for XDCEBs in a few days, so it has a lot of GEM specific things).
This firmware is divided into two parts:
   * board specific firmware (called system firmware) is stored in ctp7/hdl directory. This part implements things like top level ports, clocking, MGTs, AXI inteface with the Zynq processor in CTP7 case (including DCFEB firmware streamer interface running on Zynq).
   * board-agnostic / generic firmware is stored in common directory. The top file there is common/hdl/gem_amc.vhd. This part implements the rest of the logic, which includes GEM stuff, GBT cores, and XDCFEB specific GBT driver for loading firmware and controlling the various switches on the XDCFEB.

The XDCFEB has a lot of switches related to FPGA programming, like selecting the clock source, the bus width, master/slave mode. These are all controlled with a VIO in this firmware, called vio_dcfeb_config (found in common/hdl/gem_amc.vhd).
The firmware loader is called oh_fpga_loader (common/hdl/misc/oh_fpga_loader.vhd), which is instantiated in gem_amc.vhd. It is loading the firmware using 8bits @ 80MHz, and is connected to GBT frame in a generate block called i_glue_fpga_loader_to_gbt in gem_amc.vhd.

Note that this repository excludes Vivado project directories entirely to avoid generated files, although XPR file is included ctp7/work_dir. This XPR file is referencing the source code outside work_dir and once opened will generate a Vivado project in the work_dir. Please do not commit any files from the work_dir other than XPR. IPs are stored in ctp7/ip. 

In scripts directory you'll find a python application called generate_registers.py which takes an address_table (provided) and inserts the necessary VHDL code to expose those registers in the firmware as well as uHAL address table, documentation and some bash scripts for quick CTP7 testing.

CTP7 Notes
==========

To run chipscope follow these steps:
   * SSH into CTP7 and run: xvc \<ip_of_the_machine_directly_talking_to_ctp7\>
   * If running Vivado on a machine that's not directly connected to the MCH, open a tunnel like so: ssh -L 2542:eagle34:2542 \<host_connected_to_ctp7\>
   * Open Vivado Hardware Manager, click Auto Connect
   * In TCL console run: open_hw_target -xvc_url localhost:2542 (if not using tunnel, just replace localhost with CTP7 IP or hostname)
   * Once you see the FPGA, click refresh device to get all the Chipscope cores in your design

CTP7 doesn't natively support IPbus, but it can be emulated. In this firmware the GEM_AMC IPbus registers are mapped to AXI address space 0x64000000-0x67ffffff using an AXI-IPbus bridge. IPbus address width is 24 bits wide and it's shifted left by two bits in AXI address space (in AXI the bottom 2 bits are used to refer to individual bytes, so they're not usable for 32bit word size). So to access a given IPbus register from Zynq, you should do: mpeek (0x64000000 + (\<ip_bus_reg_address\> \<\< 2)) for reading and mpoke (0x64000000 + (\<ip_bus_reg_address\> \<\< 2)) for writing. So e.g. to write 0x1 to IPbus reg 0x300001, you would run mpoke 0x64c00004 0x1 (or simply use the provided ipb_read.sh and ipb_write.sh which will do that for you. You can also use the provided ctp7_status.sh script (generated), which will read and print all the readable registers of a given firmware module. In IPbus address the top 4 bits [23:20] are dedicated to selecing a GEM_AMC module (see scripts/address_table for more info).

To read and write the IPbus registers using uHAL, you can use an application, developed by WU that can be run on Zynq linux and emulates IPBus master. Compiled binary is included in scripts directory here, the source repository is here: https://github.com/uwcms/uw-appnotes/  (see docs directory for instructions on compiling applications for the Zynq processor)
