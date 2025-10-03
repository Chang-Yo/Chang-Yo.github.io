---
layout: single
title: "半年之后再回首：关于xmake在C++环境配置中的简单实践&配置文件使用指南"
# date: 2025-09-01 15:00
categories: 
    - Tech
tags:
    - content
    - Zhihu
---

```
文章为原创，最初发表在知乎，发表时间为2025-09-20
全文总计 1 万字左右，预计阅读时间为 30~40 min
```

## 前言：

### 文章动机

在半年之前，我曾经写了一篇关于WSL中VScode的C++环境配置的教程。在这个教程里，我主要聚焦于launch.json和tasks.json的代码含义。那篇文章是在我努力摸索了很多天后对VScode环境配置初步理解的总结。

[WSL下VSCode的C++环境配置（代码解析版）](2025-09-05-Instructions-on-WSL.md)


不过，如今回首再看那篇文章，发现那篇文章其实还是有很多不合理的地方。而且文章中对于代码从编译到形成可执行文件这个过程的描述十分的模糊，某些部分甚至是错误的！

最近我尝试着配置在Windows环境下配置C++环境，在知乎上读到了一篇利用xmake配置环境的教程，然后又读到了 @xq114 写的xmake教程，尝试了一段时间却屡屡出错。于是又去读了一些介绍代码编译过程的资料，主要是《程序员的自我修养》这本书，终于对代码编译、链接过程有了一个比较系统和清晰的认识，最后在大模型的帮助下尝试了一种新的C++环境配置方式。而现在这篇文章，就是对这个过程的总结。

### 这篇教程有什么：

1. 对xmake的简单介绍。如果你不知道xmake是什么但想了解，或者你已经知道xmake是什么想看看如何使用，都可以阅读这篇教程

2. 对代码编译、链接过程的进一步深化。如果你对环境配置文件（比如launch.json和tasks.json）当中的一些属性不太了解并也不清楚这些文件之间的通讯过程，那么这篇文章可以让你对这个代码的构建过程有一个基础的认识
3. 对tasks.json和launch.json配置文件的再认识

4. 作者的一点点小感悟和教程链接⁄(⁄ ⁄•⁄ω⁄•⁄ ⁄)⁄

OK，那我们就进入正题吧！

---

## Xmake 一种构建系统工具

xmake是一种构建系统，它并不会直接参与对代码的编译和链接，而是作为一个“管理者”，向它的手下们（编译器，汇编器，链接器）分配任务。你可以把他理解为公司里面的高管，负责人员的工作分配。高管的能力有高有低，带领公司的水平也有高有低。从构建系统诞生至今，总共出现了4代工具：

- **Make**
- **(GNU) Autotools**
- **Cmake**
- **GN、Ninja （3.5代）**
- **Xmake**

这里面大家可能看到比较熟悉的工具如Cmake，当然像Make这样的第一代构建工具你也常常可以在一些老项目中看见。但我们要讨论的是Xmake，也就是目前最新一代的构建工具。

Xmake可以自动帮你分配需要的编译器、链接器，当然你也可以自己设置。同时Xmake具有增量编译功能，并行编译的功能。增量编译可以理解为每次build他都只会重新编译发生过更改的文件，可以提高效率；并行编译则可以显著提升编译速度。

总之，先进的工具就是先进的生产力。

---

## 从编译到执行文件，代码发生了什么

这部分内容可能涉及到一些编译原理的知识，但我尽量简单描述。

![image](/assets/images/compile_process.jpg)

这张图可以清晰的反映代码运行的背后机理。了解这张图，有助于了解我们环境配置的前后逻辑，也有助于你理解IDE的报错信息。

1. 首先这个简单的项目有一个头文件(header) stdio.h和源文件hello.c，他们第一步会先通过预编译过程(Propressing)，在这个过程中主要是处理“#”开头的预编译指令，比如将所有宏定义展开，将#include文件插入代码，删除所有注释等等

2. 接着进入到正式的编译过程(Compilation)，这个部分主要是进行词法分析、语法分析、语义分析和优化产生的汇编代码，这也是整个程序构建最重要的一环

3. 然后进入汇编过程(Assembly)，这个过程比较简单，相当于将汇编代码转化为机器可执行码，这个过程只需要对照着汇编指令和机器指令的对照表翻译即可

