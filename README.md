# LLVM Tutor Note

- 本人学习[banach-space/llvm-tutor](https://github.com/banach-space/llvm-tutor)的一点笔记

## Environment

首先安装 LLVM-17。Ubuntu 的包版本普遍滞后，22.04 到目前只有 LLVM-14，如果想使用更现代的 LLVM-17 就需要添加第三方源（其实是因为 LLVM project 的 API 更换太频繁到现在都没稳定下来）

- Ubuntu 22.04

```shell
wget -O - https://apt.llvm.org/llvm-snapshot.gpg.key | sudo apt-key add -
sudo apt-add-repository "deb http://apt.llvm.org/jammy/ llvm-toolchain-jammy-17 main"
sudo apt update
sudo apt install -y llvm-17 llvm-17-dev llvm-17-tools clang-17
```

安装完就能在`/usr/lib/llvm-17`找到所需的头文件、库和二进制文件了

- 推荐给 shell 加个配置

```shell
export LLVM_DIR=/usr/lib/llvm-17
```

- 如果用的 vscode，建个`.vscode`后`F1`输入`C++: Edit Configurations (UI)`，加上编译器路径、Include Path 和 C++ 标准
  - Include Path 加上：`/usr/include/llvm-17`、`/usr/include/llvm-c-17`

## HelloWorld

### First Taste

编译 HelloWorldPass

```shell
cd /path/to/llvm-tutor
mkdir build && cd build
cmake -DLT_LLVM_INSTALL_DIR=$LLVM_DIR ../HelloWorld/
make
```

编译完成后就可以在`build`目录看到 Pass `libHelloWorld.so`了

在使用 Pass 之前，首先需要准备一个小白鼠作为输入文件

```shell
clang-17 -O1 -S -emit-llvm ../inputs/input_for_hello.c -o input_for_hello.ll
```

小白鼠长这样

```c
int foo(int a) {
  return a * 2;
}

int bar(int a, int b) {
  return (a + foo(b) * 2);
}

int fez(int a, int b, int c) {
  return (a + bar(a, b) * 2 + c * 3);
}

int main(int argc, char *argv[]) {
  int a = 123;
  int ret = 0;

  ret += foo(a);
  ret += bar(a, ret);
  ret += fez(a, ret, 123);

  return ret;
}
```

运行完成后可以在`build`目录下发现`input_for_hello.c`的 IR 文件`input_for_hello.ll`

生成出的 IR 长这样

```llvm
; ModuleID = '../inputs/input_for_hello.c'
source_filename = "../inputs/input_for_hello.c"
target datalayout = "e-m:e-p270:32:32-p271:32:32-p272:64:64-i64:64-f80:128-n8:16:32:64-S128"
target triple = "x86_64-pc-linux-gnu"

; Function Attrs: mustprogress nofree norecurse nosync nounwind willreturn memory(none) uwtable
define dso_local i32 @foo(i32 noundef %0) local_unnamed_addr #0 {
  %2 = shl nsw i32 %0, 1
  ret i32 %2
}

; Function Attrs: mustprogress nofree norecurse nosync nounwind willreturn memory(none) uwtable
define dso_local i32 @bar(i32 noundef %0, i32 noundef %1) local_unnamed_addr #0 {
  %3 = shl i32 %1, 2
  %4 = add nsw i32 %3, %0
  ret i32 %4
}

; Function Attrs: mustprogress nofree norecurse nosync nounwind willreturn memory(none) uwtable
define dso_local i32 @fez(i32 noundef %0, i32 noundef %1, i32 noundef %2) local_unnamed_addr #0 {
  %4 = shl i32 %1, 3
  %5 = mul nsw i32 %2, 3
  %6 = mul i32 %0, 3
  %7 = add i32 %6, %4
  %8 = add nsw i32 %7, %5
  ret i32 %8
}

; Function Attrs: mustprogress nofree norecurse nosync nounwind willreturn memory(none) uwtable
define dso_local i32 @main(i32 noundef %0, ptr nocapture noundef readnone %1) local_unnamed_addr #0 {
  ret i32 12915
}

attributes #0 = { mustprogress nofree norecurse nosync nounwind willreturn memory(none) uwtable "min-legal-vector-width"="0" "no-trapping-math"="true" "stack-protector-buffer-size"="8" "target-cpu"="x86-64" "target-features"="+cmov,+cx8,+fxsr,+mmx,+sse,+sse2,+x87" "tune-cpu"="generic" }

!llvm.module.flags = !{!0, !1, !2, !3}
!llvm.ident = !{!4}

!0 = !{i32 1, !"wchar_size", i32 4}
!1 = !{i32 8, !"PIC Level", i32 2}
!2 = !{i32 7, !"PIE Level", i32 2}
!3 = !{i32 7, !"uwtable", i32 2}
!4 = !{!"Ubuntu clang version 17.0.6 (++20231209124227+6009708b4367-1~exp1~20231209124336.77)"}
```

最后把 IR 和 Pass 交给 opt，看看输出的效果如何

```shell
$ opt-17 -load-pass-plugin ./libHelloWorld.so -passes=hello-world -disable-output input_for_hello.ll
(llvm-tutor) Hello from: foo
(llvm-tutor)   number of arguments: 1
(llvm-tutor) Hello from: bar
(llvm-tutor)   number of arguments: 2
(llvm-tutor) Hello from: fez
(llvm-tutor)   number of arguments: 3
(llvm-tutor) Hello from: main
(llvm-tutor)   number of arguments: 2
```

可以看到 Pass 只是将函数名和参数数量输出了一遍

### Code Analysis

首先看 Pass 的主体

```c++
struct HelloWorld : PassInfoMixin<HelloWorld> {
  // Main entry point, takes IR unit to run the pass on (&F) and the
  // corresponding pass manager (to be queried if need be)
  PreservedAnalyses run(Function &F, FunctionAnalysisManager &) {
    visitor(F);
    return PreservedAnalyses::all();
  }

  // Without isRequired returning true, this pass will be skipped for functions
  // decorated with the optnone LLVM attribute. Note that clang -O0 decorates
  // all functions with optnone.
  static bool isRequired() { return true; }
};
```

可以看到 Pass 的主体`HelloWorld`继承自模板`PassInfoMixin<PassT>`。`PassInfoMixin`是新版的 PassManager 的实现，它是一个 CRTP 模板，Pass 需要继承它并实现`run`方法，接收一些 IR 单元和一个分析管理器，返回类型为`PreservedAnalyses`

- 旧版的 PM 是什么样的我没看不知道

- CRTP 是 C++ 里的一种很有趣的技巧，其特点就是先实现一个模板类，然后实现一个派生类，但派生类声明的父类类型是该派生类本身。这样做可以实现很多很有趣的特性，比如父类可以访问派生类的方法、实现静态多态（省去虚函数绑定的同时实现多态）、方便地进行子类计数等

  ```c++
  template <typename Derived>
  class Base {
      // ...
  };
  
  class Derived : public Base<Derived> {
      // ...
  };
  ```

- C++ 的 struct 和 class 除了成员的默认访问权限不同外完全一样，同样有构造函数和虚构函数，同样有 OOP 特性，但要注意的是 C++ 的结构体与 C 结构体的内存布局（用 C++ 的描述说，C struct 是 POD 类型的）已经完全不同，各个成员的内存存储不再是连续的，因此 C 也无法再兼容调用 C++ 实现的结构体，需要手动做一些兼容设计

继续看`visitor`方法，其实就是在 stderr 输出函数名和 argc

```c++
void visitor(Function &F) {
    errs() << "(llvm-tutor) Hello from: "<< F.getName() << "\n";
    errs() << "(llvm-tutor)   number of arguments: " << F.arg_size() << "\n";
}
```

最后是关于新 PM 下 Pass 的注册方面的内容。注册 Pass 的核心接口是要实现`llvmGetPassPluginInfo`函数。 这是一个无参并返回`PassPluginLibraryInfo`的函数。`PassPluginLibraryInfo`是一个纯 C 结构体（定义前带`extern "C"`，因此各成员在内存里是连续的），结构体内包含：32 位整数`APIVersion`、字符串`PluginName`、字符串`PluginVersion`和一个在 Pass 被装载时调用的回调函数指针

```c++
llvm::PassPluginLibraryInfo getHelloWorldPluginInfo() {
  return {LLVM_PLUGIN_API_VERSION, "HelloWorld", LLVM_VERSION_STRING,
          [](PassBuilder &PB) {
            PB.registerPipelineParsingCallback(
                [](StringRef Name, FunctionPassManager &FPM,
                   ArrayRef<PassBuilder::PipelineElement>) {
                  if (Name == "hello-world") {
                    FPM.addPass(HelloWorld());
                    return true;
                  }
                  return false;
                });
          }};
}

// This is the core interface for pass plugins. It guarantees that 'opt' will
// be able to recognize HelloWorld when added to the pass pipeline on the
// command line, i.e. via '-passes=hello-world'
extern "C" LLVM_ATTRIBUTE_WEAK ::llvm::PassPluginLibraryInfo
llvmGetPassPluginInfo() {
  return getHelloWorldPluginInfo();
}
```

- `extern "C"`用于告诉编译器以 C 的规则处理该处定义，由于 C++ 在 C 的基础上做了很多改动（比如编译时会修饰函数、结构体的内存布局等），于是用 `extern "C"`就可以告诉编译器此处使用 C 标准，从而解决 C/C++ 混合编程带来的兼容性问题

最后看看 Pass 的编译，不太懂 CMake，找了半天最后找到下面这条，其中`c++`是个指向`g++`的符号链接，所以最后 Pass 是被编译成一个带位置无关代码的动态链接库

```cmake
/usr/bin/c++ -fPIC -shared -Wl,-soname,libHelloWorld.so -o libHelloWorld.so CMakeFiles/HelloWorld.dir/HelloWorld.cpp.o
```

## OpcodeCounter

### First Taste

这次编译要将所有示例全部编译完

先将整个`build`删掉，然后重新生成 CMake 文件重新 make

```shell
rm -r build
mkdir build && cd build
cmake -DLT_LLVM_INSTALL_DIR=$LLVM_DIR ../
make
```

中间可能会缺库报错，比如我就缺了 libedit 和 libzstd

```shell
sudo apt install -y libedit-dev
sudo apt install -y libzstd-dev
```

然后在`build/lib`里就能看到所有编译完的 Pass 了

然后我们生成小白鼠的 bitcode，并运行 Pass

```shell
clang-17 -emit-llvm -c ../inputs/input_for_cc.c -o input_for_cc.bc
opt-17 -load-pass-plugin ./lib/libOpcodeCounter.so -passes="print<opcode-counter>" -disable-output input_for_cc.bc
```

小白鼠长这样

```c
void foo() { }
void bar() {foo(); }
void fez() {bar(); }

int main() {
  foo();
  bar();
  fez();

  int ii = 0;
  for (ii = 0; ii < 10; ii++)
    foo();

  return 0;
}
```

输出长这样，把每个函数里的各种指令统计了一遍

```
Printing analysis 'OpcodeCounter Pass' for function 'foo':
=================================================
LLVM-TUTOR: OpcodeCounter results
=================================================
OPCODE               #TIMES USED
-------------------------------------------------
ret                  1
-------------------------------------------------

Printing analysis 'OpcodeCounter Pass' for function 'bar':
=================================================
LLVM-TUTOR: OpcodeCounter results
=================================================
OPCODE               #TIMES USED
-------------------------------------------------
call                 1
ret                  1
-------------------------------------------------

Printing analysis 'OpcodeCounter Pass' for function 'fez':
=================================================
LLVM-TUTOR: OpcodeCounter results
=================================================
OPCODE               #TIMES USED
-------------------------------------------------
call                 1
ret                  1
-------------------------------------------------

Printing analysis 'OpcodeCounter Pass' for function 'main':
=================================================
LLVM-TUTOR: OpcodeCounter results
=================================================
OPCODE               #TIMES USED
-------------------------------------------------
add                  1
call                 4
ret                  1
load                 2
br                   4
alloca               2
store                4
icmp                 1
-------------------------------------------------
```

除了上面这种直接指定调用 Pass 的方法外，还可以通过指定优化级别（`-On`）来自动调用 Pass，前提是在 Pass 里写了对应的规则，后面会讲到

```shell
opt-17 -load-pass-plugin ./lib/libOpcodeCounter.so --passes='default<O1>' -disable-output input_for_cc.bc
```

### Code Analysis

代码开始变复杂了。首先看看注册部分，可以看到这里有三个注册操作

- 首先注册整个 Plugin，其可以通过命令行传入字符串`print<opcode-counter>`向 PM 添加 Pass
- 然后将`OpcodeCounterPrinter`注册进现有的 pipeline 的一部分，意思大概就是开优化时（`-O{1|2|3|s}`）自动调用它
- 最后将`OpcodeCounter`注册为 Analysis Pass，注册完其就可以被`FAM.getResult`调用了
  - 关于 Pass 下面的几个派生类，大体上分为 AnalysisPass、TransformPass 和 UtilityPass
    - AnalysisPass 用于计算相关 IR 单元的信息，但不做修改，通常作为查询和分析的接口供其他 Pass 使用
    - TransformPass 会检视 IR、调 AnalysisPass 查询 IR 的信息，然后按规则优化或改变 IR
    - UtilityPass 主要是一些功能性的程序，故单独将其分离出来一个派生类

```c++
llvm::PassPluginLibraryInfo getOpcodeCounterPluginInfo() {
  return {
    LLVM_PLUGIN_API_VERSION, "OpcodeCounter", LLVM_VERSION_STRING,
        [](PassBuilder &PB) {
          // #1 REGISTRATION FOR "opt -passes=print<opcode-counter>"
          // Register OpcodeCounterPrinter so that it can be used when
          // specifying pass pipelines with `-passes=`.
          PB.registerPipelineParsingCallback(
              [&](StringRef Name, FunctionPassManager &FPM,
                  ArrayRef<PassBuilder::PipelineElement>) {
                if (Name == "print<opcode-counter>") {
                  FPM.addPass(OpcodeCounterPrinter(llvm::errs()));
                  return true;
                }
                return false;
              });
          // #2 REGISTRATION FOR "-O{1|2|3|s}"
          // Register OpcodeCounterPrinter as a step of an existing pipeline.
          // The insertion point is specified by using the
          // 'registerVectorizerStartEPCallback' callback. To be more precise,
          // using this callback means that OpcodeCounterPrinter will be called
          // whenever the vectoriser is used (i.e. when using '-O{1|2|3|s}'.
          PB.registerVectorizerStartEPCallback(
              [](llvm::FunctionPassManager &PM,
                 llvm::OptimizationLevel Level) {
                PM.addPass(OpcodeCounterPrinter(llvm::errs()));
              });
          // #3 REGISTRATION FOR "FAM.getResult<OpcodeCounter>(Func)"
          // Register OpcodeCounter as an analysis pass. This is required so that
          // OpcodeCounterPrinter (or any other pass) can request the results
          // of OpcodeCounter.
          PB.registerAnalysisRegistrationCallback(
              [](FunctionAnalysisManager &FAM) {
                FAM.registerPass([&] { return OpcodeCounter(); });
              });
          }
        };
}

extern "C" LLVM_ATTRIBUTE_WEAK ::llvm::PassPluginLibraryInfo
llvmGetPassPluginInfo() {
  return getOpcodeCounterPluginInfo();
}
```

然后来看看两个 Pass 的主体

- `FAM.getResult`会在函数对象`Func`上运行模板指定的 Pass 的`run`方法，并将结果作为引用返回
  - Pass 的具体运作方式还得看看 PM 的源码
- `OpcodeCounterPrinter`的输出流`OS`在`includeOpcodeCounter.h`里有定义，是一个 LLVM 自己封装的输入输出流`raw_ostream`，`raw_ostream`不仅支持标准流，还支持文件流和 string 流，常用于内部打印
- 当 Pass 运行后，其有可能会改变 IR，`PreservedAnalyses`表示的就是哪些改变将被保留，`PreservedAnalyses::all()`表示所有分析结果都被保留

```c++
OpcodeCounter::Result OpcodeCounter::run(llvm::Function &Func,
                                         llvm::FunctionAnalysisManager &) {
  return generateOpcodeMap(Func);
}

PreservedAnalyses OpcodeCounterPrinter::run(Function &Func,
                                            FunctionAnalysisManager &FAM) {
  auto &OpcodeMap = FAM.getResult<OpcodeCounter>(Func);

  // In the legacy PM, the following string is printed automatically by the
  // pass manager. For the sake of consistency, we're adding this here so that
  // it's also printed when using the new PM.
  OS << "Printing analysis 'OpcodeCounter Pass' for function '"
     << Func.getName() << "':\n";

  printOpcodeCounterResult(OS, OpcodeMap);
  return PreservedAnalyses::all();
}
```

最后来看看 counter 的主逻辑

- `OpcodeMap`的具体定义在头文件里，是一个`llvm::StringMap<unsigned>`，LLVM 似乎把与 string 关联的很多数据结构自行封装了一遍
- 代码逻辑就是先遍历`Func`的 Basic Block，容纳后遍历 Basic Block 里的所有指令，获取指令的名称后在哈希表里做记录，最后返回存有结果的哈希表

```c++
OpcodeCounter::Result OpcodeCounter::generateOpcodeMap(llvm::Function &Func) {
  OpcodeCounter::Result OpcodeMap;

  for (auto &BB : Func) {
    for (auto &Inst : BB) {
      StringRef Name = Inst.getOpcodeName();

      if (OpcodeMap.find(Name) == OpcodeMap.end()) {
        OpcodeMap[Inst.getOpcodeName()] = 1;
      } else {
        OpcodeMap[Inst.getOpcodeName()]++;
      }
    }
  }

  return OpcodeMap;
}
```

综合前面的代码，可知这个 Pass 的粒度是单个 Function，于是其就会在 opt 将每个函数放进 pipeline 时都被调用一遍

## InjectFuncCall

### First Taste

这个 Pass 会在每个函数里插个桩，输出函数名和 argc

```c++
printf("(llvm-tutor) Hello from: %s\n(llvm-tutor)   number of arguments: %d\n", FuncName, FuncNumArgs)
```

首先还是生成 bitcode、过 pipeline

```shell
clang-17 -O0 -emit-llvm -c ../inputs/input_for_hello.c -o input_for_hello.bc
opt-17 -load-pass-plugin ./lib/libInjectFuncCall.so --passes="inject-func-call" input_for_hello.bc -o instrumented.bin
```

- 最后生成的 bin 文件似乎是 bitcode

然后我们可以用 lli 测试工具来运行它

```shell
$ lli-17 instrumented.bin
(llvm-tutor) Hello from: main
(llvm-tutor)   number of arguments: 2
(llvm-tutor) Hello from: foo
(llvm-tutor)   number of arguments: 1
(llvm-tutor) Hello from: bar
(llvm-tutor)   number of arguments: 2
(llvm-tutor) Hello from: foo
(llvm-tutor)   number of arguments: 1
(llvm-tutor) Hello from: fez
(llvm-tutor)   number of arguments: 3
(llvm-tutor) Hello from: bar
(llvm-tutor)   number of arguments: 2
(llvm-tutor) Hello from: foo
(llvm-tutor)   number of arguments: 1
```

- lli 还可以测试运行 IR 文件
- IR 和 bitcode 似乎并没有什么不同，当我没说（）

用 llvm-dis 工具可以将 bitcode 转回可读的 IR。diff 一下看看原来的 IR 和现在的有何不同

```diff
$ diff input_for_hello.ll instrumented.bin.ll
1c1
- ; ModuleID = 'input_for_hello.bc'
---
+ ; ModuleID = './instrumented.bin'
5a6,11
+ @PrintfFormatStr = global [68 x i8] c"(llvm-tutor) Hello from: %s\0A(llvm-tutor)   number of arguments: %d\0A\00"
+ @0 = private unnamed_addr constant [4 x i8] c"foo\00", align 1
+ @1 = private unnamed_addr constant [4 x i8] c"bar\00", align 1
+ @2 = private unnamed_addr constant [4 x i8] c"fez\00", align 1
+ @3 = private unnamed_addr constant [5 x i8] c"main\00", align 1
+
8,12c14,19
-   %2 = alloca i32, align 4
-   store i32 %0, ptr %2, align 4
-   %3 = load i32, ptr %2, align 4
-   %4 = mul nsw i32 %3, 2
-   ret i32 %4
---
+   %2 = call i32 (ptr, ...) @printf(ptr @PrintfFormatStr, ptr @0, i32 1)
+   %3 = alloca i32, align 4
+   store i32 %0, ptr %3, align 4
+   %4 = load i32, ptr %3, align 4
+   %5 = mul nsw i32 %4, 2
+   ret i32 %5
...
```

- 可以看到除了那些由于 SSA 改变的临时寄存器名外主要还是在函数前多了条`call @printf`

### Code Analysis

注册没什么特别的，那就先看看`run`。其实就是调用`runOnModule`处理 module，然后根据 IR 是否有所改变返回`llvm::PreservedAnalyses`

- 按照新 PM 的规定，`run`返回`llvm::PreservedAnalyses.all()`表示当前 Pass 没有修改任何 IR，也就意味着当前 Pass 执行后所有分析结果依然有效，这是新 PM 用来提高效率的做法

```c++
PreservedAnalyses InjectFuncCall::run(llvm::Module &M,
                                       llvm::ModuleAnalysisManager &) {
  bool Changed =  runOnModule(M);

  return (Changed ? llvm::PreservedAnalyses::none()
                  : llvm::PreservedAnalyses::all());
}
```

然后看看主逻辑

- step 0
  - 首先获取 module 的 context
  - 然后`Type::getInt8Ty`从 context 获取一个 8 比特（1 字节）的 int 类型，即 char 类型
  - `PointerType::getUnqual`构造一个指向 0 地址的指定类型的指针，代表`printf`的第一个参数的类型`char *`
- step 1
  - `FunctionType::get`用于创建一个新的函数类型，第一个参数是函数的返回类型（这里是 32 位的 int），第二个参数是一个参数类型的列表（这里只指明了第一个参数的类型`char *`），第三个参数表明函数是否是可变参数的
  - `getOrInsertFunction`用于在 module 的符号表里查找同名函数，如果不存在则插入，存在则比对 prototype 并用旧的 prototype 替换之，返回`FunctionCallee`
  - 然后设置 printf 的一些属性，从上往下是：
    - printf 不会抛出异常，可以删掉异常处理代码
    - 第一个参数添加属性`NoCapture`，即参数不会被任何函数内部的指针捕获，方便优化时别名分析
    - 第一个参数填加属性`ReadOnly`，方便优化
- step 2
  - 创建全局变量`PrintfFormatStr`，用于保存 printf 的格式化字符串`PrintfFormatStr`
  - `getOrInsertGlobal`将全局变量`PrintfFormatStrVar`添加进 module 中，同样如果曾经声明过旧用旧定义替换，这里`PrintfFormatStrVar`的类型应该是一个 array
  - `setInitializer`将变量`PrintfFormatStrVar`的初始值设置为`PrintfFormatStr`
- step 3
  - 遍历 module 里的 function
  - 判断这个 function 是函数声明还是函数定义，如果是声明旧跳过了
  - 然后创建一个 IR Builder，位置在 function 的第一个 basic block 的第一个插入点（返回第一个指令的迭代器）
  - 随后创建一个内容是函数名的全局字符串变量
  - 随后创建一个格式化字符串的局部变量`formatStr`，`Builder.CreatePointerCast`将 array 类型的`PrintfFormatStrVar`转换为`char *`类型以适应 printf 的定义
  - 最后调用`Builder.CreateCall`创建一个 printf 的调用，并将是否已插桩标记为 True

```c++
bool InjectFuncCall::runOnModule(Module &M) {
  bool InsertedAtLeastOnePrintf = false;

  auto &CTX = M.getContext();
  PointerType *PrintfArgTy = PointerType::getUnqual(Type::getInt8Ty(CTX));

  // STEP 1: Inject the declaration of printf
  // ----------------------------------------
  // Create (or _get_ in cases where it's already available) the following
  // declaration in the IR module:
  //    declare i32 @printf(i8*, ...)
  // It corresponds to the following C declaration:
  //    int printf(char *, ...)
  FunctionType *PrintfTy = FunctionType::get(
      IntegerType::getInt32Ty(CTX),
      PrintfArgTy,
      /*IsVarArgs=*/true);

  FunctionCallee Printf = M.getOrInsertFunction("printf", PrintfTy);

  // Set attributes as per inferLibFuncAttributes in BuildLibCalls.cpp
  Function *PrintfF = dyn_cast<Function>(Printf.getCallee());
  PrintfF->setDoesNotThrow();
  PrintfF->addParamAttr(0, Attribute::NoCapture);
  PrintfF->addParamAttr(0, Attribute::ReadOnly);


  // STEP 2: Inject a global variable that will hold the printf format string
  // ------------------------------------------------------------------------
  llvm::Constant *PrintfFormatStr = llvm::ConstantDataArray::getString(
      CTX, "(llvm-tutor) Hello from: %s\n(llvm-tutor)   number of arguments: %d\n");

  Constant *PrintfFormatStrVar =
      M.getOrInsertGlobal("PrintfFormatStr", PrintfFormatStr->getType());
  dyn_cast<GlobalVariable>(PrintfFormatStrVar)->setInitializer(PrintfFormatStr);

  // STEP 3: For each function in the module, inject a call to printf
  // ----------------------------------------------------------------
  for (auto &F : M) {
    if (F.isDeclaration())
      continue;

    // Get an IR builder. Sets the insertion point to the top of the function
    IRBuilder<> Builder(&*F.getEntryBlock().getFirstInsertionPt());

    // Inject a global variable that contains the function name
    auto FuncName = Builder.CreateGlobalStringPtr(F.getName());

    // Printf requires i8*, but PrintfFormatStrVar is an array: [n x i8]. Add
    // a cast: [n x i8] -> i8*
    llvm::Value *FormatStrPtr =
        Builder.CreatePointerCast(PrintfFormatStrVar, PrintfArgTy, "formatStr");

    // The following is visible only if you pass -debug on the command line
    // *and* you have an assert build.
    LLVM_DEBUG(dbgs() << " Injecting call to printf inside " << F.getName()
                      << "\n");

    // Finally, inject a call to printf
    Builder.CreateCall(
        Printf, {FormatStrPtr, FuncName, Builder.getInt32(F.arg_size())});

    InsertedAtLeastOnePrintf = true;
  }

  return InsertedAtLeastOnePrintf;
}
```

以上就完成了一次简单的插桩（这么多类型和 API 谁记得住啊）

## StaticCallCounter

### First Taste

所谓 Static Call 指的就是在编译期的调用，相反在运行时的调用就是 Dynamic Call，比如在循环里有函数调用，Static Call 就只会计数一次，而 Dynamic Call 的计数则取决于循环次数

```c
for (ii = 0; ii < 10; ii++)
	foo();
```

用的还是之前的 bitcode，这里直接跳到 opt

```shell
opt-17 -load-pass-plugin ./lib/libStaticCallCounter.so -passes="print<static-cc>" -disable-output input_for_cc.bc
```

输出如下

```
=================================================
LLVM-TUTOR: static analysis results
=================================================
NAME                 #N DIRECT CALLS
-------------------------------------------------
foo                  3
bar                  2
fez                  1
-------------------------------------------------
```

### Code Analysis

其实没什么好说的，Static Call Counter 的流程和前面的非常相像，这里简单分析下

首先注册 Printer 和 Counter

```c++
llvm::PassPluginLibraryInfo getStaticCallCounterPluginInfo() {
  return {LLVM_PLUGIN_API_VERSION, "static-cc", LLVM_VERSION_STRING,
          [](PassBuilder &PB) {
            // #1 REGISTRATION FOR "opt -passes=print<static-cc>"
            PB.registerPipelineParsingCallback(
                [&](StringRef Name, ModulePassManager &MPM,
                    ArrayRef<PassBuilder::PipelineElement>) {
                  if (Name == "print<static-cc>") {
                    MPM.addPass(StaticCallCounterPrinter(llvm::errs()));
                    return true;
                  }
                  return false;
                });
            // #2 REGISTRATION FOR "MAM.getResult<StaticCallCounter>(Module)"
            PB.registerAnalysisRegistrationCallback(
                [](ModuleAnalysisManager &MAM) {
                  MAM.registerPass([&] { return StaticCallCounter(); });
                });
          }};
};
```

然后直接看 Counter

- 其实就是遍历 module 里的 function 里的 Basic Block 里的指令
- 函数调用用`CallBase`类表示，尝试 cast 转换一下就知道指令是不是函数调用了
- `getCalledFunction()`方法获取被调用的函数
- 最后在哈希表里处理一下就结束了

```c++
StaticCallCounter::Result StaticCallCounter::runOnModule(Module &M) {
  llvm::MapVector<const llvm::Function *, unsigned> Res;

  for (auto &Func : M) {
    for (auto &BB : Func) {
      for (auto &Ins : BB) {

        // If this is a call instruction then CB will be not null.
        auto *CB = dyn_cast<CallBase>(&Ins);
        if (nullptr == CB) {
          continue;
        }

        // If CB is a direct function call then DirectInvoc will be not null.
        auto DirectInvoc = CB->getCalledFunction();
        if (nullptr == DirectInvoc) {
          continue;
        }

        // We have a direct function call - update the count for the function
        // being called.
        auto CallCount = Res.find(DirectInvoc);
        if (Res.end() == CallCount) {
          CallCount = Res.insert(std::make_pair(DirectInvoc, 0)).first;
        }
        ++CallCount->second;
      }
    }
  }

  return Res;
}

StaticCallCounter::Result
StaticCallCounter::run(llvm::Module &M, llvm::ModuleAnalysisManager &) {
  return runOnModule(M);
}
```

至于函数指针这种能不能分析出来就不知道了

## DynamicCallCounter

### First Taste

```shell
opt-17 -load-pass-plugin ./lib/libDynamicCallCounter.so -passes="dynamic-cc" input_for_cc.bc -o instrumented.bin
lli-17 instrumented.bin
```

输出如下

```
=================================================
LLVM-TUTOR: dynamic analysis results
=================================================
NAME                 #N DIRECT CALLS
-------------------------------------------------
bar                  2
main                 1
foo                  13
fez                  1
```

可以发现 Dynamic Counter 比 Static Counter 给 foo 的统计多了循环里的部分，而且 main 也有了一次计数

### Code Analysis

直接看主体部分`runOnModule`。代码比较长，这里分几步贴出

step 0 先建两个哈希表，一个用于保存计数，另一个用于保存函数名，最后获取 context

```c++
bool Instrumented = false;

// Function name <--> IR variable that holds the call counter
llvm::StringMap<Constant *> CallCounterMap;
// Function name <--> IR variable that holds the function name
llvm::StringMap<Constant *> FuncNameMap;

auto &CTX = M.getContext();
```

step 1

- 开始遍历 module 内的每个函数（跳过函数声明）
- 在函数首部建一个 IR Builder，然后给函数建一个 string，调用`CreateGlobalCounter`创建用于计数的全局变量，并将函数名和计数器变量记录在 step 0 创建的哈希表`CallCounterMap`中
- 最后用 Builder 插入 load、add、store 指令，并将插桩 flag 置为真

```c++
// STEP 1: For each function in the module, inject a call-counting code
// --------------------------------------------------------------------
for (auto &F : M) {
if (F.isDeclaration())
  continue;

// Get an IR builder. Sets the insertion point to the top of the function
IRBuilder<> Builder(&*F.getEntryBlock().getFirstInsertionPt());

// Create a global variable to count the calls to this function
std::string CounterName = "CounterFor_" + std::string(F.getName());
Constant *Var = CreateGlobalCounter(M, CounterName);
CallCounterMap[F.getName()] = Var;

// Create a global variable to hold the name of this function
auto FuncName = Builder.CreateGlobalStringPtr(F.getName());
FuncNameMap[F.getName()] = FuncName;

// Inject instruction to increment the call count each time this function
// executes
LoadInst *Load2 = Builder.CreateLoad(IntegerType::getInt32Ty(CTX), Var);
Value *Inc2 = Builder.CreateAdd(Builder.getInt32(1), Load2);
Builder.CreateStore(Inc2, Var);

// The following is visible only if you pass -debug on the command line
// *and* you have an assert build.
LLVM_DEBUG(dbgs() << " Instrumented: " << F.getName() << "\n");

Instrumented = true;
}

// Stop here if there are no function definitions in this module
if (false == Instrumented)
return Instrumented;
```

step 2、step 3

- 其实就是构造一个 printf 的原型并设置好相关属性，然后构造格式化字符串全局变量，此前讲述过

```c++
// STEP 2: Inject the declaration of printf
// ----------------------------------------
// Create (or _get_ in cases where it's already available) the following
// declaration in the IR module:
//    declare i32 @printf(i8*, ...)
// It corresponds to the following C declaration:
//    int printf(char *, ...)
PointerType *PrintfArgTy = PointerType::getUnqual(Type::getInt8Ty(CTX));
FunctionType *PrintfTy =
  FunctionType::get(IntegerType::getInt32Ty(CTX), PrintfArgTy,
                    /*IsVarArgs=*/true);

FunctionCallee Printf = M.getOrInsertFunction("printf", PrintfTy);

// Set attributes as per inferLibFuncAttributes in BuildLibCalls.cpp
Function *PrintfF = dyn_cast<Function>(Printf.getCallee());
PrintfF->setDoesNotThrow();
PrintfF->addParamAttr(0, Attribute::NoCapture);
PrintfF->addParamAttr(0, Attribute::ReadOnly);

// STEP 3: Inject a global variable that will hold the printf format string
// ------------------------------------------------------------------------
llvm::Constant *ResultFormatStr =
  llvm::ConstantDataArray::getString(CTX, "%-20s %-10lu\n");

Constant *ResultFormatStrVar =
  M.getOrInsertGlobal("ResultFormatStrIR", ResultFormatStr->getType());
dyn_cast<GlobalVariable>(ResultFormatStrVar)->setInitializer(ResultFormatStr);

std::string out = "";
out += "=================================================\n";
out += "LLVM-TUTOR: dynamic analysis results\n";
out += "=================================================\n";
out += "NAME                 #N DIRECT CALLS\n";
out += "-------------------------------------------------\n";

llvm::Constant *ResultHeaderStr =
  llvm::ConstantDataArray::getString(CTX, out.c_str());

Constant *ResultHeaderStrVar =
  M.getOrInsertGlobal("ResultHeaderStrIR", ResultHeaderStr->getType());
dyn_cast<GlobalVariable>(ResultHeaderStrVar)->setInitializer(ResultHeaderStr);
```

step 4、step 5

- 统计的结果需要被输出出来，这里构造了一个函数`printf_wrapper`，函数会依次输出函数名和引用计数（不过代码是直接展开的，没有设置循环，不是很智能）
- 最后调用`appendToGlobalDtors`函数，将 printf_wrapper 添加到全局析构函数列表（在 ELF section `.fini_array`中，这是 ELF 的析构函数表，会被`__libc_csu_fini`调用），这样在程序正常退出时 printf_wrapper 就会被调用

```c++
// STEP 4: Define a printf wrapper that will print the results
// -----------------------------------------------------------
// Define `printf_wrapper` that will print the results stored in FuncNameMap
// and CallCounterMap.  It is equivalent to the following C++ function:
// ```
//    void printf_wrapper() {
//      for (auto &item : Functions)
//        printf("llvm-tutor): Function %s was called %d times. \n",
//        item.name, item.count);
//    }
// ```
// (item.name comes from FuncNameMap, item.count comes from
// CallCounterMap)
FunctionType *PrintfWrapperTy =
  FunctionType::get(llvm::Type::getVoidTy(CTX), {},
                    /*IsVarArgs=*/false);
Function *PrintfWrapperF = dyn_cast<Function>(
  M.getOrInsertFunction("printf_wrapper", PrintfWrapperTy).getCallee());

// Create the entry basic block for printf_wrapper ...
llvm::BasicBlock *RetBlock =
  llvm::BasicBlock::Create(CTX, "enter", PrintfWrapperF);
IRBuilder<> Builder(RetBlock);

// ... and start inserting calls to printf
// (printf requires i8*, so cast the input strings accordingly)
llvm::Value *ResultHeaderStrPtr =
  Builder.CreatePointerCast(ResultHeaderStrVar, PrintfArgTy);
llvm::Value *ResultFormatStrPtr =
  Builder.CreatePointerCast(ResultFormatStrVar, PrintfArgTy);

Builder.CreateCall(Printf, {ResultHeaderStrPtr});

LoadInst *LoadCounter;
for (auto &item : CallCounterMap) {
LoadCounter = Builder.CreateLoad(IntegerType::getInt32Ty(CTX), item.second);
// LoadCounter = Builder.CreateLoad(item.second);
Builder.CreateCall(
    Printf, {ResultFormatStrPtr, FuncNameMap[item.first()], LoadCounter});
}

// Finally, insert return instruction
Builder.CreateRetVoid();

// STEP 5: Call `printf_wrapper` at the very end of this module
// ------------------------------------------------------------
appendToGlobalDtors(M, PrintfWrapperF, /*Priority=*/0);

return true;
```

一开始我不太理解代码中所说的“将代码添加到 module 的末尾”的含义，于是将 bitcode 编译为汇编代码

```shell
llc-17 -filetype=asm instrumented.bin -o instrumented.s
```

汇编代码的最后有这么一段，这段汇编的含义是：定义可写可分配的`.fini_array`段、然后用`NOP`对齐、最后在段内存储`printf_wrapper`的地址，也即我们前面分析的意思

```asm
.section	.fini_array.0,"aw",@fini_array
.p2align	3, 0x90
.quad	printf_wrapper
```

## MBASub

Mixed Boolean Arithmetic Transformations

### First Taste

```shell
clang-17 -emit-llvm -S ../inputs/input_for_mba_sub.c -o input_for_sub.ll
opt-17 -load-pass-plugin ./lib/libMBASub.so -passes="mba-sub" -S input_for_sub.ll -o out.ll
```

先来看看原来的 IR，还只是正常的 Sub 指令完成减法

```llvm
define dso_local i32 @main(i32 noundef %0, ptr noundef %1) #0 {
  ...
  %30 = sub nsw i32 %28, %29
  ...
  %33 = sub nsw i32 %31, %32
  ...
  %36 = sub nsw i32 %34, %35
  ret i32 %36
}
```

然后看看后面的，现在减法已经混淆成这种形式：`a-b = (a+~b)+1`

```llvm
define dso_local i32 @main(i32 noundef %0, ptr noundef %1) #0 {
  ...
  %30 = xor i32 %29, -1
  %31 = add i32 %28, %30
  %32 = add i32 %31, 1
  ...
  %35 = xor i32 %34, -1
  %36 = add i32 %33, %35
  %37 = add i32 %36, 1
  ...
  %40 = xor i32 %39, -1
  %41 = add i32 %38, %40
  %42 = add i32 %41, 1
  ret i32 %42
}
```

### Code Analysis

- 遍历 Basic Block 里的每条指令，提取其中的所有 Binary Operator，并细化提取所有的 Sub 指令
- 然后将`a+b`转换为`(a+~b)+1`，`getOperand`函数可以提取 Binary Operator 的 LHS 和 RHS
- 最后调用`ReplaceInstWithInst`将原来的 Sub 指令替换为刚刚构造的指令

```c++
bool MBASub::runOnBasicBlock(BasicBlock &BB) {
  bool Changed = false;

  // Loop over all instructions in the block. Replacing instructions requires
  // iterators, hence a for-range loop wouldn't be suitable.
  for (auto Inst = BB.begin(), IE = BB.end(); Inst != IE; ++Inst) {

    // Skip non-binary (e.g. unary or compare) instruction.
    auto *BinOp = dyn_cast<BinaryOperator>(Inst);
    if (!BinOp)
      continue;

    /// Skip instructions other than integer sub.
    unsigned Opcode = BinOp->getOpcode();
    if (Opcode != Instruction::Sub || !BinOp->getType()->isIntegerTy())
      continue;

    // A uniform API for creating instructions and inserting
    // them into basic blocks.
    IRBuilder<> Builder(BinOp);

    // Create an instruction representing (a + ~b) + 1
    Instruction *NewValue = BinaryOperator::CreateAdd(
        Builder.CreateAdd(BinOp->getOperand(0),
                          Builder.CreateNot(BinOp->getOperand(1))),
        ConstantInt::get(BinOp->getType(), 1));

    // The following is visible only if you pass -debug on the command line
    // *and* you have an assert build.
    LLVM_DEBUG(dbgs() << *BinOp << " -> " << *NewValue << "\n");

    // Replace `(a - b)` (original instructions) with `(a + ~b) + 1`
    // (the new instruction)
    ReplaceInstWithInst(&BB, Inst, NewValue);
    Changed = true;

    // Update the statistics
    ++SubstCount;
  }
  return Changed;
}
```

## MBAAdd

Mixed Boolean Arithmetic Transformations

### First Taste

```shell
clang-17 -O1 -emit-llvm -S ../inputs/input_for_mba.c -o input_for_mba.ll
opt-17 -load-pass-plugin ./lib/libMBAAdd.so -passes="mba-add" -S input_for_mba.ll -o out.ll
```

然后看看 IR 有何变化。可以看到普通的 Add 指令已经变成这种形式：`a + b = (((a ^ b) + 2 * (a & b)) * 39 + 23) * 151 + 111`

```diff
$ diff input_for_mba.ll out.ll
1c1
- ; ModuleID = '../inputs/input_for_mba.c'
---
+ ; ModuleID = 'input_for_mba.ll'
8,11c8,32
-   %5 = add i8 %1, %0
-   %6 = add i8 %5, %2
-   %7 = add i8 %6, %3
-   ret i8 %7
---
+   %5 = and i8 %1, %0
+   %6 = mul i8 2, %5
+   %7 = xor i8 %1, %0
+   %8 = add i8 %7, %6
+   %9 = mul i8 39, %8
+   %10 = add i8 23, %9
+   %11 = mul i8 -105, %10
+   %12 = add i8 111, %11
+   %13 = and i8 %12, %2
+   %14 = mul i8 2, %13
+   %15 = xor i8 %12, %2
+   %16 = add i8 %15, %14
+   %17 = mul i8 39, %16
+   %18 = add i8 23, %17
+   %19 = mul i8 -105, %18
+   %20 = add i8 111, %19
+   %21 = and i8 %20, %3
+   %22 = mul i8 2, %21
+   %23 = xor i8 %20, %3
+   %24 = add i8 %23, %22
+   %25 = mul i8 39, %24
+   %26 = add i8 23, %25
+   %27 = mul i8 -105, %26
+   %28 = add i8 111, %27
+   ret i8 %28
```

### Code Analysis

这里和 MBASub 逻辑是一致的，直接看代码即可理解。注意这种替换混淆只对 8 位整数有效，故需要判断 Binary Operator 操作的类型

```c++
bool MBAAdd::runOnBasicBlock(BasicBlock &BB) {
  bool Changed = false;
  
  // Loop over all instructions in the block. Replacing instructions requires
  // iterators, hence a for-range loop wouldn't be suitable
  for (auto Inst = BB.begin(), IE = BB.end(); Inst != IE; ++Inst) {
    // Skip non-binary (e.g. unary or compare) instructions
    auto *BinOp = dyn_cast<BinaryOperator>(Inst);
    if (!BinOp)
      continue;

    // Skip instructions other than add
    if (BinOp->getOpcode() != Instruction::Add)
      continue;

    // Skip if the result is not 8-bit wide (this implies that the operands are
    // also 8-bit wide)
    if (!BinOp->getType()->isIntegerTy() ||
        !(BinOp->getType()->getIntegerBitWidth() == 8))
      continue;

    // A uniform API for creating instructions and inserting
    // them into basic blocks
    IRBuilder<> Builder(BinOp);

    // Constants used in building the instruction for substitution
    auto Val39 = ConstantInt::get(BinOp->getType(), 39);
    auto Val151 = ConstantInt::get(BinOp->getType(), 151);
    auto Val23 = ConstantInt::get(BinOp->getType(), 23);
    auto Val2 = ConstantInt::get(BinOp->getType(), 2);
    auto Val111 = ConstantInt::get(BinOp->getType(), 111);

    // Build an instruction representing `(((a ^ b) + 2 * (a & b)) * 39 + 23) *
    // 151 + 111`
    Instruction *NewInst =
        // E = e5 + 111
        BinaryOperator::CreateAdd(
            Val111,
            // e5 = e4 * 151
            Builder.CreateMul(
                Val151,
                // e4 = e2 + 23
                Builder.CreateAdd(
                    Val23,
                    // e3 = e2 * 39
                    Builder.CreateMul(
                        Val39,
                        // e2 = e0 + e1
                        Builder.CreateAdd(
                            // e0 = a ^ b
                            Builder.CreateXor(BinOp->getOperand(0),
                                              BinOp->getOperand(1)),
                            // e1 = 2 * (a & b)
                            Builder.CreateMul(
                                Val2, Builder.CreateAnd(BinOp->getOperand(0),
                                                        BinOp->getOperand(1))))
                    ) // e3 = e2 * 39
                ) // e4 = e2 + 23
            ) // e5 = e4 * 151
        ); // E = e5 + 111

    // The following is visible only if you pass -debug on the command line
    // *and* you have an assert build.
    LLVM_DEBUG(dbgs() << *BinOp << " -> " << *NewInst << "\n");

    // Replace `(a + b)` (original instructions) with `(((a ^ b) + 2 * (a & b))
    // * 39 + 23) * 151 + 111` (the new instruction)
    ReplaceInstWithInst(&BB, Inst, NewInst);
    Changed = true;

    // Update the statistics
    ++SubstCount;
  }
  return Changed;
}
```

## RIV

### First Taste

原 README 没怎么将 RIV 是什么，Google 也没查到解释，最后 GPT 读完源码后给了这么一个解释：

> RIV（Reachable Integer Values）的目标是为输入函数中的每个基本块创建一个可达整数值的列表。这个列表包含了从该基本块开始，可以通过控制流图（CFG）到达的所有整数值。"可达"的含义是，从一个基本块开始，通过控制流图（CFG）的边，可以到达另一个基本块。如果一个整数值在一个基本块中被定义（例如，通过一个指令），并且这个基本块可以从另一个基本块到达，那么我们就说这个整数值对于那个基本块是"可达"的。
>
> RIV 的分析结果可以用于优化，例如常量传播和死代码消除。例如，如果一个基本块的 RIV 列表中没有一个特定的整数值，那么我们就知道在那个基本块中，没有任何代码可以引用那个整数值，因此我们可以安全地删除定义那个整数值的代码。

emmm 看着挺有说服力就姑且相信它吧（）

源码长这样

```c
int foo(int a, int b, int c) {
  int result = 123 + a;

  if (a > 0) {
    int d = a * b;
    int e = b / c;
    if (d == e) {
      int f = d * e;
      result = result - 2*f;
    } else {
      int g = 987;
      result = g * c * e;
    }
  } else {
    result = 321;
  }

  return result;
}
```

运行一下 Pass

```shell
clang-17 -O1 -emit-llvm -S ../inputs/input_for_riv.c -o input_for_riv.ll
opt-17 -load-pass-plugin ./lib/libRIV.so -passes="print<riv>" -disable-output input_for_riv.ll
```

原来的 IR 长这样

```llvm
define dso_local i32 @foo(i32 noundef %0, i32 noundef %1, i32 noundef %2) local_unnamed_addr #0 {
  %4 = add nsw i32 %0, 123
  %5 = icmp sgt i32 %0, 0
  br i1 %5, label %6, label %17

