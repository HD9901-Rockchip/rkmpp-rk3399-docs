# RK3399 上 Jellyfin 硬解 HEVC 花屏的根因与修复

环境:rk3399(smart-am40),Armbian 26.11,内核 6.12.111,Jellyfin 12.2 + jellyfin-ffmpeg8 8.1.3。

## 现象

Jellyfin 开启 rkmpp 硬件转码后,HEVC(包括 Main10 10-bit)转 H.264 的输出出现宏块花屏。花屏有几个稳定的规律:

1. 只有 P/B 帧(inter 帧)坏,I 帧始终正常;
2. 同一段视频每次转码,坏的总是同一批帧——完全可复现,不是随机竞态;
3. 画面上静态背景误差小,运动的人物区域误差大。

测试片源为 HEVC Main10 1080p(`/mnt/disk/Openlist/1.mkv`)。

## 排查过程

我们用"逐帧 PSNR 对比 CPU 软解基准"的方法一层层排除嫌疑。每轮取 48 帧:

**第一轮:确认复现,怀疑 RGA。** 走 rkmpp 硬解 → scale_rkrga(硬件缩放转格式)→ 下载到 CPU 的完整硬件链路,43/48 帧损坏(Y 平面 16–35 dB),I 帧 52 dB 干净。于是给内核 rga3 驱动加了"外部 dma-buf 映射后强制作废 CPU 缓存"的补丁重编上板——结果没效果,44/48 坏。RGA 侧缓存失效的假设被排除,内核时钟、设备树与原厂 BSP 逐项比对也无差异。

**第二轮:H.264 对照。** 用同源转出的 H.264 测试片走完全相同的链路(硬解 + RGA),48/48 帧全部 99 dB,位级无损。H.264 和 HEVC 在 RK3399 上走的是不同解码核(H.264 走 vdpu2,HEVC 走 rkvdec),这说明 RGA 本身和 rkmpp→RGA 的交接层都没问题,问题被锁定在 rkvdec 解 HEVC 这条路径上。

**第三轮:绕开 RGA 直接验证解码输出。** 这是关键一步。之前的证据都来自"解码+RGA"组合链路,分不清是谁弄坏的数据。改用 8-bit HEVC 测试片,让 rkmpp 解码后直接下载到 CPU,不经过任何硬件转换:结果 29/48 帧损坏,而且**坏的只有 Y(亮度)平面,U/V(色度)高达 70 dB 几乎无损**。下载路径由色度平面的完好得到证明,所以结论很直接:解码器吐出来的数据本身就是错的。

**第四轮:交叉验证。** 让两个完全独立的读取方——RGA 硬件和 ffmpeg 的 CPU 下载——分别去读同一批解码缓冲,两者报出的坏帧集合几乎一致。缓冲区里的内容确实是坏的,解码器的责任至此实锤。

## 根因

问题出在 **MPP 用户态库 librockchip_mpp**,不是内核。

rk3399 的视频解码是"薄内核、厚用户态"的架构:内核里的 `mpp_service`/`mpp_rkvdec` 驱动只管时钟、电源域、复位、中断和把寄存器列表写给硬件;HEVC 解码最核心的寄存器编程——包括参考帧队列(DPB)和参考图像集(RPS)的管理——全部由用户态的 librockchip_mpp 完成。这就解释了为什么解码出错时 dmesg 一声不吭,也解释了为什么换内核版本、改设备树都没用。