4. 最后就是通过链接过程(Linking)形成可执行文件。这个过程会把生成的目标文件和系统自带的运行时库链接起来，最终生成一个可以直接执行的文件

而我们常常用到的编译器比如GCC,clang其实相当于以上过程中各个工具的包装，它能够为相应的语言在相应的过程选择合适的工具执行操作。

现代的编译器基本都把编译和链接过程合在一起了，称为 **构建 (Build)**

---

## 再看tasks.json和launch.json

讲了这么久终于可以开始我们环境的配置了。我们可以先建立一个项目文件夹，姑且称为Project吧。然后按照大部分约定俗成的规矩创建这样的文件结构：

```bash
# Project file frame
Project
|
|--.vscode
|   |--launch.json 
|   |--tasks.json
|
|--src  #主要存放源代码也就是cpp/c文件
|--include  #主要存放头文件
|--out/   #可以用作测试时的输出文件夹
|--xmake.lua  #非常关键的Xmake配置文件
```

如果你看过我之前那篇教程的话，应该会对上方的两份json文件内容有所了解，不过为了严谨性，我会将它们重新讲述，并且做到比以前更精准。不过这次我们采用先引导的方式来讲述。

当按下F5时，发生了什么
debug最常用的快捷键就是F5了，当我们按下F5时，这个过程可以理解为调用launch.json。我们知道launch.json中可以设定很多configurations，**每一个configuration其实都对应着一种debug方式**

你打开左侧的run and debug选项，找到下拉框，其实他就出现很多选择，前面这两种是我自己定义configuration，所以会出现在最前方。

我们一般会让每个configuration都有一个对应的preLaunchTasks，这样可以方便过程的构建。而这个preLaunchTasks其实对应的就是tasks.json中的某一个或多个task。

说到这了，我觉得可以把代码贴出来了：

**launch.json**

```json
{
    "configurations": [
        {
            "name": "LLDB Debug",
            "type": "lldb",
            "request": "launch",
            "program": "${workspaceFolder}/out/${fileBasenameNoExtension}",
            "args": [],
            "cwd": "${workspaceFolder}",
            "preLaunchTask": "clang debug"
        },
        {
            "name": "Xmake Build",
            "type": "lldb",
            "request": "launch",
            "program": "${workspaceFolder}/build/windows/x64/debug/output",
            "args": [],
            "cwd": "${workspaceFolder}",
            "preLaunchTask": "xmake build"
        }

    ]
}
```

**tasks.json**

```json
{
    "version": "2.0.0",
    "tasks": [
        {
            "label": "xmake build",
            "type": "shell",
            "command": "xmake",
            "args": [
                "build",
                "-v",
                "-j6"
            ],
            "group": {
                "kind": "build",
                "isDefault": false
            },
            "presentation": {
                "echo": true,
                "reveal": "always",
                "focus": false,
                "panel": "shared",
                "showReuseMessage": true,
                "clear": false
            },
            "problemMatcher": "$gcc"
        },
        {
            "label": "clang debug",
            "type": "shell",
            "command": "clang++",
            "args": [
                "-v",
                "-g",
                "${fileWorkspaceFolder}/src/*.cpp",
                "-o",
                "${workspaceFolder}/out/${fileBasenameNoExtension}",
                "-I",
                "${workspaceFolder}/include",
            ],
            "group": {
                "kind": "test",
                "isDefault": true
            },
            "presentation": {
                "echo": true,
                "reveal": "always",
                "focus": false,
                "panel": "shared",
                "showReuseMessage": true,
                "clear": false
            },
            "problemMatcher": "$gcc"
        }
    ]
}
```

我们就以其中的Xmake build为例来理解其中的互相调用的过程

下面是从launch.json中截取的一部分：

```json
    {
        "name": "Xmake Build",
        "type": "lldb",
        "request": "launch",
        "program": "${workspaceFolder}/build/windows/x64/debug/output",
        "args": [],
        "cwd": "${workspaceFolder}",
        "preLaunchTask": "xmake build"
    }
```

下面是task.json中的一部分：

```json
    {
        "label": "xmake build",
        "type": "shell",
        "command": "xmake",
        "args": [
            "build",
            "-v",
            "-j6"
        ],
        "group": {
            "kind": "build",
            "isDefault": false
        },
        "presentation": {
            "echo": true,
            "reveal": "always",
            "focus": false,
            "panel": "shared",
            "showReuseMessage": true,
            "clear": false
        },
        "problemMatcher": "$gcc"
    }
```

