---
title: Arch下iphone的备份与恢复
description: 基于pymobiledivice3的备份,刷机与恢复教程
date: 2026-05-24T12:00:00+08:00
author: Saki酱的通信学习之路
cover: '{"image":"img/arisa_cover1.png"}'
catalog: true
share: true
filename1: 2026-05-24-arch下iphone的备份与恢复.md
tags:
  - ios
  - Arch
cover_image: "[[arisa_cover1.png]]"
---
最近我的iPhone更新完ios26.4.1后遇到了重启之后就会bootloop的问题,不过通过进入恢复模式/诊断模式后再重启就可以正常启动。查看iphone诊断日志,发现可能是nvram出现了问题。于是希望通过不保留数据刷机来解决这个问题。在选择备份数据的工具上,我选择了pymobiledevice3作为我的备份工具。
注:我可能感觉选择基于libimobiledevice的idevicebackup2工具比较好,但是未经过测试,不予评论。

## 安装流程
因为pymobiledevice3需要涉及到物理usb访问的问题,且PEP668不让直接运行`pip install`,所以我这里基于一个已经停止维护的AUR仓库`python-pymobiledevice3`进行改包。
找个合适的位置,先git clone下来
```
git clone https://aur.archlinux.org/python-pymobiledevice3.git
cd python-pymobiledevice3
```
编辑`PKGBUILD`,找到
```
pkgver=6.0.1
```
字段,将其改成github中的最新版本号(我这里是9.13.0),最新版本号可在[pymobiledevice3_releases](https://github.com/doronz88/pymobiledevice3/releases)中查看。
然后,仿照[requirements.txt](https://raw.githubusercontent.com/doronz88/pymobiledevice3/refs/heads/master/requirements.txt),修改下面的`depends=...`一行,这里提供ai生成的依赖行。
```
depends=(
  'openssl'
  'libusb'
  'python'
  'python-construct'
  'python-asn1'
  'python-click'
  'python-coloredlogs'
  'ipython'
  'python-bpylist2'
  'python-pygments'
  'python-hexdump'
  'python-daemonize'
  'python-gpxpy'
  'python-pykdebugparser'
  'python-pyusb'
  'python-tqdm'
  'python-requests'
  'xonsh'
  'python-parameter-decorators'
  'python-packaging'
  'python-pygnuutils'
  'python-cryptography'
  'python-pycrashreport'
  'python-fastapi'
  'uvicorn'
  'python-nest-asyncio'
  'python-pillow'
  'python-inquirer3'
  'python-ifaddr'
  'python-hyperframe'
  'python-srptools'
  'python-qh3'
  'python-developer_disk_image'
  'python-opack2'
  'python-psutil'
  'python-pytun-pmd3'
  'python-prompt_toolkit'
  'python-sslpsk-pmd3'
  'python-python-pcapng'
  'python-plumbum'
  'python-wsproto'
  'python-typer'
  'python-defusedxml'
  'python-pyimg4'
  'python-construct-typing'
  'python-typer-injector'
  'pyton-ipsw_parser'
)
```
修改后保存,运行
```
updpkgsums
```
来更改新包的checksum
现在我们针对几个aur源中不存在的包手搓几个PKGBUILD。创建一个目录,然后创建一个PKGBUILD文件。以下PKGBUILD文件均为ai生成,不保证其正确性,但本人测试没有问题。
python-construct-typing
```
pkgname=python-construct-typing

_pkgname=construct-typing

pkgver=0.7.0

pkgrel=1

pkgdesc='Extension for construct that adds typing features'

arch=('any')

url='https://github.com/timrid/construct-typing'

license=('MIT')

depends=('python' 'python-construct')

makedepends=('python-build' 'python-wheel' 'python-installer' 'python-setuptools')

source=("https://files.pythonhosted.org/packages/8c/0c/2db6f7e1ae9795e436c6a0dc0bc38b12b8c8a228cb63203e24190b755b3b/construct_typing-0.7.0-py3-none-any.whl")

sha256sums=('c92383c6e8e5d07ba25811c8d5163820458d821e73bb1006541f43f89788646c')

build() {

cd "$srcdir"

# wheel 不需要编译，直接解包作为构建产物

python -m wheel unpack "construct_typing-${pkgver}-py3-none-any.whl" -d build

}

package() {

cd "$srcdir"

python -m installer --destdir="$pkgdir" "construct_typing-${pkgver}-py3-none-any.whl"

}
```
`python-typer-injector`更改source链接和`python -m wheel unpack`行即可。
接下来是两个包的特殊处理:
### 1.python-ipsw_parser
由于这个包更新略微滞后,导致如果使用aur源直接安装,库会不支持pymobiledevice3的刷机功能。因此我们还是通过更改PKGBUILD的方式来解决此问题
```
cd /tmp
git clone https://aur.archlinux.org/python-ipsw_parser.git
cd python-ipsw_parser
```
编辑PKGBUILD,更改`pkgver`为`1.7.0`(可以在[python-ipsw_parser](https://pypi.org/project/ipsw-parser/#files)上查看最新版本号)
```
makepkg -si
```
安装这个包即可,同样的,再安装`python-typer-injector`和`python-construct-typing`两个包
接下来,我们需要手动处理依赖关系(**关键,因为如果不处理会导致后面pyimg4无法正常安装**),我们需要安装2.8.0的python-asn1来解决依赖冲突。
```
git clone https://aur.archlinux.org/python-asn1.git /tmp/python-asn1-old
cd /tmp/python-asn1-old
git log --oneline  # 找到 asn1 3.0.0 之前那个 commit
git checkout <commit_before_3.0.0>
makepkg -si
```
安装完后,回到`python-pymobiledevice3`的目录下,这时候可以运行`makepkg -si`了。
## 备份iphone中的数据
先安装usbmux
```
sudo pacman -S usbmuxd
sudo systemctl enable --now usbmuxd
```
连接iphone,运行
```
pymobiledevice3 usbmux list
```
如果出现iPhone的设备型号,UDID等信息则说明成功了,如果发现运行时提示`usbmuxd`没有开启,重新运行`pymobiledevice3 usbmux list`即可。
建议先为iPhone设置一版备份密码
```
pymobiledevice3 backup2 encryption on '你的密码'
```
设置完成后,直接选择全量备份(后续备份时可以基于这一版数据库进行增量备份)。
```
pymobiledevice3 backup2 backup --full /path/to/backup_dir
```
即可完成备份,我是256G的iPhone 13 Pro,用了150多G,大概备份了半小时左右,速度还是挺快的。

## 刷机
备份完成后,确认数据完整,即可进行刷机操作。
先去[ipsw.me](https://ipsw.me/)上下载固件
iphone进入恢复模式/dfu模式,运行
```
sudo pymobiledevice3 restore update -i /path/to/ipsw
```
如果不想保留数据,加上`--erase`参数
```
sudo pymobiledevice3 restore update -i /path/to/ipsw --erase
```
如果连接成功,手机上应该出现经典的白苹果读条,电脑上应该会有详细进度表示正在刷机
## 恢复
刷机完成后,正常激活iphone,**不要**在激活界面选择从mac/pc恢复数据,等设置完成进入桌面后,在账号设置中,**关闭**"查找我的iphone"(恢复完数据可以再打开)后,将iphone连接电脑
```
pymobiledevice3 usbmux list
```
确认能连接设备后,运行
```
pymobiledevice3 backup2 restore --system --settings --password "你的加密密码" --source <原设备UDID> --remove /path/to/backup
```
手机上有"正在恢复"字样即算作正在恢复数据。恢复之后,跟爱思助手备份的恢复一样,联网下载好所有的APP即算恢复完成了。
## 版权声明

[Arch下iphone的备份与恢复](https://blog.05262025.xyz/posts/2026-05-24-arch下iphone的备份与恢复/) © 2026 by [高三小祥](https://github.com/jmc0x68) is licensed under [CC BY-NC-SA 4.0](https://creativecommons.org/licenses/by-nc-sa/4.0/)
文中所有图片著作权均归版权方所有
