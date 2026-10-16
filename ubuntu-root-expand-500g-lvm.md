# Ubuntu 根分区扩容到 500 GiB（Windows + Ubuntu 双系统 · LVM 方案）

> 记录日期：2026-10-16
> 适用机器：Lenovo 笔记本（单块 1 TB NVMe，UEFI + GPT，Windows 11 + Linux 双系统）
> 这是一份可执行手册：动手前请先读第 8 节的风险与回滚点。

---

## 1. 目标与结论

- **目标**：把 Ubuntu 根分区从 200 GiB 扩到约 500 GiB（+300 GiB），并且以后还能继续加。
- **做法**：不合并 C/D、不移动任何现有分区。把 D 让出的 300 GiB 建成一块新的 LVM PV，把旧根**复制**进新 LV；验证能正常启动后，再把旧根所在分区也并入同一个卷组，在线把根分区拉到约 500 GiB。
- **收益**：整个过程没有「移动分区」这种高风险动作，旧系统在验证通过前一直完好可启动；以后的扩容是几条命令、在线完成、不搬数据、不需要 U 盘。

---

## 2. 现状快照

磁盘：1 TB NVMe，容量 `1024209543168` 字节 = **953.87 GiB**，GPT，**全盘已分配，没有任何未分配空间**。

| 分区号 | 内容 | 起始 | 大小 | 说明 |
|---|---|---|---|---|
| p1 | EFI System | 0 | 0.44 GiB | 450 MiB |
| p2 | MSR | 0.44 GiB | 0.02 GiB | 16 MiB |
| p3 | **C: Windows** | 0.46 GiB | 400 GiB | NTFS，空闲约 185 GiB |
| p4 | **D: Data** | 400.46 GiB | 351.46 GiB | NTFS，已清理 Steam 库，现用约 26 GiB、空闲约 325 GiB |
| p6 | **Ubuntu 根** | 751.92 GiB | 200 GiB | 类型为 Linux filesystem，预期是单个 ext4，无独立 `/boot`、`/home` |
| p5 | **WinRE** | 951.92 GiB | 1.95 GiB | 位于盘尾，**本方案不动它** |

执行前请在 Linux 侧补全这张表（命令见第 6 节第 0 步）：

| 待确认项 | 期望值 | 实测 |
|---|---|---|
| 发行版版本 | 与本机一致（`cat /etc/os-release`） | |
| 根文件系统类型 | `ext4`（`df -T /`） | |
| 是否有独立 `/boot`、`/home`、swap 分区 | 无 | |
| `lvm2` 是否已安装 | 已安装 | |
| `/etc/fstab` 中根分区写法 | `UUID=...` | |

---

## 3. 为什么不能按最初的想法做

最初的设想是「合并 C、D 两块盘，再给 Ubuntu 划 200 G」。有两个问题：

**一、算不过来。** 合并后的 C+D 是 751.46 GiB，Ubuntu 再要 200 GiB（合计 400 GiB），加上 EFI/MSR/WinRE 共 2.41 GiB，需要 1153.87 GiB，而磁盘只有 953.87 GiB —— 正好差 200 GiB。这两个要求只能二选一。

**二、合并纯亏。** D 上还留着 Adobe Photoshop / Premiere / AutoCAD / Autodesk / Drivers 等约 26 GiB 已安装软件，合并要把它们搬进 C，注册表与安装路径 `D:\...` 失效，基本等于重装；工作区 `D:\CANN` 也会跟着失效。而 D 现在几乎全空，腾空间根本不需要合并。

**三、真正的难点在几何位置。** Ubuntu 分区在盘尾，右边只有 1.95 GiB 的 WinRE，所以新增空间只能来自它**左边**（D 的区域）。把分区起点往左挪，Windows 磁盘管理做不到，只能靠 GParted 做「向左移动 + 扩容」——那是一次全量数据搬移（几十分钟），中途中断会造成文件系统损坏。

结论：**放弃合并，用 LVM 绕开「移动分区」这个唯一的高风险动作。**

---

## 4. 方案选择：为什么用 LVM

传统做法（GParted 左移 ext4 根分区）与本方案都要搬一次数据，但风险和后续收益完全不同：

| | GParted 左移根分区 | 本方案（LVM） |
|---|---|---|
| 数据搬移方式 | 原地移动（先缩到最小、再整块搬） | `rsync` 复制到新卷，**旧的根本不动** |
| 中断后果 | 文件系统损坏，可能起不来 | 重做复制即可，旧根完好 |
| 出错的退路 | 只能靠备份 | 验证通过前旧根可直接启动回去 |
| 以后再加空间 | 每次都要重新左移，且越搬越久 | 新建 PV → 扩 VG/LV → 在线 `resize2fs`，几分钟 |
| 新空间位置要求 | 必须紧贴 Ubuntu 分区左边 | **任意位置**（从 C 或 D 挤出来的都行） |

