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

待完成后补充。

## 今日完成情况

待完成后补充。

## 遇到的问题

待完成后补充。

## 今日复盘

今天最重要的收获：

待完成后补充。

还不清楚的点：

待完成后补充。

## 明日计划

- 学习 WSL2 中的 Python 环境管理。
- 明确 `conda`、`venv`、`pip` 的区别。
- 创建第一个用于模型部署学习的 Python 3.10 环境。
