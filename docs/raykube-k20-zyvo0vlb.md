# Raykube K20 smart lock (`zyvo0vlb`)

Local BLE support for the Raykube K20 (Tuya model F302), a `jtmspro` smart lock.

## Tested device

- Category: `jtmspro`
- Product ID: `zyvo0vlb`
- Model: `F302`, firmware 0.3 (protocol 3.3)
- BLE advertisement service data: `0000a201-0000-1000-8000-00805f9b34fb`
  (classic Tuya BLE framing, **not** the FD50 service used by the A1 Ultra)
- Tested through an ESPHome Bluetooth proxy (ESP32-C3)
- The tested unit is powered from an external supply instead of batteries, so the
  battery sensor always reads a constant value there

## What works (verified on a real lock)

- Lock state (`lock_motor_state`, DP 47): `false` = locked, `true` = unlocked
- Remote lock (`manual_lock`, DP 46)
- Remote unlock (see below)

## Implemented but not yet verified

These use the same DP write path as lock/unlock, but have not been confirmed on
hardware yet. Please report back if they work for you:

| Entity | Type | DP | Notes |
|---|---|---|---|
| Passage Mode | switch | 33 (`automatic_lock` in the Tuya schema) | on = door stays unlocked; behaves as passage mode, not auto-lock (confirmed on hardware) |
| Lock Volume | select | 31 `beep_volume` | mute / low / normal / high |
| Auto-lock Delay | select | 89 `autolock_delay_level` | off / 4 / 6 / 8 / 10 / 12 s |
| Battery | sensor | 8 `residual_electricity` | constant 100 % on mains-powered units |
| Alarm | sensor | 21 `alarm_lock` | disabled by default, needs *Keep connected* |
| Last fingerprint / password / card / Bluetooth unlock | sensors | 12 / 13 / 15 / 19 | disabled by default, needs *Keep connected* |
| Doorbell | binary sensor | 24 `doorbell` | disabled by default, needs *Keep connected* |
| Locked From Inside | binary sensor | 32 `reverse_lock` | inner deadbolt engaged |

The Czech translation (`translations/cs.json`) is included as a partial file;
missing keys fall back to English.

DP 34 (`unlock_switch`, single/combined unlock) is intentionally **not**
exposed, because a wrong value can lock you out of every unlock method.

## Required `devices.json` fields

```json
{
  "XX:XX:XX:XX:XX:XX": {
    "address": "XX:XX:XX:XX:XX:XX",
    "uuid": "<device UUID>",
    "local_key": "<device local key>",
    "device_id": "<device ID>",
    "category": "jtmspro",
    "product_id": "zyvo0vlb",
    "device_name": "Front door",
    "product_model": "F302",
    "product_name": "Smart lock",
    "ble_unlock_check": "<base64 raw Tuya status value of DP 71>"
  }
}
```

`ble_unlock_check` is required for remote unlock. Without it, lock and state
work, but unlock fails with a configuration error.

### Getting `ble_unlock_check`

1. Open the Tuya IoT Platform → Cloud → API Explorer for the project the lock is linked to.
2. Run **Query Properties** (`/v2.0/cloud/thing/{device_id}/shadow/properties`) for your device.
3. In the result find the property with `"code": "ble_unlock_check"` (`dp_id` 71) and copy the whole
   `value` string (base64, about 28 characters) into `devices.json`.

The same API response also contains the DP list of your device, which is useful for
checking that it matches the table above.

> **Security:** `ble_unlock_check` contains the unlock code of your lock. Treat it like
> `local_key`: never post it in issues, logs or screenshots, and keep `devices.json` out
> of shared backups.

## How remote unlock works

The K20 has no "unlock" DP. In the integration the lock entity writes the
internal trigger DP 6, which `tuya_ble.py` converts into a normal Tuya BLE DPS
write of **DP 71** (raw, 19 bytes):

```
<2 byte prefix> 00 01 <8 ASCII digits> 01 <4 byte key> 00 01
```

taken from `ble_unlock_check` (the same layout the Raykube A1 Ultra uses in its V4 framing,
here sent with the classic V3 DPS command). Lock uses the plain `manual_lock` DP 46.

## Behaviour and limitations

- The lock connects on demand (`keep_connect` is off). After a long idle period the
  first command can take a few seconds while the BLE link is established.
- Entities of type sensor/binary sensor only receive new values while a BLE session is open,
  so they can be stale or unavailable between commands. The settings entities (switch/select)
  stay available and are written on demand.
- **Alarm, doorbell and last-unlock sensors** are event data points: the lock only pushes
  them while a BLE session is open. They are therefore disabled by default. To use them,
  enable the entities and turn on *Keep connected (24/7)* in the device options. The K20
  has no DP 52 (the no-op used by other locks), so the keep-alive requests the status every
  60 s. This mode is experimental: an earlier test with a 20 s interval did not improve
  stability, so leave it off if it makes the lock less reliable.
- **Jammed state**: a sleeping lock may run a queued command only after waking up. The
  lock entity waits 90 s (instead of 12 s) before reporting *jammed*, and the jammed flag
  is cleared as soon as the lock reaches the commanded state later, changes state on its
  own, or a new command is sent.
- `Unexpcted packet` / `Missing packet` errors in the Home Assistant log (a BLE notification
  fragment was lost while reassembling a multi-packet reply) appeared occasionally during
  testing, together with the occasional delay, even with a strong signal (-39 dBm).
  Lock, unlock and state still worked. A Bluetooth proxy close to the lock is recommended.
- The advertisement-gated on-demand mode used for `j3n6jw2y` is not enabled for this lock,
  because it has not been tested with the K20.

## Supported device data points

| DP | Code | Type | Used for |
|---|---|---|---|
| 8 | `residual_electricity` | value 0-100 | battery sensor |
| 12 / 13 / 15 / 19 | `unlock_fingerprint` / `unlock_password` / `unlock_card` / `unlock_ble` | value | last unlock sensors |
| 21 | `alarm_lock` | enum | alarm sensor |
| 24 | `doorbell` | bool | doorbell |
| 31 | `beep_volume` | enum | volume |
| 32 | `reverse_lock` | bool | locked from inside |
| 33 | `automatic_lock` | bool | passage mode (stays unlocked) |
| 46 | `manual_lock` | bool | remote lock |
| 47 | `lock_motor_state` | bool | lock state |
| 71 | `ble_unlock_check` | raw | remote unlock |
| 89 | `autolock_delay_level` | enum | auto-lock delay |
