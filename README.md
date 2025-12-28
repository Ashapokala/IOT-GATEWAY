```md
# Modbus RTU → CoAP + MQTT Gateway (Raspberry Pi)

A lightweight gateway on Raspberry Pi that:
- Polls Modbus RTU sensors (Arduino DHT22 slave over RS-485)
- Exposes sensor data via CoAP endpoints for constrained clients
- Publishes aggregated telemetry to an MQTT broker
- Supports bi-directional control (CoAP PUT / MQTT commands → Modbus writes)
- Logs all Modbus/CoAP/MQTT activity

---

## Architecture

```

(Arduino UNO/NANO + DHT22)  <--RS-485 Modbus RTU-->  (Raspberry Pi Gateway)
|
| CoAP (UDP/5683)
v
CoAP clients / constrained devices

```
                                         |
                           | MQTT publish/subscribe
                                         v
                                           Mosquitto broker (local)
```

````

**Data flow**
1. `gateway_core` (C++) polls Modbus holding registers:
   - HREG 0 → temp*10
   - HREG 1 → hum*10
2. `gateway_core` serves the latest values via CoAP endpoints.
3. `mqtt_bridge.js` (Node.js) consumes JSON lines from `gateway_core` and:
   - Publishes telemetry every 5 seconds to MQTT
   - Subscribes to command topics and forwards them to `gateway_core` → Modbus writes

---

## Requirements

### Hardware
- Raspberry Pi 4 with Debian/Raspberry Pi OS (tested on Debian 13 “trixie”)
- USB → RS485 adapter (CH340/CH341)
- Arduino UNO/NANO as Modbus slave
- RS485 TTL transceiver (HW-519 auto direction) or similar
- DHT22 sensor

### Software
- C++17 toolchain, CMake
- `libmodbus`
- `libcoap3` (notls)
- Node.js + npm
- Mosquitto (MQTT broker)
- `mbpoll` for Modbus testing
- `coap-client-notls` for CoAP testing

---

## Install Dependencies (Debian 13 / trixie)

```bash
sudo apt update
sudo apt install -y git build-essential cmake pkg-config libmodbus-dev
sudo apt install -y libcoap3-dev libcoap3-bin
sudo apt install -y mosquitto mosquitto-clients
sudo systemctl enable --now mosquitto
sudo apt install -y nodejs npm
sudo apt install -y mbpoll
````

Verify CoAP client:

```bash
command -v coap-client-notls
```

---

## Repository Structure

```
modbus-coap-mqtt-gateway/
  cpp/
    gateway_core.cpp
    CMakeLists.txt
    build/                (generated)
  node/
    mqtt_bridge.js
    package.json
    node_modules/         (generated)
  scripts/
    test_modbus.sh
    test_coap.sh
    test_mqtt_sub.sh
    test_mqtt_cmd.sh
  logs/
    gateway_core.log
    mqtt_bridge.log
  proof/                  (optional: captured outputs/screenshots)
  README.md
```

---

## Build (C++ core)

```bash
cd cpp
rm -rf build
cmake -S . -B build
cmake --build build -j
```

Binary:

```bash
./build/gateway_core
```

---

## Run

### 1) Run CoAP + Modbus core

```bash
cd ~/modbus-coap-mqtt-gateway
./cpp/build/gateway_core /dev/ttyUSB0 1
```

* Modbus device: `/dev/ttyUSB0`
* Slave ID: `1`
* CoAP server: `udp/5683`

### 2) Run MQTT bridge (publishes telemetry + handles commands)

```bash
cd ~/modbus-coap-mqtt-gateway/node
npm install
node mqtt_bridge.js
```

---

## Modbus Register Map

Holding Registers (Function Code 0x03):

* **HREG 0 (40001)**: Temperature × 10 (signed int16 stored in uint16)
* **HREG 1 (40002)**: Humidity × 10 (uint16)

Example:

* `291` → 29.1°C
* `608` → 60.8%

---

## CoAP API

Base URL: `coap://<pi-ip>:5683`

### Read

* `GET /sensors` → `{"ts":..., "temp_c":..., "hum_pct":...}`
* `GET /sensors/temp` → `{"ts":..., "temp_c":...}`
* `GET /sensors/hum` → `{"ts":..., "hum_pct":...}`

### Write (bi-directional control)

* `PUT /modbus/hreg/0` payload: integer (e.g. `"300"`)
* `PUT /modbus/hreg/1` payload: integer

Example:

```bash
coap-client-notls -m put -e "305" coap://127.0.0.1/modbus/hreg/0
```

---

## MQTT Topics

Broker (default): `mqtt://localhost:1883`

### Telemetry (published every 5 seconds)

* Topic: `gateway/pi1/telemetry`
* Payload:

```json
{"ts":173..., "temp_c":29.1, "hum_pct":60.8}
```

### Commands (subscribed)

* Topic: `gateway/pi1/cmd/hreg/<addr>`
* Payload: integer value

Examples:

```bash
mosquitto_pub -t 'gateway/pi1/cmd/hreg/0' -m '300'
mosquitto_pub -t 'gateway/pi1/cmd/hreg/1' -m '650'
```

---

## Logging

Logs are stored in `logs/`:

* `logs/gateway_core.log` → Modbus polling, CoAP requests, Modbus writes, errors
* `logs/mqtt_bridge.log` → MQTT connect, publish events, received commands, core stderr

View:

```bash
tail -n 30 logs/gateway_core.log
tail -n 30 logs/mqtt_bridge.log
```

---

## Testing (Deliverables)

### 1) Modbus RTU test (Pi → Arduino)

```bash
./scripts/test_modbus.sh
```

Manual:

```bash
sudo mbpoll -0 -m rtu -a 1 -b 9600 -P none -t 4 -r 0 -c 2 /dev/ttyUSB0
```

### 2) CoAP tests (translation + write)

```bash
./scripts/test_coap.sh
```

Manual:

```bash
coap-client-notls -m get coap://127.0.0.1/sensors
coap-client-notls -m put -e "300" coap://127.0.0.1/modbus/hreg/0
```

### 3) MQTT subscribe/publish tests

Subscriber:

```bash
./scripts/test_mqtt_sub.sh
```

Command:

```bash
./scripts/test_mqtt_cmd.sh
```

Manual:

```bash
mosquitto_sub -t 'gateway/pi1/#' -v
mosquitto_pub -t 'gateway/pi1/cmd/hreg/0' -m '305'
```

---

## Notes / Troubleshooting

* If Modbus times out, verify:

  * A/B lines are correct (swap once if needed)
  * Slave ID, baud rate, 8N1 match
  * Only one Modbus master is active
* CoAP uses UDP port **5683**. Ensure no firewall blocks it.
* For HW-519 auto-direction RS485 TTL modules: hardware UART (D0/D1) on Arduino is more reliable than SoftwareSerial.

---

## License

MIT (or replace with your preferred license).

````

### Next 2 things you should do
1) Create the file:
```bash
cd ~/modbus-coap-mqtt-gateway
nano README.md
````

Paste the README above, save.

2. Commit it:

```bash
git add README.md
git commit -m "Add README with architecture, APIs, topics, and test steps"
```

If you want, I can also give you a **systemd service** so the gateway auto-starts on boot (looks great for job submissions).