6:                                                ; preds = %3
  %7 = mul nsw i32 %1, %0
  %8 = sdiv i32 %1, %2
  %9 = icmp eq i32 %7, %8
  br i1 %9, label %10, label %14

10:                                               ; preds = %6
  %11 = mul i32 %7, -2
  %12 = mul i32 %11, %8
  %13 = add i32 %4, %12
  br label %17

14:                                               ; preds = %6
  %15 = mul nsw i32 %2, 987
  %16 = mul nsw i32 %15, %8
  br label %17

17:                                               ; preds = %3, %10, %14
  %18 = phi i32 [ %13, %10 ], [ %16, %14 ], [ 321, %3 ]
  ret i32 %18
}
```

输出如下

```llvm
=================================================
LLVM-TUTOR: RIV analysis results
=================================================
BB id      Reachable Ineger Values
-------------------------------------------------
BB %3
             i32 %0
             i32 %1
             i32 %2
BB %6
               %4 = add nsw i32 %0, 123
               %5 = icmp sgt i32 %0, 0
             i32 %0
             i32 %1
             i32 %2
BB %17
               %4 = add nsw i32 %0, 123
               %5 = icmp sgt i32 %0, 0
             i32 %0
             i32 %1
             i32 %2
BB %10
               %7 = mul nsw i32 %1, %0
               %8 = sdiv i32 %1, %2
               %9 = icmp eq i32 %7, %8
               %4 = add nsw i32 %0, 123
               %5 = icmp sgt i32 %0, 0
             i32 %0
             i32 %1
             i32 %2