关键点：LVM 的逻辑卷可以横跨多块物理分区（PV），所以「扩容」不再等于「移动分区」。

---

## 5. 目标布局

| 分区号 | 内容 | 起始 | 大小 | 变化 |
|---|---|---|---|---|
| p1 / p2 | EFI / MSR | 0 | 0.44 / 0.02 GiB | 不变 |
| p3 | C: Windows | 0.46 GiB | 400 GiB | 不变 |
| p4 | D: | 400.46 GiB | **51.46 GiB** | 让出 300 GiB（剩约 25 GiB 空闲） |
| p7 | **LVM PV #1** | **451.92 GiB** | **300 GiB** | 新建，来自 D 让出的空间 |
| p6 | **LVM PV #2** | 751.92 GiB | 200 GiB | 由原 Ubuntu 根分区改造 |
| p5 | WinRE | 951.92 GiB | 1.95 GiB | 不变 |

卷组与逻辑卷：

```
VG: vgubuntu        （约 500 GiB，横跨 p7 + p6）
└── LV: root        （约 499.9 GiB，ext4，label = ubuntu-root，挂载为 /）
```

最终 `df -h /` 显示约 **496 GiB 可用**（`mkfs.ext4 -m 1` 只保留 1% 的 root 保留块，比默认 5% 多出约 15 GiB）。

> 注意：新分区编号是 **p7**，但物理位置在 **p6 左边**。分区号由 GPT 槽位决定，与位置无关；GRUB 和系统靠 UUID、卷组名识别，不受影响。

---

## 6. 执行步骤

### 第 0 步：前置检查与准备（在现有 Linux 里）

```bash
cat /etc/os-release                      # 确认发行版版本，U 盘要按这个版本准备
lsblk -o NAME,SIZE,FSTYPE,MOUNTPOINT     # 确认 / 就是单个 200G 分区，没有独立 /boot、/home
df -T /                                  # 确认 / 是 ext4（若为 btrfs/xfs，本方案需调整）
dpkg -l lvm2 | tail -1                   # 必须已安装 lvm2，没有就先 sudo apt install lvm2
grep -n . /etc/fstab                     # 记下根分区与 swap 的写法
sudo blkid | grep -i swap; swapon --show # 确认 swap 是文件而不是分区

# 顺手存一份当前状态，排错时非常有用
lsblk -o NAME,SIZE,FSTYPE,UUID,MOUNTPOINT > ~/disk-before.txt
sudo blkid >> ~/disk-before.txt; cp /etc/fstab ~/fstab.before
```

同时做两件事：**备份不可再生的数据**，以及**准备一个 ≥ 8 GB 的 U 盘**（详见第 9 节 FAQ 的前两问：为什么要做、怎么做、版本怎么选）。

- 备份：项目/代码、`~/.ssh`、`~/.gnupg`、关键配置、文档，以及训练脚本与数据清单；不要备份到同一块物理盘上。
- U 盘：用与本机 Linux 同版本或更新的桌面 ISO（本机是 26.x 就写 26.04 的 ISO，**不要用旧的 22.04.5**）；用 Rufus/balenaEtcher 写入，U 盘数据会被清空。
- 笔记本插上电源。

### 第 1 步：Windows 侧缩小 D，腾出 300 GiB（需要管理员权限）

图形界面：`Win+X` → 磁盘管理 → 右键 D: → 压缩卷 → 输入 `307200`（MB，等于 300 GiB）→ 压缩。

命令行（管理员 PowerShell / cmd）：

```
diskpart
select volume D
shrink desired=307200
```

预期结果：D 变成 **51.46 GiB**，右侧出现 **300 GiB 未分配空间**。

若可压缩量不足 300 GiB，先整理再重试：

```powershell
Optimize-Volume -DriveLetter D -Verbose
vssadmin delete shadows /for=D: /all      # 如果 D 上存在卷影副本
```

> 本步可逆：未分配空间紧贴 D 右侧，随时可以用「扩展卷」还给它。

### 第 2 步：U 盘启动，在未分配空间上建一个 LVM 分区

完全关机（用「关机」，不要用「重启」，避免快速启动把 NTFS 卷锁在休眠态）→ 开机按 F12 选 U 盘启动 → 选择 **Try Ubuntu**（不要点 Install）→ 打开终端。

