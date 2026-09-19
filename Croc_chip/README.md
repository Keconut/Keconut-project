# Croc 芯片实验记录：把 helloworld 跑起来，再做一次综合

## 一、实验开始前的情况

我是大一新生，之前对芯片只有零散的认识，大部分概念都是自己在网上搜的入门文章。
我想照着 `pulp-platform/croc` 这个教学用的 SoC 走一遍流程，看看一个"芯片"从代码到门电路大概经过哪几步。

我在自己的 Windows 电脑上试过。电脑上没有 Verilator，没有 Yosys，也没有 RISC-V 的交叉编译器。
Windows 上还找不到 `apt` 或者 `yum` 这类包管理器，网络下载也不通。
所以我没有在本机装任何东西，改用 GitHub Actions 的云端机器来跑。云端机器是 Ubuntu 24.04，工具都用 `apt` 装得上。

接下来记录的就是云端那次运行的经过。

> 先说明一件事。写这份文档的时候，工作流还没有在 GitHub 上真正跑过。我本机没有 Linux 环境，也没法在本地起容器，所以终端输出我还没有拿到手。下面第二节和第三节里的命令，是我照着 Croc 仓库里脚本的实际内容整理出来的，脚本里确实会打印那些行。等我把这个仓库推到 GitHub、Actions 跑完一次，再把真实的日志和截图贴进来，替换掉现在的位置说明。我不想凭空编一堆没跑过的日志来充数。

## 二、这次真正要执行的代码在哪里

这次实验真正要跑起来的东西，是仓库里的一个工作流文件。

```
.github/workflows/croc_test.yml
```

它写在 GitHub 的云端机器上执行，我本机不用装任何工具。工作流跑完会自动打包一份证据出来，里面是每一步完整的终端输出，还有三张由日志生成的截图。

这次一共做两件事。第一件是把前端跑通，也就是让 helloworld 这个程序在 Verilator 仿真里真正运行起来，并且打印出结果。第二件是把后端跑通，也就是用 Yosys 把同一份硬件代码综合一遍，得到一个由标准单元组成的电路清单。

## 三、前端干了什么

前端做的事情是：把写好的硬件代码交给 Verilator 编译成一个能在电脑上运行的程序，然后把这个程序跑起来，看里面的 CPU 有没有老老实实执行我们给它的软件。

这里有一个我之前没想明白的地方。Croc 里面有一个 RISC-V 的 CPU 核，这个核不能直接运行 C 语言写的 `helloworld.c`。所以先要用 RISC-V 的交叉编译器把 C 代码编译成 RISC-V 的机器码，再把这个机器码变成一串十六进制的文本文件。仿真的时候，测试台会通过 JTAG 把这串文本写进仿真里的内存，然后放开 CPU 让它跑。

我执行的具体命令是这样的。

第一步，用 Bender 生成 RTL 的文件清单。Croc 的硬件代码分散在很多文件里，Bender 会把这些文件按正确顺序整理成一个列表文件 `croc_rtl.f`。

```
cd croc/verilator
./run_verilator.sh --flist-rtl
```

第二步，编译 helloworld。

```
cd croc/sw
make all
```

跑完之后 `sw/bin/` 下面多出了三个文件：`helloworld.elf`、`helloworld.dump`、`helloworld.hex`。那个 `.hex` 就是后面要送进内存的东西。

第三步，用 Verilator 编译硬件代码。这是整个前端里最花时间的一步。

```
cd croc/verilator
./run_verilator.sh --build
```

这一步 Verilator 要先把 SystemVerilog 读进去，然后生成一大堆 C++ 代码，再调用 g++ 把它们编译成可执行文件。
最后生成了 `verilator/obj_dir_rtl/Vtb_croc_soc`，这个就是仿真程序。

第四步，运行仿真。

```
cd croc/verilator
./run_verilator.sh --run ../sw/bin/helloworld.hex
```

运行的输出里有几条对我来说比较重要。

```
Running program: ../sw/bin/helloworld.hex
@... | [JTAG] Initialization success
@... | [UART] Hello World from Croc!
@... | [JTAG] Simulation finished: SUCCESS
```