BB %14
               %7 = mul nsw i32 %1, %0
               %8 = sdiv i32 %1, %2
               %9 = icmp eq i32 %7, %8
               %4 = add nsw i32 %0, 123
               %5 = icmp sgt i32 %0, 0
             i32 %0
             i32 %1
             i32 %2
```

感觉还是不是很清晰，我的理解如下

- 基本块`BB %3`中可达的只有参数中的`%0`、`%1`和`%2`
  - `BB %3`的后继为`BB %6`和`BB %17`
- 而由于基本块`BB %6`是`BB %3`的后继，所以继承参数`%0`、`%1`和`%2`，以及`BB %3`中定义的`%4`和`%5`
  - `BB %6`的后继为`BB %10`和`BB %14`
- 后面的基本块同样如此

### Code Analysis

- step 0

  - 定义了一个双端队列，用于后续遍历 Basic Block

  - `NodeTy`是一个指向包含一个 Basic Block 的支配树节点的指针

  - > 支配树（Dominator Tree）是一种在编译器优化和静态程序分析中常用的数据结构，它表示了程序的控制流图（Control Flow Graph，CFG）中的支配关系。
    >
    > 在控制流图中，一个节点 A 支配另一个节点 B，如果从入口节点到节点 B 的所有路径都必须经过节点 A。换句话说，如果我们从程序的入口开始执行，那么在每次到达 B 之前，我们都必须经过 A。
    >
    > 支配树是将这种支配关系表示为树形结构的一种方式。在支配树中，每个节点都支配它的所有子节点。这种结构使得查询一个节点是否支配另一个节点变得非常高效。

- step 1

  - 遍历 function 内部的所有 Basic Block，再遍历其内部的所有指令
  - 将所有整数变量的定义加入`llvm::MapVector`类型的`DefinedValuesMap`中
    - `llvm::MapVector`是一个通过将 Map 和 Vector 包装在一起实现一键多值存储的数据结构

- step 2

  - 首先第一个循环遍历了 function 所在 module（`getParent`）内的全局变量，其中所有整数变量会被加入入口基本块的 RIV 集合哈希表`ResultMap`中
  - 然后第二个循环遍历函数参数列表，并将整数形参加入入口基本块的 RIV 集合中

- step 3

  - 从入口基本块开始遍历 CFG，这里是通过向双端队列尾部 push 子基本块来完成遍历
  - 从之前的`DefinedValuesMap`中获取本基本块内定义的整数变量
  - 再从哈希表`ResultMap`中获取对于本基本块而言的全局变量
  - 最后遍历子基本块，将本基本块的 RIV 添加到子基本块的 RIV 中，并将子基本块添加进要遍历的基本块队列中

```c++
// DominatorTree node types used in RIV. One could use auto instead, but IMO
// being verbose makes it easier to follow.
using NodeTy = DomTreeNodeBase<llvm::BasicBlock> *;
// A map that a basic block BB holds a set of pointers to values defined in BB.
using DefValMapTy = RIV::Result;

