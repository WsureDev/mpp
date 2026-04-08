# RK3588s 10-bit 视频硬解绿屏问题分析与修复说明（AFBC 强制方案）

## 1. 问题背景
在 Rockchip RK3588s (Android 12) 平台上，使用 MPP (Media Process Platform) 进行硬件解码 10-bit 色深的 H.265 视频时，出现了显示异常。
具体表现为：播放 1080p 10-bit 视频时，画面出现绿屏/绿线。4K HDR10+ 10-bit 视频显示正常。

## 2. 错误方案回顾（已废弃）

### 方案 A-旧：10-bit→8-bit 格式降级 + FBC 全局禁用
**此方案有两个致命错误：**

1. **VDPU383 硬件不支持 10-bit→8-bit 输出转换**
   - HAL 代码 `hal_h265d_vdpu383.c` 第 214 行：`bit_depth = 10;` 硬编码
   - PPS 中的 `bit_depth_luma_minus8=2` 告诉硬件按 10-bit 解码
   - 硬件始终输出 NV15 (10-bit packed) 数据
   - 仅修改 `pix_fmt` 和 `hor_stride` 导致缓冲区过小，硬件写溢出 → 绿线

2. **FBC 全局禁用导致 4K 回归**
   - 4K HDR10+ 文件在原厂固件上通过 AFBC 路径正常显示
   - 禁用 FBC 后强制走线性 NV15 路径 → 触发同样的 stride bug → 绿线

### 关键证据：stride 溢出计算

| 分辨率 | 硬件实际输出 stride | 错误 patch 设的 stride | 溢出量/行 |
|--------|-------|--------|---------|
| 3840x2160 | 4800 bytes (NV15) | 3840 bytes (NV12) | -960 bytes |
| 1920x1080 | 2400 bytes (NV15) | 1920 bytes (NV12) | -480 bytes |

## 3. 根本原因分析

RK3588 Android 12 的显示栈（Gralloc/HWC/DRM）处理 **线性 NV15** 缓冲区时，
stride/UV-offset 计算有 bug。但 **AFBC 压缩的 NV15** 路径不受影响。

- 4K HDR10+ 文件：播放器请求了 AFBC 输出 → AFBC NV15 → 显示正常
- 1080p 10-bit 文件：播放器没有请求 AFBC → 线性 NV15 → stride bug → 绿屏

## 4. 正确的修复方案：强制 AFBC 输出

在 MPP 解码器层，当检测到 10-bit YUV 输出时，强制添加 `MPP_FRAME_FBC_AFBC_V2` 标志，
使所有 10-bit 内容走 AFBC 路径（VOP2 对 AFBC NV15 的处理是正确的）。

### 修改点

1. **`inc/mpp_frame.h`**: 恢复原始 FBC 宏定义（不再全局禁用）
2. **`mpp/codec/dec/h265/h265d_sps.c`**: 恢复原始 10-bit 格式选择（保持 NV15）
3. **`mpp/codec/dec/h265/h265d_dpb.c`**: 恢复原始 stride 计算 + 新增 force-AFBC 逻辑
4. **`mpp/codec/dec/h264/h264d_init.c`**: 恢复原始格式/stride + 新增 force-AFBC 逻辑

### 运行时控制

```bash
# 默认启用（强制 AFBC）
setprop mpp_dec_force_10bit_fbc 1

# 禁用（例如在 Android 14+ 或已修复的显示栈上）
setprop mpp_dec_force_10bit_fbc 0
```

## 5. 收益与风险

### 收益
- 10-bit 内容保持原始色深（NV15），不丢失 HDR/色阶信息
- 4K HDR10+ 不受影响（原本就走 AFBC）
- 1080p 10-bit 被强制切到 AFBC 路径，绕过线性 NV15 的 stride bug
- 8-bit 内容完全不受影响

### 已知风险
- 如果播放器使用 buffer 模式（非 Surface 渲染），强制 AFBC 可能导致 CPU 无法直接读取帧数据
- 可通过 `mpp_dec_force_10bit_fbc=0` 环境变量禁用此 workaround