```bash
lsblk -o NAME,SIZE,FSTYPE,MOUNTPOINT
sudo umount /dev/nvme0n1p6 2>/dev/null        # 万一旧根被 live 会话自动挂载

# 读出那块 300 GiB 空闲区间的精确起止（MiB）
sudo parted /dev/nvme0n1 unit MiB print free
#   预期：起点约 462762 MiB，终点约 769961 MiB，紧贴 p6 左侧

sudo parted /dev/nvme0n1 unit MiB mkpart primary <起点MiB> <终点MiB>
sudo parted /dev/nvme0n1 print                 # 确认新分区号为 7、大小约 299.99 GiB
sudo parted /dev/nvme0n1 set 7 lvm on          # 类型标记为 Linux LVM
sudo partprobe /dev/nvme0n1
```

### 第 3 步：建 PV / VG / LV 并格式化（仍在 live 会话）

```bash
sudo pvcreate /dev/nvme0n1p7
sudo vgcreate vgubuntu /dev/nvme0n1p7
sudo lvcreate -l 100%FREE -n root vgubuntu      # 想给 LVM 快照留余量可改成 95%FREE
sudo mkfs.ext4 -m 1 -L ubuntu-root /dev/vgubuntu/root

sudo blkid -s UUID -o value /dev/vgubuntu/root   # 记下新 UUID，第 5 步要用
```

### 第 4 步：把旧根复制进新 LV

```bash
sudo e2fsck -f /dev/nvme0n1p6
sudo mkdir -p /mnt/old /mnt/new
sudo mount -o ro /dev/nvme0n1p6 /mnt/old
sudo mount /dev/vgubuntu/root /mnt/new

sudo rsync -aHAXSx --numeric-ids --info=progress2 /mnt/old/ /mnt/new/

# 复检：dry-run 应该报告没有任何需要同步的内容
sudo rsync -aHAXSnx --delete --numeric-ids /mnt/old/ /mnt/new/
# 用量比对：两个数字应接近
du -sx /mnt/old /mnt/new
```

> 如果 live 环境里没有 `rsync`：`sudo apt install -y rsync`（需联网），或直接改用 `sudo cp -a /mnt/old/. /mnt/new/`。

### 第 5 步：改 fstab、重建 initramfs、重装 GRUB（chroot）

```bash
sudo nano /mnt/new/etc/fstab
#   把 / 那一行的 UUID= 换成第 3 步记下的新 UUID，例如：
#   UUID=<新UUID>  /  ext4  errors=remount-ro  0  1

# 确认没有别的地方还在引用旧根 UUID（resume、crypttab 等）
OLD=$(sudo blkid -s UUID -o value /dev/nvme0n1p6)
sudo grep -rn "$OLD" /mnt/new/etc /mnt/new/boot 2>/dev/null

# 绑定必要目录并进入 chroot
for d in dev dev/pts proc sys run; do sudo mount --bind /$d /mnt/new/$d; done
sudo mount /dev/nvme0n1p1 /mnt/new/boot/efi
sudo chroot /mnt/new
```

chroot 内部：

```bash
dpkg -l lvm2 | tail -1          # 必须已安装，否则 apt install -y lvm2
update-initramfs -u -k all      # 让 initramfs 能在启动时激活 LVM
grub-install /dev/nvme0n1
update-grub
grep -n 'linux.*root=' /boot/grub/grub.cfg | head
exit
```

退出后卸载干净：

```bash
sudo umount /mnt/new/boot/efi
for d in dev/pts dev proc sys run; do sudo umount /mnt/new/$d; done
sudo umount /mnt/new /mnt/old
sync
```

### 第 6 步：重启验证（此时根分区约 300 GiB）

拔掉 U 盘，重启进 Linux：

```bash
findmnt /                                    # 应该是 /dev/mapper/vgubuntu-root
lsblk -o NAME,SIZE,FSTYPE,MOUNTPOINT
df -h /
sudo pvs; sudo vgs; sudo lvs
```

确认能正常启动、正常进桌面、再重启一次也正常，**才能**做第 7 步。

### 第 7 步：并入旧分区，根分区扩到约 500 GiB（在新系统里在线完成）

```bash
findmnt /dev/nvme0n1p6          # 必须没有输出，确认旧分区没被使用
sudo wipefs -a /dev/nvme0n1p6   # 不可逆的那一下，务必在第 6 步验证通过之后再执行
sudo pvcreate /dev/nvme0n1p6
sudo vgextend vgubuntu /dev/nvme0n1p6
sudo lvextend -l +100%FREE /dev/vgubuntu/root
sudo resize2fs /dev/vgubuntu/root     # 对挂载中的 ext4 扩容是受支持的，几秒完成
df -h /
```