在launch.json中有几个属性需要明白

`"type"` 是调试器，也就是方便你设置断点，单步执行，检查调用栈等等功能的，我这里使用了LLDB调试器，你也可以使用GDB。

`"request"` 这个表示启动方式，有launch和attach两种，一般都用前者

`"name"` 这个其实就是你在**run and code**界面看见的名字了，比如我取得名字是Xmake build，所以可以看见Xmake build这个选项。取个你易于理解的名字即可

`"program"` 这个比较重要，后面的路径其实就是可执行文件的位置，相当于IDE会直接找到这个可执行文件然后帮你运行。可执行文件的输出位置和名称一定要和这个对应！不管你是在task.json中设定输出位置还是在xmake.lua中设定，请一定对应准确！！

`"preLaunchTask"` 这个属性也很重要，就像我刚才说的，它会去调用tasks.json中的某个或多个任务。**靠什么查找呢？通过每个task的 "label" 标签来对应。**

我们把目光转向tasks.json

`"label"` 后面的值是Xmake build，和上方的preLaunchTask完全对应。所以在直接运行可执行文件之前，会先调用Xmake build的task，等这个task完成后再去运行执行文件。故而可以知道tasks.json一般就是用来设定编译条件的。

`"type"` 这个一般都是shell，表示调用系统的终端，不需要更改(具体而言，Windows是powershell, Linux是Bash, MacOS是zsh)

而后面的 `"command"` 和 `"args"` 其实值得好好理解。

`"command"` 其实就是会在终端中执行的命令，比如这里是xmake，相当于在shell中输入xmake指令；如果你的command是gcc，那么就相当于在shell中执行gcc指令。

当然，我们一般后面会跟上一些参数`"args"` ，用过命令行的同学应该很好理解，比如我们如果用GCC去编译C++代码，在命令行中一般是这样书写：

```bash
gcc -g hello.cpp -o hello -I .
#（这里假设头文件，源代码都在根目录下）
```

**所以我们可以看到的`"args"` 内容其实就是为`"command"`提供参数(arguments)，实际上args本就是arguments的缩写。**

这时候我们看向我的xmake build中的指令是什么：

```json
"command": "xmake",
"args": [
     "build",
     "-v",
     "-j6"
], 
```

**这段代码等效于在shell中执行 `xmake build -v -j6`**

`-v` 在这表示输出详细信息，一般报错时特别有用，你可以直接拿错误报告喂给大模型

`-j6` 在这里表示采用CPU的6个核并行编译，这主要是为了加快编译速度

`build` 就没什么好说了，就是执行build操作

当然，其实你可以直接空着不留参数，因为命令行输入xmake也相当于执行build操作

话都说到这里了，必须要贴出xmake.lua的代码了
```lua
add_rules("mode.debug","mode.release")

target("output")
    set_kind("binary")
    add_files("src/*.cpp")
    add_includedirs("include")
```

xmake.lua其实就是xmake的配置文件，也就是运行xmake指令时会执行的文件。

这里我不会讲述xmake的详细用法，仅仅从这个简单项目出发来编写，如果你想了解更多的xmake知识，可以参考文末的链接。
```lua
add_rules("mode.debug","mode.release")
```

这个一般都是标配，相当于设定了模式，可以不用太在意

```lua
target("output")
    set_kind("binary")
    add_files("src/*.cpp")
    add_includedirs("include")
```

这个可以好好讲讲。

xmake.lua中可以执行很多任务，每个任务都是一个target，而后面括号内的内容，其实就是这个target的名称。如果xmake.lua中有很多target，一般默认是执行第一个target，除非在xmake 后面指定，比如`xmake output`

`set_kind("binary")`表示设定目标类型为可执行文件（二进制文件）

`add_files`表示要参与编译的文件，我这里相当于把src下的所有cpp都编译了。实际上如果有多个cpp文件的话也确实应该这样做