第一条说明测试台确实找到了我要跑的程序。第二条说明 JTAG 调试通道连上了。第三条是 UART 打印出来的字符串，也就是 CPU 真的把 `printf` 执行完了。最后一条 `Simulation finished: SUCCESS` 是测试台检查完 CPU 的状态寄存器之后给出的结论。

我看到 `Hello World from Croc!` 出来的时候才确认前端是真通了，不是编译通过就算数。

前端这一段的完整终端输出，跑完之后会在这里：

- `logs/frontend_01_flist.log`
- `logs/frontend_02_sw_build.log`
- `logs/frontend_03_verilator_build.log`
- `logs/frontend_04_helloworld_run.log`

云端跑的截图会在这里：

- `logs/01_frontend_verilator_compile.png`（Verilator 编译过程）
- `logs/02_frontend_helloworld_run.png`（helloworld 运行成功）

## 四、后端干了什么

后端做的事情是：把同一份硬件代码交给 Yosys，让它算出一个由具体标准单元搭起来的电路清单。

前端关心的是"功能对不对"，后端关心的是"要用多少电路来实现"。Yosys 读进去的是行为描述，吐出来的是一个门级网表。网表里不再有 `always` 这种写法，只剩下一个个标准单元和它们之间的连线。

Croc 用的工艺是 IHP 的 130nm。仓库里带着一份 Liberty 文件，里面写清楚了每种标准单元长什么样、有多少个引脚、大概占多少面积、跑得快不快。Yosys 就是照着这份文件把逻辑映射到具体单元上的。

后端我执行了这一条命令。

```
cd croc/yosys
./run_synthesis.sh --synth
```

这条命令实际做了两件事，先重新生成一次文件清单，然后启动 Yosys 跑 `scripts/yosys_synthesis.tcl` 这个脚本。

这里我卡过一次，值得记下来。我一开始想用 Ubuntu 自带的 Yosys，结果不行。因为 Croc 的脚本里写着 `yosys plugin -i slang.so`，它需要一个叫 slang 的 SystemVerilog 前端插件，而 Ubuntu 软件源里的 Yosys 包不含这个插件。后来我改用 OSS CAD Suite，那里面自带的 Yosys 是带 slang 的，而且 abc 这些工具也一起给了，就不用一个个自己装。

Yosys 跑完后，`yosys/out/` 目录下多了两个网表文件。`croc_yosys.v` 是给后续布线流程用的，`netlist_debug.v` 是留了信号名的版本，方便对着看。同一时间 `yosys/reports/` 下面生成了一批报告，我留了几份：

- `logs/yosys_reports/croc_area.rpt`：面积报告，能看出各个模块大概占了多少单元
- `logs/yosys_reports/croc_registers.rpt`：寄存器列表
- `logs/yosys_reports/croc_synth.rpt`：综合后的检查结果
- `logs/yosys_reports/croc_instances.rpt`：实例列表

后端这一段的完整终端输出，跑完之后会在这里：

- `logs/backend_01_yosys_synthesis.log`（终端 stdout，带时间戳）
- `logs/backend_02_croc_yosys_internal.log`（脚本自己写的日志）
- `logs/backend_03_yosys_out.txt`（产出的网表文件列表）
- `logs/backend_05_area_report_tail.txt`（面积报告末尾）

云端跑的截图会在这里：

- `logs/03_backend_yosys_synthesis.png`（Yosys 综合跑完）

## 五、我这次实际用到的工具

| 工具 | 用途 | 来源 |
| --- | --- | --- |
| verilator | 把硬件代码编译成可执行的仿真程序 | `apt install verilator` |
| riscv64-unknown-elf-gcc | 把 helloworld.c 编译成 RISC-V 机器码 | `apt install gcc-riscv64-unknown-elf` |
| bender | 整理硬件代码的文件清单 | GitHub release 里的预编译二进制 |
| yosys + slang | 把 RTL 综合成门级网表 | OSS CAD Suite |
| imagemagick | 把终端日志转成 png 截图 | `apt install imagemagick` |

Croc 的版本记录在 `logs/croc_commit.txt` 里，是 clone 时的那一刻的 commit。

## 六、这次没做的事情

为了让边界清楚，这里写一下我这次没碰的部分。