可选（只让类型标记更好看，不影响功能）：

```bash
sudo sgdisk -t 6:8e00 /dev/nvme0n1    # 8e00 = Linux LVM
```

到这一步，`df -h /` 应该在 496 GiB 左右。

---

## 7. 以后继续给 Ubuntu 加空间

这是选 LVM 的主要收益：**新空间来自哪里都行，不需要和 Ubuntu 分区相邻，也不需要搬任何数据。**

1）Windows（管理员）缩 C 或 D，并在空出来的地方建一个**不格式化**的分区：

```
diskpart
select volume C            （或 select volume D）
shrink desired=102400      （MB；102400 = 100 GiB）
select disk 0
create partition primary
```

如果 Windows 弹出「需要格式化」的提示，**点取消**（我们只要一块裸分区）。

2）在 Linux 里把它并入卷组并在线扩容：

```bash
lsblk -o NAME,SIZE,TYPE                    # 确认新分区号，例如 nvme0n1p8
sudo pvcreate /dev/nvme0n1pN
sudo vgextend vgubuntu /dev/nvme0n1pN
sudo lvextend -l +100%FREE /dev/vgubuntu/root
sudo resize2fs /dev/vgubuntu/root
df -h /
```

不用重启、不用 U 盘、不用重装 GRUB。按现在的布局，C 还能再让出约 150 GiB、D 还有约 25 GiB 余量，也就是说根分区将来可以继续长到约 670 GiB 而不需要任何搬迁。

---

## 8. 风险、回滚点与验证清单

**回滚点**

- 第 1 步之后：未分配空间紧贴 D，磁盘管理里「扩展卷」即可还原。
- 第 6 步之前：旧根 p6 一直完好。即使 chroot 阶段把 fstab / GRUB 改坏，用 live 盘把根 UUID 改回旧值就能启动回去。
- **第 7 步的 `wipefs` 是唯一的不可逆操作**，只在第 6 步验证通过后才执行。

**主要风险**

| 风险 | 影响 | 处置 |
|---|---|---|
| `rsync` 期间断电 | 复制不完整 | 旧根未动，重做第 4 步 |
| fstab / GRUB 写错 | 系统起不来 | live 盘启动修复（旧根仍在，可回退） |
| 忘记 `update-initramfs` 或系统没装 `lvm2` | 启动时找不到根卷组 | live 盘启动，chroot 里装 lvm2 并重建 initramfs |
| live 盘版本过旧 | `e2fsck` 报 Unsupported feature，进不了第 4 步 | 改用与本机同版本或更新的 live 盘 |
| 单盘无冗余 | 任何误操作都可能影响 Windows | 动手前备份重要数据 |

**验证清单**

- [ ] 第 0 步：确认 `/` 是单个 ext4 分区、`lvm2` 已安装、fstab 用 UUID
- [ ] 第 1 步：D 变成 51.46 GiB，右侧出现 300 GiB 未分配
- [ ] 第 2 步：新分区创建成功、类型为 LVM、大小约 299.99 GiB
- [ ] 第 4 步：`rsync` dry-run 无差异、`du` 用量一致
- [ ] 第 5 步：根 UUID 已替换、initramfs 已重建、GRUB 已重装、无旧 UUID 残留引用
- [ ] 第 6 步：`findmnt /` 指向 `vgubuntu-root`，能正常重启两次
- [ ] 第 7 步：`df -h /` 约 496 GiB 可用

---

## 9. 常见问题

### Q1 第 0 步的「备份」是干什么的？怎么做？

**为什么**：这次操作会改 `/etc/fstab` 和 GRUB、把 300 GiB 变成 LVM PV，最后还会 `wipefs` 掉旧的 200 GiB 分区。方案设计上旧根在验证通过前一直完好，但仍然要备份，原因是：单块盘没有任何冗余；一旦第 5 步把引导配置改坏、系统进不去，你需要在「一边修引导」的同时确信数据还有另一份。

**怎么做**（在现有 Linux 里，把外接盘挂到 `/media/<user>/Backup` 之类的位置）：

```bash
# 只备份不可再生的东西，不要整盘拷（同一块物理盘上的备份没有意义）
rsync -aHAX --info=progress2 ~/work ~/.ssh ~/.gnupg ~/Documents \
      /media/$USER/Backup/backup-$(date +%F)/

# 空间紧张时先打包
tar --acls --xattrs -czf /media/$USER/Backup/work-$(date +%F).tar.gz ~/work
```

