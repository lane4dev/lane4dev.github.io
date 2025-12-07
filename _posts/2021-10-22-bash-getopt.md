---
layout: post
title: "Bash getopt 模板备忘"
date: 2021-10-22 18:14:00 +0800
categories: Code
tags: ["Bash"]
comments: true
---

这段脚本是一个 “用 getopt 解析短/长选项” 的通用模板，适合在写 Bash 命令行工具时直接复制改造。

先贴原始脚本，方便以后直接用：

```bash
OPTS=$(getopt --options abg:h --long alpha,beta,gamma:,help -- "$@")

if [[ $? -ne 0 ]]; then
    echo "Exit with no options"
    exit 1;
fi

eval set -- "$OPTS"

while [ : ]; do
    case "$1" in
        -a | --alpha)
            echo "Processing 'alpha' option"
            shift
            ;;
        -b | --beta)
            echo "Processing 'beta' option"
            shift
            ;;
        -g | --gemma)
            echo "Processing 'gamma' option. Input argument is '$2'"
            shift 2
            ;;
        -h | --help)
            echo "Help Command"
            shift
            ;;
        --) shift;
            break
            ;;
        *)
            echo "Unexpected option: $1"
            ;;
    esac
done
```


## 功能概览

支持的选项（短 + 长）：

- `-a` / `--alpha`：无参数
- `-b` / `--beta`：无参数
- `-g` / `--gamma <value>`：需要一个参数
- `-h` / `--help`：显示帮助

每个选项在 `case` 里只做一件事：打印一行提示，可按需换成真正逻辑。


## getopt 调用

```bash
OPTS=$(getopt --options abg:h --long alpha,beta,gamma:,help -- "$@")
```

- `--options abg:h`

  - `a`、`b`：无参数
  - `g:`：后面要跟一个参数
  - `h`：无参数

- `--long alpha,beta,gamma:,help`

  - `gamma:` 表示 `--gamma` 也要带一个值

`if [[ $? -ne 0 ]]; then ... fi` 用来在 `getopt` 失败（非法参数）时提前退出。


## 把解析结果「塞回」参数列表

```bash
eval set -- "$OPTS"
```

这一步很关键：

- `getopt` 会把参数整理成标准形式，例如：
  `-a -g foo --beta` → `-a -g 'foo' --beta --`
- `eval set -- "$OPTS"` 把整理好的参数重新写回 `$1 $2 ...`，方便后面用 `while ... case` 统一处理。
- `--` 是分隔符，表示「选项结束，后面都是普通参数」。


## while + case 解析循环

```bash
while [ : ]; do
    case "$1" in
        -a | --alpha)
            echo "Processing 'alpha' option"
            shift
            ;;
        -b | --beta)
            echo "Processing 'beta' option"
            shift
            ;;
        -g | --gamma)
            echo "Processing 'gamma' option. Input argument is '$2'"
            shift 2
            ;;
        -h | --help)
            echo "Help Command"
            shift
            ;;
        --)
            shift
            break
            ;;
        *)
            echo "Unexpected option: $1"
            shift
            ;;
    esac
done
```

记几个要点：

- `while [ : ]; do ... done`：死循环，直到遇到 `--` 分支 `break`。
- 每个选项分支结束都要 `shift`：

  - 只考虑选项本身：`shift`
  - 考虑选项 + 参数：`shift 2`

- `--)`：遇到 `--` 时退出选项解析循环，后面如果还有参数，就当成普通参数处理。
- `*)`：兜底分支，遇到未预期的选项时给个提示。
