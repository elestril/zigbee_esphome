# Design: On/Off commands to bound devices, and bindings in YAML

Status: draft for a local proof of concept (PoC). Not yet discussed upstream.
Branch: `feature/onoff-send-binding` on `elestril/zigbee_esphome` (fork of `luar123/zigbee_esphome`).
Base commit: `d31c2f2` (upstream `master`, 2026-09-17).

## 1. Goal

Let ESP32-C6 nodes running this component control each other directly over Zigbee,
so that control keeps working when Home Assistant, ZHA and the coordinator stick are down.

Target system:

- Several mains-powered ESP32-C6 nodes, Zigbee routers, no Wi-Fi in normal operation.
- Each node reads 24 VAC thermostat call lines (W, Y, G, O/B) through AC optocouplers,
  and drives relays from GPIO outputs.
- A thermostat call on one node must switch relays on other nodes.
- HA/ZHA is used for commissioning and monitoring only. It is never in the control path.
- Wi-Fi is acceptable only temporarily, for OTA firmware updates.

This is critical infrastructure (HVAC). Fail-safe behaviour matters more than features.

## 2. What already exists

### 2.1 In this component (upstream `master`)

| Capability | Where | Notes |
|---|---|---|
| On/Off **server** endpoint driving an ESPHome switch/light | `endpoints:` + `on_value` on attribute 0 of `ON_OFF` | Incoming On/Off commands from any source update the attribute, which fires `on_value`. The relay side needs no new code. |
| Client-role clusters | `role: CLIENT` in endpoint schema | Creates the cluster, nothing sends from it. |
| Incoming attribute reports on client clusters | `on_report` trigger | Gives source address and endpoint. |
| Thread-safe hand-off from Zigbee task to ESPHome loop | `ZBEvent` + `LockFreeQueue` + `EventPool` in `esp_zb_event.h`, `zigbee.cpp` | New callbacks from the stack must use this path. |
| Reading own binding table after reboot | `ZigBeeComponent::searchBindings()` / `bindingTableCb()` | Logs entries, then discards them. Leftover from Espressif's switch example. |
| Wi-Fi coexistence | `CONFIG_WIFI_COEX` path in `setup()` | Router + Wi-Fi triggers a warning in `final_validate`. |

### 2.2 Not planned upstream, with an open invitation

README, section "Not planned (feel free to submit a pull request)":

- Binding config in yaml
- Control device support like switches

In discussion #18 (Jan 2025) the maintainer proposed a `zigbee.sendCmd` action modelled
on `zigbee.setAttr`, with command enums generated from SDK headers by
`files_to_parse/parse_zigbee_headers.py`, and pointed at Espressif's Arduino
`ZigbeeSwitch.cpp` as reference. The workaround posted there (a lambda calling the
SDK directly) uses SDK 1.x names and no longer compiles against `master` (SDK 2.x, `ezb_*`).

### 2.3 SDK 2.x APIs we will use

Header paths are relative to `espressif__esp-zigbee-lib/include/ezbee/`. Component pins `esp-zigbee-lib` `2.0.4`.

| Purpose | API | Header |
|---|---|---|
| Send On / Off / Toggle | `ezb_zcl_on_off_on_cmd_req`, `..._off_cmd_req`, `..._toggle_cmd_req` taking `ezb_zcl_on_off_cmd_t` | `zcl/cluster/on_off.h` |
| Send On-with-timed-off | `ezb_zcl_on_off_on_with_timed_off_cmd_req` taking `ezb_zcl_on_off_on_with_timed_off_cmd_t` (`on_time`, `off_wait_time` in 1/10 s) | `zcl/cluster/on_off.h` |
| Addressing / endpoints | `ezb_zcl_cluster_cmd_ctrl_t { dst_addr, dst_ep, src_ep, dis_default_rsp, cnf_ctx }` | `zcl/zcl_common.h` |
| "Send to bound targets" | `dst_addr.addr_mode = EZB_ADDR_MODE_NONE` (APS: no destination, use binding table) | `core_types.h` |
| Explicit target | `EZB_ADDR_MODE_EXT` (IEEE + endpoint), `EZB_ADDR_MODE_SHORT`, `EZB_ADDR_MODE_GROUP` | `core_types.h` |
| Delivery confirmation | `cnf_ctx = { cb, user_ctx }`, callback `void (*)(ezb_af_user_cnf_t *cnf, void *user_ctx)`, `cnf->status`, `tsn`, `dst_addr` | `af.h` |
| Create binding | `ezb_zdo_bind_req(ezb_zdo_bind_req_t *)`, fields `src_addr`, `src_ep`, `cluster_id`, `dst_addr_mode` (0x01 group, 0x03 IEEE) | `zdo/zdo_bind_mgmt.h` |
| Read binding table | `ezb_zdo_nwk_mgmt_bind_req` (already used) | `zdo/zdo_nwk_mgmt.h` |
| Stack lock | `esp_zigbee_lock_acquire(ticks)` / `esp_zigbee_lock_release()` | `esp_zigbee.h` |

