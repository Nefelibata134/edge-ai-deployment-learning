# Day 03 学习记录

日期：2026-07-03

## 今日目标

- 理解 Linux 权限的基本概念：用户、用户组、读写执行权限。
- 学会安全使用 `sudo`，知道什么时候需要管理员权限。
- 学会使用 `apt` 安装、更新、查询软件包。
- 掌握 `nano` 这个简单文本编辑器，能在终端里改配置文件。
- 初步理解环境变量和 `PATH`。
- 为后续 Python 3.10、ONNX、TensorRT、Jetson 环境管理打基础。

## 今日 7 小时计划

### 第 1 段：复习路径与文件命令（0.5 小时）

任务：

- 进入 WSL2 Ubuntu。
- 回到昨天的练习目录，复习基本命令。

需要执行：

```bash
cd ~
pwd
ls
cd ~/model-deploy-day02
ls
cat notes.txt
```

需要理解：

- `~` 是 Ubuntu 当前用户家目录。
- 操作前先确认自己在哪个目录，避免误删或误改文件。

### 第 2 段：理解 Linux 权限（1 小时）

任务：

- 查看文件权限。
- 理解 `r`、`w`、`x` 的含义。

需要执行：

```bash
cd ~/model-deploy-day02
ls -l
touch permission_test.sh
ls -l permission_test.sh
```

需要理解：

- `r`：read，读取。
- `w`：write，写入。
- `x`：execute，执行。
- `-rw-r--r--` 这类格式表示文件所有者、用户组、其他人的权限。

产出：

- 能看懂 `ls -l` 输出的前 10 个字符大概是什么意思。

### 第 3 段：练习 `chmod` 和脚本执行（1 小时）

任务：

- 创建一个简单脚本。
- 给脚本添加执行权限。
- 执行脚本。

需要执行：

```bash
cd ~/model-deploy-day02
nano permission_test.sh
```

在 `nano` 中写入：

```bash
#!/usr/bin/env bash
echo "Hello Linux permission"
```

保存方式：

- `Ctrl + O`：保存。
- 回车确认文件名。
- `Ctrl + X`：退出。

继续执行：

```bash
cat permission_test.sh
./permission_test.sh
chmod +x permission_test.sh
ls -l permission_test.sh
./permission_test.sh
```

需要理解：

- 没有执行权限时，`./permission_test.sh` 可能会提示权限不足。
- `chmod +x` 是给文件添加执行权限。
- `./xxx.sh` 表示执行当前目录下的脚本。

### 第 4 段：理解 `sudo` 和系统目录（1 小时）

任务：

- 理解普通用户和管理员权限的区别。
- 知道 `sudo` 的使用边界。

需要执行：

```bash
whoami
id
sudo whoami
ls /root
sudo ls /root
```

需要理解：

- 普通用户是 `nefelibata`。
- `sudo` 临时用管理员权限执行命令。
- 不是所有命令都要加 `sudo`。
- 不要随便用 `sudo rm`、`sudo rm -rf`。

产出：

- 能说清楚为什么 `sudo whoami` 会输出 `root`。

### 第 5 段：学习 `apt` 包管理（1.5 小时）

任务：

- 学会查询软件是否安装。
- 学会更新软件源列表。
- 安装一个小工具。

需要执行：

```bash
which tree
tree --version
sudo apt update
sudo apt install -y tree
tree --version
tree ~/model-deploy-day02
```

如果 `tree` 已经安装，也正常。

需要理解：

- `which`：查命令位置。
- `apt update`：更新软件包索引，不是升级系统。
- `apt install`：安装软件。
- `tree`：用树状结构查看目录。

安全提醒：

- 安装软件前先知道它是做什么的。
- 不要复制网上看不懂的安装脚本直接执行。

### 第 6 段：环境变量和 `PATH` 初见（1 小时）

任务：

- 查看常见环境变量。
- 理解为什么输入 `python`、`git`、`code` 这些命令系统能找到程序。

需要执行：

```bash
echo $HOME
echo $PATH
which python
which python3
python --version
python3 --version
which pip
which pip3
```

需要理解：

- `$HOME` 是当前用户家目录。
- `$PATH` 是系统寻找命令的位置列表。
- 后续配置 CUDA、TensorRT、Python 环境时，经常会遇到环境变量。

产出：

- 记录 `python` 和 `python3` 分别指向哪里、版本是多少。

### 第 7 段：整理 Day 03 输出并复盘（1 小时）

任务：

- 把关键命令输出发给 Codex。
- 记录今天学会的命令。
- 总结今天最容易混淆的点。

需要整理：

- `ls -l permission_test.sh` 的输出。
- `./permission_test.sh` 加权限前后的表现。
- `sudo whoami` 的输出。
- `tree ~/model-deploy-day02` 的输出。
- `python --version` 和 `python3 --version` 的输出。

## 动手记录

- Ubuntu 家目录：`/home/nefelibata`。
- Day 02 练习目录：`/home/nefelibata/model-deploy-day02`。
- `notes.txt` 内容仍然保留：

```text
Day 02 Linux practice
23213133
onnx and tensorrt need environment management
linux is important for model deployment
```

- `ls -l` 初始输出：

```text
total 4
-rw-r--r-- 1 nefelibata nefelibata 117 Jul  3 10:46 notes.txt
```

