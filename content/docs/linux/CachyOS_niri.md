---
title: CachyOS与niri配置
---

## 字体替换

因为默认英文字体好像不是等宽，导致终端巨巨巨丑，所以当务之急先配个英文字体再说

安装JetBrains Moni
```fish
sudo pacman -S ttf-jetbrains-mono
fc-list | grep "JetBrains Mono" #这里是确认是否安装成功

mkdir -p ~/.config/fontconfig/
micro ~/.config/fontconfig/fonts.conf
```

我这里用的终端编辑器是micro，比较符合现代使用习惯

然后在该文件夹中写入配置
```xml
<?xml version="1.0"?>
<!DOCTYPE fontconfig SYSTEM "fonts.dtd">
<fontconfig>
  <match target="pattern">
    <test name="family" qual="any">
      <string>monospace</string>
    </test>
    <!-- 第一位：JetBrains Mono -->
    <edit name="family" mode="prepend" binding="strong">
      <string>JetBrains Mono</string>
    </edit>
    <!-- 最后一位：文楷等宽 Nerd Font（作为中文回退） -->
    <edit name="family" mode="append" binding="strong">
      <string>LXGWWenKaiMono Nerd Font</string>
    </edit>
  </match>
</fontconfig>
```

可以先暂时把霞鹜文楷的部分去掉，也可以先执行下面这个安装一下霞鹜文楷

```fish
paru -S ttf-lxgw-wenkai ttf-lxgw-wenkai-mono-nerd
```

## 增加archlinuxcn源
```fish
micro /etc/pacman.conf
```
在文件的末尾增加
```xml
[archlinuxcn]
Server = https://mirrors.ustc.edu.cn/archlinuxcn/$arch
```
也可以增加自己喜欢的源

## 安装clash
```fish
sudo paru -S clash-verge-rev-bin
```

关机前记得关clash，否则有概率在开机后网络无法使用

## 配置中文输入法
```fish
sudo pacman -S fcitx5-im fcitx5-rime rime-ice-pinyin-git
mkdir -p ~/.local/share/fcitx5/rime
micro ~/.local/share/fcitx5/rime/default.custom.yaml
```

在用户配置文件中写入
```yaml
patch:
  __include: rime_ice_suggestion:/
```

在niri配置文件的环境变量部分写入
```xml
environment{
	LC_MESSAGES "zh_CN.UTF-8"
	XMODIFIERS "@im=fcitx"
}
```

然后后台启动fictx5
```fish
fcitx5 -d
```
打开fcitx5配置，添加中州韵，可以顺手把英文键盘删掉，有概率出bug，在终端中使用ctrl+L清屏，等待一会儿之后就可以使用雾凇输入法了

## niri配置文件
```fish
~/.config/niri/cgf/
```

## 补全nautilus功能
```fish
sudo pacman -S nautilus-open-any-terminal gst-plugins-base gst-plugins-good gst-libav
gsettings set com.github.stunkymonkey.nautilus-open-any-terminal terminal alacritty
```
## 将系统文件名替换为英文
```fish
LC_ALL=C.UTF-8 xdg-user-dirs-update --force
```

## 配置代理

1. 临时代理

```fish
# HTTP/HTTPS 代理
export http_proxy=http://127.0.0.1:7897
export https_proxy=http://127.0.0.1:7897

# 如果还需要 SOCKS5 代理
export all_proxy=socks5://127.0.0.1:7897
```
2. 创建代理开启指令
在`~/.config/fish/functions/proxy_on.fish`输入
```fish
function proxy_on
    set -gx http_proxy http://127.0.0.1:7897
    set -gx https_proxy http://127.0.0.1:7897
    set -gx all_proxy socks5://127.0.0.1:7897
    echo "代理已开启"
end
```

在`~/.config/fish/functions/proxy_off.fish`输入
```fish
function proxy_off
    set -e http_proxy
    set -e https_proxy
    set -e all_proxy
    echo "代理已关闭"
end
```