## 3. Scope

### In scope for the PoC

1. **`zigbee.sendCmd` action, On/Off cluster only:** commands `ON`, `OFF`, `TOGGLE`, `ON_WITH_TIMED_OFF`.
   Default target is "bound devices"; explicit IEEE/short/group targets are optional.
2. **Delivery result trigger** so YAML can react to failed sends (log, retry, local fallback).
3. **`bindings:` in YAML:** the node ensures its own binding table contains the configured
   entries after every join and reboot, without ZHA.
4. Example YAML and README section for all of the above.

### Out of scope

- Other clusters' commands (Level, Color, Window Covering). The design keeps `sendCmd`
  generic so they can be added later via the header parser.
- Group management (`ADD_GROUP` on receivers), scenes, reporting configuration.
- Coordinator role. The network is formed by ZHA's coordinator (option A, section 7).
- Zigbee OTA (stays "not planned"; Wi-Fi OTA is used).

## 4. YAML interface

### 4.1 Sender endpoint

The sender needs an endpoint with an On/Off **client** cluster. One endpoint per independent
output signal, because bindings are per (endpoint, cluster).

```yaml
zigbee:
  id: zb
  router: true
  endpoints:
    - num: 10
      device_type: ON_OFF_SWITCH
      clusters:
        - id: ON_OFF
          role: CLIENT
    - num: 11
      device_type: ON_OFF_SWITCH
      clusters:
        - id: ON_OFF
          role: CLIENT
```

### 4.2 `zigbee.sendCmd`

```yaml
binary_sensor:
  - platform: gpio
    id: call_w
    pin: {number: GPIO0, mode: {input: true, pullup: true}, inverted: true}
    filters: [delayed_on: 50ms, delayed_off: 100ms]
    on_press:
      - zigbee.sendCmd:
          id: zb
          endpoint: 10          # source endpoint (must have ON_OFF client)
          cluster: ON_OFF
          command: ON_WITH_TIMED_OFF
          on_time: 90s          # receiver turns itself off after 90 s without refresh
    on_release:
      - zigbee.sendCmd: {id: zb, endpoint: 10, cluster: ON_OFF, command: OFF}
```

Schema:

| Key | Type | Default | Notes |
|---|---|---|---|
| `id` | `ZigBeeComponent` id | required (or single instance) | |
| `endpoint` | int 1–240 | required | Must have the cluster in client role. Validated at config time. |
| `cluster` | enum | required | PoC: `ON_OFF` only. |
| `command` | enum, templatable | required | `ON`, `OFF`, `TOGGLE`, `ON_WITH_TIMED_OFF`. |
| `on_time` | time period, templatable | none | Required for `ON_WITH_TIMED_OFF`. Encoded in 1/10 s, max 6553.5 s. |
| `off_wait_time` | time period | 0 | `ON_WITH_TIMED_OFF` only. |
| `target` | map | bound devices | `{ieee: "58:e6:...", endpoint: 1}`, `{short: 0x3C69, endpoint: 1}` or `{group: 0x0101}`. |
| `default_response` | bool | true | Ask the receiver for a ZCL Default Response. |

Naming follows the existing camelCase actions (`setAttr`, `reportAttr`) and the
maintainer's proposed name. Enum values come from a small hand-written table in the PoC;
section 9 notes moving them to the header parser before upstreaming.

### 4.3 Delivery result trigger

```yaml
zigbee:
  on_send_result:
    - if:
        condition:
          lambda: "return !x.success;"
        then:
          - logger.log:
              format: "send failed ep=%u tsn=%u status=0x%02x"
              args: [x.src_endpoint, x.tsn, x.status]
```

`x` is a struct `{bool success; uint8_t status; uint8_t tsn; uint8_t src_endpoint; uint16_t cluster_id; ezb_address_t dst_addr;}`.
Fires in the ESPHome main loop, never in the Zigbee task.

### 4.4 `bindings:`

```yaml
zigbee:
  bindings:
    - endpoint: 10
      cluster: ON_OFF
      target: {ieee: "58:e6:c5:ff:fe:14:43:7c", endpoint: 1}
    - endpoint: 11
      cluster: ON_OFF
      target: {group: 0x0101}
  bindings_mode: ensure     # ensure | off
```

