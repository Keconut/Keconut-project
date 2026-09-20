# Croc 运行故障修复包

本包针对公开仓库 `Keconut/Keconut-project` 的 Croc 实验工作流。诊断基于真实 GitHub Actions 日志、对应上游源码，以及本会话中的独立复现。

## 怎么使用

1. 打开朋友仓库根目录的 `.github/workflows/croc_test.yml`，用本包的 `croc_test.yml` **完整替换其内容**并提交。注意不是 `Croc_chip/.github/workflows/croc_test.yml`；后者是子目录里的副本。
2. 在 Actions 中选择 `Croc frontend + backend`，查看新运行；需要手动启动时点 `Run workflow`。重新运行旧的失败任务不会自动使用这份修改。
3. 成功标准是前端步骤输出 `[FRONTEND] PASSED`，后端步骤输出 `[BACKEND] PASSED`。运行页的 `croc-evidence` artifact 保存真实日志、软件产物、门级网表和综合报告。

替换工作流即可；它会自行检出固定版本的 Croc，并应用两个补丁，不需要手动修改下载后的源码。仓库现有网页和其他项目不需要调整。

## 已定位的故障

### 1. 卡住的是程序结束流程

原 Actions 运行中，软件交叉编译和 Verilator 编译均已成功，JTAG 写入 SRAM 后也读回了正确数据，UART 已输出 `Hello World from Croc!`。随后 CPU 持续报异常，测试台等不到 CORESTATUS 中的程序结束状态，因此第 6 步一直运行，Yosys 步骤被阻塞。

日志中的第一条重复异常为：

```text
Illegal instruction (hart 0) at PC 0x00000000: 0x00010413
```

实际根因在上游 `sw/crt0.S` 的 `_start`：`la sp, __stack_pointer$` 位于 `.option norelax` 之前，且执行时 `gp` 尚未初始化。Ubuntu 的 RISC-V GCC 13.2 / binutils 2.42 将它进行 GP-relative relaxation，链接为：

```asm
1000000c: 6d818113    addi sp,gp,1752
```

Boot ROM 此时已将 `gp` 清零，所以 `sp` 实际变成 `0x000006D8`，而不是 SRAM 末端的 `0x10001000`。栈没有落在预期 SRAM 中，函数调用与返回无法可靠工作。仅看到了 UART 字符串，不能作为整个程序正常结束的证明。

修复是让 `.option norelax` 同时覆盖 SP 和 GP 初始化：

```asm
_start:
  .option push
  .option norelax
  la      sp, __stack_pointer$
  la      gp, __global_pointer$
  .option pop
  tail    main
```

修复后的前两条指令为 `auipc sp,0x1` 和 `addi sp,sp,-12`，正确得到 `0x10001000`。

### 2. 后端还有工具版本兼容问题

原工作流下载 OSS CAD Suite 的 `latest`，工具版本会变化。实测 2026-09-19 套件的 slang 已经不接受 `--ignore-unknown-modules`，`--compat-mode` 也已废弃。Croc 的技术初始化脚本已经先通过 Liberty 导入标准单元和 SRAM 的 blackbox 定义，因此删除这两个旧参数后可以正常展开设计，未知模块仍会被报错。

同一套件中，Yosys 默认寻找 ABC 的路径也失败。补丁让综合脚本显式调用 `yosys abc -exe yosys-abc ...`，通过套件提供的 wrapper 加载正确运行库。

本包固定 Croc 提交、Bender 版本和套件日期，直接使用套件中的 Verilator、Yosys、ABC，不再混用旧 apt Verilator，也不全局设置套件的 `LD_LIBRARY_PATH`。

### 3. 旧 SRAM 补丁没有解决此问题

旧工作流搜索的是 `ts_sram.sv`，上游实际文件叫 `tc_sram.sv`，所以全盘搜索并没有找到目标。这里保留原 SRAM RTL，使用已经实测能构建它的 Verilator 5.053。

## 验证依据

- 朋友仓库提交：`833e2ddd4116e7de38cb58bbdc8bf4165def23d7`。
- Croc 提交：`b301ef4439ab9b55322d2ac31d672fdcc67bc0c6`，与失败日志检出的版本一致。
- 软件编译器：Ubuntu RISC-V GCC `13.2.0-11ubuntu1+12`，binutils `2.42-1ubuntu1+6`。
- Bender：`v0.32.1`。
- OSS CAD Suite：`2026-09-19`；Verilator `5.053 devel v5.052-119-g014c9820d`；Yosys `0.69+75 f0b945f63`。
- 原程序在同一仿真器中复现异常，8 秒后由测试命令超时终止，退出码 124。
- 仅修复启动代码后，同一仿真器约 1.749 秒正常结束，退出码 0，出现 UART Hello World 与 `Simulation finished: SUCCESS`，没有 `Illegal instruction`。
- 应用综合工具兼容补丁后，Yosys 退出码 0，生成两个非空门级网表和面积报告；`croc_synth.rpt` 报告 `Found and reported 0 problems.`。

`validation/` 保存原始 Actions 摘录、本次修复前后反汇编、仿真日志、编译记录与综合验证产物。具体检查结果见 `validation/verification.json`。

这些是本会话 Ubuntu 环境中的实测记录。新 YAML 已做语法检查并核对补丁内容；尚未在朋友的 GitHub Actions 中远程重跑，也没有修改其远程仓库。

## 前端、后端各做了什么

| 阶段 | 输入和操作 | 可验证的结果 |
| --- | --- | --- |
| 软件编译 | 将 C 程序和启动汇编交给 RISC-V 交叉编译器 | ELF、反汇编 dump、加载用 hex |
| 前端 RTL 仿真 | Verilator 编译 SoC 与测试台；JTAG 将 hex 装入模拟 SRAM；模拟 CPU 执行 | UART 字符串、程序结束状态、正常退出 |
| Yosys 综合 | 读取 SoC RTL 和 IHP 工艺 Liberty，完成逻辑综合、ABC 映射 | `croc_yosys.v`、`netlist_debug.v`、面积及其他综合报告 |

本包后端验证范围为 Yosys 综合。布局布线属于后续 OpenROAD 阶段，综合报告不代表已经完成物理设计或流片验证。

helloworld 及本版本支持库没有使用数学函数，因此示例工作流取消了不必要的 `-lm`，保留 `-lgcc` 和项目原有编译参数；不需要安装 picolibc 或创建编译器包装脚本。日后若程序加入数学库函数，需要重新配置相应的库。

## 文件说明

- `croc_test.yml`：可直接替换的完整 Actions 工作流。
- `fix-crt0.patch`：最小启动代码补丁。
- `fix-synthesis-tools.patch`：slang 与 ABC 调用兼容补丁。
- `validation/`：实测记录与产物。

## 来源

- [原始失败运行与完整日志](https://github.com/Keconut/Keconut-project/actions/runs/35450994899/job/105917998062)
- [失败运行使用的工作流](https://github.com/Keconut/Keconut-project/blob/833e2ddd4116e7de38cb58bbdc8bf4165def23d7/.github/workflows/croc_test.yml)
- [对应 Croc 启动代码](https://github.com/pulp-platform/croc/blob/b301ef4439ab9b55322d2ac31d672fdcc67bc0c6/sw/crt0.S)
- [GNU 汇编器对 relaxation / norelax 的说明](https://sourceware.org/binutils/docs/as/RISC_002dV_002dDirectives.html)
- [slang/sv-elab 关于移除 unknown-modules 参数的说明](https://github.com/povik/sv-elab/wiki/No-unknown-modules)
