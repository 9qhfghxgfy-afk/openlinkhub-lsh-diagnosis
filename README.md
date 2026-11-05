# Corsair iCUE LINK System Hub reporting **"0 devices"** / **firmware `0.0.0`**?

### It's usually a **transient enumeration state**, not a dead hub — how to tell the difference and clear it.

> **TL;DR** — If your Corsair iCUE LINK System Hub (LSH) suddenly reports **0 downstream devices**, **firmware `0.0.0`**, and every fan / pump / RGB control in your iCUE LINK control software (e.g. OpenLinkHub) is "zeroed", **don't assume the hardware is dead**. In many cases it is a **transient USB enumeration / "hub-not-ready" state**. A clean re-enumeration (restart the control app, or power-cycle the hub) restores it. Prove it with a real RGB round-trip **before** you RMA anything.

---

## Symptom

After a reboot (or on its own), the iCUE LINK System Hub appears to the host as an *empty* USB hub:

- Dashboard / device list: **"0 devices found"** / **"No RGB compatible devices found"**.
- Hub **firmware reads `0.0.0`**.
- Every downstream control table is **zeroed**: fans show no RPM, pump absent, RGB absent.
- Another iCUE LINK hub on the same system (e.g. one driving a CPU AIO) still works fine — so the host USB stack is clearly functional.

It looks exactly like the hub is dead. Often it isn't.

## Why it happens (root cause)

The iCUE LINK System Hub, when it **loses and regains USB**, is **mid-boot**, or **loses its secondary (Molex/SATA) power**, presents itself to the host as a hub with **zero downstream endpoints** and **firmware `0.0.0`**. The host — and any control software such as OpenLinkHub — faithfully reflects that: *0 devices, all control tables zeroed.*

This is a **host-side enumeration race / hub-not-ready condition**, **not** a failure of the fan motors, LED strips, or the hub ASIC. Because the control software caches the "zero" snapshot, it keeps showing 0 devices **until the hub re-enumerates cleanly**. The hardware is usually perfectly fine underneath.

The tell-tale sign of the *benign* (transient) case in the host's USB log is **repeated disconnect → connect cycles** for the hub (i.e. it's being re-enumerated), as opposed to a hard "device not accepting address" / controller reset loop, which would point to a real fault.

## How to diagnose (is it actually broken?)

Run these in order. The goal: find out whether a clean re-enumeration brings the hub back.

1. **Check the host USB log** — are there flapping disconnect/connect cycles for the hub?
   ```bash
   dmesg | grep -iE 'usb [0-9]+-[0-9.]+' | tail -30
   ```
   Repeated `disconnect` / `new USB device found` on the same port = the hub is being re-enumerated (benign).

2. **Confirm the hub has its secondary power.** Corsair LINK hubs need their **secondary power** (Molex / SATA / 5.25") *in addition to* the USB data link. A data-connected but unpowered hub reads as 0 devices.

3. **Force a clean re-enumeration.** Either **restart the control software** (OpenLinkHub, etc.) so it re-queries the hub, or **power-cycle the hub** (unplug, wait, replug).

4. **Re-query the hub.** It should now expose its real downstream devices with a **real firmware version** (not `0.0.0`).

5. **Prove the control path end-to-end** with a real RGB round-trip (adjust the port / profile / control port to yours):
   ```bash
   curl -s -X POST -H 'Content-Type: application/json' \
     -d '{"port":1,"profile":"rainbow"}' \
     http://127.0.0.1:27003/api/color
   ```
   The set response returns the resulting per-port profile map — **if your port now shows `RAINBOW`, the hub + fans + RGB are healthy** and no RMA is needed.

## The fix (clearing the transient state)

1. *(Optional, for capture)* Enable debug logging in OpenLinkHub so you can see the enumeration sequence:
   ```json
   { "debug": true, "log_level": "debug" }
   ```
2. **Restart the control app** (or restart the whole host) to trigger a fresh enumeration.
3. If it still reports 0 devices, **power-cycle the hub** and make sure its **secondary power** is firmly connected.
4. Re-open the dashboard — the downstream fans should appear with **real firmware**.
5. **Verify** with the RGB round-trip above (and tach/RPM reads for the fans).
6. *(If you enabled it)* Set `"debug": false` again before leaving it running.

> The single most reliable "fix" is simply a **clean re-enumeration of the hub** — restart the control software, power-cycle the hub, or reboot. The zero-device state is transient and clears on a good re-enumeration.

## Example verified topology

A concrete case where this resolved with **no hardware action**:

- **Hub A** — Corsair iCUE LINK System Hub: **3 × iCUE LINK RX RGB** case fans.
- **Hub B** — older Corsair iCUE LINK hub: **1 × CPU AIO pump + 2 × radiator fans**.

Hub A reported **0 devices / firmware 0.0.0** while Hub B worked normally. After a clean re-enumeration, Hub A enumerated all **3 fans** with real firmware and **RGB writes round-tripped** over the API. The hardware was healthy the whole time.

## When it *is* a hardware problem

Treat it as a real fault if, after **multiple** clean re-enumerations, you still get:

- The hub **never** exposes its downstream devices, **or**
- Firmware **stays `0.0.0`** across several full power cycles, **or**
- The host log shows the device **failing to be recognized at all** (no `new USB device found`).

At that point try, in order: a **different USB port/controller**, a **known-good secondary power source**, a **different USB cable** (if applicable). If it's still dead, the hub is a candidate for **RMA**.

## Notes

- Keep `"debug": false` in your production config.
- Observed on Linux over USB with OpenLinkHub (a self-hosted iCUE LINK control server). The same class of transient applies to any iCUE LINK host.
- Example endpoints (e.g. `/api/color`) and the default control port (e.g. `27003`) vary slightly by OpenLinkHub version — point the examples at your own port and routes.

---

*Counter-example that turned out to be real hardware, or an extra symptom? Open an issue / PR.*
