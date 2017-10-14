TSmart-DATE Fuzzer
==================

## 配置环境
TSmart-DATE 的Fuzzer 工具包位于 /opt/tsmart-date/。请先配置 shell 的 `PATH` 环境变量。

在 shell 中执行

```sh
export PATH="/opt/tsmart-date:$PATH"
```

或是直接将上述内容添加至 `~/.bashrc` 等配置文件中。

## 符号执行

### 配置和构建
在配置好之后，就可以使用 TSmart-DATE 的内置工具了。TSmart-DATE 使用符号执行构建高质量种子，作为动态测试的输入，因此需要先配置符号化输入的长度。以 10 字节为例，执行：

```sh
~/test/c-ares root@HghDTG
❯ date-prepare sym 10
[*] Toolchain is at /tmp/toolchain
[+] Saving configuration (symbolic, size is 10)
```

接着进行项目在编译前的配置 (configure) 操作。类似于一般的配置，但需要在命令前插入 `date-configure`。

```sh
~/test/c-ares/src heads/master* root@HghDTG
❯ date-configure ./configure
[*] Config: sym 10
[+] Setting up bitcode toolchain (unfiltered)
[*] LLVM is at /opt/llvm
[*] Toolchain is at /tmp/toolchain
[*] Additional argument(s) to pass:
[*]     CXXFLAGS=-stdlib=libc++ -nostdinc -I/opt/llvm/include/c++/v1 -Qunused-arguments
[*]     LDFLAGS=-stdlib=libc++ -nostdinc -L/opt/llvm/lib -Wl,-rpath,/opt/llvm/lib
[*]     CC=/tmp/toolchain/clang
[*]     CXX=/tmp/toolchain/clang++
[+] Running target program
checking whether to enable maintainer-specific portions of Makefiles... no
[...]
config.status: executing libtool commands
configure: amending ./Makefile
```

接着正式编译了。类似于一般的编译，但需要在命令前插入 `date-build`。


```sh
❯ date-build make -j
[*] Config: sym 10
[+] Setting up bitcode toolchain
[*] WLLVM is at /opt/wllvm
[*] Toolchain is at /tmp/toolchain
[+] Custom build comand, ignoring
[+] Running target program
make  all-recursive
[...]
make[1]: Leaving directory '/home/hugh/test/c-ares/src'
```

在构建完库文件后，可能还需要编译额外的测试文件。类似于调用标准的 `g++` 或 `clang` 命令，但需要在命令前插入 `date-build`。

```sh
~/test/c-ares root@HghDTG
❯ date-build g++ target.cc -I src -c
[*] Config: sym 10
[+] Setting up bitcode toolchain
[*] WLLVM is at /opt/wllvm
[*] Toolchain is at /tmp/toolchain
[+] C++ compiler intercepted
[*] Command to execute:
[*]     /tmp/toolchain/clang++
[*]     -stdlib=libc++
[*]     -nostdinc++
[*]     -I/opt/llvm/include/c++/v1
[*]     -Qunused-arguments
[*]     target.cc
[*]     -I
[*]     src
[*]     -c
```

最后执行链接操作。需要注意的是，TSmart-DATE 没有使用 Clang 内置的驱动，因此不能使用标准工具链直接链接，必须使用 `date-link` 命令。

下面的例子将测试目标文件 `target.o` 和库文件 `src/.libs/libcares.a` 链接出二进制文件 `app`，并生成中间代码 `app.bc`。

```sh
~/test/c-ares root@HghDTG 
❯ date-link target.o src/.libs/libcares.a
[*] Config: sym 10
[+] Setting up bitcode toolchain
[*] WLLVM is at /opt/wllvm
[*] Toolchain is at /tmp/toolchain
[+] Generating driver, symbolic argument size = 10
[*] Command to execute:
[*]     /tmp/toolchain/clang
[*]     -xc
[*]     -c
[*]     -o
[*]     /tmp/toolchain/driver.o
[*]     /tmp/toolchain/driver.c
[+] Linking objects
[*] Command to execute:
[*]     /tmp/toolchain/clang++
[*]     -stdlib=libc++
[*]     -nostdinc++
[*]     -I/opt/llvm/include/c++/v1
[*]     -Qunused-arguments
[*]     -stdlib=libc++
[*]     -nostdinc++
[*]     -L/opt/llvm/lib
[*]     -Wl,-rpath,/opt/llvm/lib
[*]     /tmp/toolchain/driver.o
[*]     target.o
[*]     src/.libs/libcares.a
[*]     -o
[*]     app
[+] Extracting bitcode
Loading '/tmp/toolchain/.driver.o.bc'
Loading '/home/hugh/test/c-ares/.target.o.bc'
Linking in '/home/hugh/test/c-ares/.target.o.bc'
Loading '/home/hugh/test/c-ares/src/.libcares_la-ares_create_query.o.bc'
Linking in '/home/hugh/test/c-ares/src/.libcares_la-ares_create_query.o.bc'
Loading '/home/hugh/test/c-ares/src/.libcares_la-ares_library_init.o.bc'
Linking in '/home/hugh/test/c-ares/src/.libcares_la-ares_library_init.o.bc'
Writing bitcode...
```

首先构建符号执行需要的二进制文件。

### 启动符号执行

创建一个新的目录，用于存放符号执行的中间文件，同时将链接好的中间文件 (`app.bc`) 放在这里。这里以 `sym` 为例。

```sh
~/test/c-ares root@HghDTG
❯ mkdir sym && cd sym

~/test/c-ares/sym root@HghDTG
❯ cp ../app.bc .
```

