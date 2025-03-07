# Blitz

Blitz is a compact, low-power  M.2 card designed to offload digital signature verification from software, accelerating transaction validation in Bitcoin-based cryptocurrencies like Nexa. It currently targets Schnorr signatures—a compact, efficient cryptographic scheme used in modern blockchain protocols—with plans to support ECDSA and quantum-secure methods in future iterations. Using a PCIe 3.0 x1 interface for high-throughput operation (up to 8 Gbps) and a USB interface for development, Blitz aims to exceed 1 million signature operations per second. This README serves as both a specification for its ongoing development and future documentation.

## Form Factor

Blitz (V0.2) is a FPGA-based 22mm x 30mm M.2 card with A + E keying, a form factor common to Bluetooth/Wi-Fi cards and accelerators like the [Google Coral M.2 Accelerator A+E](https://coral.ai/products/m2-accelerator-ae). Most modern computers have a compatible M.2 slot, ensuring wide adaptability.

## Communication Interfaces

Blitz supports two host communication channels: PCIe for regular operation and USB for bitstream uploads, research, and debugging.

### PCIe Interface

The PCIe 3.0 x1 interface in Blitz (V0.2) supports bandwidth sufficient for over 1 million signature operations per second with an optimized kernel driver, though current throughput is limited by the FPGA’s accelerator cores.

### USB Interface

An FTDI FT4232H chip provides:
- **JTAG Master**: Hardwired to the FPGA’s JTAG port for bitstream loading and debugging.
- **UART 1**: Connected to FPGA GPIO for low-bandwidth core communication.
- **UART 2**: Configured in loopback mode (TX tied to RX) to verify USB functionality.

## Status

### History

Blitz originated in 2019 as a Bitcoin Unlimited project within UBC’s Engineering Physics program. A student team built a simplified elliptic curve point multiplier (15,000 multiplications/s), earning the "Roy Nodwell Award." The first prototype, Blitz V0.1 (22mm x 60mm, B + M keying), debuted at the Australian Crypto Convention in Sydney, November 2024. Feedback revealed M-key slots often lack USB and B-key slots are obsolete, prompting a redesign to the 22mm x 30mm A+E form factor in V0.2 for better compatibility.

Recent work by @hypernyan (early 2024) integrated Schnorr validation, while Peter Rizun (late 2024) optimized elliptic curve circuits. Currently, USB enables basic interaction and debugging, with PCIe under development for primary operation.

### Current Performance

Preliminary benchmarking of Blitz V0.2 suggests a throughput of approximately 184,000 signature operations per second. This estimate is derived from the latency of a single Schnorr signature verification, multiplied by the five elliptic curve multiplier cores, each supporting two threads. Final measurements, including power consumption, await PCIe integration and comprehensive testing planned for the short-term roadmap.

### Current USB Functionality

Currently, Blitz leverages its USB interface for FPGA configuration and basic debugging. When connected to a computer with the Efinix Efinity Software installed, Blitz appears as an Efinix development kit. Users should follow the installation instructions for the Efinix Ti60 development kit (available from Efinix documentation), particularly the section on installing USB FTDI drivers, to ensure proper recognition and communication. The JTAG master enables bitstream loading, while UART 1 supports low-bandwidth debug commands (e.g., core status checks), with full signature verification pending PCIe implementation.

## Data Flow (Planned)

Blitz, a memory-mapped PCIe device, will enable high-throughput signature verification via two access methods: a kernel driver through `/dev/blitz` or direct memory access as root using two registers.

### Access Through the Kernel Driver

The driver offers a simple file-like interface:

--- Python Code Start ---
import os
fd = os.open("/dev/blitz", os.O_RDWR)
os.write(fd, pubkey + signing_hash + signature)  # 160-byte string: 64 + 32 + 64 bytes
result = os.read(fd, 1)
if ord(result) == 1:
    print("VERIFIED")
os.close(fd)
--- Python Code End ---

A 160-byte write (64-byte public key, 32-byte hash, 64-byte signature) triggers verification. The driver writes a single byte to the user-space buffer: 1 for a valid signature (PASS bit), 0 for an invalid signature (FAIL bit), blocking until complete. Errors (e.g., faults, timeouts) raise an `OSError`, detailed later.

#### Multi-threading

Blitz supports up to 256 simultaneous pending requests. The kernel driver manages response matching, ensuring each thread receives its corresponding result. To the application, Blitz appears identical across all threads—each can write and read concurrently without explicit synchronization. This design enables significantly higher throughput than single-threaded operation, as unblocked threads remain active while others await read completion.

### Direct Access Via Memory

Two registers—162-byte input FIFO (BAR0) and 2-byte output FIFO (BAR1)—are assigned dynamic addresses at boot, queryable via `lspci` or `/sys/bus/pci/devices/<device>/resource`. Root access via `/dev/mem` allows signatures to be written and results to be read manually, bypassing the kernel driver:

--- Python Code Start ---
import os
import mmap
fd = os.open("/dev/mem", os.O_RDWR | os.O_SYNC)
bar0_addr = 0xd0000000  # BAR0: input FIFO (example)
bar1_addr = 0xd0001000  # BAR1: output FIFO (example)
bar0 = mmap.mmap(fd, 4096, offset=bar0_addr)
bar1 = mmap.mmap(fd, 4096, offset=bar1_addr)
bar0[0:160] = pubkey + signing_hash + signature
if ord(bar1[0:1]) & 0x01:
    print("VERIFIED")
bar0.close(); bar1.close(); os.close(fd)
--- Python Code End ---

#### Address Assignment and Mapping

BAR0 and BAR1 map as 4 KiB regions due to the PCIe IP block’s default, aligning with PCIe’s minimum allocation for host compatibility, offering space for future enhancements like ECDSA support.

### Accelerator Input and Output FIFOs

Blitz (V0.2) has five elliptic curve multiplier cores, each with two threads, verifying up to 10 signatures in parallel with single-signature latency. Round-robin sequencers manage input FIFO distribution and output FIFO collection, preserving request order without throughput loss.

The input FIFO (162 bytes wide, 16 entries, 2592 bytes total) and 2-byte output FIFO interface with the host. The driver’s 256-request buffer exceeds V0.2’s capacity, with ASICs planned to scale hardware closer to this limit.

--- Diagram Start ---
[Diagram: Data Flow with PCIe and FIFOs]
User Space -> Kernel Driver (160B + Tag + Checksum) -> PCIe -> Input FIFO (162B x 16) -> 5 Cores (10 Threads) -> Output FIFO (2B) -> Driver -> User Space
--- Diagram End ---

The accelerator core’s module declaration is:

--- Verilog Code Start ---
module schnorr_accelerator (
    output reg [15:0] out,      // 2-byte output (STATUS, TAG)
    output full,
    output empty,
    input [162*8-1:0] in,       // 162-byte input (160-byte signature, TAG, checksum)
    input push,                 // PUSH from PCIe
    input pull                  // PULL from accelerator
);
--- Verilog Code End ---

#### Input FIFO

The input FIFO is 162 bytes wide, rather than the 160 bytes used for signature data alone. The additional bytes serve specific purposes:
- **Bytes 0–159**: Signature data (public key, hash, signature).
- **Byte 160 (Tag)**: 8-bit thread identifier.
- **Byte 161 (Checksum)**: 8-bit XOR of bytes 0–160 for corruption detection.

#### Output FIFO

The 2-byte output comprises:
- **Byte 1 (Status)**: `{OUTPUT_CHECKSUM[4:0], INPUT_CHECKSUM_BAD, FAIL, PASS}`:
  - Bits 7–3: OUTPUT_CHECKSUM (5-bit XOR of pre-encoded result and tag).
  - Bit 2: INPUT_CHECKSUM_BAD (1 = corrupt input).
  - Bit 1: FAIL (1 = invalid).
  - Bit 0: PASS (1 = valid).
- **Byte 2 (Tag)**: Echoed input tag for matching.

The driver uses these to confirm results and detect errors.

### Error Handling

Error handling is managed exclusively by the Linux kernel driver, ensuring robust signature verification across the PCIe interface. The process is as follows:

- The kernel driver assigns each request an 8-bit ID tag (byte 160) and calculates an 8-bit checksum (byte 161) by XORing bytes 0–160 (signature data + tag), then sends the 162-byte string to the card via PCIe.
- The PCIe module reassembles and latches this string into the input FIFO.
- The input FIFO verifies the checksum; if it fails, the accelerator sets the INPUT_CHECKSUM_BAD bit (bit 2) in the STATUS byte, though it still performs the signature check and sets PASS (bit 0) and FAIL (bit 1) normally.
- The output FIFO computes a 5-bit checksum by XORing the 3 status bits (PASS, FAIL, INPUT_CHECKSUM_BAD) with the ID tag, appending it to the STATUS byte (bits 7–3), alongside the echoed tag (byte 2).
- The PCIe module returns these 2-byte responses to the kernel driver via BAR1, or fails to respond if the card malfunctions.
- The kernel driver inspects BAR1, matches responses to threads using the ID tag, and applies a timeout mechanism.

The driver processes the response and returns to user space:
- **Valid**: PASS bit is set, FAIL bit is clear, INPUT_CHECKSUM_BAD bit is clear, and output checksum matches → writes 1 to buffer, returns 1 (bytes read).
- **Invalid**: FAIL bit is set, PASS bit is clear, INPUT_CHECKSUM_BAD bit is clear, and output checksum matches → writes 0 to buffer, returns 1 (bytes read).
- **Fault**: Any response that is neither a valid Pass nor Fail (e.g., INPUT_CHECKSUM_BAD set, checksum mismatch, or other invalid STATUS combinations) → returns `-EIO` (-5).
- **Timeout**: No response within 10 ms → returns `-ETIMEDOUT` (-110).

--- Diagram Start ---
[Diagram: Error Handling States]
Pass: [PASS=1, FAIL=0, INPUT_CHECKSUM_BAD=0, Checksum OK] -> Buffer: 1, Return: 1
Fail: [PASS=0, FAIL=1, INPUT_CHECKSUM_BAD=0, Checksum OK] -> Buffer: 0, Return: 1
Fault: [Any response not Pass or Fail, e.g., INPUT_CHECKSUM_BAD=1, Checksum mismatch, or invalid STATUS] -> Return: -EIO (-5)
Timeout: [No response within 10 ms] -> Return: -ETIMEDOUT (-110)
--- Diagram End ---

### Detailed Usage Example

For robust applications, handle errors explicitly. In C:

--- C Code Start ---
#include <unistd.h>
#include <fcntl.h>
#include <stdio.h>
#include <errno.h>
int fd = open("/dev/blitz", O_RDWR);
if (fd < 0) { perror("open failed"); return 1; }
unsigned char buf[160] = { /* 64-byte pubkey, 32-byte hash, 64-byte sig */ };
unsigned char result;
write(fd, buf, 160);  // Send 160-byte request
ssize_t n = read(fd, &result, 1);  // Read 1-byte response
if (n == 1) {
    if (result == 1) printf("VERIFIED\n");
    else printf("FAILED\n");
} else if (n == -EIO) {
    printf("I/O Error\n");
} else if (n == -ETIMEDOUT) {
    printf("Timeout Error\n");
} else {
    printf("Unexpected Error: %zd\n", n);
}
close(fd);
--- C Code End ---

In Python:

--- Python Code Start ---
import os
import errno
fd = os.open("/dev/blitz", os.O_RDWR)
os.write(fd, pubkey + signing_hash + signature)  # 160-byte string: 64 + 32 + 64 bytes
try:
    result = os.read(fd, 1)  # Read 1 byte into bytes object
    if ord(result) == 1:
        print("VERIFIED")
    else:
        print("FAILED")
except OSError as e:
    if e.errno == errno.EIO:
        print("I/O Error")
    elif e.errno == errno.ETIMEDOUT:
        print("Timeout Error")
    else:
        print(f"Unexpected Error: {e}")
os.close(fd)
--- Python Code End ---

## Roadmap

### Short Term

The goal is an "experimenter’s kit" release in summer 2025 with these steps (not necessarily sequential):

1. **Complete PCIe Integration**: Finalize kernel driver and PCIe link.

2. **Validate Hardware**: Test V0.2, respin if needed.

3. **Benchmark Performance**: Measure signature operations per second and power consumption.

4. **Showcase at Conference**: Demo V0.2 at the "2025 BCH Bliss Conference" in Ljubljana, Slovenia.

5. **Integrate with Nexa**: Embed Blitz into Nexa software.

6. **Release Experimenter’s Kit**: Include card, debug cradle, and manual for Nexa use and FPGA development.

### Long Term

Focus shifts to scalability and functionality:

1. **Enhance Verilog Validation**: Target near-100% cryptographic coverage.

2. **Optimize Performance**: Improve speed and lower energy use, targeting 1 million signature operations per second for an ASIC-based Blitz.

3. **Advance ASIC Design**: Develop an ASIC implementation.

4. **Develop Low-Cost Version**: Create an affordable ASIC-based Blitz.

5. **Support Additional Signatures**: Add ECDSA, quantum-secure schemes, and other signature operations.