`add_includedirs`表示包括头文件的目录。这里需要强调的是头文件一般不需要指定到某个具体的文件，像这里只需要填include这个上级目录而不需要填include/*.h

当然还有很多可以添加的操作，具体可以去查阅xmake的官方文档，我可以挑几个比较实用的：

`set_targetdir`表示目标输出目录，默认情况下输出文件会在build文件内部，你可以通过这个显式的指定路径

`set_filename`表示输出的目标文件的名称，默认情况下它和target名字相同

比如这里我其实可以这样设定：
```lua
set_targetdir("out")
set_file("outcomes") 
```

这样你的可执行文件就在out目录下且名为outcomes了，**这时候要注意将launch.json中program路径做出修改！！**

有时候，你还会还有一些外部库，那么你可以使用add_linkdir和add_links这两种操作。此外如果你想指定编译器的话可以在全局位置用set_toolset和set_toolchain来制定编译器和链接器。我这里已经下载了MSVC，所以就直接用了其中工具

**总之，以官方文档的介绍为准！**

---

## 简单回顾

到了这里，我们可以回顾一下这个代码的具体运行过程是什么了

- 按下F5，开始运行和调试(Run and Debug)
- 执行launch.json中program路径下的文件
- 发现有preLaunchTask，于是转到tasks.json中寻找同名task
- Xmake build这个task在命令行输出xmake build -v -j6，调用xmake
- xmake 根据自身的配置文件xmake.lua开始构建系统，为各个环节配置合适工具
- 代码通过编译、链接，最终执行，输出在Terminal

这就是整个的工作链。

---

## 其他补充

到这里，其实文章的关于xmake主要部分已经差不多讲完了。可以对其他内容做一个简单的补充。

我的launch设定了两种，一种是Xmake build，一种是clang++，因为build是将所有cpp文件都编译，比较耗时，所以有时候不太想全部编译而是想调试某个具体的文件时可以用clang++，这样更精细一些。至于这部分怎么写，相信我的读者肯定已经会写了。

当然，如果没有这个需求，其实都不需要这两个json文件，理论上只要有xmake.lua就可以完成环境配置了。

如果你是第一次执行xmake build，那么应该会发现项目中多了几个文件夹或文件，比如compile_commands.json和build文件夹，前者可以查看自己的编译相关选项，后者存着一些中间文件和缓存，当然也会有你的可执行文件。

---

## 写在结尾

这篇文章可以理解为是对我半年前那篇教程的纠正，补充与深化。半年过去了，其实我的编程技巧并没有太大的长进，对工具的使用也并没有太熟练。就像这个Xmake的使用，我也是花了好几小时来理解运用的，而且目前只是用在简单的场景下，更复杂的场景我依然无法掌握。

人对一些事物的认识总是不断地深化的，前人之言，尚容易被后人反驳，更何况自己还处在一个不断学习知识的过程中，接触的越多，就越发现自己之前的想法的稚嫩，不过有时倒也觉得几分可爱。

或许过了几个月或者几年，我又会写一写文章来修正甚至是反驳自己的观点吧。

计算机学习之路漫漫，我这篇教程其实并不对你的编程技巧有太多的提升，现在网上配置环境的教程纷繁复杂，人们对新工具的探索有时候并不多，因为新工具使用时总是容易遇到各种各样的闻所未闻的问题，而网上对新工具的经验介绍又比较少。

不过我可以相信，读完我这篇文章的读者们，以后如果遇到了IDE给你的报错信息时，你至少能够理解它大概那个环节出了问题还是没有衔接上；再不济，你也可以通过修改一些参数让报错信息更加详细，然后将它喂给大模型。某种程度上对你的效率有所提升。

最后希望这篇不算太难的教程能够帮助需要的伙伴(*^▽^*)

---

#### 相关资料

我半年前写过的环境配置文章: 
    [WSL下VSCode的C++环境配置（代码解析版）](2025-09-05-Instructions-on-WSL.md)

强烈推荐几位大佬的教程：

[@blackbird](https://www.zhihu.com/people/shu-xue-da-wang-25):
    [[万字长文] Visual Studio Code 配置 C/C++ 开发环境的最佳实践(VSCode + Clangd + XMake)](https://zhuanlan.zhihu.com/p/398790625)

[@xq114](https://www.zhihu.com/people/xq114):
    [A Tour of xmake](https://www.zhihu.com/column/c_1537535487199281152)

以及Xmake的官方文档:    [Xmake Introduction](https://xmake.io/guide/introduction.html)

还有*《程序员的自我修养：链接、装载与库》*这本书

再次感谢读到这的读者！！(\*^▽^\*)