使用命令 `date-fuzz` 启动符号执行。注意，符号执行没有时间限制，需要手动停止运行。可以多次启动符号执行，无需清理中间文件。

```sh
~/test/c-ares/sym root@HghDTG
❯ date-fuzz app
[*] Config: sym 10
[*] Symbolic argument size = 10
[*] Application is at app.bc
[+] Starting symbolic execution
[*] KLEE cmdline:
[*]     /opt/klee/bin/klee
[*]     -only-output-states-covering-new
[*]     -libc=uclibc
[*]     -posix-runtime
[*]     -simplify-sym-indices
[*]     app.bc
[*]     -sym-arg
[*]     10
KLEE: NOTE: Using klee-uclibc : /opt/klee/lib64/klee/runtime/klee-uclibc.bca
KLEE: NOTE: Using model: /opt/klee/lib64/klee/runtime/libkleeRuntimePOSIX.bca
KLEE: output directory is "/home/hugh/test/c-ares/sym/klee-out-0"
KLEE: Using Z3 solver backend
[...]
^CKLEE: ctrl-c detected, requesting interpreter to halt.
KLEE: halting execution, dumping remaining states

KLEE: done: total instructions = 224686
KLEE: done: completed paths = 1371
KLEE: done: generated tests = 6
```

### 可选：执行不同长度的输入

由于符号执行的长度是固定的，可能需要重新指定输入的长度。此时，只需要重新配置长度并链接即可，无需重新构建。

```sh
~/test/c-ares root@HghDTG
❯ date-prepare sym 50
[*] Toolchain is at /tmp/toolchain
[+] Saving configuration (symbolic, size is 50)

~/test/c-ares root@HghDTG
❯ date-link target.o src/.libs/libcares.a
[*] Config: sym 50
[+] Setting up bitcode toolchain
[*] WLLVM is at /opt/wllvm
[*] Toolchain is at /tmp/toolchain
[+] Generating driver, symbolic argument size = 50
[*] Command to execute:
[*]     /tmp/toolchain/clang
[*]     -xc
[*]     -c
[*]     -o
[*]     /tmp/toolchain/driver.o
[*]     /tmp/toolchain/driver.c
[+] Linking objects
[*] Command to execute:
[*]     /tmp/toolchain/clang++
[*]     -stdlib=libc++
[*]     -nostdinc++
[*]     -I/opt/llvm/include/c++/v1
[*]     -Qunused-arguments
[*]     -stdlib=libc++
[*]     -nostdinc++
[*]     -L/opt/llvm/lib
[*]     -Wl,-rpath,/opt/llvm/lib
[*]     /tmp/toolchain/driver.o
[*]     target.o
[*]     src/.libs/libcares.a
[*]     -o
[*]     app
[+] Extracting bitcode
Loading '/tmp/toolchain/.driver.o.bc'
Loading '/home/hugh/test/c-ares/.target.o.bc'
Linking in '/home/hugh/test/c-ares/.target.o.bc'
Loading '/home/hugh/test/c-ares/src/.libcares_la-ares_create_query.o.bc'
Linking in '/home/hugh/test/c-ares/src/.libcares_la-ares_create_query.o.bc'
Loading '/home/hugh/test/c-ares/src/.libcares_la-ares_library_init.o.bc'
Linking in '/home/hugh/test/c-ares/src/.libcares_la-ares_library_init.o.bc'
Writing bitcode...
```

此时新生成二进制文件 `app` 和中间文件 `app.bc`。用新文件启动符号执行即可。

### 生成种子
符号执行会生成测试用例。需要使用 `date-test-gen` 命令，将测试用例转换为可以进行动态测试的种子。下面的例子在 `seed` 目录生成了 6 个种子。

```sh
~/test/c-ares/sym root@HghDTG
❯ ls
klee-out-0  app.bc  klee-last

~/test/c-ares/sym root@HghDTG
❯ mkdir seed

~/test/c-ares/sym root@HghDTG
❯ date-test-gen klee-out-*/*.ktest --output seed

~/test/c-ares/sym root@HghDTG
❯ ls seed
test000001.symseed  test000003.symseed  test000005.symseed
test000002.symseed  test000004.symseed  test000006.symseed
```

这些种子可以作为动态测试的输入。

## 动态测试

### 构建和编译
在进行动态测试前，请先清理构建目录，删除生成的对象和二进制文件。

```sh
~/test/c-ares/src heads/master* root@HghDTG
❯ make clean
make[1]: Entering directory '/home/hugh/test/c-ares/src'
[...]
rm -f *.o
rm -f *.lo
make[1]: Leaving directory '/home/hugh/test/c-ares/src'
```

接着重新配置 TSmart-DATE，切换到动态测试模式。

```sh
~/test/c-ares/src heads/master* root@HghDTG
❯ date-prepare dyn
[*] Toolchain is at /tmp/toolchain
[+] Saving configuration (dynamic)
```

接着执行和符号执行阶段的构建和编译过程完全相同的命令：

```sh
# 构建库
date-configure ./configure
date-build make -j

# 构建测试文件
cd ..
date-build g++ target.cc -I src -c
date-link target.o src/.libs/libcares.a 
```

### 启动动态测试

```sh
❯ mkdir dyn && cd dyn
```
#### 默认模式下直接执行
需要事先配置tmux运行环境 （sudo apt-get install tmux）

直接执行以下指令即可
```sh
date-fuzz ./app 
```

#### 本工具同样提供afl模式的运行方式
只需在date-fuzz 后面加上-afl-mode 即可
```sh
date-fuzz -afl-mode  -afl参数  ./app
```

可支持所有afl的参数调优，afl参数调优详见afl官方文档
