# Day 02 学习记录

日期：2026-07-03

## 今日目标

- 启动 WSL2 Ubuntu-22.04。
- 理解 Linux 和 Windows 路径的区别。
- 掌握最基础的 Linux 文件/目录命令。
- 在 Ubuntu 中完成一次小练习：创建目录、创建文件、查看内容、搜索文本、移动/复制文件。
- 形成第一份 Linux 命令记录。

## 今日 7 小时计划

### 第 1 段：启动 Ubuntu 与理解路径（1 小时）

任务：

- 打开 Windows Terminal 或 PowerShell。
- 执行 `wsl` 进入 Ubuntu。
- 查看当前目录。
- 理解 Linux 家目录和 Windows 目录的区别。

需要执行：

```bash
pwd
whoami
ls
cd ~
pwd
ls /mnt/c/Users/86137/Desktop
```

需要理解：

- `~` 是 Linux 当前用户的家目录。
- `/mnt/c/...` 是 WSL 里访问 Windows C 盘的路径。
- Windows 路径 `C:\Users\86137\Desktop` 在 WSL 里对应 `/mnt/c/Users/86137/Desktop`。

产出：

- 能说清楚当前 Ubuntu 家目录在哪里。
- 能找到 Windows 桌面目录。

### 第 2 段：目录和文件基础命令（1.5 小时）

任务：

- 学会查看目录、进入目录、创建目录、创建文件。

需要执行：

```bash
mkdir -p ~/model-deploy-day02
cd ~/model-deploy-day02
pwd
touch notes.txt
ls
echo "Day 02 Linux practice" > notes.txt
cat notes.txt
```

需要理解：

- `mkdir`：创建目录。
- `cd`：进入目录。
- `pwd`：显示当前路径。
- `touch`：创建空文件。
- `echo > file`：把文本写入文件。
- `cat`：查看文件内容。

产出：

- 创建 `~/model-deploy-day02/notes.txt`。

### 第 3 段：复制、移动、删除的安全练习（1 小时）

任务：

- 在练习目录中练习 `cp`、`mv`、`rm`。
- 只在 `~/model-deploy-day02` 目录中操作。

需要执行：

```bash
cp notes.txt notes_backup.txt
ls
mv notes_backup.txt linux_notes_backup.txt
ls
rm linux_notes_backup.txt
ls
```

安全提醒：

- 今天只删除自己创建的练习文件。
- 暂时不要使用 `rm -rf`。
- 不要在 `/`、`/mnt/c` 或系统目录里随便删除东西。

产出：

- 能说清 `cp`、`mv`、`rm` 分别做什么。

### 第 4 段：查看文件与搜索文本（1 小时）

任务：

- 学会用 `cat`、`less`、`grep` 查看和搜索内容。

需要执行：

```bash
echo "linux is important for model deployment" >> notes.txt
echo "onnx and tensorrt need environment management" >> notes.txt
cat notes.txt
grep "linux" notes.txt
grep "tensorrt" notes.txt
```

可选：

```bash
less notes.txt
```

退出 `less`：按 `q`。

产出：

- 能用 `grep` 在文件中搜索关键词。

### 第 5 段：进程、网络和系统信息初见（1 小时）

任务：

- 了解几个常见系统信息命令。

需要执行：

```bash
uname -a
df -h
free -h
ps aux | head
ip addr
```

需要理解：

- `uname -a`：查看系统内核信息。
- `df -h`：查看磁盘空间。
- `free -h`：查看内存。
- `ps aux`：查看进程。
- `ip addr`：查看网络信息。

产出：

- 记录 Ubuntu 系统信息。

### 第 6 段：整理今日命令清单（1 小时）

任务：

- 把今天用过的命令按功能分类。
- 在本日志的“动手记录”中补充关键输出。
- 把不理解的命令列到“遇到的问题”。

需要整理：

- 路径命令：`pwd`、`cd`、`ls`
- 文件命令：`touch`、`cat`、`echo`
- 目录命令：`mkdir`
- 文件操作：`cp`、`mv`、`rm`
- 搜索命令：`grep`
- 系统信息：`uname`、`df`、`free`、`ps`、`ip`

