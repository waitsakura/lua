# AGENTS.md

面向 AI 编码代理的仓库指南。本仓库是 **Lua 官方开发仓库（Lua development code）**，当前版本为 **Lua 5.5.1**（工作版本，见 `lua.h`）。

## 1. 项目性质与最重要的规则

- 这是 PUC-Rio Lua 团队的**主开发仓库**，包含全部提交历史，按官方节奏不定期镜像。
- **不要提交 pull request**（见 `README.md`）。发现问题请发到 [Lua 邮件列表](https://www.lua.org/lua-l.html)。
- 因此在本仓库工作时应**只修改源码并保持最小、可审查的 diff**；不要引入第三方依赖、构建脚本重写或格式化工具。
- Lua 的核心卖点之一是**可移植的纯 ANSI C（C99，且保留 C89 兼容路径）**。任何改动都必须保持这一点：不用编译器扩展、不用平台专有 API（除非已由 `LUA_USE_*` 宏保护）、不依赖库（除 `-lm`，以及 Linux 下 `-ldl`）。

## 2. 目录与文件布局

| 路径 | 说明 |
| --- | --- |
| `l*.c` / `l*.h` | 解释器、编译器、VM、GC、标准库的全部实现（扁平放在根目录） |
| `lua.c` | 独立解释器 `lua` 的 `main` |
| `onelua.c` | **单文件合并构建**入口（将核心 + 库 + 解释器合成一个文件） |
| `luaconf.h` | 配置中心：平台、数值类型、路径、兼容性开关 |
| `lprefix.h` | 必须最先 include 的前缀头 |
| `ltests.c` / `ltests.h` | 内部测试库 + 测试期编译配置（`LUA_DEBUG`、`LUAI_ASSERT`） |
| `makefile` | Linux/POSIX 开发者 makefile（默认 `-DLUA_USE_LINUX`） |
| `testes/` | 官方测试套件（Lua 编写），入口 `all.lua` |
| `testes/libs/` | 测试 C 模块（`require` 相关），有独立 `makefile` 生成 `.so` |
| `manual/` | 参考手册源文件 `manual.of` 与转换脚本 `2html`；`manual.html` 为生成物 |

## 3. 构建

没有 `configure`，没有 CMake，直接用 `make`（POSIX/Linux 默认）。**所有平台相关的量都集中在 `makefile` 顶部的 “USER SETTINGS” 区域**（`MYCFLAGS` / `MYLDFLAGS` / `CC` / `CFLAGS`），修改构建选项就改那里。

```sh
# 默认：构建 liblua.a + 解释器 lua（含 ltests.o，默认关闭断言）
make            # 等价于 make all

# 部分目标
make o          # 只编译所有 .o
make a          # 只构建 liblua.a
make clean
make depend     # 重新生成 makefile 末尾的依赖列表（gcc -MM）
make echo       # 打印当前生效的 CC/CFLAGS/...
```

Windows 下无官方 makefile：使用 `onelua.c` 单文件构建，或在 `luaconf.h` 中启用 `LUA_USE_WINDOWS` 后自行用 MSVC/MinGW 编译全部 `l*.c`。

**本机的 Windows 构建方式（已实测通过）**：本仓库应使用 `D:\environments\envpath\w64devkit\bin`（w64devkit 2.10.0，GCC 16.2.0 + GNU Make 4.4.1 + busybox `sh`）而非系统 PATH，构建时须覆盖 makefile 的 Linux 默认值：

```sh
export PATH="/d/environments/envpath/w64devkit/bin:$PATH"
make clean
make MYCFLAGS='-std=c99' MYLIBS='-lm' MYLDFLAGS=
# 产出 lua.exe、liblua.a 与标记文件 all
```

直接 `make`（默认 `-DLUA_USE_LINUX`）会失败：MinGW 定义了 `_WIN32`，使 `luaconf.h` 自动启用 `LUA_USE_C89`，与 `LUA_USE_POSIX` 冲突（`luaconf.h:93` 的 `#error "POSIX is not compatible with C89"`）；链接阶段还会因 `-ldl` 不存在而失败。**无需手动定义 `LUA_USE_WINDOWS`**，`_WIN32` 已自动处理。

```sh
gcc -O2 -std=c99 -o lua onelua.c -lm                 # 单文件构建
gcc -O2 -std=c99 -DLUA_USE_LINUX -Wl,-E -o lua onelua.c -lm -ldl
gcc -O2 -std=c89 -DLUA_USE_C89 -o lua onelua.c -lm   # C89 兼容构建
```

`onelua.c` 顶部宏可切换产物：默认 `MAKE_LUA`，也可定义 `MAKE_LIB`（库）或 `MAKE_LUAC`（编译器）。

## 4. 测试

测试套件用 Lua 编写，从 **`testes/` 目录内运行**（脚本内有 `#!../lua`，并依赖相对路径）。

```sh
make                      # 先在仓库根目录构建出 ./lua
cd testes
../lua all.lua            # 完整运行；结束时打印 "final OK !!!"
```

要点：

- `all.lua` 会校验 `_VERSION == "Lua 5.5"`；版本不符直接退出。
- 它把每个测试文件都经过 `string.dump`/`load` 走一遍，因此**序列化/反序列化路径也被覆盖**。
- **测试期开关（应在运行前作为全局变量注入）**：`_soft`（跳过耗时/耗内存的测试）、`_port`（跳过不可移植的测试）、`_nomsg`（不打印跳过信息）。
- 运行结果的时间记录写入 `testes/time.txt`（debug 构建为 `time-debug.txt`），两者都在 `.gitignore` 中；与上次耗时相差超过 5% 会发出警告。
- `testes/libs/` 需要单独 `make` 生成 `lib1.so` 等，`require` 相关测试才会执行。
- **内部测试模式**：以 `TESTS= -DLUA_USER_H='"ltests.h"' -Og -g` 构建，会启用 `ltests.c` 中的 `T` 库（`T.checkmemory`、`T.totalmem`、`T.stacklevel` 等）以及 `LUA_DEBUG`/`LUAI_ASSERT`。`ltests.c` 的代码全部包在 `#if defined(LUA_DEBUG)` 内。
  - ⚠️ 在 Windows 上通过命令行传入 `TESTS=` 有坑：`MYCFLAGS= $(LOCAL) -std=c99 -DLUA_USE_LINUX`，其中 `LOCAL = $(TESTS) $(CWARNS)`。**一旦在命令行覆盖 `MYCFLAGS`，其中的 `$(TESTS)` 就被一并丢弃**，`ltests.h` 不会被包含。因此需把 `-DLUA_USER_H` 直接写进 `MYCFLAGS`（见第 8 节的实测命令）。
- 其他有用的调试宏见 `makefile` 注释：`LUAI_ASSERT`、`HARDSTACKTESTS`、`HARDMEMTESTS`、`EMERGENCYGCTESTS`、`EXTERNMEMCHECK`、`MAXINDEXRK=k`，以及 `-fsanitize=address,undefined` + `ASAN_OPTIONS="detect_invalid_pointer_pairs=2"`。

## 5. 代码风格与约定

- 缩进用**两个空格**，不用 Tab（`\t` 仅用于头文件里的注释对齐）。
- 函数定义写作 `static int foo (lua_State *L) {`：**函数名与左括号之间有空格**，函数调用则没有（`foo(x)`）。
- 每个文件以 `/* ** $Id: xxx.c $ ** <标题> ** See Copyright Notice in lua.h */` 开头，随后 `#define <文件名>_c` 与 `#define LUA_CORE`（库文件用 `LUA_LIB`）。
- **第一个 include 必须是 `"lprefix.h"`**，其他头文件不得出现在它之前（`lprefix.h` 自身除外）。
- 头文件用 `#ifndef lxxx_h` / `#define lxxx_h` 卫哨。
- include 顺序约定：系统头 → `lua.h` → `lauxlib.h`/`lualib.h` → 内部 `l*.h`。
- 私有内部实现符号用 `lua_` / `l_` / `luai_` 前缀区分：`lua_` 是公开 API，`l_`/`luai_` 是内部宏与函数，**`lapi.c` 将内部符号暴露给库**，库只应通过 `lauxlib.h` + 内部头访问。
- 注释风格是多行 `/* ... */` 块注释，解释“为什么”而不只是“做什么”，保持 Lua 原有的高密度注释习惯。
- 保持 C89 兼容编码习惯（声明在块首、不使用 `//` 注释、不使用变长数组），即使默认按 C99 编译。
- 建议的警告集已在 `makefile` 中定义（`CWARNSCPP`/`CWARNSC`/`CWARNGCC`），提交前确保**零新增警告**；`-Werror` 默认未开启，但不应依赖这一点。

## 6. 改动时的关键不变式

- **版本号同步**：`LUA_VERSION_MAJOR_N` / `LUA_VERSION_MINOR_N` / `LUA_VERSION_RELEASE_N` 与 `LUA_VERSION`、`LUA_RELEASE`、`LUA_VERSION_NUM` 都在 `lua.h`，`testes/all.lua` 中的 `local version = "Lua 5.5"` 也需同步。
- **公开 API 变更必须同时更新**：`lua.h`/`lauxlib.h`/`lualib.h`（声明）、实现文件、以及 `manual/manual.of` 文档；同时确认 `luaconf.h` 中受影响的配置宏。
- **新增/修改 VM 指令**要同时改 `lopcodes.h`、`lopcodes.c`（`luaP_opnames`、`luaP_opmodes`）、`lvm.c`、`ljumptab.h`，以及 `lcode.c` 的生成逻辑；`CODE` 测试会校验指令集一致性。
- **GC 相关改动**需同时考虑增量与分代模式（`lgc.c`、`testes/gc.lua`、`testes/gengc.lua`）。
- 栈/指针字段在 5.5 中使用 `StkIdRel`/union 形式，改动栈操作时注意 `L->top.p` 等新访问形式，不要写回旧版 `L->top` 语义。
- `makefile` 末尾的依赖列表（`# DO NOT EDIT` 段）由 `make depend` 生成，手工加 `.o` 后请一并更新依赖，或说明原因。
- `all`、`lua`（根目录）与 `*.o`、`*.a`、`*.so`、`manual/manual.html` 都是**构建产物且已在 `.gitignore` 中**，不要提交。

## 7. 提交与协作

- 提交信息简洁、聚焦单一改动（对比现有历史风格，例如 `Bug: UB when applying GC parameter`、`'mktime' can return -1`）。
- 上游风格偏好“小而精确的修复 + 清晰说明”，避免大规模重构性提交。
- **不要**在未经确认的情况下改动 `testes/` 中现有断言来让代码通过——应视为测试是正确的一方去修实现。

## 8. 本工作环境的已知限制（重要）

工具链**可用**，位于 `D:\environments\envpath\w64devkit\bin`（w64devkit 2.10.0）。但它**不在系统 PATH 中**，需显式加入；系统 PATH 里确实没有 `gcc` / `make` / `cl` / `cmake`，`wsl` 也因权限被拒绝（`E_ACCESSDENIED`）。

**已实测可行的验证闭环（在临时目录中构建，避免污染仓库）**：

```sh
export PATH="/d/environments/envpath/w64devkit/bin:$PATH"
# 1) 构建（约 1 分钟）
make clean && make MYCFLAGS='-std=c99' MYLIBS='-lm' MYLDFLAGS=
# 2) 冒烟测试
./lua.exe -v          # 期望 "Lua 5.5.1  Copyright (C) 1994-2026 Lua.org, PUC-Rio"
./lua.exe -e "print(_VERSION, math.type(math.maxinteger), math.maxinteger)"
# 3) 内部测试模式（启用 ltests.c 的 T 库）
make clean && make MYCFLAGS='-DLUA_USER_H='"'"'"ltests.h"'"'"' -Og -g -std=c99' MYLIBS='-lm' MYLDFLAGS=
./lua.exe -e "print(type(T), type(T.checkmemory))"   # 期望 table function
```

**`testes/all.lua` 无法在本机跑完**，且原因分两类，不要误判为代码缺陷：

1. **沙箱限制**（与本仓库无关，是运行环境造成的）。以下测试会因临时文件写入/子进程调用被拒绝而失败，均出现在 `all.lua` 的前段之后：
   - `main.lua`：第 5–6 行有 `if _port then return end`，它以 `_port` 作为"需要类 Unix shell"的开关；`_soft` + `_port` 可跳过它。
   - `verybig.lua`、`memerr.lua`：需要写入 `os.tmpname()` 给出的路径，报 `Permission denied`。
   - `files.lua`：`os.tmpname()` 打不开文件，脚本自身会打印 `'os.tmpname' file cannot be open; skipping file tests` 并跳过大部分文件测试。
   - `testes/libs/` 的 `make` 在本机也会失败（`undefined reference to 'lua_gettop'`）：其 makefile 为 Linux 设计，依赖主程序用 `-Wl,-E` 导出符号；Windows/MinGW 下需改为链接 `liblua.a`。
2. **真实的平台差异**（不是沙箱问题）：`files.lua:858-861` 测试 `os.date("*t", -1)` 与 `os.time` 的往返，该断言**未被 `_port` 保护**；MinGW 的 C 运行时不支持 1970 年前的时间戳（`os.date("*t", -1)` 与 `os.time{year=1969,...}` 都报 `cannot be represented in this installation`，而 `os.date("!%Y-%m-%d", -1)` 正常）。所以在 Windows 上套件会在 `files.lua:859` 中断。

**可用的部分验证手段**（在 `testes/` 内执行，已实测）：

```sh
# 通过 LUA_INIT 注入 _port，使子进程也继承该设置（比命令行传参可靠，因为 all.lua
# 自身会吞掉命令行参数）
echo '_port = true' > setport.lua
LUA_INIT='@setport.lua' ../lua.exe all.lua     # 可跳过 main.lua，直到 files.lua/memerr.lua 处中断
../lua.exe db.lua                              # 单个测试文件可独立运行，实测通过
```

在内部测试模式下加 `_port` 运行 `all.lua`，实测可依次通过 `main`（跳过）→ `gc` → `db` → `calls` → `strings` → `literals` → `tpack` → `attrib` → `gengc` → `locals` → `constructs` → `code` → `cstack` → `nextvar` → `pm` → `utf8` → `api`，在 `memerr.lua` 处因沙箱中断（约 6 秒）。

因此修改 C 代码后的正确做法是：

- 用上面的命令**确实编译**，并跑冒烟测试与能单独通过的测试文件（如 `db.lua`）；
- 对无法在本机验证的部分（完整 `testes`、内存/栈压力测试、`testes/libs` 的 `require` 测试）**如实声明"未验证"**，并给出应在 Linux/POSIX 上执行的命令，不要声称已通过：
  ```sh
  make && (cd testes && ../lua all.lua)
  ```
- 可以做纯静态验证：用文件读取与搜索检查符号一致性（例如新增 API 是否在所有调用点同步、头文件声明与实现是否匹配）。
