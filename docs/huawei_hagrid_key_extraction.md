# Huawei Hagrid device keys (CAK / C1 / C2)

Huawei "Hagrid" scales — e.g. Huawei Smart Scale 3 and Scale 3 Pro —
authenticate over BLE with three per-device 16-byte keys:

- **CAK** – device authentication key (used for the HMAC challenge/response)
- **C1 / C2** – combined with the Bluetooth address to derive the
  encryption root key

openScale needs each key as **32 hexadecimal characters**, entered in the
device settings panel ("Huawei Hagrid WSP secrets").

The keys are provisioned to the scale once. They do not change when you
unpair or re-pair it, and they are not tied to a specific phone — the
auth exchange runs between the app and the scale itself. One extraction
is enough.

> Only extract keys from a device you own. The values are device
> credentials — treat them like a password and don't post them publicly.

## Requirements

- A rooted Android phone
- Huawei Health installed, with the scale added and synced at least once
- `frida-inject` (or a frida server) matching your device

## Where the keys live

Huawei Health keeps them in its white-box store, reachable through
`health.compact.a.WhiteBoxManager` (which wraps
`com.huawei.whitebox.NdkJniUtils.getStorageInfo`). Three storage slots
each hold a complete 32-char hex string:

| Slot  | Key |
|-------|-----|
| 33    | CAK |
| 1033  | C1  |
| 2033  | C2  |

These are the exact slots the app's own auth code (`Ldja` / `Ldjb` in the
obfuscated app) loads the values from.

## Extraction

1. Pair the scale in Huawei Health and run one sync so the app is fully
   started.
2. Push `frida-inject` to the device and attach to the **main process**
   `com.huawei.health` (not the `:DaemonService` or `:PhoneService`
   processes).
3. Run:

   ```js
   Java.perform(function () {
     var wbm = Java.use('health.compact.a.WhiteBoxManager');
     var mgr = wbm.a();            // singleton accessor
     send('CAK = ' + mgr.a(1, 33));
     send('C1  = ' + mgr.a(1, 1033));
     send('C2  = ' + mgr.a(1, 2033));
   });
   ```

4. Each call returns a 32-character hex string. Enter them in openScale's
   Hagrid settings — CAK, C1, C2 in that order, with no spaces or `0x`
   prefix.

## Tips

- Inject only after the app is fully running. Injecting during startup or
  instrumenting broadly can destabilise Huawei Health mid-sync; a
  minimal, static-call script like the one above is safest.
- The obfuscated wrapper class name (`Lcsc`/`Ldja`/`Ldjb` etc.) varies
  between app versions, but `health.compact.a.WhiteBoxManager` is a real
  (unobfuscated) class name and has been stable across releases.
- Older app versions stored the CAK split across fragment slots
  ({23, 1023, 2023}) and additionally base64-decoded/decrypted it — that
  path was for an older authentication variant. Slots 33/1033/2033 are
  the direct read used by current releases.
