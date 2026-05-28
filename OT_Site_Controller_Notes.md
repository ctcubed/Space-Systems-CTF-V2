# OT Site Controller — Register Map & Field Notes

Internal reference for the four ground-station PLCs (antenna ACU + equipment-room controller, bundled image). Maintained by site ops. If you're on-call and the dish is parked at the limit switches, start here.

> Reminder from CR-0117: the OT segment is **not** firewalled off from the engineering subnet. We never got around to it. Treat every register as readable by anyone with a route in. Don't stage anything sensitive in plain registers.

---

## Sites

| Code  | Site        | Host IP      | Notes                                  |
|-------|-------------|--------------|----------------------------------------|
| OT-HI | Hawai'i     | 10.4.0.210   | primary CONUS-adjacent                 |
| OT-CO | Colorado    | 10.4.0.211   | warm spare                             |
| OT-UK | UK          | 10.4.0.212   | EU coverage                            |
| OT-AU | Australia   | 10.4.0.213   | south-hemi                             |

All four run the same image — register map below applies everywhere. Modbus/TCP on **port 502**, unit ID **1**. No per-site auth — segment-level only (and see CR-0117 above).

---

## Holding registers (40001–40220) — read/write

These are the writeable control points. The SCADA HMI polls them at 10 Hz.

### Antenna control (40001–40009)

| Addr     | Name              | Type   | Values                            | Notes                                          |
|---------:|-------------------|--------|-----------------------------------|------------------------------------------------|
| 40001    | CONTROL_MODE      | uint16 | 1=Auto-Track, 2=Manual            | default 1                                      |
| 40002    | AZ_SETPOINT_X10   | uint16 | 0–3599 (deg × 10)                 | manual mode only                               |
| 40003    | EL_SETPOINT_X10   | uint16 | 0–900 (deg × 10)                  | manual mode only                               |
| 40004    | TARGET_SAT        | uint16 | 1=MOUSE-1, 2=MOUSE-2              | auto-track only                                |
| 40005    | SLEW_RATE_X10     | uint16 | 10–100 (deg/s × 10)               | default 50                                     |
| **40006**| **E_STOP**        | uint16 | **0=normal, 1=ASSERTED**          | **kill-switch interlock — see Ops Notes**      |
| 40007    | PRIORITY_MODE     | uint16 | site-configured                   | dual-sat handoff policy                        |
| 40008    | M1_CONTACT        | uint16 | 0/1                               | written by GDT — do not poke                   |
| 40009    | M2_CONTACT        | uint16 | 0/1                               | written by GDT — do not poke                   |

### Equipment room (40100–40120)

| Addr   | Name              | Type   | Values                            | Notes                                          |
|------:|-------------------|--------|-----------------------------------|------------------------------------------------|
| 40100 | HVAC_MODE         | uint16 | 1=Cool, 2=Heat, 3=Auto, 4=Off     | default 3                                      |
| 40101 | TEMP_SETPOINT_X10 | uint16 | 180–260 (°C × 10)                 | default 220 (22.0°C)                           |
| 40102 | CHILLER_ENABLE    | uint16 | 0/1                               | default 1                                      |
| 40106 | ANTENNA_POWER     | uint16 | 0/1                               | default 1 — clearing drops drive power silently|
| 40120 | ENV_RESET         | uint16 | 0/1 (auto-clears)                 | write 1 to snap rack temps back to setpoint    |

### Diagnostic / asset nameplate (40201–40210)

20-byte ASCII block, **big-endian packed** (2 chars per 16-bit register). Originally a CIM asset tag — left in place so the vendor's diagnostic tool still picks it up on factory reset.

Read it as text:

```python
from pymodbus.client import ModbusTcpClient
c = ModbusTcpClient('10.4.0.210', port=502); c.connect()
r = c.read_holding_registers(200, 10, device_id=1)  # 40201..40210
print(b''.join(x.to_bytes(2,'big') for x in r.registers).decode().rstrip('\x00'))
c.close()
```

---

## Input registers (30001–30124) — read-only telemetry

### Antenna feedback (30001–30017)

| Addr        | Name              | Type    | Units                         |
|------------:|-------------------|---------|-------------------------------|
| 30001       | AZ_CURRENT_X10    | uint16  | deg × 10                      |
| 30002       | EL_CURRENT_X10    | uint16  | deg × 10                      |
| 30003       | TRACK_STATE       | uint16  | 0=Idle, 1=Slew, 2=Track, 3=Err|
| 30004       | AZ_ERROR_X100     | int16   | deg × 100                     |
| 30005       | EL_ERROR_X100     | int16   | deg × 100                     |
| 30006       | IN_VIEW           | uint16  | 0/1                           |
| 30007       | SIGNAL_LOCK       | uint16  | 0/1                           |
| 30008–30009 | AZ_TARGET         | f32 BE  | deg                           |
| 30010–30011 | EL_TARGET         | f32 BE  | deg                           |
| 30012       | ACTIVE_SAT        | uint16  | 1/2                           |
| 30013       | PRIO_MODE_FB      | uint16  | 0/1                           |
| 30014       | M1_CONTACT_FB     | uint16  | 0/1                           |
| 30015       | M2_CONTACT_FB     | uint16  | 0/1                           |
| 30016       | HANDOFF_STATE     | uint16  | enum                          |
| 30017       | CTRL_MODE_FB      | uint16  | 1/2                           |

