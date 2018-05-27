# LAB1

##练习1:理解通过make生成执行文件的过程

###说明

列出本实验各练习中对应的OS原理的知识点,并说明本实验中的实现部分如何对应和体现了原理中的基本概念和关键知识
点。
在此练习中,大家需要通过静态分析代码来了解:

1.	 操作系统镜像文件ucore.img是如何一步一步生成的?(需要比较详细地解释Makefile中每一条相关命令和命令参数的含
  义,以及说明命令导致的结果)
2.	 一个被系统认为是符合规范的硬盘主引导扇区的特征是什么?

###回答

Makefile中创建 ucore.img 的代码：

```

UCOREIMG	:= $(call totarget,ucore.img)#向totarget函数传'ucore.img'参数

$(UCOREIMG): $(kernel) $(bootblock)
	$(V)dd if=/dev/zero of=$@ count=10000
	$(V)dd if=$(bootblock) of=$@ conv=notrunc
	$(V)dd if=$(kernel) of=$@ seek=1 conv=notrunc

$(call create_target,ucore.img)

```

其中：`UCOREIMG	:= $(call totarget,ucore.img)` 向 totarget 函数传 'ucore.img' 参数，totarget 函数原型为：

`totarget = $(addprefix $(BINDIR)$(SLASH),$(1))`，其中 `BINDIR := bin`，` SLASH	:= /`，所以这个函数调用实际上是:

`totarget = $(addprefix bin/,ucore.img)`

addprefix 是为参数加前缀函数，所以这一句就是将 ucore.img 变为 bin/ucore.img。

然后是这一段代码：

```
$(UCOREIMG): $(kernel) $(bootblock) #/bin/ucore.img 依赖于 kernel 和 bootblock
    #$(V) 中 V       := @，加在命令前面的意思是终端不显示出命令
	$(V)dd if=/dev/zero of=$@ count=10000 #先向ucore.img写入 10000个字节，内容全是0
	$(V)dd if=$(bootblock) of=$@ conv=notrunc 
	#再将bootblock的内容写入到img前面，覆盖前面的内容，如果去掉了conv=notrunc 则效果是删除之前的所有内容(即10000个字节全为0)，而不仅仅是覆盖前面的字节
	$(V)dd if=$(kernel) of=$@ seek=1 conv=notrunc #和上一句相同，只是从开头跳过1个block以后再覆盖
```



再来看看bootblock如何生成：

```
# create bootblock
bootfiles = $(call listf_cc,boot) #列出boot目录下面所有后缀为 c 或者为 S 的文件
$(foreach f,$(bootfiles),$(call cc_compile,$(f),$(CC),$(CFLAGS) -Os -nostdinc))
#将boot目录下面的所有 .c 和 .S 文件进行编译，即文件 bootmain.c ，bootasm.S
## bootmain.c 编译的命令为：
#gcc -Iboot -fno-builtin -Wall -ggdb -m32 -gstabs -nostdinc \
-fno-stack-protector  -Ilibs\
-Os -nostdinc -c boot/bootmain.c -o obj/boot/bootmain.o

##-fno-builtin  只识别以__builtin_开头的内置函数
##-ggdb 生成供gdb使用的调试信息
##-m32生成适用于32位的代码，ucore为32位的
##-gstabs 以stabs格式生成调试信息
##-nostdinc 不使用标准库，因为ucore只调用自己的库
##-Os 优化大小，使用了所有-O2的优化选项，但又不增加代码尺寸
##-Ilibs -Iboot 添加搜索头文件的路径

## bootasm.S 同理
bootblock = $(call totarget,bootblock)
#和上面的 ucore.img 同理，将bootblock 改为 bin/bootblock
$(bootblock): $(call toobj,$(bootfiles)) | $(call totarget,sign)#依赖于boot目录下编译好的 .o 文件或者 bin/sign 文件
	@echo + ld $@ #将目标文件 bin/bootblock 链接为可执行文件
	$(V)$(LD) $(LDFLAGS) -N -e start -Ttext 0x7C00 $^ -o $(call toobj,bootblock)
	#@i386-elf-ld -m elf_i386 -nostdlib -N -e start -Ttext 0x7C00 *.o -o *.o    编译到指定物理位置
-m <emulation>  模拟为i386上的仿真链接器
-nostdlib  不使用标准库
-N  设置代码段和数据段均可读写
-e <entry>  指定入口
-Ttext  制定代码段开始位置
	@$(OBJDUMP) -S $(call objfile,bootblock) > $(call asmfile,bootblock)
	@$(OBJCOPY) -S -O binary $(call objfile,bootblock) $(call outfile,bootblock)
	#拷贝二进制代码bootblock.o到bootblock.out
	@$(call totarget,sign) $(call outfile,bootblock) $(bootblock)
	#使用sign工具处理bootblock.out，生成bootblock

$(call create_target,bootblock)
```

生成sign的代码为：

```
$(call add_files_host,tools/sign.c,sign,sign)
$(call create_target_host,sign,sign)
```

实际命令为：

```
gcc -Itools -g -Wall -O2 -c tools/sign.c -o obj/sign/tools/sign.o
gcc -g -Wall -O2 obj/sign/tools/sign.o -o bin/sign
```

生成 kernel 内核的代码为：

```
$(kernel): tools/kernel.ld
$(kernel): $(KOBJS)
@echo + ld $@
$(V)$(LD) $(LDFLAGS) -T tools/kernel.ld -o $@ $(KOBJS)
@$(OBJDUMP) -S $@ > $(call asmfile,kernel)
@$(OBJDUMP) -t $@ | $(SED) '1,/SYMBOL TABLE/d; s/ .* / /; /^$$/d' > $(call symfile,kernel)
```

为了生成kernel，首先需要 

kern/debug : kdebug.o kmonitor.o panic.o 

kern/driver : clock.o console.o intr.o picirq.o 

kern/init : init.o

kern/libs : readline.o stdio.o 

kern/mm : pmm.o  

kern/trap : trap.o trapentry.o vectors.o

tools : kernel.ld

libs :  printfmt.o string.o

生成这些目标文件的代码如下

```
KINCLUDE	+= kern/debug/ \
			   kern/driver/ \
			   kern/trap/ \
			   kern/mm/

KSRCDIR		+= kern/init \
			   kern/libs \
			   kern/debug \
			   kern/driver \
			   kern/trap \
			   kern/mm

KCFLAGS		+= $(addprefix -I,$(KINCLUDE))

$(call add_files_cc,$(call listf_cc,$(KSRCDIR)),kernel,$(KCFLAGS))

KOBJS	= $(call read_packet,kernel libs)
```



一个被系统认为是符合规范的硬盘主引导扇区的特征是什么?

因为使用sign工具处理bootblock.out，生成bootblock，所以阅读sign.c代码发现符合规范的硬盘主引导扇区的大小为512字节，并且第511字节为 0x55，第512字节为0xAA，代码如下：

```
char buf[512];
    memset(buf, 0, sizeof(buf));
    FILE *ifp = fopen(argv[1], "rb");
    int size = fread(buf, 1, st.st_size, ifp);
    if (size != st.st_size) {
        fprintf(stderr, "read '%s' error, size is %d.\n", argv[1], size);
        return -1;
    }
    fclose(ifp);
    buf[510] = 0x55;
    buf[511] = 0xAA;
```

