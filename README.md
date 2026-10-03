# ESP32 BLE WiFi HTTPUpdate LittleFS Preferences

ESP32 firmware for BLE-based WiFi provisioning and OTA updates to the app
partition (HTTPUpdate), the data partition (LittleFS), and NVS (Preferences),
with a companion Web Bluetooth control page. No companion app needed — just
Chrome/Edge with Web Bluetooth support.

## Features

- **WiFi provisioning over BLE** — send SSID & password to the device without hardcoding them in firmware.
- **OTA updates for three partitions** — the device can download and apply updates to:
  - the **app** partition, via **HTTPUpdate**
  - the **data** partition, via **LittleFS**
  - **NVS**, via **Preferences**
- **Manual read/write over BLE** — send or retrieve arbitrary raw data for the data partition or NVS directly from the browser, without going through OTA.
- **Anti-clone binding** — the firmware locks itself to the chip's MAC address on first boot, stored in its own NVS namespace, separate from the NVS blob used by the read/write/update feature above.
- **Serial console over BLE notify** — all device logs can be monitored straight from the web page, no USB cable required.
- **RAM-conscious by design** — BLE is torn down during any OTA (app, data, or NVS) so HTTP/TLS has headroom to run without fragmentation; the data partition is mounted only while actually needed, not held open for the device's whole runtime.

## Repo structure

| File | Purpose |
|---|---|
| `firmware.ino` | ESP32 firmware |
| `index.html` | Web Bluetooth control page |

## Usage

1. Open `firmware.ino` and adjust the **GLOBAL CONFIG** section at the top: BLE UUIDs, BLE name prefix, and the three OTA URLs (stored as hex bytes — see the comments in the code for how to generate them).
2. Compile & upload to the ESP32.
3. Open `index.html` in Chrome/Edge (over HTTPS or localhost — required by Web Bluetooth).
4. Click **Connect Bluetooth** and pick the device from the list.
5. Enter SSID & password, click **Send WiFi**.
6. Click **Update App** / **Update Data** / **Update NVS** to trigger an OTA update for that partition (only if a new file is available on the server).
7. Use **Write Data** / **Write NVS** to send whatever is in the Command box as the new contents of that partition; use **Read Data** / **Read NVS** to pull the current contents back into the Monitor.

## BLE protocol

All communication goes through a single characteristic (write + notify).
Text commands are sent from the browser; logs and returned data come back as
notifications into the Monitor textarea.

| Direction | Command | Notes |
|---|---|---|
| Browser → ESP | `WIFI:ssid\|pass` | Store credentials & start connecting to WiFi |
| Browser → ESP | `UPDATE:APP` | Trigger an OTA update of the app partition via HTTPUpdate |
| Browser → ESP | `UPDATE:DATA` | Trigger an OTA update of the data partition via LittleFS |
| Browser → ESP | `UPDATE:NVS` | Trigger an OTA update of the NVS blob via Preferences |
| Browser → ESP | `DATA:WRITE:<size>` | Followed by `<size>` bytes of raw data (automatically chunked by the browser); overwrites the data partition's file |
| Browser → ESP | `DATA:READ` | Ask the device to open the data partition's file and report its size |
| Browser → ESP | `DATA:NEXT` | Ask for the next chunk of the data file currently being read |
| Browser → ESP | `NVS:WRITE:<size>` | Followed by `<size>` bytes of raw data; overwrites the NVS blob |
| Browser → ESP | `NVS:READ` | Ask the device to report the size of the current NVS blob |
| Browser → ESP | `NVS:NEXT` | Ask for the next chunk of the NVS blob currently being read |
| Browser → ESP | *(anything else)* | Echoed back to Monitor, useful for connection debugging |
| ESP → Browser | `SIZE:<n>` | Initial reply to `DATA:READ` / `NVS:READ`, content size in bytes |
| ESP → Browser | *(raw chunk)* | Content, sent one chunk per `DATA:NEXT` / `NVS:NEXT` |

Only one read or write transfer (data or NVS) is expected to run at a time —
there's no handling for overlapping transfers. OTA and manual read/write for
the same partition share the same underlying storage: whichever happens last
is what's there — there's no separate staging location.

## Key configuration

All of these live in the **GLOBAL CONFIG** section at the top of the `.ino`:

- `BLE_SERVICE_UUID` / `BLE_CHAR_UUID` — must match exactly between the firmware and `index.html`.
- `BLE_NAME_PREFIX` — advertised BLE name prefix, combined with the chip's MAC address.
- `OTA_PATH_APP_HEX` / `OTA_PATH_DATA_HEX` / `OTA_PATH_NVS_HEX` — the three OTA URLs, each stored as hex bytes (light obfuscation, not encryption).
- `DATA_LOCAL_PATH` — the file path on the data partition used by `DATA:WRITE` / `DATA:READ` / `UPDATE:DATA`.
- `NVS_BLOB_NAMESPACE` / `NVS_BLOB_KEY` — the NVS namespace and key used by `NVS:WRITE` / `NVS:READ` / `UPDATE:NVS` (kept separate from the anti-clone binding's own namespace/key).
- Feature toggles (`ENABLE_SERIAL_PRINT`, `ENABLE_BLE_NOTIFY`, `ENABLE_HTTPS`, `ENABLE_DEVICE_BINDING`) — turn specific parts of the code on/off at compile time.

## Notes

- Anti-clone binding permanently locks the device to the first MAC address stored in NVS, in its own namespace/key separate from the NVS blob feature above. To reset it, erase the NVS partition or change its namespace/key.
- The data and NVS partitions may not exist at all, depending on the partition scheme flashed to the device — in that case the relevant commands fail with a logged error rather than crashing. Same goes for the app partition's second OTA slot.
- The data partition's file is streamed in and out in small chunks, so RAM use stays low regardless of file size. NVS blobs cannot be streamed — the full blob is held in RAM for one `putBytes()` call — so keep NVS blobs small; the data partition is the better fit for larger content.
- The `DATA:WRITE` / `DATA:READ` / `NVS:WRITE` / `NVS:READ` features are currently meant for development/debugging — there's no access control (anyone connected over BLE who knows the UUIDs can read/write them).