RIV::Result RIV::buildRIV(Function &F, NodeTy CFGRoot) {
  Result ResultMap;

  // Initialise a double-ended queue that will be used to traverse all BBs in F
  std::deque<NodeTy> BBsToProcess;
  BBsToProcess.push_back(CFGRoot);

  // STEP 1: For every basic block BB compute the set of integer values defined
  // in BB
  DefValMapTy DefinedValuesMap;
  for (BasicBlock &BB : F) {
    auto &Values = DefinedValuesMap[&BB];
    for (Instruction &Inst : BB)
      if (Inst.getType()->isIntegerTy())
        Values.insert(&Inst);
  }

  // STEP 2: Compute the RIVs for the entry BB. This will include global
  // variables and input arguments.
  auto &EntryBBValues = ResultMap[&F.getEntryBlock()];

  for (auto &Global : F.getParent()->globals())
    if (Global.getValueType()->isIntegerTy())
      EntryBBValues.insert(&Global);

  for (Argument &Arg : F.args())
    if (Arg.getType()->isIntegerTy())
      EntryBBValues.insert(&Arg);

  // STEP 3: Traverse the CFG for every BB in F calculate its RIVs
  while (!BBsToProcess.empty()) {
    auto *Parent = BBsToProcess.back();
    BBsToProcess.pop_back();

    // Get the values defined in Parent
    auto &ParentDefs = DefinedValuesMap[Parent->getBlock()];
    // Get the RIV set of for Parent
    // (Since RIVMap is updated on every iteration, its contents are likely to
    // be moved around when resizing. This means that we need a copy of it
    // (i.e. a reference is not sufficient).
    llvm::SmallPtrSet<llvm::Value *, 8> ParentRIVs =
        ResultMap[Parent->getBlock()];

    // Loop over all BBs that Parent dominates and update their RIV sets
    for (NodeTy Child : *Parent) {
      BBsToProcess.push_back(Child);
      auto ChildBB = Child->getBlock();

      // Add values defined in Parent to the current child's set of RIV
      ResultMap[ChildBB].insert(ParentDefs.begin(), ParentDefs.end());

      // Add Parent's set of RIVs to the current child's RIV
      ResultMap[ChildBB].insert(ParentRIVs.begin(), ParentRIVs.end());
    }
  }

  return ResultMap;
}
```

## DuplicateBB

### First Taste

这个 Pass 的功能似乎是将有 RIV 的基本块复制成两份并用 if-then-else 结构串起来

```
BEFORE:                     AFTER:
-------                     ------
                              [ if-then-else ]
             DuplicateBB           /  \
