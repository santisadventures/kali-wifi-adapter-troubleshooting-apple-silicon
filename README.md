# Kali USB Wi-Fi Adapter Troubleshooting on Apple Silicon Macs

![Platform](https://img.shields.io/badge/Apple%20Silicon-M--series-informational)
![Kali](https://img.shields.io/badge/Kali%20Linux-ARM64-informational)
![Virtualization](https://img.shields.io/badge/UTM-Tested-success)
![Monitor Mode](https://img.shields.io/badge/Monitor%20Mode-Tested-success)
![Packet Injection](https://img.shields.io/badge/Packet%20Injection-Tested-success)

Troubleshooting guide for using external USB Wi-Fi adapters with **Kali Linux ARM64 virtualized on Apple Silicon (M-series) Macs**.

This repository documents a working solution for cases where macOS detects a USB Wi-Fi adapter, but the adapter cannot be passed directly to a Kali Linux ARM64 virtual machine.

The tested workaround uses **VirtualHere** to expose the physical USB Wi-Fi adapter from macOS to the Kali guest.

> [!NOTE]
> This is a troubleshooting guide. Not every Apple Silicon or UTM configuration requires VirtualHere. USB passthrough capabilities depend on the virtualization backend, UTM version, macOS version, and guest configuration.

---

## TL;DR

If your USB Wi-Fi adapter is detected by macOS but does not appear inside a Kali Linux ARM64 VM on an Apple Silicon Mac, this setup provides a tested workaround:

```text
USB Wi-Fi Adapter
        ↓
macOS / Apple Silicon
        ↓
VirtualHere USB Server
        ↓
UTM / Kali Linux ARM64
        ↓
VirtualHere ARM64 Client
        ↓
vhci_hcd
        ↓
Linux Wi-Fi driver
        ↓
wlan0
        ↓
Monitor Mode + Packet Injection
```

**Tested successfully with a Mercusys AC650 (`2c4e:0105`) using the `rtw88_8821cu` driver.**

---

## Quick Start

Once VirtualHere Server is running on macOS and the ARM64 client is running inside Kali:

```bash
uname -m
./vhclientarm64 -t "LIST"
./vhclientarm64 -t "USE,<device-address>"
lsusb
iw dev
```

Verify monitor mode support:

```bash
iw list | sed -n '/Supported interface modes:/,/Band/p'
```

Enable monitor mode:

```bash
sudo airmon-ng check kill
sudo ip link set wlan0 down
sudo iw dev wlan0 set type monitor
sudo ip link set wlan0 up
iw dev
```

Test passive capture:

```bash
sudo airodump-ng wlan0
```

On networks you own or are authorized to test, verify packet injection with:

```bash
sudo aireplay-ng --test wlan0
```

For installation details and troubleshooting, continue with the full guide below.

---

## Tested Environment

This is the configuration that was actually tested:

| Component | Tested configuration |
|---|---|
| Host architecture | Apple Silicon (M-series) |
| Virtualization | UTM |
| Guest | Kali Linux ARM64 |
| Kernel | `7.1.5+kali-arm64` |
| USB transport | VirtualHere |
| Wi-Fi adapter | Mercusys AC650 |
| USB VID:PID | `2c4e:0105` |
| Linux driver | `rtw88_8821cu` |
| Monitor mode | ✅ PASS |
| Packet injection | ✅ PASS |

Results with other adapters, kernels, virtualization backends, or macOS versions may differ.

---

## Known Working Adapters

The following hardware has been physically tested with this setup:

| Adapter | USB ID | Linux Driver | Monitor Mode | Packet Injection |
|---|---|---|---|---|
| Mercusys AC650 | `2c4e:0105` | `rtw88_8821cu` | ✅ Tested | ✅ Tested |

Other adapters may work, but should not be considered confirmed until tested. Contributions with the adapter model, USB ID, driver, and test results are welcome.

---

## Tested Configuration

| Component | Status |
|---|---|
| Apple Silicon Mac (M-series) | ✅ Tested |
| Kali Linux ARM64 | ✅ Tested |
| UTM | ✅ Tested |
| VirtualHere | ✅ Tested |
| Mercusys AC650 | ✅ Tested |
| USB ID `2c4e:0105` | ✅ Tested |
| Realtek RTL8821CU | ✅ Tested |
| `rtw88_8821cu` | ✅ Tested |
| Monitor mode | ✅ Working |
| Packet injection | ✅ Working |
| `airodump-ng` | ✅ Working |
| `aireplay-ng --test` | ✅ Working |

Other USB Wi-Fi adapters may also work if the Kali ARM64 kernel has a compatible driver.

---

## The Problem

When running Kali Linux ARM64 virtualized on an Apple Silicon Mac, macOS may correctly detect an external USB Wi-Fi adapter while Kali cannot see it.

On macOS, the tested adapter appeared as:

```text
USB Product Name = "802.11ac NIC"
USB Vendor Name  = "Realtek"
idVendor         = 11342
idProduct        = 261
```

Converted to hexadecimal:

```text
2c4e:0105
```

But initially:

```bash
lsusb
```

inside Kali did not show the adapter.

Without the physical adapter being exposed to the guest, Kali cannot load its Wi-Fi driver, create `wlan0`, enter monitor mode, or use packet injection.

---

## Working Architecture

The working setup is:

```text
Mercusys AC650 / USB Wi-Fi Adapter
              │
              │ USB
              ▼
┌──────────────────────────────┐
│ Apple Silicon Mac            │
│ macOS                        │
│                              │
│ VirtualHere USB Server       │
└──────────────┬───────────────┘
               │
               │ Virtual network
               ▼
┌──────────────────────────────┐
│ UTM                          │
│ Kali Linux ARM64             │
│                              │
│ VirtualHere ARM64 Client     │
│          │                   │
│          ▼                   │
│       vhci_hcd               │
│          │                   │
│          ▼                   │
│    rtw88_8821cu              │
│          │                   │
│          ▼                   │
│        wlan0                 │
│          │                   │
│     monitor mode             │
│     packet injection         │
└──────────────────────────────┘
```

VirtualHere handles the USB transport between macOS and Kali.

On Linux, the device is exposed through the kernel's `vhci_hcd` virtual USB host controller.

---

# Step 1 — Verify macOS Detects the Wi-Fi Adapter

Before changing anything in Kali, verify that macOS can see the physical adapter.

Run on macOS:

```bash
ioreg -p IOUSB -l -w 0
```

For Realtek adapters, a more useful filtered command is:

```bash
ioreg -p IOUSB -l -w 0 | grep -iE -A12 -B5 '802.11ac|Realtek'
```

For the tested Mercusys AC650, macOS reported:

```text
USB Product Name = "802.11ac NIC"
USB Vendor Name = "Realtek"
```

If macOS cannot see the adapter, troubleshoot the physical USB connection, hub, or adapter before continuing.

---

# Step 2 — Install VirtualHere Server on macOS

Download the official **VirtualHere USB Server for macOS**:

[VirtualHere USB Server for macOS](https://www.virtualhere.com/osx_server_software)

Install and start it with the USB Wi-Fi adapter connected.

Verify that VirtualHere is running:

```bash
ps aux | grep -i virtualhere | grep -v grep
```

---

# Step 3 — Install VirtualHere ARM64 Client in Kali

Inside Kali, verify the architecture:

```bash
uname -m
```

Expected:

```text
aarch64
```

Download the ARM64 VirtualHere client:

```bash
wget https://www.virtualhere.com/sites/default/files/usbclient/vhclientarm64
chmod +x vhclientarm64
```

Verify it:

```bash
file vhclientarm64
```

Expected output should contain:

```text
ARM aarch64
```

---

# Step 4 — Start VirtualHere Client

Start the client interactively:

```bash
sudo ./vhclientarm64
```

Keep this terminal open.

Open a **second Kali terminal**.

List the available USB devices:

```bash
./vhclientarm64 -t "LIST"
```

Example:

```text
OSX Silicon Hub (MacBook.local:7575)
   --> 802.11ac NIC (MacBook.local.18096128)
```

Your device address will be different.

---

# Step 5 — Attach the Wi-Fi Adapter

Copy the address displayed by `LIST`.

For example:

```bash
./vhclientarm64 -t "USE,MacBook.local.18096128"
```

Successful result:

```text
OK
```

Now check:

```bash
lsusb
```

Our tested adapter appeared as:

```text
Bus 005 Device 002: ID 2c4e:0105 Mercucys INC 802.11ac NIC
```

At this point, the physical USB adapter is visible inside Kali.

---

# Step 6 — Verify the Driver

Check:

```bash
sudo dmesg | tail -50
```

For the tested Mercusys AC650:

```text
New USB device found, idVendor=2c4e, idProduct=0105
Product: 802.11ac NIC
Manufacturer: Realtek
rtw88_8821cu: Firmware version 24.11.0
```

Check for a Wi-Fi interface:

```bash
ip link
```

and:

```bash
iw dev
```

Our adapter created:

```text
wlan0
```

with the Linux driver:

```text
rtw88_8821cu
```

---

# Step 7 — Verify Monitor Mode Support

Run:

```bash
iw list | sed -n '/Supported interface modes:/,/Band/p'
```

Our tested adapter reported:

```text
Supported interface modes:
    * IBSS
    * managed
    * AP
    * AP/VLAN
    * monitor
```

If `monitor` is present, continue.

---

# Step 8 — Enable Monitor Mode

Stop processes that may interfere:

```bash
sudo airmon-ng check kill
```

Then:

```bash
sudo ip link set wlan0 down
sudo iw dev wlan0 set type monitor
sudo ip link set wlan0 up
```

Verify:

```bash
iw dev
```

Expected:

```text
Interface wlan0
        type monitor
```

---

# Step 9 — Test Monitor Mode

Run:

```bash
sudo airodump-ng wlan0
```

If access points begin appearing, monitor mode is working.

Our tested setup successfully detected nearby 2.4 GHz Wi-Fi networks.

---

# Step 10 — Test Packet Injection

Only perform wireless security testing on networks and devices you own or have explicit authorization to test.

Run:

```bash
sudo aireplay-ng --test wlan0
```

Our tested configuration returned:

```text
Injection is working!
```

The adapter also successfully received directed probe responses.

Therefore:

```text
Monitor mode      PASS
Packet injection  PASS
```

---

# Troubleshooting

## `FAILED: API Timeout 3 sec`

During testing, starting VirtualHere with:

```bash
sudo ./vhclientarm64 -n
```

resulted in:

```text
VirtualHere Client is running as a service
```

`LIST` worked, but:

```bash
./vhclientarm64 -t "USE,<device-address>"
```

returned:

```text
FAILED: API Timeout 3 sec
```

### Working solution

Stop the background client:

```bash
sudo pkill vhclientarm64
```

Start it interactively:

```bash
sudo ./vhclientarm64
```

Keep that terminal open.

From another terminal:

```bash
./vhclientarm64 -t "LIST"
```

The working configuration reported:

```text
VirtualHere Client not running as a service
```

Then:

```bash
./vhclientarm64 -t "USE,<device-address>"
```

returned:

```text
OK
```

---

## VirtualHere Sees the Adapter but `lsusb` Does Not

Check the Linux virtual USB host controller:

```bash
lsmod | grep vhci
```

Our system showed:

```text
vhci_hcd
usbip_core
```

If necessary:

```bash
sudo modprobe vhci-hcd
```

Then try `USE` again.

---

## Adapter Appears in `lsusb` but There Is No `wlan0`

USB forwarding is working.

The remaining problem is probably the Linux driver or firmware.

Check:

```bash
sudo dmesg | grep -iE 'wifi|wlan|rtl|rtw|firmware|usb'
```

and:

```bash
lsusb -t
```

Identify the chipset and verify that your Kali ARM64 kernel supports it.

---

## `wlan0` Exists but Monitor Mode Does Not

Check:

```bash
iw list | sed -n '/Supported interface modes:/,/Band/p'
```

If:

```text
* monitor
```

is missing, the current chipset/driver combination may not support monitor mode.

---

# Tested Hardware

## Mercusys AC650

USB ID:

```text
2c4e:0105
```

Detected as:

```text
Mercucys INC 802.11ac NIC
Realtek
```

Chipset/driver used by Kali:

```text
rtw88_8821cu
```

Results:

| Test | Result |
|---|---|
| macOS USB detection | ✅ PASS |
| VirtualHere discovery | ✅ PASS |
| USB forwarding to Kali | ✅ PASS |
| Linux driver | ✅ PASS |
| `wlan0` | ✅ PASS |
| Monitor mode | ✅ PASS |
| `airodump-ng` | ✅ PASS |
| Packet injection | ✅ PASS |
| `aireplay-ng --test` | ✅ PASS |

---

# Useful Diagnostic Commands

### macOS

```bash
ioreg -p IOUSB -l -w 0
```

### VirtualHere

```bash
./vhclientarm64 -t "LIST"
```

### USB inside Kali

```bash
lsusb
lsusb -t
```

### Kernel / driver

```bash
sudo dmesg | grep -iE 'wifi|wlan|rtl|rtw|firmware|usb'
```

### Wi-Fi interfaces

```bash
iw dev
ip link
```

### Supported modes

```bash
iw list
```

### Monitor mode

```bash
sudo airmon-ng check kill
sudo ip link set wlan0 down
sudo iw dev wlan0 set type monitor
sudo ip link set wlan0 up
```

### Passive capture

```bash
sudo airodump-ng wlan0
```

### Injection test

```bash
sudo aireplay-ng --test wlan0
```

---

# Notes

- VirtualHere device addresses are system-specific. Always obtain yours using `LIST`.
- Other USB Wi-Fi adapters may require different Linux drivers.
- Seeing the device in `lsusb` confirms USB forwarding, but does not guarantee Wi-Fi driver compatibility.
- Monitor mode and packet injection depend on the chipset and Linux driver.
- VirtualHere uses Linux's virtual USB infrastructure (`vhci_hcd`) on the Kali side.
- Future UTM or macOS versions may provide native USB passthrough capabilities that make this workaround unnecessary.
- Use wireless security tools only on systems you own or are explicitly authorized to test.

---

## Tested

This procedure was tested successfully on an **Apple Silicon M-series Mac running Kali Linux ARM64 in UTM**, using a **Mercusys AC650 (`2c4e:0105`)**.

Both **monitor mode and packet injection were confirmed working**.
