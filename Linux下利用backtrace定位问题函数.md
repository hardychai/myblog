最近遇到一个问题，程序莫名其妙崩溃，由于系统设置并没有生成core文件，因此也就不能通过gdb调试来查看出错时的调用栈信息。好在系统生成了crash.log文件，里面的backtrace信息可以帮我分析定位问题。
先来看一下当时的backtrace打印：

> 08-22 10:07:39.981 F/DEBUG   (13647): backtrace:
> 08-22 10:07:39.981 F/DEBUG   (13647):     #00 pc 000000000031744c  /usr/bin/myprogram
> 08-22 10:07:39.981 F/DEBUG   (13647):     #01 pc 0000000000315864  /usr/bin/myprogram
> 08-22 10:07:39.981 F/DEBUG   (13647):     #02 pc 0000000000315adc  /usr/bin/myprogram
> 08-22 10:07:39.981 F/DEBUG   (13647):     #03 pc 00000000002c5fb4  /usr/bin/myprogram
> 08-22 10:07:39.981 F/DEBUG   (13647):     #04 pc 00000000002c3a64  /usr/bin/myprogram
> 08-22 10:07:39.981 F/DEBUG   (13647):     #05 pc 0000000000065f88  /system/lib64/libc.so (_ZL15__pthread_startPv+36)
> 08-22 10:07:39.982 F/DEBUG   (13647):     #06 pc 000000000001ed24  /system/lib64/libc.so (__start_thread+68)

然后使用addr2line工具，addr2line工具是一个可以将指令的地址和可执行映像转换为文件名、函数名和源代码行数的工具。这在内核执行过程中出现崩溃时，可用于快速定位出出错的位置，进而找出代码的bug。

##用法
addr2line [-a| --addresses ] [-b bfdname | --target=bfdname] [-C | --demangle[=style]] [-e filename | --exe=filename] [-f | --function] [-s | --basename] [-i | --inlines] [-p | --pretty-print] [-j | --section=name] [-H | --help] [-V | --version] [addr addr ...]

##参数
-a --addresses：在函数名、文件和行号信息之前，显示地址，以十六进制形式。
-b --target=<bfdname>：指定目标文件的格式为bfdname。
-e --exe=<executable>：指定需要转换地址的可执行文件名。
-i --inlines ： 如果需要转换的地址是一个内联函数，则输出的信息包括其最近范围内的一个非内联函数的信息。
-j --section=<name>：给出的地址代表指定section的偏移，而非绝对地址。
-p --pretty-print：使得该函数的输出信息更加人性化：每一个地址的信息占一行。
-s --basenames：仅仅显示每个文件名的基址（即不显示文件的具体路径，只显示文件名）。
-f --functions：在显示文件名、行号输出信息的同时显示函数名信息。
-C --demangle[=style]：将低级别的符号名解码为用户级别的名字。
-h --help：输出帮助信息。
-v --version：输出版本号。

backtrace里给出了地址`000000000031744c`，使用命令：
```shell
addr2line 000000000031744c -e /usr/bin/myprogram -f -C -s
```

输出信息：
```shell
divide(int, int)
test.cpp:3
```
可以很轻松得到崩溃函数信息。

但有时也有例外，输出`??:?`或`??:0`，这表示崩溃程序没有包含符号表。