- `ensure`: after `EZB_BDB_SIGNAL_STEERING` success and after `DEVICE_REBOOT`, compare the
  configured list with the local binding table (from `searchBindings()`), and send
  `ezb_zdo_bind_req` to the node's own short address for each missing entry.
- Extra entries (for example made in ZHA) are left alone. No unbinding in the PoC.
- Target IEEE addresses are stable per chip, so they can be written into YAML once,
  from the device label or from ZHA's device page.

## 5. Implementation

### 5.1 Files

| File | Change |
|---|---|
| `components/zigbee/__init__.py` | Register `zigbee.sendCmd`, schema and validation (endpoint has client cluster, `on_time` present for timed command). Add `bindings`, `bindings_mode`, `on_send_result` to `CONFIG_SCHEMA`; codegen calls `add_binding(...)`. |
| `components/zigbee/automation.h` | `SendCmdAction<Ts...>` with templatable `command`, `on_time`, `off_wait_time`; `play()` calls `parent_->send_on_off(...)`. `SendResultTrigger`. |
| `components/zigbee/zigbee.h` / `zigbee.cpp` | `send_on_off(...)`, `add_binding(...)`, `ensure_bindings_()`, confirmation callback, store binding table entries instead of discarding them. |
| `components/zigbee/esp_zb_event.h` | New `ZBEvent` kind for send confirmations (fixed size, no heap). |
| `example_esp32c6_onoff_sender.yaml`, `..._receiver.yaml` | PoC examples. |
| `README.md` | New section; move the two items out of "Not planned" when done. |

### 5.2 Sending (main loop context)

```cpp
bool ZigBeeComponent::send_on_off(uint8_t src_ep, OnOffCmd cmd, const ZbTarget &target,
                                  uint16_t on_time_ds, uint16_t off_wait_ds, bool default_rsp) {
  ezb_zcl_cluster_cmd_ctrl_t ctrl{};
  ctrl.src_ep = src_ep;
  ctrl.dis_default_rsp = !default_rsp;
  target.fill(&ctrl.dst_addr, &ctrl.dst_ep);       // NONE for bound, EXT/SHORT/GROUP otherwise
  ctrl.cnf_ctx.cb = &ZigBeeComponent::send_cnf_cb_;
  ctrl.cnf_ctx.user_ctx = nullptr;

  if (!esp_zigbee_lock_acquire(pdMS_TO_TICKS(20))) {
    ESP_LOGW(TAG, "sendCmd: stack busy, dropped");   // caller sees false; YAML can retry
    return false;
  }
  ezb_err_t err;
  switch (cmd) { /* on / off / toggle / on_with_timed_off */ }
  esp_zigbee_lock_release();
  return err == EZB_ERR_NONE;
}
```

Rules:

- Never block the ESPHome loop with `portMAX_DELAY`. The existing commissioning code uses a
  short timeout plus `set_timeout` retry; do the same, or drop and report.
- Refuse to send before `is_connected()`; log at warning level.
- `send_cnf_cb_` runs in the Zigbee task. It only copies the fields into a pooled `ZBEvent`
  via `enqueue_zb_event(...)`. `loop()` then fires `on_send_result`.

### 5.3 Bindings

- Change `bindingTableCb` to fill a `std::vector<BindingEntry>` (src_ep, cluster, mode, addr, dst_ep) instead of only logging.
- After the table is complete, `ensure_bindings_()` issues one `ezb_zdo_bind_req` per missing entry,
  with `dst_nwk_addr = ezb_get_short_address()` and `field.src_addr` = own IEEE.
- Bind results arrive in the Zigbee task; log them via the event queue.
- The steering-success path currently does not call `searchBindings()`; add it, so `ensure`
  also runs after the first join.

### 5.4 Receiver side

No new C++ is required. A relay node uses the existing pattern:

```yaml
zigbee:
  endpoints:
    - num: 1
      device_type: ON_OFF_OUTPUT
      clusters:
        - id: ON_OFF
          attributes:
            - attribute_id: 0
              type: bool
              report: true
              on_value:
                then:
                  - switch.control: {id: relay_1, state: !lambda "return x;"}
```

Fail-safe on the receiver uses one of two mechanisms, decided by PoC test T6:

1. **Preferred:** the stack implements `OnWithTimedOff` (attributes `OnTime` 0x4001 and
   `OffWaitTime` 0x4002 on the server). The sender refreshes `ON_WITH_TIMED_OFF` well inside
   `on_time`; if refreshes stop, the stack turns the attribute off and `on_value` fires.
