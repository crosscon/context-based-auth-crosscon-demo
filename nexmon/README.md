# NEXMON VM

This is the folder where the automated build process is looking for the NEXMON files.

Please follow the instructions in [this repo](https://github.com/crosscon/context-based-auth-nexmon-vm/tree/kernel_5.10.92-v8%2B_nexmon_automated) (branch kernel_5.10.92-v8+_nexmon_automated) for how to build the NEXMON VM .bin file and put it here as `nexmon.bin`.

In case the build process for the NEXMON Linux image fails or yields bad results, a precompiled image is provided as `nexmon-image`. Please note that this is not the final VM image and the `lloader` part of the instructions mentioned above must still be executed.
