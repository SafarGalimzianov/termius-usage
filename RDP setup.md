# Abstract
 We’ll (1) install a lightweight desktop and xrdp on Ubuntu, (2) enable audio/clipboard channels (PipeWire), (3) lock xrdp to localhost, (4) use **Termius** to open an SSH tunnel (local port → server 3389), and (5) RDP to `127.0.0.1:<local-port>` from your client with audio + clipboard enabled. This gives you a secure, SSH-tunneled RDP workflow.

---

# Secure RDP to Ubuntu via Termius (SSH Tunnel + xrdp)

## Checklist (high-level flow)

* Pick a desktop (XFCE) and install **xrdp** on the server.
* Enable audio + clipboard redirection (PipeWire module and xrdp channels).
* Bind xrdp to `127.0.0.1` and restart the service.
* In **Termius**, create a host and a **Local** port-forward (e.g., `13389 → 127.0.0.1:3389`).
* From your local PC/phone, RDP to `127.0.0.1:13389` with audio & clipboard turned on.
* Verify session, then maintain & troubleshoot with logs if needed.

---

## Table of Contents

- [Abstract](#abstract)
- [Secure RDP to Ubuntu via Termius (SSH Tunnel + xrdp)](#secure-rdp-to-ubuntu-via-termius-ssh-tunnel--xrdp)
  - [Checklist (high-level flow)](#checklist-high-level-flow)
  - [Table of Contents](#table-of-contents)
  - [Prerequisites](#prerequisites)
  - [Step-by-step Instructions](#step-by-step-instructions)
    - [Step 1: Install a desktop and xrdp on Ubuntu](#step-1-install-a-desktop-and-xrdp-on-ubuntu)
    - [Step 2: Enable audio \& clipboard redirection](#step-2-enable-audio--clipboard-redirection)
    - [Step 3: Lock xrdp to localhost and restart](#step-3-lock-xrdp-to-localhost-and-restart)
    - [Step 4: Create an SSH tunnel in Termius](#step-4-create-an-ssh-tunnel-in-termius)
    - [Step 5: RDP to the tunneled port from your client](#step-5-rdp-to-the-tunneled-port-from-your-client)
    - [Step 6: Verify features and optimize](#step-6-verify-features-and-optimize)
    - [Step 7: Maintain \& update safely](#step-7-maintain--update-safely)
  - [Troubleshooting](#troubleshooting)
  - [Additional Resources](#additional-resources)
  - [Further read](#further-read)

---

## Prerequisites

**Server (Ubuntu 25.x)**

* A sudo user and SSH access (public IP or via jump host).
* Network egress to fetch packages.
* Enough RAM/disk for a lightweight desktop (XFCE recommended). DigitalOcean’s reference walkthrough also uses XFCE and xrdp together. ([DigitalOcean][1])

**Local**

* **Termius** installed (desktop or mobile). Termius supports an easy Port Forwarding wizard. ([termius.com][2])
* An RDP client:

  * **Windows 11**: built-in *Remote Desktop Connection* (`mstsc`). Audio/clipboard redirection is supported in RDP clients. ([Microsoft Learn][3])
  * **Linux desktop**: **Remmina** or **FreeRDP (xfreerdp)**. FreeRDP’s current man page documents `+clipboard`, `/sound`, `/microphone`, and dynamic resolution. ([openSUSE Manpages][4])
  * **Pixel phones (Linux Terminal / Android)**: You can run a normal RDP client app on Android, or use the new Linux Terminal (Debian VM) on Pixel for Linux tooling; Google’s own guidance notes it’s not intended as a full desktop by itself. ([How-To Geek][5])

---

## Step-by-step Instructions

### Step 1: Install a desktop and xrdp on Ubuntu

**Beginner view**

1. SSH to the server.
2. Install a light desktop and xrdp in one go:

```bash
sudo apt update
sudo apt install -y xfce4 xfce4-goodies xrdp
```

3. Set your xrdp session to start XFCE:

```bash
echo "xfce4-session" | tee ~/.xsession
```

4. Restart xrdp:

```bash
sudo systemctl restart xrdp
```

You now have a GUI + RDP server ready. A widely used guide uses the same approach (`xfce4-session` in `~/.xsession`). ([DigitalOcean][1])

**Pro breakdown**

* Modern Ubuntu xrdp integrates with **Xorg** via the `xorgxrdp` modules (usually pulled in automatically when you install `xrdp`). If needed, ensure `xorgxrdp` is present (Ubuntu 25.04 shows it in universe). ([GitHub][6], [Ubuntu Updates][7])
* The `~/.xsession` forces a consistent session manager for xrdp; without it, you can hit black/blank screens. This pattern is consistent across quality references. ([DigitalOcean][1])

**Validation**

* `systemctl status xrdp` should be **active (running)**. If not, fix service errors before moving on. ([DigitalOcean][1])

---

### Step 2: Enable audio & clipboard redirection

**Beginner view**

1. Install the PipeWire audio module for xrdp (if available on Ubuntu 25.x):

```bash
sudo apt install -y libpipewire-0.3-modules-xrdp
```

2. Make sure xrdp’s clipboard and audio features are on (we’ll confirm in Step 3).
   PipeWire’s xrdp module allows audio in RDP sessions on PipeWire systems. ([Launchpad][8])

**Pro breakdown**

* The **PipeWire xrdp module** is the upstream way to get audio from xrdp with PipeWire (superseding the old PulseAudio module). If your distro doesn’t package it, the official project documents building the module. ([GitHub][9])
* On the **client** side, make sure your RDP client is set to *play audio on this computer* and (optionally) *record from this computer* for mic input. Microsoft’s RDP docs & admin guides detail audio capture/playback redirection. ([Microsoft Learn][3])

**Validation**

* After connecting later, you should see a sound device in the remote session; if not, revisit this step.

---

### Step 3: Lock xrdp to localhost and restart

**Beginner view**
Edit `/etc/xrdp/xrdp.ini` and in the `[Globals]` section **bind to loopback**:

```ini
[Globals]
; ...other settings...
address=127.0.0.1
; port defaults to 3389, leave it
```

Ensure clipboard/audio channels are enabled in `[Channels]`:

```ini
[Channels]
; ...other channels...
rdpsnd=true
cliprdr=true
drdynvc=true
```

Then:

```bash
sudo systemctl restart xrdp
```

**Pro breakdown**

* `address=127.0.0.1` is documented in **xrdp.ini(5)** and forces xrdp to listen only on loopback, so it’s **not exposed** to the internet. We’ll reach it through SSH tunneling only. The same man page documents channel toggles (`cliprdr`, `rdpsnd`, `drdynvc`). ([Ubuntu Manpages][10])

**Validation**

* `sudo ss -ltnp | grep 3389` should show `127.0.0.1:3389` listening. If you see `0.0.0.0:3389`, recheck the `address=` line and restart xrdp.

---

### Step 4: Create an SSH tunnel in Termius

**Beginner view (Termius UI)**

* Open **Termius** → *Port Forwarding* → **New Port Forwarding** → choose **Local**.

  * **Local port**: `13389` (or any free port)
  * **Destination**: `127.0.0.1`
  * **Destination port**: `3389`
  * **Host**: select the SSH Host you use for the server
* Save, then **Connect** the port forward. Termius’ docs show the Port Forwarding wizard flow. ([termius.com][2])

**Pro breakdown**

* Local forwards invert exposure: RDP client connects to **localhost:13389**, which SSH encrypts to the server, then hands off to **127.0.0.1:3389** there. This neatly bypasses WAN blocks on 3389 and avoids opening that port publicly. (General notes on local port-forwarding.) ([Wikipedia][11])

**Validation**

* In Termius, you should see the forward in *Connected* state. If a port-in-use error appears, pick a different local port (e.g., `14389`). ([Super User][12])

---

### Step 5: RDP to the tunneled port from your client

**Beginner view**

* **Windows 11**: Press `Win` → type `mstsc`, run:

  * In the *Computer* box, enter `127.0.0.1:13389`.
  * Click **Show Options → Local Resources → Remote audio → Settings…** → set *Play on this computer* and enable mic if needed.
  * Ensure **Clipboard** is ticked (Local Resources → *Clipboard*). Then **Connect**. (Microsoft docs cover audio/AV redirection basics used across RDP.) ([Microsoft Learn][3])
* **Linux desktop**:

  * **Remmina**: Protocol **RDP**, Server `127.0.0.1:13389`. In profile, enable **Clipboard** and **Audio** (output + mic if needed).
  * **FreeRDP** example:

    ```bash
    xfreerdp /v:127.0.0.1:13389 +clipboard /sound /microphone
    ```

    `+clipboard`, `/sound`, `/microphone` are documented. ([openSUSE Manpages][4])
* **Pixel (Android / Linux Terminal)**:

  * Easiest: install **Microsoft Remote Desktop** (Android) and connect to `127.0.0.1:13389`, enabling audio & clipboard in the app.
  * Pixel’s Linux Terminal (Debian VM) is great for SSH/CLI, but Google clarifies it’s **not** intended to provide a full desktop experience on its own—use an Android RDP app for GUI RDP. ([How-To Geek][13])

**Pro breakdown**

* FreeRDP supports dynamic resolution (`+dynamic-resolution`) and granular clipboard directions; see the current man page for advanced flags. ([openSUSE Manpages][4])

**Validation**

* You should reach the **xrdp** login and land in **XFCE**. If the screen is blank or session drops, see **Troubleshooting**.

---

### Step 6: Verify features and optimize

**Beginner view**

* Test **clipboard** both ways (copy local → remote and remote → local).
* Play test audio in the remote session (e.g., a short video).
* If performance is choppy, reduce color depth or resolution in your RDP client. A popular guide notes reducing resolution/bit-depth improves xrdp performance. ([DigitalOcean][1])

**Pro breakdown**

* For FreeRDP, try `/gfx:AVC420:on` and `/network:auto` on high-latency links; enable `+dynamic-resolution` for resizes. (Options are documented in the man page; many admins share performance presets publicly.) ([openSUSE Manpages][4], [Wapnet Blog][14])

**Validation**

* Clipboard and audio should work; otherwise, verify Step 2 and Step 3 settings.

---

### Step 7: Maintain & update safely

**Beginner view**

* Keep packages updated:

```bash
sudo apt update && sudo apt upgrade
```

* Restart xrdp after config changes:

```bash
sudo systemctl restart xrdp
```

**Pro breakdown**

* Logs to check when things misbehave:

  * `/var/log/xrdp.log` and `/var/log/xrdp-sesman.log` (service/session)
  * `~/.local/share/xrdp/xrdp-chansrv.$DISPLAY.log` (channels like clipboard/audio), as per chansrv docs. ([Super User][15], [Debian Manpages][16])
* Wayland vs Xorg: xrdp works with Xorg virtual sessions. If you ever switch desktops and hit black screens, ensure Xorg is used for xrdp sessions (a common pitfall documented in community guides). ([DigitalOcean][1])

**Validation**

* After updates, reconfirm you can RDP via the tunnel and that audio/clipboard still work.

---

## Troubleshooting

* **Black/blank screen after login**

  * Ensure `~/.xsession` contains `xfce4-session`. Restart `xrdp`. Confirm you’re not trying to remote the active physical Wayland session. ([DigitalOcean][1])
* **Clipboard doesn’t work**

  * Confirm `[Channels]` has `cliprdr=true`. Check the per-display **chansrv** log for errors (see paths above). Reconnect to spawn a fresh chansrv. ([Ubuntu Manpages][10], [Debian Manpages][16])
* **No sound devices**

  * Verify `libpipewire-0.3-modules-xrdp` is installed, and the client has audio redirection enabled (Windows: Local Resources → Remote audio). Some distros require logging out and back in to load modules. ([Launchpad][8], [Microsoft Learn][17])
* **Tunnel connects but RDP fails**

  * Confirm xrdp is bound to `127.0.0.1:3389` and running. Double-check the Termius forward is **Local 13389 → 127.0.0.1:3389** and connected. Termius’ wizard simplifies this flow. ([termius.com][2])
* **Local port already in use**

  * Choose another local port (e.g., `14389`). This is a typical fix if something is already listening on 3389 locally. ([Super User][12])
* **Session ends immediately**

  * Check `/var/log/xrdp-sesman.log` for session startup errors. Make sure the user’s home dir perms aren’t too restrictive (e.g., `chmod 755 ~` can help in some cases) and that `xfce4-session` is installed. (See DO guide’s notes.) ([DigitalOcean][1])

---

## Additional Resources

* **xrdp manual (xrdp.ini)** – binding to `address=`, channels (`cliprdr`, `rdpsnd`, `drdynvc`). ([Ubuntu Manpages][10])
* **FreeRDP (xfreerdp) man page** – `+clipboard`, `/sound`, `/microphone`, dynamic resolution. ([openSUSE Manpages][4])
* **DigitalOcean: Enable RDP (xrdp) on Ubuntu** – practical walkthrough using XFCE and `.xsession`. ([DigitalOcean][1])
* **Termius: Port Forwarding** – wizarded setup of local forwards. ([termius.com][2])
* **PipeWire module for xrdp** – Ubuntu package & upstream project. ([Launchpad][8], [GitHub][9])
* **Pixel Linux Terminal (How-To Geek)** – how to enable it & its intended scope (not a full desktop). ([How-To Geek][5])

## Further read
1. [Debian Manpages — xrdp-chansrv(8) — xrdp — Debian unstable](https://manpages.debian.org/unstable/xrdp/xrdp-chansrv.8.en.html)
2. [DigitalOcean — Enable Remote Desktop Protocol Using xrdp on Ubuntu 22.04](https://www.digitalocean.com/community/tutorials/how-to-enable-remote-desktop-protocol-using-xrdp-on-ubuntu-22-04)
3. [GitHub — neutrinolabs/pipewire-module-xrdp](https://github.com/neutrinolabs/pipewire-module-xrdp)
4. [GitHub — neutrinolabs/xrdp: xrdp: an open source RDP server](https://github.com/neutrinolabs/xrdp)
5. [How-To Geek — Google Explains Why It Added a Linux VM to Pixel Phones](https://www.howtogeek.com/linux-terminal-on-android-can-run-desktop-guis/)
6. [How-To Geek — How to Use Your Pixel's Hidden Linux Terminal (and Should You?)](https://www.howtogeek.com/how-to-use-pixel-hidden-linux-terminal/)
7. [Launchpad — libpipewire-0.3-modules-xrdp (amd64, Ubuntu)](https://launchpad.net/ubuntu/oracular/amd64/libpipewire-0.3-modules-xrdp)
8. [Microsoft Learn — Configure audio and video redirection over the Remote ...](https://learn.microsoft.com/en-us/azure/virtual-desktop/redirection-configure-audio-video)
9. [Microsoft Q&A — Remote Desktop Connection Stops Audio on ...](https://learn.microsoft.com/en-us/answers/questions/4049359/remote-desktop-connection-stops-audio-on-remote-co)
10. [openSUSE Manpages — xfreerdp(1) — freerdp](https://manpages.opensuse.org/Tumbleweed/freerdp/xfreerdp.1.en.html)
11. [Super User — Windows 10 SSH "Cannot listen to port 3389"](https://superuser.com/questions/1303186/windows-10-ssh-cannot-listen-to-port-3389)
12. [Super User — XRDP rejecting login - remote desktop](https://superuser.com/questions/1264096/xrdp-rejecting-login)
13. [Termius Docs — Port Forwarding](https://termius.com/documentation/port-forwarding)
14. [Ubuntu Manpages — xrdp.ini - Configuration file for xrdp(8) (focal)](https://manpages.ubuntu.com/manpages/focal/man5/xrdp.ini.5.html)
15. [Ubuntu Manpages — xrdp.ini - Configuration file for xrdp(8) (noble)](https://manpages.ubuntu.com/manpages/noble/en/man5/xrdp.ini.5.html)
16. [Ubuntu Packages — xorgxrdp package](https://packages.ubuntu.com/search?keywords=xorgxrdp)
17. [UbuntuUpdates — Package "xorgxrdp" (plucky 25.04)](https://www.ubuntuupdates.org/xorgxrdp)
18. [Wapnet Blog — Optimizing RDP Performance on Linux: My Best Settings with xfreerdp](https://blog.wapnet.nl/2024/07/optimizing-rdp-performance-on-linux-my-best-settings-with-xfreerdp/)
19. [Wikipedia — Port forwarding](https://en.wikipedia.org/wiki/Port_forwarding)

[1]: https://www.digitalocean.com/community/tutorials/how-to-enable-remote-desktop-protocol-using-xrdp-on-ubuntu-22-04 "Enable Remote Desktop Protocol Using xrdp on Ubuntu 22.04 | DigitalOcean"
[2]: https://termius.com/documentation/port-forwarding "Port Forwarding"
[3]: https://learn.microsoft.com/en-us/azure/virtual-desktop/redirection-configure-audio-video "Configure audio and video redirection over the Remote ..."
[4]: https://manpages.opensuse.org/Tumbleweed/freerdp/xfreerdp.1.en.html "xfreerdp(1) — freerdp"
[5]: https://www.howtogeek.com/how-to-use-pixel-hidden-linux-terminal/ "How to Use Your Pixel's Hidden Linux Terminal (and Should You?)"
[6]: https://github.com/neutrinolabs/xrdp "neutrinolabs/xrdp: xrdp: an open source RDP server"
[7]: https://www.ubuntuupdates.org/xorgxrdp "Package \"xorgxrdp\" (plucky 25.04)"
[8]: https://launchpad.net/ubuntu/oracular/amd64/libpipewire-0.3-modules-xrdp "libpipewire-0.3-modules-xrdp : amd64 - Ubuntu - Launchpad"
[9]: https://github.com/neutrinolabs/pipewire-module-xrdp "neutrinolabs/pipewire-module-xrdp"
[10]: https://manpages.ubuntu.com/manpages/focal/man5/xrdp.ini.5.html "xrdp.ini - Configuration file for xrdp(8)"
[11]: https://en.wikipedia.org/wiki/Port_forwarding "Port forwarding"
[12]: https://superuser.com/questions/1303186/windows-10-ssh-cannot-listen-to-port-3389 "Windows 10 SSH \"Cannot listen to port 3389\""
[13]: https://www.howtogeek.com/linux-terminal-on-android-can-run-desktop-guis/ "Google Explains Why It Added a Linux VM to Pixel Phones"
[14]: https://blog.wapnet.nl/2024/07/optimizing-rdp-performance-on-linux-my-best-settings-with-xfreerdp/ "Optimizing RDP Performance on Linux: My Best Settings with ..."
[15]: https://superuser.com/questions/1264096/xrdp-rejecting-login "XRDP rejecting login - remote desktop"
[16]: https://manpages.debian.org/unstable/xrdp/xrdp-chansrv.8.en.html "xrdp-chansrv(8) — xrdp — Debian unstable"
[17]: https://learn.microsoft.com/en-us/answers/questions/4049359/remote-desktop-connection-stops-audio-on-remote-co "Remote Desktop Connection Stops Audio on ..."
[18]: https://packages.ubuntu.com/search?keywords=xorgxrdp "xorgxrdp package — Ubuntu Packages"
[19]: https://manpages.ubuntu.com/manpages/noble/en/man5/xrdp.ini.5.html "xrdp.ini - Configuration file for xrdp(8)"
