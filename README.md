# GKI 内核构建包 (KSU + 隐藏)

目标设备: Redmi K60 (mondrian) / Android 16
目标内核: 5.10.257-android12-9-Kirara-RiverSouth-26Y06LTSR01-nemo
源码基座: NetizenNemo/android_gki_kernel_5.10_common (branch android12-5.10)

## 目录结构

```
gki-build/
├── .github/workflows/build.yml   GitHub Actions 工作流（已写好的全自动构建）
├── device.config                 从设备 boot 镜像提取的完整 config（6756 行）
├── final.config                  本机已配好的最终 config（LTO/CFI/KSU/vermagic 全对）
├── build-info.txt                设备指纹与构建元数据
├── boot_template.img             *你需要自己放*
└── README.md                     本文档
```

## 使用步骤

### 1. 建公开仓库

github.com/new → Repository name: gki-build → 选 Public → Create

公开仓库的 Actions 分钟数为无限免费。

### 2. 上传文件

把本目录全部内容推到仓库，额外放入两个文件：

- boot_template.img   ← 你的 boot_Nemo.img 改名而来（必须）
- （.github/workflows/build.yml 保持路径不变）

.gitignore 建议内容:

```
*.img
!boot_template.img
*.apk
kernelsrc/
```

### 3. 触发构建

仓库 → Actions → 左侧 "Build KSU-GKI" → Run workflow

参数说明:

| 参数 | 默认 | 说明 |
|---|---|---|
| full_lto | true | true=复现原厂 FULL LTO；false=THIN 省内存 |
| sublevel | 257 | 与设备对齐，不能改 |
| ksu_branch | main | KernelSU 分支 |

### 4. 取产物

构建约 40~90 分钟。完成后在该次 Run 页面底部下载 artifact `ksu-gki-boot`。

## 四条红线（缺一不可）

```
1. clang 必须 19.x      镜像由 clang 19.0.1 编译，LTO 位码格式绑定主版本
2. SUBLEVEL 必须 257    否则 modversions 符号 CRC 不匹配，vendor 模块拒载
3. LOCALVERSION 逐字一致  否则 vermagic 不匹配，模块全部拒载
   -android12-9-Kirara-RiverSouth-26Y06LTSR01-nemo
4. 内核段差异 > 300 块   否则是尾部注入而非 GKI 内置
```

## 刷入前校验

拿到 new_boot.img 后在本机跑：

```bash
python3 - <<'EOF'
import struct
d = open('/sdcard/Download/gki-build/new_boot.img','rb').read()
ks, rs = struct.unpack('<II', d[8:16])
off, real = 0x1000, 0
while off < len(d):
    blk = d[off:off+4096]
    if not blk or not any(blk): break
    real += 4096; off += 4096
print('header_kernel_size =', ks)
print('real_kernel_size   =', real)
print('CHECK:', 'PASS' if abs(ks-real) < 0x2000 else 'FAIL 弃用')
EOF
```

GKI 内置判定（差异块数）:

```bash
python3 - <<'EOF'
A = open('/sdcard/Download/boot_Nemo.img','rb').read()
B = open('/sdcard/Download/gki-build/new_boot.img','rb').read()
CH = 65536
n = sum(1 for i in range(0,min(len(A),len(B)),CH) if A[i:i+CH] != B[i:i+CH])
print('差异块数 =', n, '→', 'GKI OK' if n > 300 else 'WARN 疑似尾部注入')
EOF
```

## 刷入

```bash
# 备份当前 boot
dd if=/dev/block/bootdevice/by-name/boot$(getprop ro.boot.slot_suffix) \
   of=/sdcard/boot_backup_$(date +%Y%m%d_%H%M%S).img

# 刷入新内核
dd if=/sdcard/Download/gki-build/new_boot.img \
   of=/dev/block/bootdevice/by-name/boot$(getprop ro.boot.slot_suffix)
```

## 常见失败与对策

| 现象 | 原因 | 对策 |
|---|---|---|
| LTO 链接报 invalid bitcode | clang 主版本不对 | 确认用 19.x |
| 开机断在 vendor 阶段 | modversions CRC 不匹配 | 确认 SUBLEVEL=257 且源码分支正确 |
| vermagic 带 git hash | LOCALVERSION_AUTO 未关 | 已在工作流中关闭 |
| 模块全部拒载 | LOCALVERSION 不一致 | 逐字核对 |
| 编译 OOM | FULL LTO 内存不足 | full_lto 改 false |
| 内核段差异过少 | 打包方式错误 | 用 magiskboot repack |

## 备选：Oracle 永久免费 ARM 机器

```
实例    VM.Standard.A1.Flex
配置    4 OCPU / 24 GB RAM / 200 GB
系统    Ubuntu 24.04 ARM64
区域    韩国/日本（延迟低）
```

24GB 内存可无压力跑 FULL LTO，是唯一能 1:1 复现原厂编译条件的免费方案。

```bash
sudo apt update && sudo apt install -y git bc bison flex libssl-dev \
  libelf-dev cpio rsync clang-19 lld-19 llvm-19
git clone --depth=1 -b android12-5.10 \
  https://github.com/NetizenNemo/android_gki_kernel_5.10_common.git ksrc
# 后续步骤与 build.yml 完全一致
```

## 备注

- device.config 是从设备实际运行的 boot 镜像中提取的内嵌 config，
  比任何公开 defconfig 都更接近真实编译条件。
- final.config 是本机已完成配置对齐的最终产物，可直接替换使用。
- 编译器需 19.x（Ubuntu 24.04 arm64 的 clang-19 已足够；
  如需完全一致可用 AOSP r536225）。
