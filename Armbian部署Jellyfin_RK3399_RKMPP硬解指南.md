# 在原版 Armbian 镜像上部署 Jellyfin RK3399 rkmpp 硬解(HEVC 不花屏)

目标:从原版 `Armbian_26.11.0_rockchip_smart-am40_resolute_6.12.109_server` 镜像出发,让 Jellyfin 通过 rkmpp 硬件转码 HEVC(含 Main10)到 H.264——不花屏,也不用软解。全部步骤在 rk3399 smart-am40 实机验证过(2026-10-09)。如果不想动手,直接跳到文末用做好的镜像。

## 原理:三层各缺一不可

rkmpp 硬解链路由三层组成,任何一层不对都会失败:

1. **内核**要提供 BSP 的 MPP 和 RGA 驱动。原版 6.12.109 内核没有这套驱动,需要换成自编译的 6.12.111。为什么是 rga3(multi_rga)驱动:Jellyfin 自带的 librga 是 1.10.4,只说"句柄式"新协议,内核里的老 rga2 驱动听不懂,必须让 rga3 驱动接管 `/dev/rga`(相关移植:[MPP 驱动](https://github.com/HD9901-Rockchip/linux-6.12.y/commit/3255926ce0d53b0afabb0c2d78c12b4bfda9180c)、[rga3 驱动](https://github.com/HD9901-Rockchip/linux-6.12.y/commit/2a672b02d2cc9f549bb3e6879e0587b49415bf7d)、[rga3 RK3399 修复](https://github.com/HD9901-Rockchip/linux-6.12.y/commit/98cec323f1fa0381fa99c22e80d627a0de32866d)、[设备树](https://github.com/HD9901-Rockchip/linux-6.12.y/commit/3ee96b7dfc3bc37c58a5980e8650879534c76ccd))。
2. **用户态 MPP库**(librockchip_mpp)必须换成上游版本。Jellyfin 捆绑的库有 HEVC 解码 bug,是花屏的真正根源,详见《HEVC花屏根因分析与修复.md》。
3. **Jellyfin 四件套**:jellyfin(元包)、jellyfin-server、jellyfin-web、jellyfin-ffmpeg8。转码用的 ffmpeg 在 `/usr/lib/jellyfin-ffmpeg/`。

材料清单:

1. 自编译内核三件:`Image`、`rk3399-am40.dtb`、`/lib/modules/<版本>`。内核源码在 [HD9901-Rockchip/linux-6.12.y 的 rkmpp 分支](https://github.com/HD9901-Rockchip/linux-6.12.y/tree/rkmpp),配置用 [`rk3399_mpp_defconfig`](https://github.com/HD9901-Rockchip/linux-6.12.y/commit/9b06a38e28603240b8043f4c63ad04349a7db84e)。
2. Jellyfin arm64 的 4 个 deb(板上 `/var/cache/apt/archives`,或官方源 `apt-get download`)。
3. 上游编译出的 `librockchip_mpp.so.0`。
4. 原版 Armbian 镜像。

## 一、安装 Jellyfin

有网时直接用官方源:

```bash
apt-get install -y jellyfin-ffmpeg8 jellyfin-server jellyfin-web jellyfin
systemctl enable --now jellyfin
```

离线场景则把 4 个 deb 拷过去,一条命令装完:

```bash
apt-get install -y ./jellyfin-ffmpeg8_*.deb ./jellyfin-server_*.deb \
                   ./jellyfin-web_*.deb ./jellyfin_*.deb
```

装完核对一下 ffmpeg 认不认识硬件:

```bash
/usr/lib/jellyfin-ffmpeg/ffmpeg -hide_banner -hwaccels | grep rkmpp    # 应有 rkmpp
/usr/lib/jellyfin-ffmpeg/ffmpeg -hide_banner -encoders | grep h264_rkmpp
ls /usr/lib/jellyfin-ffmpeg/lib/librga.so*    # librga 1.10.x
```

## 二、替换内核

> 以下操作在一台 Linux 主机上对镜像文件进行;如果板子已经在跑,直接对运行系统操作的方法见 2.5。

### 2.1 挂载镜像

```bash
IMG=Armbian_26.11.0_rockchip_smart-am40_resolute_6.12.109_server_2026.09.14.img
sudo losetup -P /dev/loop0 "$IMG"
mkdir -p boot_mnt root_mnt
sudo mount /dev/loop0p1 boot_mnt     # boot 分区
sudo mount /dev/loop0p2 root_mnt     # rootfs
```

### 2.2 安装模块和内核镜像

```bash
KDIR=/path/to/linux-6.12.y; BDIR=/path/to/build-612   # 源码与 O= 构建目录
VER=$(cat $BDIR/include/config/kernel.release)        # 如 6.12.111-g3026e52b6d39-dirty

cd $KDIR
sudo -E make O=$BDIR INSTALL_MOD_PATH=$PWD/root_mnt INSTALL_MOD_STRIP=1 DEPMOD=true modules_install
sudo cp $BDIR/arch/arm64/boot/Image boot_mnt/vmlinuz-$VER
sudo cp $BDIR/System.map boot_mnt/System.map-$VER
# boot.scr 固定加载 dtb/rockchip/rk3399-smart-am40.dtb
sudo cp $BDIR/arch/arm64/boot/dts/rockchip/rk3399-am40.dtb \
        boot_mnt/dtb/rockchip/rk3399-smart-am40.dtb
```

两个容易踩的地方:`INSTALL_MOD_STRIP=1` 必须加,否则模块树 2.5GB,initramfs 会把 boot 分区撑爆;原 6.12.109 的 `vmlinuz/uInitrd/dtb` 保留不动,留作启动失败的退路。

### 2.3 生成 initrd/uInitrd

宿主机和镜像架构不同,用 qemu-user 模拟进 chroot,让 initramfs 工具在"原生"环境里跑:

```bash
sudo cp /usr/bin/qemu-aarch64-static root_mnt/usr/bin/
sudo mount --bind /dev  root_mnt/dev
sudo mount -t proc proc root_mnt/proc
sudo mount -t sysfs sys root_mnt/sys
sudo mount --bind boot_mnt root_mnt/boot

sudo chroot root_mnt /usr/bin/qemu-aarch64-static /bin/bash -c "update-initramfs -c -k $VER"
sudo chroot root_mnt /usr/bin/qemu-aarch64-static /sbin/depmod -a $VER

# U-Boot 要求 imageram 格式
mkimage -A arm64 -O linux -T ramdisk -C none -n uInitrd \
        -d boot_mnt/initrd.img-$VER boot_mnt/uInitrd-$VER
```

### 2.4 切换启动链接,收尾

```bash
sudo ln -sf vmlinuz-$VER  boot_mnt/Image
sudo ln -sf uInitrd-$VER  boot_mnt/uInitrd

sudo umount root_mnt/{boot,sys,proc,dev}
sudo rm -f root_mnt/usr/bin/qemu-aarch64-static
sudo umount boot_mnt root_mnt && sudo losetup -d /dev/loop0
```

Armbian 的启动逻辑是 U-Boot 读 `/boot/boot.scr` 和 `armbianEnv.txt`,再加载固定名的 `/boot/Image`、`/boot/uInitrd` 和 `/boot/dtb/rockchip/rk3399-smart-am40.dtb`。所以只需要改符号链接和 dtb,启动脚本本身不用碰。

### 2.5 替代做法:板子在跑时直接换

```bash
scp vmlinuz-$VER root@board:/boot/
ssh root@board 'ln -sf vmlinuz-$VER /boot/Image; sync; reboot'
# 模块用 rsync 同步到板的 /lib/modules/$VER/ 后执行 depmod -a $VER
```

## 三、替换 librockchip_mpp(不做必花屏)

这是整套方案里最容易被忽略、但最关键的一步。Jellyfin 捆绑的 MPP 库自带 HEVC 解码 bug,不换它,前面一切努力都会白费。在板子上编译上游 MPP(约三分钟):

```bash
git clone https://github.com/rockchip-linux/mpp.git /root/mpp-upstream
cd /root/mpp-upstream && mkdir -p build/board && cd build/board
cmake -DCMAKE_BUILD_TYPE=Release ../.. && make -j6 rockchip_mpp

JF=/usr/lib/jellyfin-ffmpeg/lib/librockchip_mpp.so.0
cp -n $JF ${JF}.jellyfin-orig     # 备份
cp mpp/librockchip_mpp.so.0 $JF
```

换完跑一次 ffmpeg,日志出现 `mpp version: 14729dd` 之类的新版本号才算生效。

## 四、验证

### 4.1 硬件就绪

```bash
uname -r                                  # 应为 6.12.111-...
ls /dev/rga /dev/mpp_service /dev/dri     # 三个节点都应在
dmesg | grep -E "mpp_service|mpp_rkvdec|rga2? "
# 期望看到:mpp_service probe success
#         mpp_rkvdec ff660000.rkvdec: probing finish
#         rga2 ff680000.rga: probe successfully ... hw_version:3.2.18218
```

### 4.2 转码冒烟测试

```bash
FF=/usr/lib/jellyfin-ffmpeg/ffmpeg
# 10-bit HEVC 硬解 → RGA 转格式 → 硬编 H.264,即 Jellyfin 的实际链路
$FF -hide_banner -y -hwaccel rkmpp -hwaccel_output_format drm_prime \
    -i test_hevc10.mkv -an -vf "scale_rkrga=format=nv12" \
    -c:v h264_rkmpp -b:v 4M -f mp4 /tmp/out.mp4
```

想严格验收,可以把输出和 CPU 软解逐帧比 PSNR:不应出现 Y 低于 40 dB 的离群帧,更不应有"I 帧全好、inter 帧成批坏"的固定模式。

### 4.3 Jellyfin 控制台

控制台 → 播放 → 转码:硬件加速选 **RKMPP**,勾选 HEVC 解码和 H.264 编码,保存后播一段 HEVC 片子,转码日志里应走 rkmpp 且画面无花屏。

## 五、容易踩的坑

1. **不换 librockchip_mpp 必花屏**,而且坏的是固定的那几帧——别被"I 帧正常"带偏去查 RGA,根因在用户态库。
2. `ROCKCHIP_RGA2` 和 `ROCKCHIP_MULTI_RGA` 都绑 `rockchip,rga2` 兼容名、都注册 `/dev/rga`,内核里**只能开一个**。
3. `modules_install` 忘了 `INSTALL_MOD_STRIP=1`,initramfs 会到 169MB,直接溢出 boot 分区。
4. jellyfin-ffmpeg 升级会把有 bug 的库写回去,升级后记得重换(见第三节)。
5. 如果 dmesg 里 rga2 probe 报 Oops 但 `/dev/rga` 还在,这是设备树的 `&rga` 时钟名不对造成的僵尸节点——用 [rkmpp 分支](https://github.com/HD9901-Rockchip/linux-6.12.y/tree/rkmpp)编译的 dtb 即可。