2. **Fallback, YAML only:** each `on_value` with `true` restarts a `script` (`mode: restart`)
   that waits the timeout and then turns the relay off. This needs `on_value` to fire on
   repeated `ON` commands even when the value does not change, which T5 checks.

## 6. Fail-safe and safety rules

These belong in every node's YAML regardless of transport:

1. Relays boot `ALWAYS_OFF`.
2. Senders refresh state periodically (`interval:` resend of the current input state, for example every 30 s),
   not only on edges. A lost edge then heals within one period.
3. Receivers time out to off (section 5.4).
4. Interlocks that protect equipment (heat and cool never together, compressor minimum off time)
   run on the node that drives the equipment, from local state only.
5. `TOGGLE` is never used for HVAC calls.
6. The XIAO ESP32-C6 needs GPIO3 driven low (RF switch power) and GPIO14 low (onboard antenna)
   before Zigbee starts, or it cannot join. Use internal GPIO switches with `restore_mode: ALWAYS_OFF`.

## 7. Network topology and what depends on the coordinator

PoC uses **option A**: nodes join the existing ZHA network (coordinator: Home Assistant Connect ZBT-1, channel 25, PAN 0xDB44).

| Situation | Needs coordinator? | Expected behaviour |
|---|---|---|
| Steady-state bound On/Off traffic between routers | No | Works. |
| Node reboot with network data in NVRAM | No | "Device rebooted", resumes without rejoin. |
| Node factory reset / new node | Yes (trust center) | Cannot join until ZHA is back. |
| Target's short address changed | No (address discovery) | Bound-by-IEEE sends should resolve. Verify in T8. |
| Network key rotation, channel change | Yes | Initiated by coordinator only; nothing happens while it is down. |

Option B (one node acts as coordinator for a separate, HA-free network) is out of scope:
the component lists coordinator devices as not planned, and it would remove ZHA monitoring.

## 8. PoC plan

Hardware: two XIAO ESP32-C6. Node S (sender): one input on a GPIO (button or opto), endpoint 10 ON_OFF client.
Node R (receiver): onboard LED (GPIO15, active low) as the "relay", endpoint 1 ON_OFF server.
Both: routers, Wi-Fi off, antenna pins set.

Build note: the dashboard uses the native ESP-IDF toolchain. Basic mode (`components:`)
is rejected without PlatformIO (`final_validate`), so the PoC uses advanced mode (`endpoints:`) only.
Endpoint limit is 10 per device.

| # | Test | Pass criterion |
|---|---|---|
| T1 | Both nodes join ZHA; signatures show ON_OFF client on S/ep10 and server on R/ep1 | Visible in ZHA device page |
| T2 | Bind S/ep10 to R/ep1 in ZHA; press input | LED follows within 100 ms; `on_send_result` success |
| T3 | Same binding created by `bindings:` YAML on a fresh node pair (no ZHA binding) | Same as T2 |
| T4 | `ha core stop` (or unplug ZBT-1); repeat T2 | Still works |
| T5 | Send `ON` twice in a row | Record whether `on_value` fires twice |
| T6 | `ON_WITH_TIMED_OFF` with 10 s; stop refreshing | LED off after about 10 s; ZHA shows off once back |
| T7 | Power off R; press input on S | `on_send_result` reports failure |
| T8 | Reboot S and R while coordinator is down | Both resume; T2 passes |
| T9 | Unbound send (no binding entry) | Clear warning, no crash |
| T10 | 1000 sends at 5 Hz | No dropped-event warnings, no heap growth |

Record results in `docs/poc-results.md`.

## 9. Upstreaming plan (after the PoC)

1. Open an issue on `luar123/zigbee_esphome` with PoC results, referencing discussion #18.
2. Split into three PRs, smallest first: `sendCmd` for ON_OFF, `on_send_result`, `bindings:`.
3. Before the `sendCmd` PR, generate command IDs and request functions from SDK headers with
   `parse_zigbee_headers.py`, as the maintainer suggested, so other clusters follow without hand-written tables.
4. Provide example YAML; the maintainer asked for one on PR #80.
5. Note the related open PR #80 (custom command receive hooks, channel retention) and avoid overlapping its event changes.

## 10. Open questions

- Does `EZB_ADDR_MODE_NONE` produce one confirmation per bound destination, or one for the request?
- Does the SDK's On/Off server implement `OnWithTimedOff` timing on its own (T6)?
- Is `on_value` fired for an unchanged value (T5)?
- How do routers behave with Wi-Fi enabled briefly for OTA? Plan: `wifi: enable_on_boot: false`, enabled by a local long-press or a ZHA-written attribute, never automatically.
- Should `sendCmd` be refused while `is_connected()` is false, or queued?