另外把第 0 步生成的 `~/disk-before.txt`、`~/fstab.before` 一起拷走。如果手边没有外接盘，退而求其次：先把关键代码推到远端仓库、把资料传到网盘，再动手。

### Q2 第 0 步的「准备 U 盘」是干什么的？为什么必须要有？

**为什么**：有三件事没法在运行中的系统里做，而且它们都必须在第 2～5 步完成：

1. 新建分区（不能随便改正在使用的磁盘的分区表）；
2. 把旧根**完整、一致**地复制到新 LV（运行中的系统文件一直在变，拷贝可能不一致）；
3. 万一第 5 步把 GRUB / fstab 改坏，需要一个外部环境把系统修回来——没有 live 盘就没有救援手段。

LVM 的 `pvcreate` / `vgcreate` / `lvcreate` 也在 live 会话里执行。

**怎么做**：

1. 准备 ≥ 8 GB U 盘（里面的数据会被清空）；
2. 用 Rufus 或 balenaEtcher 写入与本机 Linux **同版本或更新**的桌面 ISO；分区方案选 GPT、目标系统选 UEFI；
3. 完全关机（不要「重启」）→ 开机按 F12 → 选 U 盘 → 选择 **Try Ubuntu**（不要点 Install）；
4. 打开终端，先用 `lsblk` 确认能正常看到这块 NVMe。

### Q3 用 Ubuntu 22.04 的 live 盘，会不会和本机的 26 冲突？

**不会冲突的部分**：live 盘默认只在内存里运行，不会安装第二个系统、不会动 Windows、不会覆盖你现有的 Linux；第 5 步 chroot 里执行的是**你系统自带的** `update-initramfs` 和 `grub-install`，不是 live 盘里的版本，所以也不会把旧版工具写进系统。

**但有工具链风险，这次不建议用 22.04 的盘**：新版 e2fsprogs（Ubuntu 24.04 起）给新建 ext4 默认打开了 `orphan_file` 之类的新特性，旧版 `e2fsck` / `resize2fs` 会直接报 `Unsupported feature` 拒绝操作，旧内核也可能挂不上。本方案第 4 步正好要读那个 200 GiB 的根分区，用 22.04 的 live 盘很可能就卡在这一步。

**结论**：

- 本机是 26.x → 请写 **26.04 的桌面 ISO**（`cat /etc/os-release` 确认版本），不要用 Downloads 里那个 22.04.5。
- 备选：最新版 GParted Live（内核与工具都新，但没有 `rsync`，第 4 步改用 `cp -a`）。
- 反过来，如果确认本机就是 22.04，那个 22.04.5 的 ISO 完全够用。

### Q4 会不会影响 Windows？要不要重装系统？

不动 C:、EFI、MSR、WinRE，D 只是缩小；Windows 侧唯一的改动是分区表。GRUB 重装只改 ESP 里的 Linux 引导文件。方案全程复制而非移动，两边系统都不需要重装。

### Q5 中途断电怎么办？

- 第 4 步复制期间断电：旧根没动，重做即可；
- 第 7 步 `wipefs` 之后断电：新根已经能启动，重新执行 `pvcreate` / `vgextend` / `lvextend` / `resize2fs` 即可。

### Q6 需要联网吗？

用同版本桌面 ISO 时不需要（live 会话自带 `parted`、`rsync`、`lvm2`）；只有补装工具或改用 GParted Live 时才需要。

---

## 10. 附录：参数速查

| 项目 | 值 |
|---|---|
| 目标：Ubuntu 根分区 | 200 GiB → **约 499.9 GiB**（+300 GiB） |
| Windows 缩 D 的目标 | `shrink desired=307200`（MB） |
| 空闲区起点（= D 新末址） | 约 462762 MiB |
| 空闲区终点（= 原 p6 起点） | 约 769961 MiB |
| 新分区号 | p7（物理位置在 p6 左侧） |
| 卷组 / 逻辑卷 | `vgubuntu` / `root`，设备为 `/dev/mapper/vgubuntu-root` |
| 新文件系统 | `mkfs.ext4 -m 1 -L ubuntu-root` |
| WinRE | 保持不动（盘尾 1.95 GiB） |
| 单位换算 | 1 GiB = 1024 MiB；磁盘管理 / diskpart 里的「MB」实际是 MiB |

**没有 U 盘时的替代路径（风险更高，不推荐）**：分区改用 Windows 的 `diskpart create partition primary` 创建，`pvcreate` 之后的步骤全部在运行中的 Linux 里完成。缺点：`rsync` 拷贝运行中的系统一致性不保证，且一旦 GRUB 起不来就没有救援环境。
