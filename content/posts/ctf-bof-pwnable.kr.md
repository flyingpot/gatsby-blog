+++
categories = []
tags = ["CTF"]
date = 2020-08-15T16:00:00Z
title = "CTF从零单排（二）—— bof (pwnable.kr)"
url = "/post/ctf2"

+++
# 一、题目分析

查看题目给出的信息，一个C代码文件和一个可执行文件，C代码文件如下：

    #include <stdio.h>
    #include <string.h>
    #include <stdlib.h>
    void func(int key){
    	char overflowme[32];
    	printf("overflow me : ");
    	gets(overflowme);	// smash me!
    	if(key == 0xcafebabe){
    		system("/bin/sh");
    	}
    	else{
    		printf("Nah..\n");
    	}
    }
    int main(int argc, char* argv[]){
    	func(0xdeadbeef);
    	return 0;
    }

可以看出这道题考的是栈溢出，从标准输入读取的数据覆盖掉func传入的参数值即可提权。关键问题就是如何构造这个数据。

## 二、题解

使用gdb对可执行文件bof进行分析：

首先使用start开始执行，方便之后使用地址打断点

然后使用disas func查看func函数的汇编代码

找到get函数调用后的比较语句并打断点

使用c（continue）继续执行代码

输入AAAAAA，使用x /40xw $esp查看栈数据。A用16进制表示是41，可以看到第一个A到deadbeef相差52个字节。因此我们只需要构造52个A加上cafebabe即可。

使用Python的pwn库：

成功拿到flag

由此可见，C代码中使用gets有多危险，使用gcc编译时也会提示gets的危险性。

## 三、遗留问题

虽然题目参考着其他人的题解做了出来，但是目前还是有两个问题我还没想明白，在这里记录一下：

1. 发现如果使用gcc默认编译选项编译出来的可执行文件（可能与64位有关），deadbeef参数在低地址，标准输入参数在高地址，不符合栈帧是从高地址向低地址生长（申请）的原则，很奇怪
2. 为什么0xdeadbeef写入栈中的时候没有按照小端原则？

   问题已解决，因为0xdeadbeef是int类型，占据了4个字节，所以无所谓大端小端，在内存中就是以0xdeadbeef形式保存的