[ BB ]      ------------>   [clone 1] [clone 2]
                                   \  /
                                 [ tail ]

LEGEND:
-------
[BB]           - the original basic block
[if-then-else] - a new basic block that contains the if-then-else statement (inserted by DuplicateBB)
[clone 1|2]    - two new basic blocks that are clones of BB (inserted by DuplicateBB)
[tail]         - the new basic block that merges [clone 1] and [clone 2] (inserted by DuplicateBB)
```

先跑起来看看

```shell
clang-17 -O1 -emit-llvm -S ../inputs/input_for_duplicate_bb.c -o input_for_duplicate_bb.ll
opt-17 -load-pass-plugin ./lib/libRIV.so -load-pass-plugin ./lib/libDuplicateBB.so -passes=duplicate-bb -S input_for_duplicate_bb.ll -o duplicate.ll
```

原来的 IR 只有一个最基本的 Basic Block

```llvm
define dso_local i32 @foo(i32 noundef %0) local_unnamed_addr #0 {
  ret i32 1
}
```

跑过 Pass 之后变成下面的样子了

```llvm
define dso_local i32 @foo(i32 noundef %0) local_unnamed_addr #0 {
lt-if-then-else-0:
  %1 = icmp eq i32 %0, 0
  br i1 %1, label %lt-clone-1-0, label %lt-clone-2-0