后端我只跑到 Yosys 综合为止，没有再往下走布线。Croc 完整流程后面还有 OpenROAD 做布局布线、KLayout 做版图收尾，那两步我没有跑。Yosys 出来的网表已经能说明综合这一步是通的。

我也没有跑门级仿真的对比。Croc 支持把 Yosys 出来的网表再送回 Verilator 跑一遍，看看功能是不是还一样，这个我也没做。

## 七、怎么把它传到 GitHub 上跑起来

这一节写给我自己以后看。我本机没有环境，所以东西要推到 GitHub 上去跑。

### 需要先准备的东西

一个 GitHub 账号，还有本机的 git。这两样我都有。推送的时候要输入账号和 token，如果忘了密码，可以去 GitHub 的 Settings 里生成一个 Personal Access Token 当密码用。

### 上传的命令

在 `D:\Croc_chip` 这个目录下打开终端，依次执行下面几条。把 `你的用户名` 换成自己的 GitHub 用户名。

```
cd D:\Croc_chip
git init
git add .github/workflows/croc_test.yml README.md .gitignore
git commit -m "add croc frontend and backend workflow"
git branch -M main
git remote add origin https://github.com/你的用户名/croc-chip.git
git push -u origin main
```

`git remote add` 这一步要求 GitHub 上已经有一个叫 `croc-chip` 的空仓库。这个仓库要自己先去 GitHub 网页上新建，建的时候不要勾选添加 README，不然第一次 push 会因为两边都有内容而报错。

### 推送之后会发生什么

因为我给工作流写了 `push` 触发，所以文件推上去之后它会自己开始跑，不需要我再去点按钮。

想看进度的话，打开仓库页面，点上面的 Actions 标签，左边选 `Croc frontend + backend`，右边就能看到这次运行。点进去之后每个步骤都能展开，展开后就是那一步的终端输出。

如果它没有自动开始，点进这个工作流，右边有一个 `Run workflow` 按钮，点一下也能跑。这就是我在 YAML 里写 `workflow_dispatch` 的作用。

整个流程我估计要跑二十分钟到四十分钟。Verilator 编译那一步最慢。

### 跑完之后去哪里拿日志和截图

一次完整跑完之后，在那次运行的页面最下面有一个 `Artifacts` 区域，里面有一个叫 `croc-evidence` 的下载项。点它就会下载一个 `croc-evidence.zip`。

把这个压缩包解开，里面就是我要的东西：

```
01_frontend_verilator_compile.png    前端 Verilator 编译的截图
02_frontend_helloworld_run.png       前端 helloworld 运行成功的截图
03_backend_yosys_synthesis.png       后端 Yosys 综合跑完的截图
frontend_01_flist.log                前端第一步的完整终端输出
frontend_02_sw_build.log             前端第二步的完整终端输出
frontend_03_verilator_build.log      前端第三步的完整终端输出
frontend_04_helloworld_run.log       前端第四步的完整终端输出
backend_01_yosys_synthesis.log       后端综合的完整终端输出
backend_02_croc_yosys_internal.log   后端脚本自己写的日志
backend_03_yosys_out.txt             后端产出的网表文件列表
backend_04_netlist_head.txt          网表文件开头几行
backend_05_area_report_tail.txt      面积报告末尾
croc_commit.txt                      clone 到的 croc commit 号
yosys_reports/                       综合报告
```

想看某一步有没有真的成功，可以直接在运行页面上点开对应的步骤，日志就显示在那里，不用等下载。下载只是为了让日志能长期留着。

我在第三节和第四节里写的路径，就是我打算把这份日志和截图放的位置。等工作流在 GitHub 上跑完一次，我把证据补齐。

## 八、总结

这次做完，我对流程有了一个比较具体的印象。前端拿 C 代码和 Verilog 一起喂给仿真器，验证的是功能。后端拿同一份 Verilog 去算电路，验证的是实现代价。两边读的是同一份硬件描述，但关心的事情不一样。

对我个人来说，最大的收获是知道了 `Simulation finished: SUCCESS` 这个结论不是凭空出现的，它背后要经过交叉编译、文件清单生成、Verilator 编译、JTAG 装载、UART 打印这一串步骤，少一步都到不了。
