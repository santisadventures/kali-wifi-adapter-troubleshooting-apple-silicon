# Contributing

Contributions testing additional USB Wi-Fi adapters on Kali Linux ARM64 virtualized on Apple Silicon Macs are welcome.

## Reporting a Working Adapter

Please include:

- Adapter manufacturer and model
- USB VID:PID (`lsusb`)
- Wi-Fi chipset
- Linux driver
- Mac / Apple Silicon generation
- Kali Linux version
- Kernel version (`uname -r`)
- Virtualization software and version
- USB transport method
- Monitor mode: PASS / FAIL
- Packet injection: PASS / FAIL

Please include relevant command output where possible.

## Pull Requests

If adding a confirmed adapter, update the **Known Working Adapters** table in `README.md`.

Only mark monitor mode or packet injection as tested when you have verified the functionality on physical hardware.

## Security

Only perform wireless security testing on networks and devices you own or have explicit authorization to test.
