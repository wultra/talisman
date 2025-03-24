# Talisman Installation on Linux

## Support Matrix

| OS / Browser      | Firefox               | Chromium              | Vivaldi               |
|-------------------|-----------------------|-----------------------|-----------------------|
| Ubuntu 24.04 LTS  | Failed. Snap sandbox. | Failed. Snap sandbox. | Failed. Snap sandbox. |
| Debian 12         | OK (Note1)            | OK                    | OK                    |
| Fedora 40         | OK (Note1)            | OK                    | OK                    |
| Kali Linux 2024.2 | OK (Note1)            | OK                    | OK                    |

## Ubuntu Compatibility

Ubuntu Linux ships the browser applications as a “snap” package by default and therefore, the applications themselves are sandboxed and equiped with limited access to the hardware. To make the Talisman usage possible for the Ubuntu users, administrators have to implement a workaround by updating `udev` allowlist.

Add an appropriate hardware identifier to per-application UDEV rules stored at `/etc/udev/rules.d`. For example, the following lines should be added to firefox rules:

```
SUBSYSTEM=="hidraw", KERNEL=="hidraw*", ATTRS{idVendor}=="0483", ATTRS{idProduct}=="a4b3", TAG+="snap_firefox_firefox"
SUBSYSTEM=="hidraw", KERNEL=="hidraw*", ATTRS{idVendor}=="0483", ATTRS{idProduct}=="a4b3", TAG+="snap_firefox_geckodriver"
```

We maintain a simple script that updates the rules for some well known browsers at once. You have to run that script as superuser after the browser application is updated by the snap package manager.

## Firefox Compatibility

**Note 1:** Firefox has a bug where the `userSelected` flag renders the entire response invalid when returned in the response to the `authenticatorGetNextAssertion` command. The issue specifically affects Talisman devices with firmware version 0.6.0 and higher. It does not impact older devices or scenarios where non-discoverable credentials are used.

Issue: [#343: Unexpected keys in CTAP2 responses should be ignored](https://github.com/mozilla/authenticator-rs/issues/343)

## Chromium Compatibility

In general, Chromium browser works very well with Talisman. We don’t see any compatibility issues so far.

## Vivaldi Compatibility

In general, Vivaldi browser works very well with Talisman. We don’t see any compatibility issues so far.