- 创建脚本前误创建了一个拼写错误的空文件 `permissipn_test.sh`，后续已删除。
- 用 `nano permission_test.sh` 写入脚本：

```bash
#!/usr/bin/env bash
echo "Hello Linux permission"
```

- 脚本没有执行权限时直接运行：

```text
-bash: ./permission_test.sh: Permission denied
```

- 添加执行权限后查看权限：

```text
-rwxr-xr-x 1 nefelibata nefelibata 50 Jul  3 14:47 permission_test.sh
```

- 添加执行权限后运行成功：

```text
Hello Linux permission
```

- 用户和权限相关输出：

```text
whoami -> nefelibata
sudo whoami -> root
ls /root -> Permission denied
sudo ls /root -> 无报错
```

- `id` 输出显示当前用户属于 `sudo` 等用户组：

```text
uid=1000(nefelibata) gid=1000(nefelibata) groups=1000(nefelibata),4(adm),24(cdrom),27(sudo),30(dip),46(plugdev)
```

- 安装 `tree` 前：

```text
which tree -> 无输出
tree --version -> Command 'tree' not found
```

- `sudo apt update` 成功，使用清华 Ubuntu 源和 ROS2 源，关键结果：

```text
Fetched 11.4 MB in 3s
Reading package lists... Done
43 packages can be upgraded.
```

- `sudo apt install -y tree` 成功安装 `tree`：

```text
Setting up tree (2.0.2-1) ...
```

- `tree --version`：

```text
tree v2.0.2
```

- 清理拼写错误文件后，练习目录结构：

```text
/home/nefelibata/model-deploy-day02
├── notes.txt
└── permission_test.sh

0 directories, 2 files
```

- 环境变量与 Python 记录：

```text
echo $HOME -> /home/nefelibata
which python -> /home/nefelibata/miniconda3/bin/python
which python3 -> /home/nefelibata/miniconda3/bin/python3
python --version -> Python 3.13.12
python3 --version -> Python 3.13.12
which pip -> /home/nefelibata/miniconda3/bin/pip
which pip3 -> /home/nefelibata/miniconda3/bin/pip3
pip --version -> pip 26.0.1 from /home/nefelibata/miniconda3/lib/python3.13/site-packages/pip (python 3.13)
pip3 --version -> pip 26.0.1 from /home/nefelibata/miniconda3/lib/python3.13/site-packages/pip (python 3.13)
```

- 当前终端提示符带有 `(base)`，说明 WSL 中默认进入了 Miniconda 的 base 环境。

## 今日完成情况

- 已复习 `cd`、`pwd`、`ls`、`cat` 等基础命令。
- 已理解 `ls -l` 中 `-rw-r--r--`、`-rwxr-xr-x` 这类权限格式的基本含义。
- 已用 `nano` 创建并编辑 `permission_test.sh`。
- 已验证没有执行权限时脚本会报 `Permission denied`。
- 已用 `chmod +x` 给脚本添加执行权限，并成功运行脚本。
- 已理解 `whoami` 和 `sudo whoami` 的区别。
- 已验证普通用户无法直接查看 `/root`，但可以通过 `sudo` 临时获得管理员权限。
- 已完成 `apt update` 和 `apt install`，成功安装 `tree`。
- 已用 `tree` 查看练习目录结构。
- 已查看 `$HOME`、`$PATH`、`python`、`python3`、`pip`、`pip3` 的位置和版本。
- 已发现当前 WSL 默认 Python 来自 Miniconda base，版本为 Python 3.13.12。

## 遇到的问题

- `touch permissipn_test.sh` 中 `permission` 拼写错误，创建了一个无用空文件；已用 `rm permissipn_test.sh` 删除。
- 一开始执行 `./permission_test.sh` 报 `Permission denied`，原因是脚本没有执行权限；通过 `chmod +x permission_test.sh` 解决。
- 输入了 `whosmi`，系统提示可能想输入 `whoami`；这是普通拼写错误。
- `which tree` 没有输出，说明 `tree` 当时未安装；通过 `sudo apt install -y tree` 解决。
- 当前 Python 是 Conda base 中的 Python 3.13.12，后续深度学习部署不建议直接使用，需要单独创建 Python 3.10 环境。

## 今日复盘

今天最重要的收获：

- Linux 权限不是抽象概念，可以通过“脚本能不能执行”直接感受到。
- `chmod +x` 的作用是给文件增加执行权限，脚本权限从 `-rw-r--r--` 变成 `-rwxr-xr-x` 后才能用 `./permission_test.sh` 运行。
- `sudo` 是临时管理员权限，适合安装软件、查看系统目录等操作，但删除和修改系统文件时必须谨慎。
- `apt update` 是更新软件包索引，`apt install` 才是安装软件。
- `$PATH` 决定系统从哪些目录寻找命令；当前 `python`、`pip` 都来自 Miniconda。

还不清楚的点：

- Conda 的 `base` 环境和普通系统 Python 的关系还需要继续学习。
- `pip`、`pip3`、`conda install` 的区别还需要系统梳理。
- 后续需要弄清楚为什么模型部署更推荐 Python 3.10，而不是直接使用 Python 3.13。

## 明日计划

- 学习 WSL2 中的 Python 环境管理。
- 明确 `conda`、`venv`、`pip` 的区别。
- 创建第一个用于模型部署学习的 Python 3.10 环境。
