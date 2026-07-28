# Project Context

ESP8266-based irrigation controller with 2 valves, HomeKit support, UDP logging.

## Critical Constraints

- **Always propose changes and ask permission before applying.** Do not modify files, restart containers, or change config without explicit approval.
- Explain exactly which file will be changed and what the change does before making it.

## Device Info

- MAC: `f4:cf:a2:e3:c9:6f`, IP: `192.168.1.152`
- Serial port: `/dev/cu.usbserial-210`
- Flash: 1MB, dout, 40MHz — use `-fs 1MB -fm dout -ff 40m` with esptool
- Flash layout: rboot at `0x0`, blank at `0x1000`, app at `0x2000`
- Known ESP8266 bug: after serial flash, RTS reset is unreliable — power cycle required.

## Build & Flash

```bash
# Build inside Docker
./build.sh firmware esp-homekit-irrigation

# Sign firmware
openssl sha384 -binary -out src/firmware/main.bin.sig src/firmware/main.bin
printf "%08x" $(wc -c < src/firmware/main.bin) | xxd -r -p >> src/firmware/main.bin.sig

# Flash app only
esptool.py -p /dev/cu.usbserial-210 -b 115200 write_flash -fs 1MB -fm dout -ff 40m 0x2000 src/firmware/main.bin
```

Docker image: `espbuilder`, SDK_PATH: `/project/esp-open-rtos`. Run from `/Users/dave/Documents/OpenCode/esp-firmwares`.

## Key Findings

### UDP Logger & Pre-Trigger Buffer
- The UDP logger buffers printf output in a 768-byte internal buffer before the trigger packet arrives from the PC's udplog-client.
- Pre-trigger data that exceeds 768 bytes is silently discarded (no client to send to yet).
- `get_sysparam_info()` was called in `standard_init()` (early, pre-UDP) and in `on_wifi_ready()` (after `udplog_init`).
- The early call from `standard_init()` overflows the 768-byte buffer before WiFi/UDP is ready — those sysparam lines are lost.
- The `on_wifi_ready()` call was moved to after the reset info / exception cause dump to give the trigger more time to arrive.

### Sysparam Address
- Base address 1011712 decimal = `0xF7000` hex. Both formats now print consistently as hex.

### HAP Compliance
- VALVE characteristics `ACTIVE` and `IN_USE` must use `.uint8_value`, not `.bool_value`.
- Second valve service is non-primary, without custom characteristics (ota_trigger, wifi_reset, etc.).

### Versioning
- FW_VERSION in `src/main.c` follows semver (currently `0.0.6`).
- Release naming: tag `x.y.z`, title `vx.y.z-pre`, prerelease=true.
- `latest-pre-release` file (version string only, no newline) uploaded to latest stable release asset for OTA discovery.

## Relevant Files

- `src/main.c`: firmware entry, two-valve logic, FW_VERSION
- `src/Makefile`: build config, component paths, signature target
- `../../components/UDPlogger/udplogger.c`: UDP log buffer and trigger logic
- `../../components/UDPlogger/udplogger.h`: defines `UDPLOGSTRING_SIZE` (default 768)
- `../../components/esp-homekit-common-functions/shared_functions/shared_functions.c`: `on_wifi_ready` and `standard_init` — `get_sysparam_info()` placement
- `../../components/esp-homekit-common-functions/custom_characteristics/custom_characteristics.c`: `get_sysparam_info()` implementation, log level changes
- `../../components/esp-homekit-common-functions/shared_functions/shared_functions.h`: shared headers (FreeRTOS, task.h, etc.)