### 第 7 段：复盘与提交（0.5 小时）

任务：

- 把今天完成情况发给 Codex。
- Codex 更新本日志和 README 进度索引。
- 提交并推送到 GitHub。

## 动手记录

- `pwd` 输出：`/home/nefelibata`。
- `whoami` 输出：`nefelibata`。
- Ubuntu 家目录：`/home/nefelibata`，也可以写作 `~`。
- Windows 用户目录在 WSL 中的路径：`/mnt/c/Users/86137`。
- Windows 桌面在 WSL 中的路径：`/mnt/c/Users/86137/Desktop`。
- 练习目录：`/home/nefelibata/model-deploy-day02`。
- `notes.txt` 内容：

```text
Day 02 Linux practice
23213133
onnx and tensorrt need environment management
linux is important for model deployment
```

- `grep "linux" notes.txt` 输出：

```text
linux is important for model deployment
```

- `grep "tensorrt" notes.txt` 输出：

```text
onnx and tensorrt need environment management
```

- `uname -a` 输出：

```text
Linux Nefelibata 6.6.87.2-microsoft-standard-WSL2 #1 SMP PREEMPT_DYNAMIC Thu Jun  5 18:30:46 UTC 2025 x86_64 x86_64 x86_64 GNU/Linux
```

- `df -h` 关键输出：
  - Linux 根目录 `/`：`/dev/sdd`，总大小约 `1007G`，已用 `19G`。
  - Windows C 盘挂载点：`/mnt/c`，总大小约 `933G`，已用 `815G`，剩余 `119G`。
- `free -h` 输出：

```text
Mem: 11Gi total, 634Mi used, 10Gi free, 10Gi available
Swap: 8.0Gi total, 0B used, 8.0Gi free
```

- `ps aux | head`：成功查看进程列表，看到 `/sbin/init`、`systemd-journald`、`cron` 等进程。
- `ip addr` 关键输出：
  - `lo`：本地回环地址 `127.0.0.1`。
  - `eth2`：当前可用网卡，IPv4 地址 `192.168.3.44/24`。
- 今天最不理解的命令：`less notes.txt` 已理解，作用是用分页查看器打开文件，按 `q` 退出。

## 今日完成情况

- 已成功进入 WSL2 Ubuntu-22.04。
- 已理解 Ubuntu 家目录 `~` 和 Windows 目录 `/mnt/c/...` 的区别。
- 已在 `/home/nefelibata/model-deploy-day02` 中完成文件和目录练习。
- 已掌握 `mkdir`、`cd`、`pwd`、`touch`、`ls`、`echo`、`cat` 的基础用法。
- 已完成 `cp`、`mv`、`rm` 的安全练习。
- 已理解 `>` 是覆盖写入，`>>` 是追加写入。
- 已学会用 `grep` 搜索文件中的关键词。
- 已初步了解 `uname`、`df`、`free`、`ps`、`ip` 这些系统信息命令。

## 遇到的问题

- 一开始 `grep "linux" notes.txt` 没有输出，因为文件中还没有包含 `linux` 的行。
- `mv notes_backup.txt linux_notes_backup.txt` 容易误以为只是移动文件，今天理解了它也可以用于重命名。
- `less notes.txt` 和 `cat notes.txt` 的区别需要记住：短文件用 `cat`，长文件用 `less`。

## 今日复盘

今天最重要的收获：

- Linux 不是只背命令，核心是理解“我在哪个目录、我要操作哪个文件、命令会改变什么”。
- WSL2 里的 Linux 家目录适合做学习和开发练习，Windows 文件可以通过 `/mnt/c/...` 访问。
- 删除命令要谨慎，今天只在练习目录里使用 `rm` 是正确做法。

还不清楚的点：

- Linux 权限、`sudo`、`apt` 包管理还没有系统学习。
- 后续需要学习如何在 WSL2 里创建 Python 3.10 项目环境。

## 明日计划

- 继续 Linux：权限、环境变量、包管理和常见文本编辑。
- 学会 `chmod`、`chown`、`sudo`、`apt`、`nano` 或 `vim` 的基础用法。
- 开始整理 Python 3.10 环境方案。
