# Dasharo BenchRack

Dasharo BenchRack is [3mdeb](https://3mdeb.com/)'s modular test-bench platform
for open-source firmware development and validation. This case study describes
how a device under test (DUT) hosted in a Dasharo BenchRack boots with
[coreboot](https://www.coreboot.org/) (as distributed by Dasharo) performing
silicon initialization and LinuxBoot serving as the payload, following the same
LinuxBoot adoption approach described in the
[Ampere](Ampere_study.md) and [Google](Google_study.md) case studies.

It contains the following sections:

* [Platform overview](#platform-overview)
* [Firmware architecture](#firmware-architecture)
* [Build process](#build-process)
  * [Prerequisites](#prerequisites)
  * [Build the firmware image](#build-the-firmware-image)
* [Flashing and operation](#flashing-and-operation)
* [Booting with LinuxBoot](#booting-with-linuxboot)
* [Support](#support)
  * [Hardware support](#hardware-support)
  * [Community support](#community-support)
  * [Professional support](#professional-support)
* [See also](#see-also)

## Platform overview

The device under test (DUT) covered by this case study is the
[ASRock Rack TURIND8UD](https://www.asrockrack.com/), a single-socket AMD EPYC
server board:

* **CPU**: single socket SP5 (LGA 6096), supporting AMD EPYC 9005 series
  processors.
* **Memory**: 8 DDR5 RDIMM slots (1 DIMM per channel).
* **Expansion**: PCIe 5.0 slots and two M.2 (PCIe 5.0 NVMe) slots.
* **Networking**: onboard 10 GbE.

The DUT is mounted in a Dasharo BenchRack, which supplies remote power control,
firmware flashing, and serial capture so the board can be validated
automatically. Because both silicon initialization (coreboot) and the payload
(LinuxBoot) are open source, the platform represents a fully open boot firmware
stack, aligning with the most advanced stage of the LinuxBoot adoption model
described in the [Ampere](Ampere_study.md) case study.

## Firmware architecture

The boot firmware stack for the DUT is fully open source:

* **coreboot**: performs early hardware and silicon initialization. Source is
  available at [Dasharo/coreboot](https://github.com/Dasharo/coreboot.git).
* **LinuxBoot**: built as the coreboot payload (u-root initramfs +
  [flashkernel](../glossary.md)), responsible for boot device selection and
  `kexec` into the target OS.

## Build process

The firmware image is a single `coreboot.rom` containing coreboot plus the
LinuxBoot payload.

### Prerequisites

The recommended, reproducible build path uses the Dasharo/coreboot SDK
container, which pins the toolchain and build dependencies.

```bash
git clone https://github.com/Dasharo/coreboot.git
cd coreboot
git checkout asrock_turind8ud_linuxboot_v0.9.0
git submodule update --init --checkout

# Enter the pinned coreboot SDK container
docker run --rm -it \
  -v "$PWD:/home/coreboot/coreboot" \
  -w /home/coreboot/coreboot \
  coreboot/coreboot-sdk:2025-10-19_4a3cc37cbd \
  /bin/bash
```

### Build the firmware image

From inside the SDK container:

```bash
# Select the BenchRack DUT board target and the LinuxBoot payload
./build.sh configs/config.asrock_turind8ud_linuxboot
```

The resulting flashable image is at `asrock_turind8ud_linuxboot_<version>.rom`.

## Flashing and operation

Dasharo BenchRack provides remote flashing, power control, and serial capture
for each DUT, enabling automated (CI) firmware deployment and testing. Flashing
is performed by the [Remote Test Environment
(RTE)](https://docs.dasharo.com/transparent-validation/rte/v1.1.0/specification/),
a platform that combines power control with an SPI programmer.

To flash the DUT's SPI chip, the RTE drives three GPIO lines to take control of
the SPI bus, runs `flashrom`, and then releases the bus. The GPIO lines are:

| GPIO | Purpose | Values |
| ---- | ------- | ------ |
| 405 | SPI voltage selection | `0` = 1.8 V, `1` = 3.3 V |
| 406 | SPI connector power | `1` = enabled |
| 404 | SPI signal lines | `1` = enabled |

The full flashing sequence, using the `coreboot.rom` built above, is:

```bash
# 1. Select the chip voltage — the TURIND8UD uses a 3.3 V SPI flash chip
echo 1 > /sys/class/gpio/gpio405/value

# 2. Enable SPI power, then the SPI signal lines
echo 1 > /sys/class/gpio/gpio406/value
echo 1 > /sys/class/gpio/gpio404/value

# 3. Flash the image
flashrom -w /path/to/coreboot.rom \
  -p linux_spi:dev=/dev/spidev1.0,spispeed=16000

# 4. Release the SPI bus
echo 0 > /sys/class/gpio/gpio405/value
echo 0 > /sys/class/gpio/gpio406/value
echo 0 > /sys/class/gpio/gpio404/value
```

See the [RTE specification — How to set GPIO states to flash
SPI](https://docs.dasharo.com/transparent-validation/rte/v1.1.0/specification/#how-to-set-gpio-states-to-flash-spi)
for full details, including exporting the GPIO lines and power control.

## Booting with LinuxBoot

On power-on, the DUT runs coreboot, hands off to the LinuxBoot payload, boots
into u-root, and `kexec`s into the target OS.

```text
Welcome to LinuxBoot's Menu

Enter a number to boot a kernel:

01. Ubuntu

02. Ubuntu, with Linux 7.0.0-27-generic

03. Ubuntu, with Linux 7.0.0-27-generic (recovery mode)

04. Ubuntu, with Linux 7.0.0-22-generic

05. Ubuntu, with Linux 7.0.0-22-generic (recovery mode)

06. Memory test (mt86+x64)

07. Memory test (mt86+x64)

08. Memory test (mt86+x64, serial console)

09. Memory test (mt86+x64, serial console)

10. Memory test (mt86+ia32)

11. Memory test (mt86+ia32)

12. Memory test (mt86+ia32, serial console)

13. Memory test (mt86+ia32, serial console)

14. Reboot

15. Enter a LinuxBoot shell


Enter an option ('01' is the default, 'e' to edit kernel cmdline):
 >

Attempting to boot LinuxImage(
  Name: Ubuntu
  Kernel: file:///tmp/u-root-mounts1460215710/nvme0n1p2/boot/vmlinuz-7.0.0-27-generic
  Initrd: file:///tmp/u-root-mounts1460215710/nvme0n1p2/boot/initrd.img-7.0.0-27-generic
  Cmdline: root=UUID=d6f381c6-1549-4560-87e9-f26d9e317ab1 ro quiet splash crashkernel=2G-4G:320M,4G-32G:512M,32G-64G:1024M,64G-128G:2048M,128G-:4096M console=ttyS0,115200
  DTB: <nil>
)
```

## Support

### Hardware support

Hardware support and Dasharo BenchRack units are available from
[3mdeb](https://3mdeb.com/).

### Community support

* [Dasharo community](https://docs.dasharo.com/) — documentation and matrix
  chat channels.
* [LinuxBoot open source community](https://www.linuxboot.org/) — Slack, IRC,
  mailing list, and regular meetings for technical questions.

### Professional support

Professional support services are provided by [3mdeb](https://3mdeb.com/).

## See also

* [Dasharo documentation](https://docs.dasharo.com/)
* [Dasharo RTE specification](https://docs.dasharo.com/transparent-validation/rte/v1.1.0/specification/)
