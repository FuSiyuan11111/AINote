# IntelliJ IDEA 项目 UTF-8 正确配置指南（Windows 中文系统）

## 为什么会乱码

乱码的本质：**写出一端和读入一端的编码不一致**。Windows 中文系统默认编码是 GBK，而现代工具链默认 UTF-8，任何一环没对齐就会乱码。完整的编码链条有四处，必须全部是 UTF-8：

```
源文件存储编码 → 编译器读源码的编码 → JVM 输出编码 → 控制台解码编码
```

## 第一步：IDEA 文件编码设置（每个新项目必做）

`Ctrl + Alt + S` → **Editor → File Encodings**，右侧三个下拉框全部改成 UTF-8：

- Global Encoding（全局编码）
- Project Encoding（项目编码）
- Default encoding for properties files

## 第二步：编译器编码

`Ctrl + Alt + S` → **Build, Execution, Deployment → Compiler → Java Compiler**
→ **Additional command line parameters** 填入：

```
-encoding UTF-8
```

否则源文件里的中文字符串可能被编译器按 GBK 读取，乱码被"烤"进 class 文件（这种情况必须 Rebuild Project 才能修复）。

## 第三步：运行时 VM 参数（控制台乱码的关键，最容易被漏）

**Run → Edit Configurations** → 选中实际运行的配置 → **VM options** 填入：

```
-Dfile.encoding=UTF-8 -Dsun.stdout.encoding=UTF-8 -Dsun.stderr.encoding=UTF-8
```

> **原理（大坑）**：JDK 18+ 虽然把默认编码改成了 UTF-8，但那只是 `file.encoding`。
> `System.out` 往控制台输出用的是**独立的 `stdout.encoding`**，在中文 Windows 上依然默认 GBK——
> 所以只加 `-Dfile.encoding=UTF-8` 不够，必须把 `sun.stdout.encoding` 和 `sun.stderr.encoding` 也显式指定。

注意事项：

- 改的必须是**右上角下拉框里实际选中的那个配置**；习惯性点行首绿色三角可能用到不带参数的临时配置
- 固定用右上角的运行按钮，保证用的就是改过的配置
- 验证方法：`System.out.println("中文测试")` 输出正常即全链路通畅

## 第四步：已经乱码的文件怎么抢救

编辑器里显示乱码的文件（文件字节本身还是好的）：

1. 点 IDEA 窗口**右下角状态栏的编码名**（如 `UTF-8`）
2. 选 **Reload in GBK** —— 换编码重新读取，不改文件内容；不对就换 GB18030 再试
3. 显示恢复正常后，再点右下角 → **Convert to UTF-8** —— 真正转换落盘

> **Reload** = 换个编码"看"，不改文件；**Convert** = 转换并保存。
> 顺序永远是先 Reload 试到正常、再 Convert。试错阶段绝不点 Convert，否则乱码会被永久写入。

## 判断乱码方向的技巧

| 乱码长相 | 含义 | 修哪里 |
|---|---|---|
| `涓枃` 这样的汉字 | UTF-8 字节被按 GBK 解码 | 读入端（IDEA 编码设置） |
| `��` 混着零散字母（如 `С`、`ֵ`） | GBK 字节被按 UTF-8 解码 | 输出端（第三步的 VM 参数没做，最常见） |
| `???` 问号 | 输出端编码不支持该字符 | 输出端 |

## 可选：系统级一劳永逸方案

设置 → 时间和语言 → 语言和区域 → 管理语言设置 → 更改系统区域设置 →
勾选 **"Beta 版：使用 Unicode UTF-8 提供全球语言支持"** → 重启电脑。

整个系统改用 UTF-8，连 native DLL（如虹软 SDK）直接输出的中文都能解决，VM 参数都不用配。
缺点：少数老旧国产软件/游戏可能反过来出现乱码，玩老游戏多的话慎选。

## 附：新项目防坑总清单

1. 新建项目用 **File → New → Project**，不要裸开文件夹（会缺模块配置，引出一串问题）
2. 下载别人的项目看到 `pom.xml` / `build.gradle`，用 Maven/Gradle 方式打开，配置全自动
3. 配好项目 SDK（推荐 JDK 21）
4. 做本文第一 ~ 三步
5. 跑一个 `System.out.println("中文测试")` 验证
