# BTCTools的snapcraft工程文件

## 操作系统

请在兼容[snap](https://snapcraft.io/)的Linux发行版中构建。

已知可用的发行版：
* Ubuntu 18.04
* Ubuntu 20.04
* Debian 10
* UOS 20
* Deepin v20

### 安装snapd
```
sudo apt install snapd
sudo systemctl start snapd
```

#### 在WSL2或其他不使用systemd的系统中启动snapd

snapd依赖systemd，如果系统不通过systemd启动，则需要安装systemd容器：https://github.com/arkane-systems/genie

安装后，通过以下命令进入systemd容器：
```
sudo genie -i
genie -c /bin/bash -l
sudo systemctl start snap

# 运行后续命令
```

如果`genie -i`出错，请重命名`/etc/hosts`并新建一个空白文件：
```
sudo mv /etc/hosts /etc/hosts.old
sudo touch /etc/hosts
```

### 安装snapcraft

```
sudo snap install core
sudo snap install core18
sudo snap install snapcraft --classic
```

#### 出错

如果`snapd`损坏，用如下命令修复：
```
sudo apt purge snapd gnome-software-plugin-snap
sudo apt install snapd gnome-software-plugin-snap
sudo systemctl restart snapd
```

如果出现这种错误（注意：错误提示具有误导性）：
```
- Mount snap "snapcraft" (4282) (snap "snapcraft" assumes unsupported features: snapd2.39 (try to update snapd and refresh the core snap))
```
安装或者更新`core`可以修复：
```
sudo snap install core
sudo snap refresh core
```
所以安装`core`的命令已经写到上面的安装步骤中了。

如果还不行，你就必须自己想办法升级`snapd`软件包了。

#### 使用代理

如果`snap`网速很慢，可以这样设置代理：
```
sudo snap set system proxy.http=socks5://127.0.0.1:17070
sudo snap set system proxy.https=socks5://127.0.0.1:17070

sudo systemctl restart snapd
```

### 将 /snap/bin 加入 PATH

为了保证后续命令不出错，应该把 /snap/bin 加入 PATH 环境变量，当前用户和 root 用户都需要加入。

```
vim ~/.profile
```

```
# 添加
export PATH=$PATH:/snap/bin
```

加入后注销当前会话再重新登陆。

### 安装`LXD`

构建需要使用`LXD`运行`Ubuntu 18.04`容器，`LXD`安装方法如下：

```
sudo snap install lxd
sudo /snap/bin/lxd init
sudo usermod -aG lxd $USER
```

注意：如果你的网络不支持`IPv6`，请务必将`IPv6 address`设为`none`，否则容器可能会无法正常联网。

为了使`usermod`命令生效，需要注销当前会话再重新登陆。否则构建时会遇到如下错误：

```
An error occurred when trying to communicate with the 'LXD' provider: cannot connect to the LXD socket ('/var/snap/lxd/common/lxd/unix.socket')..
```

### 构建BTCTools

```
git clone https://github.com/btccom/btctools-snap.git
cd btctools-snap
git submodule update --init --recursive

# 设置软件源
export SNAPCRAFT_BUILD_ENVIRONMENT_PRIMARY_MIRROR=http://mirrors.aliyun.com/ubuntu

# 构建
snapcraft --use-lxd
```

#### 使用代理

```
# LXC核心设置代理（可选，作用不明确，不清楚是否支持socks代理）

lxc config set core.proxy_http=http://192.168.43.224:18080
lxc config set core.proxy_https=https://192.168.43.224:18080


# 代理`snapcraft API`和软件源
# 看起来是必选，否则`snapcraft API`会报错`There seems to be a network error`。
# 只支持HTTP代理，socks代理需要通过privoxy这样的软件转换为HTTP代理。
# 注意这里的`192.168.43.224`是本机的局域网IP，因为LXD容器具有独立的虚拟网卡，`127.0.0.1`不能用。
# 代理程序需要设置为监听`0.0.0.0`。

snapcraft --use-lxd --http-proxy=http://192.168.43.224:18080 --https-proxy=https://192.168.43.224:18080


# 给容器内的snapd设置代理
# 需要先运行`snapcraft --use-lxd --http-proxy=... --https-proxy=...`，
# 然后等每次出现`Waiting for network to be ready...`时，在另一个终端运行如下命令

lxc exec snapcraft-btctools sudo snap set system proxy.http=socks5://192.168.43.224:17070
lxc exec snapcraft-btctools sudo snap set system proxy.https=socks5://192.168.43.224:17070


# 给git设置代理
# 构建有个从github克隆代码的过程，如果嫌慢也可以代理
# 在`Cloning into '/root/parts/desktop-qt5/src'...`出现前运行

lxc exec snapcraft-btctools -- git config --global http.proxy 'socks5://192.168.43.224:17070'
lxc exec snapcraft-btctools -- git config --global https.proxy 'socks5://192.168.43.224:17070'
```

如果始终出错，找不到原因，记得去 https://status.snapcraft.io/ 看看，有可能是snapcraft的服务挂了。

![image](https://user-images.githubusercontent.com/26923506/115429740-484ef580-a236-11eb-881d-3487847c551e.png)

#### 查看`LXD`日志

如果启动 LXD 容器陷入僵局，一直没有进展，可以查看日志：

```
sudo journalctl -u snap.lxd.activate.service -f
sudo journalctl -u snap.lxd.daemon.service -f
```

#### 重装`LXD`管理程序

如果有问题，可以尝试卸载重装`lxd`：

```
lxc stop snapcraft-btctools
lxc delete snapcraft-btctools
sudo snap remove --purge lxd
sudo snap install lxd
sudo /snap/bin/lxd init
sudo usermod -aG lxd $USER
```

如果你使用`btrfs`，遇到如下错误：
```
error: cannot perform the following tasks:
- Remove data for snap "lxd" (15457) (remove /var/snap/lxd/common/lxd/storage-pools/default/images/e3ee3fe3007ea911adea6fcd47dba824459c921b5d1041ac1f1ce88858e7b579/metadata.yaml: read-only file system)
```

使用如下命令可以解决：
```
sudo btrfs sub del /var/snap/lxd/common/lxd/storage-pools/default/images/e3ee3fe3007ea911adea6fcd47dba824459c921b5d1041ac1f1ce88858e7b579
sudo snap remove --purge lxd
```

#### 在容器内执行自定义命令

你还可以通过以下命令打开交互式会话进行其他操作：
```
lxc exec snapcraft-btctools bash
```

注意容器可能会被`snapcraft`自动删除，所有更改都可能会丢失。

#### 清理

如果构建出现了奇怪的错误，比如莫名其妙的文件找不到，可以考虑用如下命令清理：

```
# 删除整个容器，完全重新开始
snapcraft clean --use-lxd

# 清除特定构建阶段
snapcraft clean --use-lxd libbtctools
snapcraft clean --use-lxd btctools-gui
```


### 不使用`LXD`容器，直接在本机构建

如果你在`LXD`容器上遇到了无法解决的问题，你也可以考虑本机安装`Ubuntu 18.04`，然后运行如下命令在本机构建：

```
sudo /snap/bin/snapcraft --destructive-mode
```

注意：[snap/snapcraft.yaml](snapcraft.yaml)中指定了`base: core18`，表示构建操作系统为`Ubuntu 18.04`。
在其他操作系统中本机构建可能会得到无法正常使用的安装包！

#### 本机构建的清理

删除`.gitignore`里面指定的目录即可。

```
sudo rm -rf parts/ prime/ stage/ *.snap *.snap-build.* *.xdelta*
```


### 在arm64上构建armhf软件包

```
# 启动容器并进入容器shell
lxc launch ubuntu:18.04/armhf core18-armhf
lxc exec core18-armhf bash
```

在容器shell内执行：
```
# 更新软件包到最新版本
sed -i 's/[a-z0-9.-]*\.[cno][oer][mtg]/mirrors.aliyun.com/g' /etc/apt/sources.list
apt update && apt upgrade -y

# 设置代理（可选）
snap set system proxy.http=socks5://192.168.43.224:17070
snap set system proxy.https=socks5://192.168.43.224:17070
git config --global http.proxy socks5://192.168.43.224:17070
git config --global https.proxy socks5://192.168.43.224:17070

# 安装snapcraft
snap install snapcraft --classic

# 编译项目
git clone --recursive https://github.com/btccom/btctools-snap.git
cd btctools-snap
snapcraft --destructive-mode
```


### 在arm64上测试armhf软件包

```
# 进入容器shell
# 该容器需要编译过BTCTools，否则可能缺少运行BTCTools所需的依赖包
lxc exec core18-armhf bash
```

在容器shell内执行：
```
# 安装编译好的软件包
snap install --dangerous ~/btctools-snap/*.snap

# 安装中文字体
apt install fonts-noto-cjk

# 启动SSH服务
systemctl start ssh
touch ~/.Xauthority

# 查看容器IP
ifconfig
```

回到宿主机执行：
```
xhost +
ssh-keygen
lxc file push ~/.ssh/id_rsa.pub core18-armhf/root/.ssh/authorized_keys
lxc exec core18-armhf chown root:root /root/.ssh/authorized_keys

# 假设容器IP为 10.125.163.245
ssh -X root@10.125.163.245
```

在SSH会话内执行：
```
btctools
```
会得到如下错误：
> qt.qpa.screen: QXcbConnection: Could not connect to display localhost:10.0
> Could not connect to any X display.

所以只能尝试直接启动BTCTools二进制：
```
/snap/btctools/current/usr/bin/btctools-gui
/snap/btctools/current/usr/bin/btctools-gui --lang=en
/snap/btctools/current/usr/bin/btctools-gui --lang=zh_CN
```
因为编译BTCTools已经安装了依赖包，所以应该是可以启动的。


### 不推荐使用`multipass`

如果在运行`snapcraft`命令时不带`--use-lxd`和`--destructive-mode`，它就会使用默认的`multipass`虚拟机进行构建。

`multipass`会用`qemu-kvm`进行完全虚拟化，速度比`LXD`稍慢，而且启动和停止用时也更久。

此外，`multipass 1.3.0`和`snapcraft 4.0.4`有重大兼容性问题，根本无法正常构建。

强烈不建议使用`snapcraft`的`multipass`整合。如果你真的想用`multipass`，建议你直接通过虚拟机启动`Ubuntu 18.04`，然后通过`sudo /snap/bin/snapcraft --destructive-mode`进行构建，不要使用本机`snapcraft`命令。

## 安装构建好的snap软件包

```
sudo snap install --dangerous ./*.snap

# 运行
btctools

# 如果找不到命令
/snap/bin/btctools
```

## 发布

如果登陆报错，请删除`～/.config/snapcraft`文件夹。

```
snapcraft login

# 选择其中一个通道
snapcraft upload --release edge *.snap
snapcraft upload --release beta *.snap
snapcraft upload --release candidate *.snap
snapcraft upload --release stable *.snap
```

如果遇到报错，请确保`*.snap`只选中了一个文件，或者提供完整的文件名，比如`snapcraft push --release stable btctools_1.2.7_amd64.snap`。

备注：已知`gnome-software-plugin-snap`(Ubuntu自带应用商店)无法正常安装只有`edge`通道发布的软件。建议在交付用户时提交到`stable`通道。