lt-clone-1-0:                                     ; preds = %lt-if-then-else-0
  br label %lt-tail-0

lt-clone-2-0:                                     ; preds = %lt-if-then-else-0
  br label %lt-tail-0

lt-tail-0:                                        ; preds = %lt-clone-2-0, %lt-clone-1-0
  ret i32 1
}
```

不知道啥作用，嗯

### Code Analysis

`run`主要是调用`findBBsToDuplicate`分析 RIV 的分析结果，然后调用`cloneBB`做基本块克隆

```c++
PreservedAnalyses DuplicateBB::run(llvm::Function &F,
                                   llvm::FunctionAnalysisManager &FAM) {
  if (!pRNG)
    pRNG = F.getParent()->createRNG("duplicate-bb");
  
  BBToSingleRIVMap Targets = findBBsToDuplicate(F, FAM.getResult<RIV>(F));

  // This map is used to keep track of the new bindings. Otherwise, the
  // information from RIV will become obsolete.
  ValueToPhiMap ReMapper;

  // Duplicate
  for (auto &BB_Ctx : Targets) {
    cloneBB(*std::get<0>(BB_Ctx), std::get<1>(BB_Ctx), ReMapper);
  }

  DuplicateBBCountStats = DuplicateBBCount;
  return (Targets.empty() ? llvm::PreservedAnalyses::all()
                          : llvm::PreservedAnalyses::none());
}

