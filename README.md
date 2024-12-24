# BTCTools Snapcraft Project Files

## Operating System

Please build in a Linux distribution that is compatible with [snap](https://snapcraft.io/).

Known compatible distributions:
* Ubuntu 18.04
* Ubuntu 20.04 
* Debian 10
* UOS 20
* Deepin v20

### Install snapd

```bash
sudo apt install snapd
sudo systemctl start snapd
```

#### Starting snapd in WSL2 or other systems not using systemd

snapd depends on systemd. If your system doesn't start with systemd, you need to install systemd container: https://github.com/arkane-systems/genie

After installation, enter the systemd container with:

```bash
sudo genie -i
genie -c /bin/bash -l
sudo systemctl start snap
# Run subsequent commands
```

If `genie -i` fails, rename `/etc/hosts` and create a new empty file:

```bash
sudo mv /etc/hosts /etc/hosts.old
sudo touch /etc/hosts
```

### Install snapcraft

```bash
sudo snap install core
sudo snap install core18
sudo snap install snapcraft --classic
```

#### Troubleshooting

If `snapd` is broken, fix it with:

```bash
sudo apt purge snapd gnome-software-plugin-snap
sudo apt install snapd gnome-software-plugin-snap
sudo systemctl restart snapd
```

If you see this error (note: the error message can be misleading):

```bash
- Mount snap "snapcraft" (4282) (snap "snapcraft" assumes unsupported features: snapd2.39 (try to update snapd and refresh the core snap))
```

Installing or updating `core` can fix it:

```bash
sudo snap install core
sudo snap refresh core
```

That's why installing `core` is already included in the installation steps above.

If it still doesn't work, you'll need to find a way to upgrade the `snapd` package yourself.

#### Using Proxy

If `snap` is slow, you can set up a proxy:

```bash
sudo snap set system proxy.http=socks5://127.0.0.1:17070
sudo snap set system proxy.https=socks5://127.0.0.1:17070

sudo systemctl restart snapd
```

### Add /snap/bin to PATH

To ensure subsequent commands work properly, add /snap/bin to PATH environment variable for both current user and root user.

```bash
vim ~/.profile
```

```bash
export PATH=$PATH:/snap/bin
```

Log out and log back in after adding.

### Install `LXD`

Building requires `LXD` to run `Ubuntu 18.04` container. Install `LXD` as follows:

```bash
sudo snap install lxd
sudo /snap/bin/lxd init
sudo usermod -aG lxd $USER
```

Note: If your network doesn't support `IPv6`, make sure to set `IPv6 address` to `none`, otherwise the container may not be able to connect to network properly.

You need to log out and log back in for the `usermod` command to take effect. Otherwise you'll encounter this error during build:

```bash
An error occurred when trying to communicate with the 'LXD' provider: cannot connect to the LXD socket ('/var/snap/lxd/common/lxd/unix.socket')..
```

### Building BTCTools

```bash
git clone https://github.com/btccom/btctools-snap.git
cd btctools-snap
git submodule update --init --recursive

# Set mirror
export SNAPCRAFT_BUILD_ENVIRONMENT_PRIMARY_MIRROR=http://mirrors.aliyun.com/ubuntu

# Build
snapcraft --use-lxd
```

#### Using Proxy

```bash
# LXC core proxy settings (optional, effect unclear, SOCKS proxy support unknown)

lxc config set core.proxy_http=http://192.168.43.224:18080 
lxc config set core.proxy_https=https://192.168.43.224:18080

# Proxy for snapcraft API and package sources
# Seems required, otherwise snapcraft API will error with "There seems to be a network error"
# Only HTTP proxy supported, SOCKS proxy needs conversion via tools like privoxy
# Note: Use local LAN IP (192.168.43.224) as LXD containers have separate virtual NICs, 127.0.0.1 won't work
# Proxy program needs to listen on 0.0.0.0

snapcraft --use-lxd --http-proxy=http://192.168.43.224:18080 --https-proxy=https://192.168.43.224:18080


# Set proxy for snapd in container
# Run after snapcraft --use-lxd --http-proxy=... --https-proxy=...
# Execute in another terminal when "Waiting for network to be ready..." appears

lxc exec snapcraft-btctools sudo snap set system proxy.http=socks5://192.168.43.224:17070
lxc exec snapcraft-btctools sudo snap set system proxy.https=socks5://192.168.43.224:17070


# Set git proxy
# For github cloning process if too slow
# Run before "Cloning into '/root/parts/desktop-qt5/src'..." appears

lxc exec snapcraft-btctools -- git config --global http.proxy 'socks5://192.168.43.224:17070'
lxc exec snapcraft-btctools -- git config --global https.proxy 'socks5://192.168.43.224:17070'
```

If errors persist without clear cause, check https://status.snapcraft.io/ as snapcraft services may be down.

![image](https://user-images.githubusercontent.com/26923506/115429740-484ef580-a236-11eb-881d-3487847c551e.png)

#### View LXD Logs

If LXD container startup hangs, check logs:

```bash
sudo journalctl -u snap.lxd.activate.service -f
sudo journalctl -u snap.lxd.daemon.service -f
```

#### Reinstall LXD Manager

If issues occur, try removing and reinstalling lxd:

```bash
lxc stop snapcraft-btctools
lxc delete snapcraft-btctools  
sudo snap remove --purge lxd
sudo snap install lxd
sudo /snap/bin/lxd init
sudo usermod -aG lxd $USER
```

For btrfs users encountering this error:

```bash
error: cannot perform the following tasks:
- Remove data for snap "lxd" (15457) (remove /var/snap/lxd/common/lxd/storage-pools/default/images/e3ee3fe3007ea911adea6fcd47dba824459c921b5d1041ac1f1ce88858e7b579/metadata.yaml: read-only file system)
```

Use these commands:

```bash
sudo btrfs sub del /var/snap/lxd/common/lxd/storage-pools/default/images/e3ee3fe3007ea911adea6fcd47dba824459c921b5d1041ac1f1ce88858e7b579
sudo snap remove --purge lxd
```

#### Execute Custom Commands in Container

Open interactive session with:

```bash
lxc exec snapcraft-btctools bash
```

Note: Container may be auto-deleted by snapcraft, all changes could be lost.

#### Cleanup

If strange build errors occur, like missing files, try cleaning:

```bash
# Delete entire container to start fresh
snapcraft clean --use-lxd

# Clean specific build stages
snapcraft clean --use-lxd libbtctools
snapcraft clean --use-lxd btctools-gui
```

### Build Locally Without LXD Container

If you have unresolvable LXD issues, consider installing Ubuntu 18.04 locally and build with:

```bash
sudo /snap/bin/snapcraft --destructive-mode
```

Note: [snap/snapcraft.yaml](snapcraft.yaml) specifies `base: core18`, meaning Ubuntu 18.04 build environment.
Building on other OS versions locally may produce unusable packages!

#### Local Build Cleanup

Delete directories specified in `.gitignore`:

```bash
sudo rm -rf parts/ prime/ stage/ *.snap *.snap-build.* *.xdelta*
```

### Building armhf packages on arm64

```bash
# Start container and enter container shell 
lxc launch ubuntu:18.04/armhf core18-armhf
lxc exec core18-armhf bash
```

Execute in container shell:

```bash
# Update packages to latest version
sed -i 's/[a-z0-9.-]*\.[cno][oer][mtg]/mirrors.aliyun.com/g' /etc/apt/sources.list
apt update && apt upgrade -y

# Set proxy (optional)
snap set system proxy.http=socks5://192.168.43.224:17070
snap set system proxy.https=socks5://192.168.43.224:17070
git config --global http.proxy socks5://192.168.43.224:17070
git config --global https.proxy socks5://192.168.43.224:17070

# Install snapcraft
snap install snapcraft --classic

# Build project
git clone --recursive https://github.com/btccom/btctools-snap.git
cd btctools-snap
snapcraft --destructive-mode
```

### Testing armhf packages on arm64

```bash
# Enter container shell
# This container needs to have BTCTools compiled, otherwise dependencies may be missing
lxc exec core18-armhf bash
```

Execute in container shell:

```bash
# Install built package
snap install --dangerous ~/btctools-snap/*.snap

# Install Chinese fonts
apt install fonts-noto-cjk

# Start SSH service
systemctl start ssh
touch ~/.Xauthority

# Check container IP
ifconfig
```

Execute on host machine:

```bash
xhost +
ssh-keygen
lxc file push ~/.ssh/id_rsa.pub core18-armhf/root/.ssh/authorized_keys
lxc exec core18-armhf chown root:root /root/.ssh/authorized_keys

# Assuming container IP is 10.125.163.245
ssh -X root@10.125.163.245
```

Execute in SSH session:

```bash
btctools
```

You will get this error:
> qt.qpa.screen: QXcbConnection: Could not connect to display localhost:10.0
> Could not connect to any X display.

So we can only try launching BTCTools binary directly:

```bash
/snap/btctools/current/usr/bin/btctools-gui
/snap/btctools/current/usr/bin/btctools-gui --lang=en
/snap/btctools/current/usr/bin/btctools-gui --lang=zh_CN
```

Since dependencies were installed when compiling BTCTools, it should be able to launch.

### Not recommended to use `multipass`

If running `snapcraft` command without `--use-lxd` and `--destructive-mode`, it will use the default `multipass` VM for building.

`multipass` uses `qemu-kvm` for full virtualization, which is slightly slower than `LXD`, and takes longer to start and stop.

Additionally, there are major compatibility issues between `multipass 1.3.0` and `snapcraft 4.0.4`, making normal builds impossible.

It is strongly discouraged to use `snapcraft`'s `multipass` integration. If you really want to use `multipass`, it's recommended to directly launch `Ubuntu 18.04` in a VM, then build using `sudo /snap/bin/snapcraft --destructive-mode`, rather than using the local `snapcraft` command.

## Installing built snap package

```bash
sudo snap install --dangerous ./*.snap

# Run
btctools

# If command not found
/snap/bin/btctools
```

## Publishing

If login fails, delete the `~/.config/snapcraft` folder.

```bash
snapcraft login

# Choose one of the channels
snapcraft upload --release edge *.snap
snapcraft upload --release beta *.snap
snapcraft upload --release candidate *.snap
snapcraft upload --release stable *.snap
```

If you encounter errors, make sure only one file is selected in `*.snap`, or provide the full filename, like `snapcraft push --release stable btctools_1.2.7_amd64.snap`.

Note: It's known that `gnome-software-plugin-snap` (Ubuntu's built-in app store) cannot properly install software published only in the `edge` channel. It's recommended to submit to the `stable` channel when delivering to users.