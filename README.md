# yocto_bbb — Yocto for BeagleBone Black

Custom Yocto Project setup for the **BeagleBone Black (AM335x)**.

- Yocto Project **6.0 "Wrynose"** (LTS, supported until May 2030)
- Distro: **Poky**, Machine: **beaglebone-yocto**
- Custom layer **meta-shijo**:
  - Kernel config fragment enabling **USB gadget networking** (`g_ether`, configfs ECM/RNDIS)

## Repository contents

| Path | Purpose |
|---|---|
| `meta-shijo/` | Custom Yocto layer (recipes, bbappends, kernel fragments) |
| `shijo-bbb.conf.json` | `bitbake-setup` configuration: layers, pinned commits, machine, distro |
| `.gitignore` | Keeps only the files above in this repository |

## 1. Prepare the build PC (once)

Requirements: native Linux (Ubuntu 22.04 or newer recommended), **≥ 140 GB free disk**, **≥ 16 GB RAM** (32 GB recommended).

```bash
sudo apt-get install build-essential chrpath cpio debianutils diffstat file gawk gcc git \
  iputils-ping libacl1 libcrypt-dev locales python3 python3-git python3-jinja2 \
  python3-pexpect python3-pip python3-subunit python3-venv python3-websockets socat \
  texinfo unzip wget xz-utils zstd bmaptool picocom
sudo dpkg-reconfigure locales     # make sure en_US.UTF-8 is enabled
```

**Ubuntu 24.04 and newer only** — allow BitBake to use user namespaces:

```bash
sudo tee /etc/apparmor.d/bitbake > /dev/null << 'EOF'
abi <abi/4.0>,
include <tunables/global>
profile bitbake /**/bitbake/bin/bitbake flags=(unconfined) {
  userns,
}
EOF
sudo apparmor_parser -r /etc/apparmor.d/bitbake
```

### SSH key for GitHub (required)

The setup clones this repository over SSH, so you need an SSH key registered with GitHub:

```bash
ls ~/.ssh/id_ed25519.pub 2>/dev/null || ssh-keygen -t ed25519 -C "you@example.com"
cat ~/.ssh/id_ed25519.pub
```

Copy the printed line into **GitHub → Settings → SSH and GPG keys → New SSH key**, then test:

```bash
ssh -T git@github.com      # answer "yes"; expect "Hi <user>! You've successfully authenticated"
```

## 2. Create the build setup (once)

Run this from the folder where you want the builds to live (e.g. `~/yocto`):

```bash
mkdir -p ~/yocto && cd ~/yocto
python3 -m venv ./bitbake-setup-venv
. ./bitbake-setup-venv/bin/activate
pip install bitbake-setup
bitbake-setup init https://raw.githubusercontent.com/shijomts4-dev/yocto_bbb/main/shijo-bbb.conf.json
deactivate
```

Press Enter to accept the setup name `shijo-bbb-wrynose`, then `y` to continue.
This clones all layers at the pinned commits and configures machine, distro and fragments — no menus.

## 3. Build

In every new terminal, first source the build environment:

```bash
. ~/yocto/bitbake-builds/shijo-bbb-wrynose/build/init-build-env
bitbake core-image-weston
```

The first build can take several hours (shared sstate mirror is enabled to speed it up).
Output: `tmp/deploy/images/beaglebone-yocto/core-image-weston-beaglebone-yocto.rootfs.wic.zst`

## 4. Flash the SD card

```bash
cd tmp/deploy/images/beaglebone-yocto
lsblk                                   # identify the SD card, e.g. /dev/sdX — double-check!
sudo umount /dev/sdX?* 2>/dev/null
sudo bmaptool copy core-image-weston-beaglebone-yocto.rootfs.wic.zst /dev/sdX
sync
```

## 5. Boot

1. Connect a **3.3 V** USB-UART to header **J1** (pin 1 GND, pin 4 → adapter TX, pin 5 → adapter RX).
2. `sudo picocom -b 115200 /dev/ttyUSB0` (exit: Ctrl+A, Ctrl+X)
3. Insert the SD card, **hold S2**, apply power.
4. Log in as `root` (no password — development image only).

## 6. USB networking (optional)

Connect the BBB mini-USB port to the PC. On the BBB:

```bash
modprobe g_ether
ip addr add 192.168.7.2/24 dev usb0
ip link set usb0 up
```

On the PC (interface name from `ip link`, usually `enx...`):

```bash
sudo ip addr add 192.168.7.1/24 dev <iface>
sudo ip link set <iface> up
ssh root@192.168.7.2
```

## 7. Getting updates

When this repository changes:

```bash
cd ~/yocto
. ./bitbake-setup-venv/bin/activate
bitbake-setup update
deactivate
```

Then rebuild as in step 3.

## For maintainers

- Edit layer content in `meta-shijo/`, build and test, then commit and push.
- To move to newer Yocto fixes, update the `rev` commit IDs in `shijo-bbb.conf.json`,
  test a fresh `bitbake-setup init`, then commit and push.
