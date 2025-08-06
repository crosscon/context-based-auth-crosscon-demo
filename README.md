# CROSSCON Hypervisor and TEE Isolation

## Overview

This repository demonstrates how to build & use the Context-based Authentication (CBA) Trusted Service on the CROSSCON Hypervisor and is part of the [CROSSCON Project](https://crosscon.eu/). It is based on the [*Hypervisor and TEE Isolation Demos* (forked by 3mdeb)](https://github.com/3mdeb/CROSSCON-Hypervisor-and-TEE-Isolation-Demos) repo published for the overall CROSSCON HV stack, and adapted to build the trusted application along the required modified OP-TEE OS. This modified OP-TEE OS enables Trusted Applications (TAs) to access Channel State Information (CS) data from the Wi-Fi peripheral via a platform-independent, GlobalPlatform API-compliant way.

The CBA Trusted Service aims to utilize CSI data collected from a device's Wi-Fi environment to create a strong authentication scheme, with which an IoT device can authenticate itself to another party and proof that it is in a secure location. More details about how it works can be found in deliverable D3.3. This implementation is part of deliverable D3.4.

Furthermore, this implementation serves as a demonstrator for how TAs can use the platform-independent interface to access CSI data. This API is specified as part of deliverable D4.3, and the implementation (specifically the [modifications to OP-TEE OS](https://github.com/crosscon/context-based-auth-optee-os)) as well as this demonstrator are part of deliverable D4.4.

**Note:** This is only one part to fully run CBA as it requires another separate VM containing a modified WiFi driver (*Nexmon VM*) to be present on the system in order to fully work. **An image is provided in this repository, which is sufficient in most situations.** Instructions for how to build this other separate VM can be found [here](https://github.com/crosscon/context-based-auth-nexmon-vm). The Linux binary created via from the other repo must be copied to `nexmon` (see `nexmon/README.md` for the exact name & location). The hypervisor configuration for running the three VMs (OP-TEE + dual Linux) can be found in `rpi4-ws/configs/rpi4-single-vTEE-dual-linux`. It is used by the build scripts (see below).

This service is only available on the Raspberry Pi 4, so all links to any other device (and especially the RISC-V architecture) are irrelevant for this demo.

**Note:** The configuration file provided contains a shared memory region that is added to all three VMs. This is done for testing the individual components *only* because it provides access to the collected data to the host Linux and allows it to bypass OP-TEE. **It should be removed in production settings!**


## Requirements

### 1. Docker Environment

It is strongly advised to use Docker to build this image -- use any other build method at your own risk! Instructions for building the Docker image and running the container can be found in [`env/README.md`](env/README.md). There you will also find scripts for building & flashing the SD card image for the Raspberry Pi.

### 2. Remote Service

In addition, this TA requires a running remote service to connect to. It must be configured before the TA is built as some keys must be added statically. Please refer to [its repository](https://github.com/crosscon/context-based-auth-remote) for more details on how to set up the remote service.


## Configuration

Before building the components, some configuration is required. In all files, the configuration options are at the very top.

| file | variable/macro name | description |
| --- | --- | --- |
| `cba_ta/ta/network_handling.c` | `CONTEXT_BASED_AUTHENTICATION_SERVER_HOST` | remote server host |
| | `CONTEXT_BASED_AUTHENTICATION_SERVER_PORT` | remote server port |
| | `CONTEXT_BASED_AUTHENTICATION_SERVER_SSL_CERT` | remote server TLS certificate |
| `cba_ta/ta/signature_handling.c` | `CONTEXT_BASED_AUTHENTICATION_SERVER_SIGNATURE_CERT` | remote server signature certificate |
| `cba_ta/ta/cba.c` | `TA_CONTEXT_BASED_AUTHENTICATION_WIFI_CHANNEL` | WiFi channel (depends on the access point) |
| | `TA_CONTEXT_BASED_AUTHENTICATION_BANDWIDTH` | WiFi channel bandwidth (leave at 20 MHz for the provided machine learning model) |
| | `TA_CONTEXT_BASED_AUTHENTICATION_RECORDING_TIMEOUT` | CSI recording timeout |
| | `TA_CONTEXT_BASED_AUTHENTICATION_SAMPLES_PER_DEVICE` | CSI samples per device (leave at 64 for the provided machine learning model) |
| `cba_ta/host/main.c` | `SERVER_TEST_SIGNATURE` | Sample signature created by the remote server to demonstrate verification (see its configuration for more info) |
| `optee_os/core/pta/csi.c` | `CSI_PHYSICAL_ADDR_START` | base address for shared memory between OP-TEE and Nexmon VM (only change if required) |
| | `CSI_PHYSICAL_ADDR_SIZE` | size of shared memory between OP-TEE and Nexmon VM (only change if required) |


## Building

Instructions for building the image and flashing it onto an SD card using the Docker environment is also found in [`env/README.md`](env/README.md). Please see there for more instructions and a provided build script.


## Testing

**Note:** Testing here refers to testing the TA *without the Nexmon VM*! It is mainly meant to confirm the configuration and to verify that the build system works. Since no Nexmon VM is present, the expected behavior regarding the communication via shared memory must be simulated using a series of `busybox devmem` commands. After adding the Nexmon VM, the commands work without manually writing the shared memory.

The following commands expect the shared memory area to be configured starting at address `0x9000000`. The hypervisor configuration provided in this repository has that value by default.

### Enrollment:

To test enrollment, invoke the TA with the following command:

```sh
context_based_authentication_demo enroll &
```

This command first enrolls the client certificate with the remote server. Check the server console output to verify that this enrollment is complete.

After enrolling the certificate, the TA attempts to collect a current set of CSI measurements to forward to the server in order to set a baseline fingerprint. This uses the shared memory to communicate with the Nexmon VM. Its behavior can be simulated using `devmem`.

Use the following command to verify that the TA sent a request for data to the Nexmon VM (output must be an odd number):
```sh
busybox devmem 0x9000000 8
```

Then, execute the following commands to simulate the collection of 5 CSI samples:
```sh
busybox devmem 0x9000008 8 5
busybox devmem 0x9000007 8 7
busybox devmem 0x9000000 8 2
```

Verify with the server console logs that 5 CSI samples were received in the CSI enrollment phase.

At this point, the TA should have returned without any error.


### Creating the prove:

This works very similar to the first step, except that this time, no client certificate is enrolled.

Execute the following command to start the proving process:
```sh
context_based_authentication_demo prove &
```

Check that the TA requests data from the Nexmon VM (output must be an odd number):
```sh
busybox devmem 0x9000000 8
```

Simulate having received 20 CSI samples:
```sh
busybox devmem 0x9000008 8 20
busybox devmem 0x9000007 8 7
busybox devmem 0x9000000 8 2
```

The number of simulated samples can be changed by writing the number as an `uint32_t` to 0x9000008 to 0x900000b (little endian).
Again, verify that the server received the correct number of samples.

The TA should have returned without any error.


### Verifying the signature

Verifying a signature is straight forward, since no communication is taking place between the TA and the remote server or the Nexmon VM. The signature to be verified statically is hard-coded in the application along the nonce it's based on.

Simply run:
```sh
context_based_authentication_demo verify
```

The command should not display any error.


## License

See LICENSE file.

## Acknowledgments

The work presented in this repository is part of the [CROSSCON project](https://crosscon.eu/) that received funding from the European Union’s Horizon Europe research and innovation programme under grant agreement No 101070537.

<p align="center">
    <img src="https://crosscon.eu/sites/crosscon/themes/crosscon/images/eu.svg" width=10% height=10%>
</p>

<p align="center">
    <img src="https://crosscon.eu/sites/crosscon/files/public/styles/large_1080_/public/content-images/media/2023/crosscon_logo.png?itok=LUH3ejzO" width=25% height=25%>
</p>


