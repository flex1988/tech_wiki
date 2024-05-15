# 如何编译加载 linux kernel

linux 允许手动编译一个 kernel 并用它替换当前的系统内核，这和用系统镜像重装相比简单很多，而且对于内核开发来说非常有用。

#### &#x20;1. 下载 linux 源码，可以在 github 或者其他地方下载

#### &#x20;2. checkout 出想要的 kernel 版本

#### &#x20;3. make menuconfig

#### &#x20;4. 安装依赖的库

```
yum groupinstall -y "development tools"
yum install -y openssh-devel elfutils-libelf-devel bc
yum install -y gcc gcc-c++ bc patch ncurese-devel
```

#### &#x20;5. make -j 8编译



#### &#x20;6. 编译安装模块

make modules\_install

#### 7. install 内核

\
&#x20;make install

#### 8. 设置为默认启动内核

```
cat /boot/grub2/grub.cfg | grep menuentry
grub2-set-default 'Tencent tlinux (5.5.0-tlinux3-0023.1) 2.6'
grub2-editenv list
```

#### 9. 重启服务器

uname -r
