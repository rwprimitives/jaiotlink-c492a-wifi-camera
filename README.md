# jaiotlink-wifi-camera

This repo contains my research notes and public write-ups for a JAIOTlink 2.4G/5G Wi-Fi IP camera, model `C492A-W6`, running an Anyka-based firmware.

The short version: Got camera from a friend who purchased it from Temu, pulled the firmware, looked at the companion Android app, started tracing the local HTTP APIs, and ended up finding a command injection bug in the `SetMAC` factory API. From there, I also documented a few related issues that made the device easier to reach, debug, and abuse.

This repo walks through the research process from initial device recon, to firmware analysis, to runtime logging, to turning the bug into a working shell in my lab.

> **Scope note:** All testing was done against a device I owned in a controlled lab network. Do not use these notes against devices you do not own or have explicit permission to test.

## Target

| Item | Value |
|---|---|
| Brand | `JAIOTlink` |
| Model | `C492A-W6` / `C2L-Y-A6` |
| Manufacturer | `GUANGZHOU` |
| Firmware provider / SDK observed | Anyka |
| Firmware version | `4.8.30.57701411` |
| Firmware released date | `2024-09-04T08:53:06` |
| Hardware version | `1.0.0` |
| SDK version observed | `2.4.3.47` |
| SDK date observed | `2024.3.14` |
| Platform | ARM 32-bit Embedded Linux |
| Main HTTP process | `anyka_ipc` |
| HTTP port | `80/tcp` |
| Other service observed | `IOTDaemon` on `10000/tcp` |
| Camera AP IP | `172.14.10.1` |

## Write-ups

Start with the first one. The others make more sense after the command injection path is clear.

1. [`writeups/01-setmac-command-injection.md`](writeups/01-setmac-command-injection.md)  
   The main vulnerability. The `Wireless` value in `/NetSDK/Factory?cmd=SetMAC` is treated as data at first, then eventually becomes shell syntax.

2. [`writeups/02-default-http-credentials.md`](writeups/02-default-http-credentials.md)  
   How I found that the camera API accepted `admin` with an empty password, and why that mattered for reaching the vulnerable API surface.

3. [`writeups/03-anyka-config-execution-trigger.md`](writeups/03-anyka-config-execution-trigger.md)  
   The `/Anyka/config` endpoint executes `/etc/jffs2/anyka_cfg.ini` through `popen()`. By itself, it is not the same as the direct `SetMAC` injection, but it becomes very useful once a script can be staged.

## Repo layout

```text
.
├── README.md
├── images
│   ├── part1_function_mac_setter.png
│   ├── part1_netstat.png
│   ├── part1_setmac_command_injection_data_flow.png
│   ├── part1_shell_wrapper.png
│   ├── part1_successful_remote_shell.png
│   ├── part1_successful_wifiself_system_log.png
│   ├── part1_truncation_and_parsing_behavior.png
│   ├── part1_wifiself_system_log.png
│   ├── part2_jadx_credentials.png
│   ├── part3_anyka_config_execution_path.png
│   ├── part3_anyka_configuration_function.png
│   └── part3_popen_function.png
└── writeups/
    ├── 01-setmac-command-injection.md
    ├── 02-default-http-credentials.md
    ├── 03-anyka-config-execution-trigger.md
    └── 04-debug-mode-telnet-binary-replacement.md
```

## Findings at a glance

| Area | What I found | Practical impact |
|---|---|---|
| `SetMAC` factory API | MAC string is partially parsed, then passed into a shell command | Command execution through the local HTTP API |
| HTTP credentials | Companion app exposed `admin` with an empty password | Makes protected APIs easy to reach on the camera network |
| `/Anyka/config` | Executes `/etc/jffs2/anyka_cfg.ini` through `popen()` | Useful staged execution trigger after a file write primitive |

## Disclosure status

Work in progress, to be continued.

## Disclaimer / Ethics

This repository is for documentation, education, and defensive security research. The testing described here was performed in a controlled lab environment against a device I owned and had authorization to analyze.

Do not use these notes, commands, or techniques to access systems or devices that you do not own or do not have explicit permission to test. Unauthorized access, modification, disruption, or surveillance of connected devices is illegal and unethical.
