# Philips SpeechMike (composite USB audio+HID) not appearing in Omnissa/VMware Horizon Client USB redirection

## Symptom

A Philips SpeechMike (III/Premium/Air-family devices are all similar composite
USB devices) is plugged into the Horizon Client machine and shows up fine in
`lsusb`, but it **never appears at all** in the Horizon Client's USB devices
menu — not greyed out, not listed as blocked, just absent. Only simple
single-function devices (e.g. a fingerprint reader) show up.

Tested with:
- Device: Philips SpeechMike III, `idVendor=0911`, `idProduct=0c1c`
- Client: Omnissa Horizon Client for Linux, `2603-8.18.0-24120621798`, on Pop!_OS 24.04

## Why this happens

The SpeechMike is a **composite USB device** with 7 interfaces:

- Interface 0: Audio Control
- Interface 1: Audio Streaming (microphone in)
- Interface 2: Audio Streaming (speaker out)
- Interfaces 3–6: four separate HID interfaces (the device's buttons/jog wheel;
  one of them advertises the USB HID **boot "Mouse" protocol**)

Horizon Client's `horizon-usbd` process runs every USB device through a
device filter (`DevFltr`) before it's ever offered in the UI. By default,
several of the relevant policy keys are unset, which means:

1. **Composite device splitting is off** (`viewusb.AllowAutoDeviceSplitting`
   unset) — the audio and HID parts of the device can't be redirected
   independently. It's forwarded as a single unit or not at all.
2. **Audio interfaces are blocked by default**
   (`viewusb.AllowAudioIn` / `viewusb.AllowAudioOut` unset).
3. **The boot-protocol "mouse" HID interface is blocked by default**
   (`viewusb.AllowKeyboardMouse` unset) — this is Horizon's standard
   anti-hijack protection against a redirected device pretending to be a
   keyboard/mouse and taking over the session.

Because splitting is off, a single blocked family anywhere in the composite
device blocks the *entire* device. You can only see this by turning on
debug-level logging — at the default `INFO` level, `horizon-usbd` never logs
the actual filter decision, just a "Device Speed" line per enumerated device.

## Diagnosis: turn on debug logging

1. System-wide log verbosity (`log.fileLevel` controls the file sink; the
   `loglevel.user.usb` key controls the USB subsystem specifically) goes in
   `/etc/omnissa/config` (root-owned):

   ```
   log.fileLevel = "debug9"
   loglevel.user.usb = "10"
   ```

2. The `horizon-usbd` process banner level (a separate knob) goes in the
   **user** config, `~/.omnissa/config`:

   ```
   view-usbd.logLevel = "DEBUG"
   ```

3. Fully quit Horizon Client (not just disconnect — the log level is only
   read once, at process startup) and relaunch it.

4. Look at the newest `horizon-usbd-<pid>.log` under
   `/tmp/omnissa-<user>-*/` (or `/tmp/omnissa-<user>/`). Grep for your
   device's name or VID:PID:

   ```
   grep -iE "block|allow|filter|split" horizon-usbd-<pid>.log
   ```

   For the SpeechMike this showed:

   ```
   DevFltr: [Combined:Phase] AutoDeviceSplitting blocked. Skipping 1(b)
   DevFltr: All audio interfaces are blocked by AutoFilters
   DevFltr: [Combined] Device blocked by AutoFilters. Family(s): audio-in,audio-out,mouse
   Filter Result: Device 'Philips SpeechMike III' is blocked
   ```

   That single block of log lines is the smoking gun — it names exactly
   which families are blocking the device.

## The fix

Add the following to `/etc/omnissa/config` (system-wide; requires root —
Horizon does not appear to honor these particular `viewusb.*` policy keys
from the per-user `~/.omnissa/config`, only the system one):

```
viewusb.AllowAudioIn = "TRUE"
viewusb.AllowAudioOut = "TRUE"
viewusb.AllowKeyboardMouse = "TRUE"
viewusb.AllowHID = "TRUE"
viewusb.AllowHIDBootable = "TRUE"
viewusb.AllowAutoDeviceSplitting = "TRUE"
```

```
printf 'viewusb.AllowAudioIn = "TRUE"\nviewusb.AllowAudioOut = "TRUE"\nviewusb.AllowKeyboardMouse = "TRUE"\nviewusb.AllowHID = "TRUE"\nviewusb.AllowHIDBootable = "TRUE"\nviewusb.AllowAutoDeviceSplitting = "TRUE"\n' | sudo tee /etc/omnissa/config
```

Then fully quit and relaunch Horizon Client again, reconnect to your VM, and
open the USB devices menu — the device should now be listed and connectable.

`AllowKeyboardMouse = TRUE` genuinely does relax Horizon's anti-hijack
protection for keyboard/mouse-emulating USB devices system-wide for this
client. That's an accepted tradeoff for using the SpeechMike's jog
wheel/buttons in a VDI session, not an oversight — know what it does before
enabling it.

## Verifying the fix worked

In the debug log, on a successful connect you should see the arbitrator hand
ownership of the device to the client's `USBD<pid>` process, and `usbd` claim
**all 7 interfaces**, e.g.:

```
usbArb  ... Claiming device for client 'USBD<pid>'.
usbArb  ... Device 3: name:Philips\ SpeechMike\ III ... owner:USBD<pid>.
horizon-usbd USBGL: ... Claimed device interface(0) successfully.
...
horizon-usbd USBGL: ... Claimed device interface(6) successfully.
```

Inside the guest VM, the SpeechMike should now be selectable as an audio
input/output device, and its buttons should register as HID input.

## Cleaning up afterward

Once confirmed working, remove the debug-logging lines — they're not needed
for the fix to keep working, only for diagnosing it:

```
# Reset /etc/omnissa/config to just the six viewusb.Allow* lines above
rm -f ~/.omnissa/config   # only if it contains just the view-usbd.logLevel debug line
```

## Notes / caveats

- The exact `viewusb.*` key names were confirmed live from the debug log's
  `DevFltr: Reading config, viewusb.<Key>=` lines, not guessed from
  documentation — they should be accurate for this client build, but key
  names have changed across VMware View → VMware Horizon → Omnissa Horizon
  rebrands over the years. If your `horizon-usbd` log shows different key
  names being read, use those instead.
- This same failure mode (composite audio+HID device invisible in the USB
  menu) will affect **any** similar composite device — other dictation
  microphones, some barcode scanners, some footswitches — not just the
  SpeechMike. The diagnosis steps above generalize; only the specific VID:PID
  differs.