```

先看`findBBsToDuplicate`

- 首先遍历函数内的 Basic Block，并跳过 landing pad 也就是用于表示异常处理的 Basic Block
- 然后获取此基本块的 RIV 分析结果
- 随后从 RIV 中随机选择一个值，若为全局变量则跳过
- 最后将这个变量的 RIV 迭代器和 Basic Block 指针放进哈希表并最终返回

```c++
DuplicateBB::BBToSingleRIVMap
DuplicateBB::findBBsToDuplicate(Function &F, const RIV::Result &RIVResult) {
  BBToSingleRIVMap BlocksToDuplicate;

  for (BasicBlock &BB : F) {
    // Basic blocks which are landing pads are used for handling exceptions.
    // That's out of scope of this pass.
    if (BB.isLandingPad())
      continue;

    // Get the set of RIVs for this block
    auto const &ReachableValues = RIVResult.lookup(&BB);
    size_t ReachableValuesCount = ReachableValues.size();

    // Are there any RIVs for this BB? We need at least one to be able to
    // duplicate this BB.
    if (0 == ReachableValuesCount) {
      LLVM_DEBUG(errs() << "No context values for this BB\n");
      continue;
    }

    // Get a random context value from the RIV set
    auto Iter = ReachableValues.begin();
    std::uniform_int_distribution<> Dist(0, ReachableValuesCount - 1);
    std::advance(Iter, Dist(*pRNG));

    if (dyn_cast<GlobalValue>(*Iter)) {
      LLVM_DEBUG(errs() << "Random context value is a global variable. "
                        << "Skipping this BB\n");
      continue;
    }

    LLVM_DEBUG(errs() << "Random context value: " << **Iter << "\n");

    // Store the binding between the current BB and the context variable that
    // will be used for the `if-then-else` construct.
    BlocksToDuplicate.emplace_back(&BB, *Iter);
  }

  return BlocksToDuplicate;
}
```

然后是`cloneBB`

- 首先找到函数的第一个非 phi 指令，phi 节点不能乱动
- 然后创建一个用于 if 的条件，调用的是`CreateIsNull`，生成的条件是 Basic Block 的 RIV 内的某个值
  - 没搞懂这是要干啥
- 然后调用`SplitBlockAndInsertIfThenElse`将当前 Basic Block 分割成 if-then-else 并隐式地创建一个收束块（后面的`Tail`调用`getSuccessor`获取 then 的后继，并 assert 它与 else的后继相同），并调用`setName`命名
- 然后遍历原始基本块（现在变成`Tail`了，`Tail`似乎会继承原基本块后续的内容）中的所有指令
  - 跳过`Terminator`指令
  - 分别为 then 和 else 克隆一份当前指令
  - 然后往 then 和 else 插入克隆的指令
  - 随后若指令类型不是 void ，那么新创建一个 phi 节点，并调用`addIncoming`将 then 和 else 插入 phi 节点
    - 猜测：非 void 指令会产生返回值（可能是意味着基本块产生了一个新的 RIV），故做控制流拆分时这里将这些指令处理成 phi 节点来进行控制流同步
    - 不过因为这里 then 和 else 是一样的，感觉没啥实际意义只是为了示例
  - 最后调用`ReplaceInstWithInst`将该指令替换成 phi 节点
  - 最后的最后删除掉前面的非 void 节点

```c++
void DuplicateBB::cloneBB(BasicBlock &BB, Value *ContextValue,
                          ValueToPhiMap &ReMapper) {
  // Don't duplicate Phi nodes - start right after them
  Instruction *BBHead = BB.getFirstNonPHI();

  // Create the condition for 'if-then-else'
  IRBuilder<> Builder(BBHead);
  Value *Cond = Builder.CreateIsNull(
      ReMapper.count(ContextValue) ? ReMapper[ContextValue] : ContextValue);

  // Create and insert the 'if-else' blocks. At this point both blocks are
  // trivial and contain only one terminator instruction branching to BB's
  // tail, which contains all the instructions from BBHead onwards.
  Instruction *ThenTerm = nullptr;
  Instruction *ElseTerm = nullptr;
  SplitBlockAndInsertIfThenElse(Cond, &*BBHead, &ThenTerm, &ElseTerm);
  BasicBlock *Tail = ThenTerm->getSuccessor(0);

  assert(Tail == ElseTerm->getSuccessor(0) && "Inconsistent CFG");

  // Give the new basic blocks some meaningful names. This is not required, but
  // makes the output easier to read.
  std::string DuplicatedBBId = std::to_string(DuplicateBBCount);
  ThenTerm->getParent()->setName("lt-clone-1-" + DuplicatedBBId);
  ElseTerm->getParent()->setName("lt-clone-2-" + DuplicatedBBId);
  Tail->setName("lt-tail-" + DuplicatedBBId);
  ThenTerm->getParent()->getSinglePredecessor()->setName("lt-if-then-else-" +
                                                         DuplicatedBBId);

  // Variables to keep track of the new bindings
  ValueToValueMapTy TailVMap, ThenVMap, ElseVMap;

  // The list of instructions in Tail that don't produce any values and thus
  // can be removed
  SmallVector<Instruction *, 8> ToRemove;

  // Iterate through the original basic block and clone every instruction into
  // the 'if-then' and 'else' branches. Update the bindings/uses on the fly
  // (through ThenVMap, ElseVMap, TailVMap). At this stage, all instructions
  // apart from PHI nodes, are stored in Tail.
  for (auto IIT = Tail->begin(), IE = Tail->end(); IIT != IE; ++IIT) {
    Instruction &Instr = *IIT;
    assert(!isa<PHINode>(&Instr) && "Phi nodes have already been filtered out");

    // Skip terminators - duplicating them wouldn't make sense unless we want
    // to delete Tail completely.
    if (Instr.isTerminator()) {
      RemapInstruction(&Instr, TailVMap, RF_IgnoreMissingLocals);
      continue;
    }

    // Clone the instructions.
    Instruction *ThenClone = Instr.clone(), *ElseClone = Instr.clone();

    // Operands of ThenClone still hold references to the original BB.
    // Update/remap them.
    RemapInstruction(ThenClone, ThenVMap, RF_IgnoreMissingLocals);
    ThenClone->insertBefore(ThenTerm);
    ThenVMap[&Instr] = ThenClone;

    // Operands of ElseClone still hold references to the original BB.
    // Update/remap them.
    RemapInstruction(ElseClone, ElseVMap, RF_IgnoreMissingLocals);
    ElseClone->insertBefore(ElseTerm);
    ElseVMap[&Instr] = ElseClone;

    // Instructions that don't produce values can be safely removed from Tail
    if (ThenClone->getType()->isVoidTy()) {
      ToRemove.push_back(&Instr);
      continue;
    }

    // Instruction that produce a value should not require a slot in the
    // TAIL *but* they can be used from the context, so just always
    // generate a PHI, and let further optimization do the cleaning
    PHINode *Phi = PHINode::Create(ThenClone->getType(), 2);
    Phi->addIncoming(ThenClone, ThenTerm->getParent());
    Phi->addIncoming(ElseClone, ElseTerm->getParent());
    TailVMap[&Instr] = Phi;

    ReMapper[&Instr] = Phi;

    // Instructions are modified as we go, use the iterator version of
    // ReplaceInstWithInst.
    ReplaceInstWithInst(Tail, IIT, Phi);
  }

  // Purge instructions that don't produce any value
  for (auto *I : ToRemove)
    I->eraseFromParent();

  ++DuplicateBBCount;
}
```

好怪，还是没看懂它想干什么

## MergeBB

### First Taste

这个 Pass 会将之前那个 Pass 改造的控制流还原回去（怪不得之前那个看起来没啥用）

```
BEFORE:                     AFTER DuplicateBB:                 AFTER MergeBB:
-------                     ------------------                 --------------
                              [ if-then-else ]                 [ if-then-else* ]
             DuplicateBB           /  \               MergeBB         |
[ BB ]      ------------>   [clone 1] [clone 2]      -------->    [ clone ]
                                   \  /                               |
                                 [ tail ]                         [ tail* ]

LEGEND:
-------
[BB]           - the original basic block
[if-then-else] - a new basic block that contains the if-then-else statement (**DuplicateBB**)
[clone 1|2]    - two new basic blocks that are clones of BB (**DuplicateBB**)
[tail]         - the new basic block that merges [clone 1] and [clone 2] (**DuplicateBB**)
[clone]        - [clone 1] and [clone 2] after merging, this block should be very similar to [BB] (**MergeBB**)
[label*]       - [label] after being updated by **MergeBB**
```

文档给的命令好像有问题，用回下面的就好了

```shell
opt-17 -load-pass-plugin ./lib/libMergeBB.so -passes=merge-bb -S duplicate.ll -o merge.ll
```

原来被拆开的 IR 变成下面的样子了，可以看到其实基本块还是没全部合并回一个而且 if 结构还在，但 then 和 else 被合二为一了

```llvm
define dso_local i32 @foo(i32 noundef %0) local_unnamed_addr #0 {
lt-if-then-else-0:
  %1 = icmp eq i32 %0, 0
  br i1 %1, label %lt-clone-2-0, label %lt-clone-2-0

lt-clone-2-0:                                     ; preds = %lt-if-then-else-0, %lt-if-then-else-0
  br label %lt-tail-0