### Equipment room (30100–30124)

| Addr        | Name              | Type    | Units      |
|------------:|-------------------|---------|------------|
| 30100–30101 | RACK1_TEMP        | f32 BE  | °C         |
| 30102–30103 | RACK2_TEMP        | f32 BE  | °C         |
| 30104–30105 | RACK3_TEMP        | f32 BE  | °C         |
| 30106–30107 | AMBIENT_TEMP      | f32 BE  | °C         |
| 30108–30109 | AMBIENT_RH        | f32 BE  | %          |
| 30110       | HVAC_STATUS       | uint16  | enum       |
| 30111       | CHILLER_STATUS    | uint16  | enum       |
| 30112–30113 | CHILLER_POWER     | f32 BE  | kW         |
| 30114       | UPS_STATUS        | uint16  | enum       |
| 30115       | UPS_BATTERY_PCT   | uint16  | %          |
| 30116–30117 | UPS_LOAD          | f32 BE  | kW         |
| 30118       | UPS_RUNTIME_MIN   | uint16  | min        |
| 30121       | DOOR_OPEN         | uint16  | 0/1        |
| 30122       | MOTION            | uint16  | 0/1        |
| 30123       | ANTENNA_POWER_FB  | uint16  | 0/1        |
| 30124       | HVAC_MODE_FB      | uint16  | 1–4        |

### Discrete inputs (10001–10003)

| Bit | Name           |
|----:|----------------|
| 0   | antenna_power  |
| 1   | tracking       |
| 2   | hvac_running   |

---

## Ops notes

### E-Stop interlock (40006)

Writing **`0 → 1`** to 40006 asserts the manual E-Stop: the ACU latches drive power off **and** the MOC-OT collector publishes a **retained** alarm on the SCADA broker:

```
broker : mqtt://10.1.0.130:1883     (no auth)
topic  : stations/<SITE>/alarm/critical    (SITE ∈ OT-HI | OT-CO | OT-UK | OT-AU)
payload: human-readable, opaque body — see vendor schema
retain : true   (last alarm stays cached for late subscribers)
```

Clearing 40006 (`1 → 0`) releases the E-Stop immediately — the antenna resumes whatever mode `CONTROL_MODE` (40001) is currently set to. There's no drive-contactor interlock on the soft side; on the physical units the contactor latch is mechanical, but the simulator (and FSW path) doesn't replicate it.

> The vendor example used an opaque payload; we made ours human-readable so on-call doesn't have to look up the meaning at 3am. Sub with `mosquitto_sub -h 10.1.0.130 -t 'stations/+/alarm/critical' -C 1` to read the latest.

### Park-and-lock (chained 40001 → 40006)

A nasty failure mode worth documenting: writing **CONTROL_MODE = 2** (stow) parks the antenna pointing at the sky (az=0, el=90), and asserting E-Stop mid-slew locks the drive wherever the slew has reached. Recovery has **three** steps **in this order**:

1. Clear 40006 (E-Stop OFF)
2. Restore 40001 to 1 (Auto-Track) — **easy to forget**; the alarm has already gone green, but the antenna stays parked in stow if you don't switch the mode back
3. Wait for the GDT to slew the antenna from stow back to satellite track + re-acquire lock

This is exactly what the standard ops procedure says NOT to do during a live contact. We've seen it happen by accident (operator clicks E-Stop while in stow for maintenance, forgets the mode flip on recovery). It's also the worst-case sequence if anyone hostile ever got Modbus write access — the alarm fires (so it's loud), but recovery is slower than it looks.

### Antenna power kill (40106)

40106 is more useful than 40006 if you just want to silently drop the antenna without firing an alarm — clearing it cuts drive power but does **not** publish to the broker. The collector only watches 40006 for alarms.

### env_reset gotcha (40120)

Auto-clears after one cycle. If you want to keep temps pinned, hold-write it; easier to just edit 40101.

---

## Quick recon (from the engineering subnet)

```bash
# Service / device discovery
nmap -p502 --script modbus-discover 10.4.0.210-213

# Raw holding-register dump (mbtget ships on the Kali image)
mbtget -r3 -a 0   -n 125 -u 1 10.4.0.210     # 40001..40125
mbtget -r3 -a 200 -n 20  -u 1 10.4.0.210     # 40201..40220 (nameplate + spare)

# pymodbus 3.x
python3 - <<'PY'
from pymodbus.client import ModbusTcpClient
c = ModbusTcpClient('10.4.0.210', port=502); c.connect()
print('antenna ctrl:', c.read_holding_registers(0,   9,  device_id=1).registers)
print('hvac/power :', c.read_holding_registers(99,  22,  device_id=1).registers)
print('nameplate  :', c.read_holding_registers(200, 10,  device_id=1).registers)
c.close()
PY
```

---

## Ops backlog (TODO)

- [ ] Rotate the nameplate ASCII tag at 40201–40210 — still the QA test string from factory bring-up.
- [ ] Move the OT segment behind the firewall (CR-0117 — punted from last quarter).
- [ ] Wire 40120 (ENV_RESET) to the SCADA reset button instead of one-shot.
- [ ] Switch broker (10.1.0.130:1883) to TLS + auth. Currently anonymous — anyone on the segment can sub to `stations/#`.
