安装好Arch Linux或Manjaro Linux系统后默认没办法正常显示emoji，通常会变成方框或者带有unicode码的方块：

![incorrect-emoji](../../images/archlinux-show-emoji/cannot-show-emoji.png)

这是因为缺失字体以及相关的字体配置导致的。

当然也有一小部分应用没有这个问题（比如Chrome），因为字体是可以在程序里单独设置的，Qt和GTK都有相应的接口，只要设置了正确的字体即可显示emoji。但很多系统程序为了兼容性是没有进行这些设置的，比如文件管理器和终端模拟器。

解决办法：

1. 安装emoji字体
2. 更新字体配置

首先是安装emoji字体，不考虑aur和自己下载安装字体的话一般会安装这个：`noto-fonts-emoji`。自测应该能正常显示所有常见emoji。

安装命令：

```bash
sudo pacman -S noto-fonts-emoji
```

这时应用程序还是不能正常显示emoji的，需要进行第二步更新字体配置。

字体的配置文件在`/etc/fonts`目录下，不同系统可能不同，在这个目录下新建`local.conf`文件，这个文件里是我们的自定义配置，**不要去修改`font.conf`文件**。

`local.conf`里写入下面的内容：

```xml
<?xml version="1.0"?>
<!DOCTYPE fontconfig SYSTEM "fonts.dtd">
<fontconfig>
 <alias>
   <family>sans-serif</family>
   <prefer>
     <family>Noto Sans</family>
     <family>Noto Color Emoji</family>
     <family>Noto Emoji</family>
     <family>DejaVu Sans</family>
   </prefer> 
 </alias>

 <alias>
   <family>serif</family>
   <prefer>
     <family>Noto Serif</family>
     <family>Noto Color Emoji</family>
     <family>Noto Emoji</family>
     <family>DejaVu Serif</family>
   </prefer>
 </alias>

 <alias>
  <family>monospace</family>
  <prefer>
    <family>Noto Mono</family>
    <family>Noto Color Emoji</family>
    <family>Noto Emoji</family>
    <family>DejaVu Sans Mono</family>
   </prefer>
 </alias>
</fontconfig>
```

保存文件后使用命令让新配置生效：

```bash
fc-cache
```

更新完配置后需要重启应用才能正常显示emoji（一部分桌面服务需要注销当前用户再次登录才会重启）。***推荐可以的话直接重启一下操作系统***。

现在可以正常显示emoji了：

![show-emojis](../../images/archlinux-show-emoji/emojis.png)

##### 参考

<https://dev.to/darksmile92/get-emojis-working-on-arch-linux-with-noto-fonts-emoji-2a9>
