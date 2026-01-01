
# Linux “含斜杠命令名”陷阱：为何 `sudo /etc/passwd` 能触发当前目录脚本

有没有想过，明明只是把 `/etc/passwd` 设成可执行，再给普通用户一个 `sudo /etc/passwd` 的权限，居然能被人玩出 root shell？更诡异的是，shell 竟然乖乖去执行一个文件名**完全等于 passwd 第一行**、且躺在当前目录的恶意脚本。

## 实操演示录屏

[![asciicast](https://asciinema.org/a/A9khgKn8WnMDwyKuh7Hus5q5a.svg)](https://asciinema.org/a/A9khgKn8WnMDwyKuh7Hus5q5a)

很多人看到后直呼：“这不是当前目录劫持吗？shell 把 passwd 每一行当成路径去执行，但居然优先执行了当前目录的文件！”

**并非如此。** 这和 PATH 劫持没关系，它是 POSIX 对“命令名含斜杠”时的正常执行规则，在特定场景下产生了精确的“命中”效果。

下面用最直观的实验和系统调用跟踪，拆解这个“含斜杠命令名”陷阱的底层原理。

## 极简复现实验

```bash
# 在家目录操作
cd ~

# 1. 创建脚本，放在 /tmp（与运行目录不同）
cat > /tmp/pwn << 'EOF'
abc/ef/g
ddd
EOF
chmod +x /tmp/pwn

# 2. 只在当前目录创建恶意脚本
mkdir -p abc/ef
echo 'echo pwned!' > abc/ef/g
chmod +x abc/ef/g

# 3. 执行脚本（注意：脚本在 /tmp，当前目录是 ~）
strace -f -e execve zsh /tmp/pwn
```

运行结果：
![](https://i-blog.csdnimg.cn/direct/de34c1db45b144da837d7f540ba33c2b.png)


第一行被成功执行并打印 `pwned!`，第二行则没有命中。

换成 zsh 或其他 POSIX shell 也一样：

![](https://i-blog.csdnimg.cn/direct/5caec6189d8b4ec48ff78b1a1efb5506.png)

strace 关键输出：

```
[pid 159661] execve("/bin/sh", ["/bin/sh", "abc/ef/g"], 0x55ce6d08d058 /* 65 vars */) = 0
pwned!

/tmp/pwn:2: command not found: ddd
# 第二行 "ddd" 不含斜杠，走正常 PATH 搜索，一路 ENOENT，最终 command not found
```

明明脚本在 /tmp，行里写的 `abc/ef/g` 也没有 `./` 前缀，为什么 shell 能在当前目录（~）找到并执行我们放的恶意文件？

## 底层过程拆解

### 1. 命令名解析

shell 执行 `/tmp/pwn` 时逐行解析：

第一行 `abc/ef/g`（无空格）→ 被整体视为**命令名**。

**关键点：命令名包含斜杠 /**

### 2. POSIX 标准铁律：含斜杠的命令名直接当作路径执行
所有 POSIX 兼容 shell 都遵守：

- 命令名**包含斜杠** → 视为路径，**直接 execve**，**绝不搜索 $PATH**。
- 命令名**不含斜杠** → 才去 $PATH 搜索。

因此 shell 直接尝试：

```
execve("abc/ef/g", ["abc/ef/g"], env)
```

路径解析起点是**当前工作目录**（运行脚本时所在的 ~）。

### 3. 第一次 execve 失败，返回 ENOEXEC
内核在当前工作目录下查找 `abc/ef/g`：
- 找到了（我们预置的恶意脚本）
- 有执行权限
- 但它是纯文本（无 shebang、无 ELF 头）
→ 无法直接运行 → 返回 **ENOEXEC**

### 4. 标准 fallback 机制触发
POSIX 明确规定：

> 如果 execve 返回 ENOEXEC，shell **必须**将该路径名当作一个 shell 脚本文件，用 shell 自身来解释执行它。

于是 shell 启动新的 shell 实例（这里是 `/bin/sh`），把原命令名作为待解释的脚本文件传给它：

即：

```
execve("/bin/sh", ["sh", "abc/ef/g"], env)
```

意思：请用 sh 解释执行名为 `abc/ef/g` 的脚本文件。

### 5. 解释执行时，依然以当前工作目录为起点
新启动的 sh 接到 `abc/ef/g`（相对路径），仍以**当前工作目录**（~）为起点查找 → 找到 → 读取 → 执行 → 输出 `pwned!`

## 这和经典 PATH 劫持的区别

| 特性                     | 经典 PATH 劫持（. 在 PATH 中）       | 本文现象（相对路径脚本解释）                  |
|--------------------------|-------------------------------------|----------------------------------------------|
| 命令名是否含 /           | 不含（如 `ls`）                     | 必须含 /（如 `abc/ef/g`）                    |
| 是否搜索 $PATH           | 是                                  | 完全不搜索 $PATH                             |
| 触发条件                 | 无路径命令 + PATH 含 .              | 含 / 路径 execve 失败且返回 ENOEXEC          |
| 查找位置                 | $PATH 中所有目录（顺序查找）        | 仅在**当前工作目录**（相对路径起点）         |
| 是否标准行为             | 非标准，安全隐患，常被禁用          | POSIX 明确要求的标准行为                     |
| 防御方式                 | 移除 PATH 中的 .                    | 无法通过 PATH 防御，必须用绝对路径或固定 cd  |

**结论：这和 PATH 劫持完全不同——不搜索 $PATH，只在当前工作目录查找精确匹配的文件名。**

## 实际安全建议

- 这不是漏洞，而是标准特性，无法“修补”。
- 防御只能从脚本编写习惯入手：
  - 关键命令使用**绝对路径**（`/bin/ls` 而不是 `bin/ls`）
  - 脚本开头强制 `cd` 到固定安全目录
  - 避免在脚本中硬编码相对路径命令
  - 在 sudoers 中使用 `NOEXEC` 限制嵌套执行（对 sudo 执行脚本场景有效）

## 总结

`/etc/passwd` 能“命中”当前目录的恶意脚本，不是 shell 在偷偷搜索，而是：

- 含斜杠的命令名被当作**相对路径名**直接 execve
- execve 遇到 ENOEXEC 后，shell 按标准要求用自身解释执行
- 解释执行时，相对路径仍从**当前工作目录**解析

看似诡异，实则是标准行为在边缘场景下的副作用——它不是魔术，只是 POSIX 在认真对待一个含斜杠的文件名而已。