从坏帧的误差分布也能看出机理:整帧结构和熵解码(CABAC)基本正确,只有部分 CTU(编码树单元)出错,而且集中在运动区域——这是帧间预测读到了错误的参考数据,而不是反量化整体失败;色度数据量只有亮度的四分之一、预测方式也不同,所以 U/V 能幸免。一句话概括:**librockchip_mpp 的 HEVC 参考帧/状态管理有缺陷,导致 inter 帧部分宏块用错了预测数据**。上游 MPP 近期修复的 HEVC 缺陷族与此吻合,例如 [8f88b50 fix[h265d]: Force PS/RPS update for the first sent task](https://github.com/rockchip-linux/mpp/commit/8f88b50df0af9dc367bf2f9943808e82a8d14482)——PPS/RPS 表更新被吞掉后,首个下硬件的任务会读到全零表(该提交针对 RK35 系列的 vdpu34x/382/383/384a,rk3399 的对应修复包含在更早的提交里)。

社区也有同样的结论。Armbian 论坛 2026 年 9 月的帖子《[HEVC root cause found — it never worked, for anyone](https://forum.armbian.com/topic/61933-hevc-root-cause-found-%E2%80%94-it-never-worked-for-anyone-fix-is-on-the-dev-branch-verified-bit-exact-testers-wanted/)》测了三代 MPP 库:发行版 PPA 版(`5275a9a6`,2024-04)长跑六万多帧后出现解码器 reset;**Jellyfin 自家 fork(`a9380ef`,2025-12,正是 jellyfin-ffmpeg 捆绑的这版)同样中招**;只有 [rockchip-linux/mpp 上游 master](https://github.com/rockchip-linux/mpp) 完全干净。8-bit VP9 不受影响、HEVC 全体出问题,和"根因在 MPP 库的 HEVC 路径"完全一致。

## 涉及的软件版本

- Jellyfin Server / Web:`12.2+ubu2604`
- jellyfin-ffmpeg:`8.1.3-1-resolute`(提供 rkmpp 硬件加速、scale_rkrga、h264_rkmpp,位于 `/usr/lib/jellyfin-ffmpeg/`)
- librockchip_mpp(ffmpeg 捆绑):Jellyfin fork,约 [`a9380ef`](https://github.com/jellyfin/jellyfin-ffmpeg/commit/a9380ef)(2025-12)——**花屏根源,必须替换**
- librockchip_mpp(替换后):[rockchip-linux/mpp](https://github.com/rockchip-linux/mpp) `develop` 分支 [`14729dd`](https://github.com/rockchip-linux/mpp/commit/14729dd578e570e5f00fd1dd2113f5429012d64b)(2026-09-18)——验证干净
- librga(捆绑):[1.10.4](https://github.com/airockchip/librga),只支持句柄式新协议,这也是内核必须用 rga3(multi_rga)驱动接管 `/dev/rga` 的原因
- 内核:6.12.111,`MPP_SERVICE/RKVDEC/VDPU2/VEPU2` 与 `ROCKCHIP_MULTI_RGA` 内置。内核与花屏无关,但硬解链路依赖这套驱动,源码见 [rkmpp 分支](https://github.com/HD9901-Rockchip/linux-6.12.y/tree/rkmpp),关键提交:[MPP 驱动移植](https://github.com/HD9901-Rockchip/linux-6.12.y/commit/3255926ce0d53b0afabb0c2d78c12b4bfda9180c)、[rga3 驱动导入](https://github.com/HD9901-Rockchip/linux-6.12.y/commit/2a672b02d2cc9f549bb3e6879e0587b49415bf7d)

## 修复方法

在板子上编译上游 MPP 并替换捆绑库,全程保持硬件转码,不引入软解:

```bash
git clone https://github.com/rockchip-linux/mpp.git /root/mpp-upstream
cd /root/mpp-upstream && mkdir -p build/board && cd build/board
cmake -DCMAKE_BUILD_TYPE=Release ../.. && make -j6 rockchip_mpp

JF=/usr/lib/jellyfin-ffmpeg/lib/librockchip_mpp.so.0
cp -n $JF ${JF}.jellyfin-orig     # 备份原版
cp mpp/librockchip_mpp.so.0 $JF   # 替换为上游修复版
```

替换后运行 ffmpeg,日志里应出现 `mpp version: 14729dd`。

## 修复前后的量化对比

每段 48 帧,与 CPU 软解基准逐帧比 PSNR:

| 测试 | 修复前 | 修复后 |
|---|---|---|
| 8-bit HEVC 硬解直接下载 | 29/48 帧坏(Y 28–38 dB) | 0/48 坏,99 dB(位级一致) |
| 10-bit HEVC 经 rkmpp→RGA 链路 | 43/48 帧坏(Y 最低 16 dB) | 0/48 坏,52 dB |
| H.264 全链路(对照组) | 0/48 坏(99 dB) | 0/48 坏(99 dB) |
| 整片 1m45s 硬件转码(硬解→RGA 十转八→h264_rkmpp 硬编) | 花屏 | 54 秒完成,约 1.9 倍实时;片头片尾抽帧视觉无差异,无解码器 reset |

整片转码输出相对源片有 34–38 dB 的差异,这是 4 Mbps 码率下编码器的正常损耗——误差曲线平滑衰减、没有结构性马赛克,肉眼看不出来。

## 几点提醒

1. **jellyfin-ffmpeg 包升级会把有问题的捆绑库覆盖回来。** 升级后要重新替换 `librockchip_mpp.so.0`(原版备份在 `.jellyfin-orig`,或者用 `/root/mpp-upstream` 重新编译)。
2. 内核 rga3 驱动里"外部缓冲缓存作废"的补丁([rga_mm.c 修改](https://github.com/HD9901-Rockchip/linux-6.12.y/commit/98cec323f1fa0381fa99c22e80d627a0de32866d))虽然事后证明不是这次花屏的根因,但对系统无害,保留作为防御性措施。
3. Jellyfin 控制台的转码设置选 RKMPP 硬件加速,勾上 HEVC 解码和 H.264 编码即可,不要回退到软解。