lt-tail-0:                                        ; preds = %lt-clone-2-0
  ret i32 1
}
```

还能再玩点花的，直接一次性将 RIV、Duplicate 和 Merge 三个 Pass 一起运行，注意`-passes`参数的顺序就是 Pass 依次运行的顺序

```shell
opt-17 -load-pass-plugin ./lib/libRIV.so -load-pass-plugin ./lib/libDuplicateBB.so -load-pass-plugin ./lib/libMergeBB.so -passes=d
uplicate-bb,merge-bb -S input_for_duplicate_bb.ll -o merge_after_duplicate.ll
```

生成出来的 IR 跟上面是一样的

### Code Analysis

`run`首先对 function 的每个 Basic Block 调用`mergeDuplicatedBlock`并生成一个`DeleteList`，然后调用`DeleteDeadBlock`消除产生的死代码

```c++
PreservedAnalyses MergeBB::run(llvm::Function &Func,
                               llvm::FunctionAnalysisManager &) {
  bool Changed = false;
  SmallPtrSet<BasicBlock *, 8> DeleteList;
  for (auto &BB : Func) {
    Changed |= mergeDuplicatedBlock(&BB, DeleteList);
  }

  for (BasicBlock *BB : DeleteList) {
    DeleteDeadBlock(BB);
  }

  return (Changed ? llvm::PreservedAnalyses::none()
                  : llvm::PreservedAnalyses::all());
}
```

先看看`mergeDuplicatedBlock`

- 首先要跳过入口块，并检查 Basic Block`BB1`的最后一条指令（`getTerminator`）是否是条件分支（`isUnconditional`），如果不是也要跳过，并且还要遍历 Basic Block 的前驱判断它们的最后一条指令是不是分支或 Switch 指令，如果不是也要跳过
- 然后获取`BB1`的后继`BBSucc`，检查`BBSucc`的第一条指令和第二条指令是否是 phi 指令，如果两条指令都是这里也不做处理（否则代码会很繁琐，作为 Tutor 还是简洁为要），用 RIV 的测试代码测试发现也确实如此
- 书接上文，若`BBSucc`的第一条指令是 phi 指令，那么获取`BB1`在这个 phi 函数里对应的变量的第一条指令（即创建指令）`InInstBB1`
- 接下来遍历`BBSucc`的所有前驱`BB2`（不包含`BB1`）
  - 对`BB2`做和`BB1`同样的检查
  - 检查`BB2`与`BB1`对 phi 函数的输入值是否相同以及是否是在它们中被定义的，若两者不符其一则跳过
    - 猜测：如果输入值不是在 phi 的前驱定义的，那么 then 和 else 就不是 Duplicate 生成的，这里就直接不合并了
  - 随后调用`canMergeInstructions`检查`BB2`和`BB1`的指令是否相同，不同就不能合并
  - 最后若是能合并的那么记录`BB1`和`BB2`，调用`updateBranchTargets`更新并返回

```c++
bool MergeBB::mergeDuplicatedBlock(BasicBlock *BB1,
                                   SmallPtrSet<BasicBlock *, 8> &DeleteList) {
  // Do not optimize the entry block
  if (BB1 == &BB1->getParent()->getEntryBlock())
    return false;

  // Only merge CFG edges of unconditional branch
  BranchInst *BB1Term = dyn_cast<BranchInst>(BB1->getTerminator());
  if (!(BB1Term && BB1Term->isUnconditional()))
    return false;

  // Do not optimize non-branch and non-switch CFG edges (to keep things
  // relatively simple)
  for (auto *B : predecessors(BB1))
    if (!(isa<BranchInst>(B->getTerminator()) ||
          isa<SwitchInst>(B->getTerminator())))
      return false;

  BasicBlock *BBSucc = BB1Term->getSuccessor(0);

  BasicBlock::iterator II = BBSucc->begin();
  const PHINode *PN = dyn_cast<PHINode>(II);
  Value *InValBB1 = nullptr;
  Instruction *InInstBB1 = nullptr;
  BBSucc->getFirstNonPHI();
  if (nullptr != PN) {
    // Do not optimize if multiple PHI instructions exist in the successor (to
    // keep things relatively simple)
    if (++II != BBSucc->end() && isa<PHINode>(II))
      return false;

    InValBB1 = PN->getIncomingValueForBlock(BB1);
    InInstBB1 = dyn_cast<Instruction>(InValBB1);
  }

  unsigned BB1NumInst = getNumNonDbgInstrInBB(BB1);
  for (auto *BB2 : predecessors(BBSucc)) {
    // Do not optimize the entry block
    if (BB2 == &BB2->getParent()->getEntryBlock())
      continue;

    // Only merge CFG edges of unconditional branch
    BranchInst *BB2Term = dyn_cast<BranchInst>(BB2->getTerminator());
    if (!(BB2Term && BB2Term->isUnconditional()))
      continue;

    // Do not optimize non-branch and non-switch CFG edges (to keep things
    // relatively simple)
    for (auto *B : predecessors(BB2))
      if (!(isa<BranchInst>(B->getTerminator()) ||
            isa<SwitchInst>(B->getTerminator())))
        continue;

    // Skip basic blocks that have already been marked for merging
    if (DeleteList.end() != DeleteList.find(BB2))
      continue;

    // Make sure that BB2 != BB1
    if (BB2 == BB1)
      continue;

    // BB1 and BB2 are definitely different if the number of instructions is
    // not identical
    if (BB1NumInst != getNumNonDbgInstrInBB(BB2))
      continue;

    // Control flow can be merged if incoming values to the PHI node
    // at the successor are same values or both defined in the BBs to merge.
    // For the latter case, canMergeInstructions executes further analysis.
    if (nullptr != PN) {
      Value *InValBB2 = PN->getIncomingValueForBlock(BB2);
      Instruction *InInstBB2 = dyn_cast<Instruction>(InValBB2);

      bool areValuesSimilar = (InValBB1 == InValBB2);
      bool bothValuesDefinedInParent =
          ((InInstBB1 && InInstBB1->getParent() == BB1) &&
           (InInstBB2 && InInstBB2->getParent() == BB2));
      if (!areValuesSimilar && !bothValuesDefinedInParent)
        continue;
    }

    // Finally, check that all instructions in BB1 and BB2 are identical
    LockstepReverseIterator LRI(BB1, BB2);
    while (LRI.isValid() && canMergeInstructions(*LRI)) {
      --LRI;
    }

    // Valid iterator  means that a mismatch was found in middle of BB
    if (LRI.isValid())
      continue;

    // It is safe to de-duplicate - do so.
    unsigned UpdatedTargets = updateBranchTargets(BB1, BB2);
    assert(UpdatedTargets && "No branch target was updated");
    OverallNumOfUpdatedBranchTargets += UpdatedTargets;
    DeleteList.insert(BB1);
    NumDedupBBs++;

    return true;
  }

  return false;
}
```

前面调用的`updateBranchTargets`、`canMergeInstructions`和其调用的`canRemoveInst`

- 其实就是检查 Operator、操作数数量、操作数是否相同
- `canRemoveInst`检查指令的唯一使用者（即目的操作数）是否在同一个 Basic Block（不知道什么用意）或是否被后继的 phi 使用（符合 Duplicate 的逻辑），如果二者满足其一则说明指令可删除
- `updateBranchTargets`首先调用`predecessors`获取所有指向要删除的基本块的前驱基本块，然后迭代更新这些前驱基本块的终结指令指向保留的 Basic Block，这样可以将要删除的 Basic Block 从 CFG 中孤立，最后消除死代码的时候可以被消除掉

```c++
bool MergeBB::canRemoveInst(const Instruction *Inst) {
  assert(Inst->hasOneUse() && "Inst needs to have exactly one use");

  auto *PNUse = dyn_cast<PHINode>(*Inst->user_begin());
  auto *Succ = Inst->getParent()->getTerminator()->getSuccessor(0);
  auto *User = cast<Instruction>(*Inst->user_begin());

  bool SameParentBB = (User->getParent() == Inst->getParent());
  bool UsedInPhi = (PNUse && PNUse->getParent() == Succ &&
                    PNUse->getIncomingValueForBlock(Inst->getParent()) == Inst);

  return UsedInPhi || SameParentBB;
}

bool MergeBB::canMergeInstructions(ArrayRef<Instruction *> Insts) {
  const Instruction *Inst1 = Insts[0];
  const Instruction *Inst2 = Insts[1];

  if (!Inst1->isSameOperationAs(Inst2))
    return false;

  // Each instruction must have exactly zero or one use.
  bool HasUse = !Inst1->user_empty();
  for (auto *I : Insts) {
    if (HasUse && !I->hasOneUse())
      return false;
    if (!HasUse && !I->user_empty())
      return false;
  }

  // Not all instructions that have one use can be merged. Make sure that
  // instructions that have one use can be safely deleted.
  if (HasUse) {
    if (!canRemoveInst(Inst1) || !canRemoveInst(Inst2))
      return false;
  }

  // Make sure that Inst1 and Inst2 have identical operands.
  assert(Inst2->getNumOperands() == Inst1->getNumOperands());
  auto NumOpnds = Inst1->getNumOperands();
  for (unsigned OpndIdx = 0; OpndIdx != NumOpnds; ++OpndIdx) {
    if (Inst2->getOperand(OpndIdx) != Inst1->getOperand(OpndIdx))
      return false;
  }

  return true;
}

unsigned MergeBB::updateBranchTargets(BasicBlock *BBToErase, BasicBlock *BBToRetain) {
  SmallVector<BasicBlock *, 8> BBToUpdate(predecessors(BBToErase));

  LLVM_DEBUG(dbgs() << "DEDUP BB: merging duplicated blocks ("
                    << BBToErase->getName() << " into " << BBToRetain->getName()
                    << ")\n");

  unsigned UpdatedTargetsCount = 0;
  for (BasicBlock *BB0 : BBToUpdate) {
    // The terminator is either a branch (conditional or unconditional) or a
    // switch statement. One of its targets should be BBToErase. Replace
    // that target with BBToRetain.
    Instruction *Term = BB0->getTerminator();
    for (unsigned OpIdx = 0, NumOpnds = Term->getNumOperands();
         OpIdx != NumOpnds; ++OpIdx) {
      if (Term->getOperand(OpIdx) == BBToErase) {
        Term->setOperand(OpIdx, BBToRetain);
        UpdatedTargetsCount++;
      }
    }
  }

  return UpdatedTargetsCount;
}
```

## FindFCmpEq

### First Taste

文档说明有些许问题，按文档运行会发现没输出没报错，后来看 IR 发现 Clang 在 -O0 下处理每个函数时都会带 optnone，给 opt 加参数`--debug-pass-manager`观察到这会导致 Pass 会跳过函数不进行优化，而开 -O1 虽然能运行 Pass ，但又会将部分代码消除掉影响观感，最后加上`-Xclang -disable-O0-optnone`后就能正常输出了

```shell
clang-17 -Xclang -disable-O0-optnone -emit-llvm -S -c ../inputs/input_for_fcmp_eq.c -o input_for_fcmp_eq.ll
opt-17 --load-pass-plugin ./lib/libFindFCmpEq.so -passes="print<find-fcmp-eq>" -disable-output input_for_fcmp_eq.ll
```

输出也和文档的描述有所不同，估计是太久没改了

```llvm
Floating-point equality comparisons in "sqrt_impl":
  %11 = fcmp oeq double %9, %10
Floating-point equality comparisons in "main":
  %9 = fcmp oeq double %8, 1.000000e+00
  %13 = fcmp oeq double %11, %12
  %19 = fcmp oeq double %17, %18
```

### Code Analysis

主体其实是下面这个`run`，`FCmpInst`类重载的`isEquality`方法判定该浮点指令是否是 equal 指令

```c++
FindFCmpEq::Result FindFCmpEq::run(Function &Func) {
  Result Comparisons;
  for (Instruction &Inst : instructions(Func)) {
    // We're only looking for 'fcmp' instructions here.
    if (auto *FCmp = dyn_cast<FCmpInst>(&Inst)) {
      // We've found an 'fcmp' instruction; we need to make sure it's an
      // equality comparison.
      if (FCmp->isEquality()) {
        Comparisons.push_back(FCmp);
      }
    }
  }

  return Comparisons;
}
```

就这么短，没了（）

## ConvertFCmpEq

### First Taste



### Code Analysis

