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
- Also reproduced and fixed on **macOS** (Horizon Client Next 8.17.1, Apple
  silicon, macOS 15.7.4) — the cause is identical but the config keys and their
  location are completely different. See
  [macOS](#macos-same-bug-completely-different-config-plumbing) below; the
  `viewusb.*` keys in this section do **not** apply there.

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

## Diagnosis: turn on debug logging (Linux)

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

## The fix (Linux)

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

## Verifying the fix worked (Linux)

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

## Cleaning up afterward (Linux)

Once confirmed working, remove the debug-logging lines — they're not needed
for the fix to keep working, only for diagnosing it:

```
# Reset /etc/omnissa/config to just the six viewusb.Allow* lines above
rm -f ~/.omnissa/config   # only if it contains just the view-usbd.logLevel debug line
```

## macOS: same bug, completely different config plumbing

Tested with:
- Device: same Philips SpeechMike III, `idVendor=0911`, `idProduct=0c1c`
- Client: Omnissa Horizon Client Next `8.17.1` (build-22261165306), Apple
  silicon, macOS 15.7.4

The device filter is the same code (`bora/apps/viewusb/...` paths appear in the
mac binary too) and fails the same way. What does **not** carry over is where
the policy lives.

**`viewusb.*` keys do not work on macOS.** The mac build reads its filter
policy via `CFPreferencesCopyAppValue` from its own preferences domain,
`com.omnissa.usb`, using **bare key names with no `viewusb.` prefix**. The
Linux-style dict files (`config`, `preferences`) do exist on macOS and load
fine — but `viewusb.*` entries in them are silently ignored by `DevFltr`. No
error, no warning; the keys just read back empty.

Path map:

| Linux | macOS |
| --- | --- |
| `/etc/omnissa/config` (`viewusb.Allow*`) | `defaults` domain `com.omnissa.usb` (bare `Allow*`) |
| `~/.omnissa/config` (`view-usbd.logLevel`) | `~/Library/Preferences/Omnissa Horizon/config` |
| `/tmp/omnissa-<user>-*/` | `~/Library/Logs/Omnissa/` |

### The fix (macOS)

```bash
for k in AllowKeyboardMouse AllowHID AllowHIDBootable \
         AllowAudioIn AllowAudioOut AllowAutoDeviceSplitting; do
  defaults write com.omnissa.usb "$k" -string TRUE
done
```

Then fully quit Horizon Client (Cmd-Q — not just disconnect) and relaunch.

System-wide (all users on the Mac), same keys against the system domain:

```bash
sudo defaults write /Library/Preferences/com.omnissa.usb AllowKeyboardMouse -string TRUE
# ...repeat for the other five keys...
sudo chmod 644 /Library/Preferences/com.omnissa.usb.plist
```

A per-user `~/Library/Preferences/com.omnissa.usb.plist` takes precedence over
the system domain, so remove it if you want the system file to be the single
source of truth.

Undo: `defaults delete com.omnissa.usb`.

`AllowKeyboardMouse = TRUE` carries exactly the same anti-hijack tradeoff on
macOS as it does on Linux — read the note in the Linux section above before
enabling it.

### Diagnosis on macOS

Debug logging goes in `~/Library/Preferences/Omnissa Horizon/config` (create
the directory; it does not exist by default):

```
log.fileLevel = "debug9"
loglevel.user.usb = "10"
view-usbd.logLevel = "DEBUG"
```

Logs land in `~/Library/Logs/Omnissa/horizon-usbd-<pid>.log`, where `<pid>` is
the client's own pid — `usbd` runs in-process with `horizon-client` on macOS,
not as a root daemon, which is why the per-user `defaults` domain is enough and
no root config file is required.

Two things to know when reading the log:

1. **Filtering runs at desktop connect, not at client launch.** There are no
   `Filter Result` lines until you actually connect to a VM. An empty grep
   right after launching the client means nothing.
2. **The `Reading config` lines tell you whether your keys landed:**

   ```
   DevFltr: Reading config, AllowKeyboardMouse=TRUE
   DevFltr: Reading config, AllowAutoDeviceSplitting=TRUE
   ```

   An empty right-hand side (`AllowKeyboardMouse=`) means the value is not
   being read at all — wrong domain or wrong key name, not a wrong value.

The block itself, on macOS:

```
IdentifyDeviceFamily(): Not implemented on OS X
DevFltr: Interface [0] - Family(s): audio
DevFltr: Interface [1] - Family(s): audio,audio-in
DevFltr: Interface [2] - Family(s): audio,audio-out
DevFltr: Interface [3] - Family(s): mouse
DevFltr: Interface [4] - Family(s): hid
DevFltr: Interface [5] - Family(s): hid
DevFltr: Interface [6] - Family(s): hid
DevFltr: [Combined:Phase] AutoDeviceSplitting blocked. Skipping 1(b)
DevFltr: Audio interface: audio-out is blocked using AutoFilter setting. The setting is ignored as there are multiple audio interfaces which shouldnt be sp
DevFltr: [Combined] Device blocked by AutoFilters. Family(s): mouse
Filter Result: Device 'Philips SpeechMike III' is blocked
```

Note the difference from the Linux log: only `mouse` blocks here. The audio
AutoFilter disables itself on a device with multiple audio interfaces. So on
this build `AllowKeyboardMouse` and `AllowAutoDeviceSplitting` are the two
load-bearing keys; the audio pair is belt-and-braces.

On success:

```
Filter Result: [UsbDeviceId: 2000000109110c1c] Device 'Philips SpeechMike III' is allowed
Claimed 'Philips SpeechMike III' device, PlugNo: 1
```

### How the `com.omnissa.usb` domain was found

Worth recording, because no documentation names it and the config file is a
dead end:

```bash
L="/Applications/Omnissa Horizon Client Next.app/Contents/MonoBundle/libhorizon-usbd.dylib"

# 1. Policy is read through CFPreferences, not the dict files:
nm -u "$L" | grep CFPreferences
#   _CFPreferencesCopyAppValue
#   _kCFPreferencesCurrentApplication

# 2. Find the call site and the global appID it passes:
otool -tV -arch arm64 "$L" | grep -n CFPreferencesCopyAppValue
```

The getter is generic — it takes a C string key, wraps it with
`CFStringCreateWithCString`, and passes a global appID `CFStringRef` loaded
from `__DATA_CONST`. Following that `__cfstring` entry's data pointer into
`__TEXT,__cstring` gives a 15-byte string: `com.omnissa.usb`. It is also
visible in plain `strings` output right next to the `Allow*` key names, but
reads as noise until the disassembly ties it to the preferences lookup.

### Cleaning up afterward (macOS)

```bash
rm -f ~/Library/Preferences/Omnissa\ Horizon/config   # debug logging only
```

The policy keys in `com.omnissa.usb` stay; the debug logging is only needed for
diagnosis.

## Notes / caveats

- The exact `viewusb.*` key names were confirmed live from the debug log's
  `DevFltr: Reading config, viewusb.<Key>=` lines, not guessed from
  documentation — they should be accurate for this client build, but key
  names have changed across VMware View → VMware Horizon → Omnissa Horizon
  rebrands over the years. If your `horizon-usbd` log shows different key
  names being read, use those instead.
- The `viewusb.` prefix is Linux-only. On macOS the same filter reads bare
  `Allow*` key names from the `com.omnissa.usb` preferences domain. Expect the
  storage mechanism, not just the file path, to differ per platform — check
  what the client actually links against (`nm -u` on the usbd library) before
  assuming a config file is even consulted.
- This same failure mode (composite audio+HID device invisible in the USB
  menu) will affect **any** similar composite device — other dictation
  microphones, some barcode scanners, some footswitches — not just the
  SpeechMike. The diagnosis steps above generalize; only the specific VID:PID
  differs.
