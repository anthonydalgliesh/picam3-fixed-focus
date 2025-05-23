# OpenHD Pi Camera Module 3 – Disable Autofocus and Lock Manual Focus
![IMPORTANT!!!](./assets/warn.png)

This guide explains how to permanently disable autofocus on the Raspberry Pi Camera Module 3 for use with OpenHD in UAV systems. It includes steps to break libcamera’s autofocus feature, manually control focus using `v4l2-ctl`, and lock focus at boot using a `systemd` service.

## Why Disable Autofocus?

OpenHD does not support autofocus cameras due to vibration-induced motor wear and unstable video. The Pi Camera Module 3 ships with a voice-coil-based autofocus system that resets every boot unless explicitly disabled.

Disabling AF and setting a fixed focus ensures:
- Sharp and stable aerial footage
- No in-flight refocusing
- Longer lens motor lifespan

## Step 1: Disable Autofocus in libcamera

Find the camera configuration:

```bash
sudo find / -name imx708.json
```

Expected path:
```
/usr/share/libcamera/ipa/rpi/vc4/imx708.json
```

Edit it:

```bash
sudo nano /usr/share/libcamera/ipa/rpi/vc4/imx708.json
```

Change this line:

```json
"rpi.af": {
```

To:

```json
"rpi.af.breakme": {
```

This effectively disables the autofocus plugin without breaking the rest of the pipeline.

## Step 2: Confirm Manual Focus Control Works

Reboot the Pi and test focus with:

```bash
v4l2-ctl -d /dev/v4l-subdev1 --set-ctrl=focus_absolute=0
```

You should hear the lens click, and the value should stick:

```bash
v4l2-ctl -d /dev/v4l-subdev1 --get-ctrl=focus_absolute
```

If it doesn’t change back automatically, autofocus is disabled.

## Step 3: Lock Focus at Boot Using systemd

Create the service:

```bash
sudo nano /etc/systemd/system/fix-focus.service
```

Paste:

```ini
[Unit]
Description=Set camera focus to 512 (test), consider 463 since hyperfocal, focus calculation is 450 + (32 * diopters)
After=multi-user.target

[Service]
Type=oneshot
ExecStart=/usr/bin/v4l2-ctl -d /dev/v4l-subdev1 --set-ctrl=focus_absolute=512
RemainAfterExit=true

[Install]
WantedBy=multi-user.target
```

Enable it:

```bash
sudo systemctl daemon-reload
sudo systemctl enable fix-focus.service
```

This ensures focus is restored every time the UAV boots.

## Focus Value Reference

| focus_absolute Value | Approximate Focus Distance |
|----------------------|----------------------------|
| 0                    | Infinity                   |
| 463                  | Long range (infinity-ish)  |
| 512                  | Mid-range test value       |
| 800+                 | Close-up (macro)           |
| 1023                 | Closest possible           |

## Future Ideas: MAVLink Focus Control

Now that autofocus is disabled, you can potentially control `focus_absolute` in flight using:

- A MAVLink-to-SSH relay
- Onboard mission scripts
- Ground station input triggers

This could allow UAVs to dynamically refocus.

## Optional: Physically Locking the Lens (Not Recommended)

While this guide disables autofocus in software and locks the focus at boot, the Pi Camera Module 3's lens can still be damaged due to vibration in flight, especially on multirotors or high-frequency airframes.

It is possible to apply a small amount of adhesive (such as E6000) to physically lock the lens position after setting the correct focus. However:

- This approach **can easily damage the lens assembly**, especially if glue seeps into the actuator mechanism.
- The voice coil in the autofocus module is not designed for permanent immobilization.
- Even a tiny misalignment or movement during gluing could degrade image quality.

**In general, this is not recommended.** If you attempt it, use extreme care and understand the risk of damaging the camera module permanently.

## License

MIT — do what you want, but no warranties. See `LICENSE` for details.

## Credits

Big thanks to my pal Audrey she figured out the entire autofocus disable trick to break autofocus cleanly. Without that, none of this would’ve worked.

- GitHub: [@audreyap](https://github.com/audreyap)
