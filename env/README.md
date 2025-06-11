# Docker for CROSSCON Hypervisor

## Overview

The purpose of this repo is to provide the environment required for
compilation of the CROSSCON hypervisor.

This README and the `build_rpi4.sh` script are modified to demonstrate the
specific build process for building the CBA TA.

## Clone the repository

Clone the repository by executing following command.

```bash
git clone --recurse-submodules git@github.com:3mdeb/CROSSCON-Hypervisor-and-TEE-Isolation-Demos.git crosscon-demos && \
cd crosscon-demos
```

## Build & run the container

Build docker container.

```bash
docker build -t crosscon_hv -f env/Dockerfile .
```

The resulting image will have a `crosscon_hv` tag.
After the image has been built, execute `env/run.sh` script to run
the container.

```bash
env/run.sh
```

## Building the rpi4-ws demo

Inside the container, use below script to build the demo `rpi4-ws` package.

```bash
env/build_rpi4.sh --all
```

The script steps are based on what can be found in
[the README](../rpi4-ws/README.md). It is the modified version and to build the CBA TA
and leave out the other TAs irrelevant for this demo.
The above command will perform all necessary steps to build the demo.
Run it without any parameters to see other options:

```bash
env/build_rpi4.sh
```


## Creating and flashing the image

The following command can be used to build the hypervisor and create an image
with all required files included.

```bash
sudo env/create_hyp_img.sh
```

The command will output the image to `/work/crosscon/crosscon-demo-img.img`.
Note: The command must be run with `sudo`.

The built image can be then flashed to SD card.

```bash
sudo dd if=./crosscon-demo-img.img of=<drive> bs=4M conv=fsync
```


## Running the image

Use UART to USB adapter to connect RPI to your machine and start up minicom.

```bash
minicom -D /dev/ttyUSB0 -b 115200
```

Supply power to RPI and hit any key when asked to stop u-boot from attempting
auto-boot.

```bash
[...]
scanning bus xhci_pci for devices... 2 USB Device(s) found
       scanning usb for storage devices... 0 Storage Device(s) found
Hit any key to stop autoboot:  0
U-Boot>
```

_Note: If you missed the timeframe, you can spam CTRL+C many times to achieve
same result._

Boot the image by manually loading it into the memory and "jumping" to it.

```bash
fatload mmc 0 0x200000 crossconhyp.bin; go 0x200000
```

## ~QEMU build~

~The docker image contains all the neccessary dependencies to build the QEMU~
~images as well (RISCV included), so all that needs to be done is following~
~the instructions from [the readme](../README.md).~
