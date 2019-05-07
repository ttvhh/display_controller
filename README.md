# Project F Display Controller 

The Project F display controller makes it easy to add video output to FPGA projects. It's written in Verilog and supports VGA, DVI, and HDMI displays. It includes full configuration for 640x480, 800x600, 1280x720, and 1920x1080, as well as the ability to define custom resolutions. This design and its documentation are licensed under the MIT License.

To get started take a look at the [demos](#demos) then read [modules](doc/modules.md) for the display controller interfaces and parameters. The design aims to be as generic as possible but does make use of Xilinx Series 7 specific features, such as SerDes. If you want advice on adapting this design to other FPGAs then take a look at [porting](doc/porting.md).

For tutorials and further information visit [projectf.io](https://projectf.io).

## Contents

- [Display Interface Support](#display-interface-support)
- [Display Resolution Support](#display-resolution-support)
- [Demos](#demos)
- [Architecture](#architecture)
- [Modules](#modules)
- [Testing](#testing)
- [TMDS Encoder Model](#tmds-encoder-model)
- [Resource Utilization](#resource-utilization)


## Display Interface Support
This design supports displays using VGA, DVI, and HDMI.

**VGA** support is straightforward; you can see an example in [`display_demo_vga.v`](hdl/demo/display_demo_vga.v). If you're building your own hardware, then [Retro Ramblings](http://retroramblings.net/?p=190) has a good example of creating a register ladder DAC. If you're looking for a ready-made VGA output then the [VGA Pmod](https://reference.digilentinc.com/reference/pmod/pmodvga/start) is a good option for around $10.

**DVI** & **HDMI** use [transition-minimized differential signalling](https://en.wikipedia.org/wiki/Transition-minimized_differential_signaling) (TMDS) to transmit video over high-speed serial links. HDMI provides extra functionality over DVI, including audio support, but all HDMI displays should accept a standard DVI signal without issue. 

The display controller offers two types of TMDS generation: 

* Direct generation on FPGA
* [Black Mesa Labs DVI Pmod](https://blackmesalabs.wordpress.com/2017/12/15/bml-hdmi-video-for-fpgas-over-pmod/)  

Direct TMDS generation on FPGA requires high-frequency clocks (742.5 MHz for 720p60) and SerDes but allows full control of the signal including HDMI features. HDMI support is currently via backwards compatibility with DVI: any standard HDMI display should accept DVI signals. However, this display controller lacks support for audio or advanced HDMI features at present.

The Black Mesa Labs (BML) Pmod is based on the Texas Instruments [TFP410](http://www.ti.com/product/TFP410). The Pmod is restricted to standard DVI features but allows even tiny FPGAs to support DVI signalling. As of February 2019, there is an unresolved issue when using a BML DVI Pmod with HDMI displays: some displays report a signal error and show nothing. This issue isn't confined to Project F but has also been reported by Black Mesa Labs themselves.


## Display Resolution Support
The following four display resolutions are tested and included by default (all at 60 Hz refresh rate):
    
     Resolution  Ratio   Clock     
     640 x  480    4:3   25.20 MHz [1]
     800 x  600    4:3   40.00 MHz
    1280 x  720   16:9   74.25 MHz     
    1920 x 1080   16:9  148.50 MHz [2]  

You can easily add timings for other resolutions; see [demos](#demos) for how to do this. 

_[1] The canonical clock for 640x480 60Hz is 25.175 MHz, but 25.2 MHz is within VESA spec and easier to generate._

_[2] The TMDS clock for 1080p60 is 1.485 GHz, which is out of spec for Xilinx 7 series FPGAs. However, 1080p60 does work, even on the slowest Artix speed grade, provided the traces are short or a TMDS buffer is used. This has been successfully tested on the Nexys Video, which uses the TMDS141._


## Demos
The [demo](hdl/demo) directory includes a demo for each supported interface:

* **[`display_demo_dvi.v`](hdl/demo/display_demo_dvi.v)** - DVI encoded on FPGA
* **[`display_demo_dvi_pmod3.v`](hdl/demo/display_demo_dvi_pmod3.v)** - DVI generated by BML 3-bit Pmod
* **`display_demo_dvi_pmod24.v`** - DVI generated by BML 24-bit Pmod (coming soon)
* **[`display_demo_vga.v`](hdl/demo/display_demo_vga.v)** - analogue VGA with 12-bit output

You can find the list of required modules for each demo in a comment at the top of its file. You'll also need suitable constraints, such as those from the Project F [hardware support](https://github.com/projf/hardware-support) repo.

There are also two simple test cards, which the demo modules can use:

* **[test_card](hdl/demo/test_card.v)** - generates a video test card based on provided resolution
* **[test_card_simple](hdl/demo/test_card_simple.v)** - generates a simple coloured border based on provided resolution

You can adjust the demo resolution by changing the parameters for `display_clocks`, `display_timings`, and `test_card` or `test_card_simple`. Comments in the demos provide settings for tested [resolutions](#display-resolution-support).


## Architecture
There are two different high-level designs. This section explains the steps used to generate the display signal in both cases.

### Analogue VGA and BML DVI Pmod

1. Display Clocks - synthesizes the pixel clock, for example, 40 MHz for 800x600
2. Display Timings - generates the display sync signals and active pixel position
3. Colour Data - the colour of a pixel, taken from a bitmap, sprite, test card etc.
4. Parallel Colour Output - external hardware converts this to analogue VGA or TMDS DVI as appropriate

### TMDS Encoding on FPGA for DVI or HDMI

1. Display Clocks - synthesizes the pixel and SerDes clocks, for example, 74.25 and 371.25 MHz for 720p
2. Display Timings - generates the display sync signals and active pixel position
3. Colour Data - the colour of a pixel, taken from a bitmap, sprite, test card etc.
4. TMDS Encoder - encodes 8-bit red, green, and blue pixel data into 10-bit TMDS values
5. 10:1 Serializer - converts parallel 10-bit TMDS value into serial form
6. Differential Signal Output - converts the TMDS data into differential form for output via two FPGA pins


## Modules

There are three modules you need to interface with for full TMDS generation:

* **[Display Clocks](#display-clocks)** ([hdl](/hdl/display_clocks.v)) - pixel and high-speed clocks for TMDS (includes Xilinx MMCM)
* **[Display Timings](#display-timings)** ([hdl](/hdl/display_timings.v)) - generates display timings, including horizontal and vertical sync
* **[DVI Generator](#dvi-generator)** ([hdl](/hdl/dvi_generator.v)) - uses `serializer_10to1` and `tmds_encode_dvi` to generate a DVI signal

If you're generating VGA or using DVI/HDMI hardware that includes its own TMDS encoder you don't need `dvi_generator`.

You need a _top_ module to operate the display controller; the project includes [demo](hdl/demo) versions for different display interfaces. When performing TMDS encoding on FPGA, the top module makes use of the Xilinx OBUFDS buffer to generate the differential output. See [demos](#demos) for details.

Details on module parameters and interfaces can be found in the [modules](doc/modules.md) doc.


## Testing

If it isn't tested it doesn't work. Project F tests its designs in simulation and on real hardware. For the display controller you can use the included [test benches](hdl/test) and [Python TMDS model](#tmds-encoder-model) to exercise the design.

We haven't formally verified the design yet, but plan to do this for the display timings and TMDS encoder during 2019. If you're interested in learning more about formal verification check out Clifford Wolf's [Formal Verification with SymbiYosys and Yosys-SMTBMC deck](http://www.clifford.at/papers/2017/smtbmc-sby/).


## TMDS Encoder Model
The display controller includes a simple [Python model](model/tmds.py) to help with TMDS encoder development. 

There are two steps to TMDS encoding: applying XOR or XNOR to the bits to minimise transitions and keeping the overall number of 1s and 0s similar to ensure DC balance. The first step depends only on the current input value, so it is easy to test. However, balancing depends on the previous values, which makes testing harder; this is where the model is particularly useful. 

By default, the Python model encodes all 256 possible 8-bit values in order, but it's easy to change the script to handle other combinations. `A0, A1, B0, or B1` show which of the four balancing options was taken: you can see what they do in the Python or Verilog code.

Sample Python output:

             1s  B   O  76543210    876543210    9876543210
    =======================================================
     30: XNOR(2, 0, A1) 00011110 -> 010100000 -> 1001011111
     31: XNOR(6, 4, B1) 00011111 -> 001011111 -> 1010100000
     32: XOR (3, 0, A0) 00100000 -> 111100000 -> 0111100000

Sample output from Verilog TMDS test bench [tmds_encoder_dvi_tb.v](hdl/test/tmds_encoder_dvi_tb.v). It should match the middle column of the Python output:

    30 010100000   2,   0, A1
    31 001011111   6,   4, B1
    32 111100000   3,   0, A0

You can also see full output from the [Python model](model/tmds-test-python.txt) and [Verilog implementation](model/tmds-test-verilog.txt) for comparison.


## Resource Utilization
The display controller is lightweight, fitting into even the smallest FPGA: 

                      Artix-7
    Demo             LUT     FF
    ---------------------------
    DVI on FPGA      125     76
    DVI BML 3-bit     67     32
    DVI BML 24-bit   TBC    TBC
    VGA               67     32

For reference an Artix A35T has 20,800 LUT6 and 41,600 FF, so even full TMDS uses well under 1% of the LUTs. Values are for demos using the simple test card module. Synthesized using Vivado 2018.3 with default options.

