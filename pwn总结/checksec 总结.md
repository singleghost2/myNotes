# checksec 总结

## 一、RELRO

relro 是一种用于加强对 binary 数据段的保护的技术。relro 分为 partial relro 和 full relro

* Partial RELRO

  * 现在gcc 默认编译就是 partial relro（很久以前可能需要加上选项 gcc -Wl,-z,relro）
  * some sections(.init_array .fini_array .jcr .dynamic .got) are marked as read-only after they have been initialized by the dynamic loader
  * non-PLT GOT is read-only (.got)
  * GOT is still writeable (.got.plt)

* Full RELRO

  * 拥有 Partial RELRO 的所有特性

  * lazy resolution 是被禁止的，所有导入的符号都在 startup time 被解析

  * bonus: the entire GOT is also (re)mapped as read-only or the .got.plt section is completely initialized with the final addresses of the target functions (Merge .got and .got.plt to one section .got). Moreover,since lazy resolution is not enabled, the `GOT[1]` and `GOT[2]` entries are not initialized.

     `GOT[0]` is a the address of the module’s DYNAMIC section. `GOT[1]` is the virtual load address of the link_map, `GOT[2]` is the address for the runtime resolver function

  * 编译时需要加上选项  gcc -Wl,-z,relro,-z,now

    其中-z 参数是把-z 后面的 keyword 传给linker ld

    >  ld manual 中关于now 选项的解释

    >  When generating an executable or shared library, mark it to tell the dynamic linker to resolve all symbols when the program is started, or when the shared library is linked to using dlopen, instead of deferring function call resolution to the point when the function is first called.

> ld manual 中关于relro选项的解释
>
> Create an ELF "PT_GNU_RELRO" segment header in the object.



## 二、Fortify

fortify 技术是 gcc 在编译源码的时候会判断程序哪些buffer会存在 可能的溢出，在 buffer 大小已知的情况下，GCC 会把 strcpy，memcpy 这类函数自动替换成相应的 __strcpy_chk(dst, src, dstlen)等函数。GCC 在碰到以下四种情况的时候会采取不同的行为。

FORTIFY_SOURCE provides buffer overflow checks for the following functions:

```
memcpy, mempcpy, memmove, memset, strcpy, stpcpy, strncpy, strcat, 
strncat, sprintf, vsprintf, snprintf, vsnprintf, gets.
```

```C
char buf[5]; //buf = malloc(5)也一样

/* 1) Known correct.
      No runtime checking is needed, memcpy/strcpy
      functions are called (or their equivalents inline).  */
memcpy (buf, foo, 5);
strcpy (buf, "abcd");

/* 2) Not known if correct, but checkable at runtime.
      The compiler knows the number of bytes remaining in object,
      but doesn't know the length of the actual copy that will happen.
      Alternative functions __memcpy_chk or __strcpy_chk are used in
      this case that check whether buffer overflow happened.  If buffer
      overflow is detected, __chk_fail () is called (the normal action
      is to abort () the application, perhaps by writing some message
      to stderr.  */
memcpy (buf, foo, n);
strcpy (buf, bar);

/* 3) Known incorrect.
      The compiler can detect buffer overflows at compile
      time.  It issues warnings and calls the checking alternatives
      at runtime.  */
memcpy (buf, foo, 6);
strcpy (buf, "abcde");

/* 4) 不知道 buf 的 size，就不会 check  */
memcpy (p, q, n);
strcpy (p, q);


```



gcc 中 -D_FORTIFY_SOURCE=2是默认开启的，但是只有开启 O2或以上优化的时候，这个选项才会被真正激活。

如果指定-D_FORTIFY_SOURCE=1，那同样也要开启 O1或以上优化，这个选项才会被真正激活。

可以使用`-U_FORTIFY_SOURCE` 或者`-D_FORTIFY_SOURCE=0`来禁用。

如果开启了-D_FORTIFY_SOURCE=2， 那么调用__printf_chk函数的时候会检查 format string 中是否存在%n，如果存在%n 而且 format string 是在一个可写的 segment 中的（不是在 read-only 内存段中），那么程序会报错并终止。如果是开启-D_FORTIFY_SOURCE=1，那么就不会报错。

```shell
➜  test_fortify ./fortify
aaa%n
*** %n in writable segment detected ***
aaa[1]    23468 abort      ./fortify
```

## 三、Canary

要检测对函数栈的破坏，需要修改函数栈的组织，在缓冲区和控制信息（如 EBP 等）间插入一个 canary word。这样，当缓冲区被溢出时，在返回地址被覆盖之前 canary word 会首先被覆盖。通过检查 canary word 的值是否被修改，就可以判断是否发生了溢出攻击。

### 常见的 canary word：

- Terminator canaries


- 由于绝大多数的溢出漏洞都是由那些不做数组越界检查的 C 字符串处理函数引起的，而这些字符串都是以 NULL 作为终结字符的。选择 NULL, CR, LF 这样的字符作为 canary word 就成了很自然的事情。例如，若 canary word 为 0x000aff0d，为了使溢出不被检测到，攻击者需要在溢出字符串中包含 0x000aff0d 并精确计算 canaries 的位置，使 canaries 看上去没有被改变。然而，0x000aff0d 中的 0x00 会使 strcpy() 结束复制从而防止返回地址被覆盖。而 0x0a 会使 gets() 结束读取。插入的 terminator canaries 给攻击者制造了很大的麻烦。


- Random canaries


- 这种 canaries 是随机产生的。并且这样的随机数通常不能被攻击者读取。这种随机数在程序初始化时产生，然后保存在一个未被隐射到虚拟地址空间的内存页中。这样当攻击者试图通过指针访问保存随机数的内存时就会引发 segment fault。但是由于这个随机数的副本最终会作为 canary word 被保存在函数栈中，攻击者仍有可能通过函数栈获得 canary word 的值。


- Random XOR canaries


- 这种 canaries 是由一个随机数和函数栈中的所有控制信息、返回地址通过异或运算得到。这样，函数栈中的 canaries 或者任何控制信息、返回地址被修改就都能被检测到了。

### 如何绕过 canary

1. 不覆盖 canary，只覆盖全局变量

2. 覆盖 canary，但是通过覆盖 canary 地址往上的函数参数、calling function 的栈帧等内容在函数 ret 之前 getshell

3. 改写__stack_chk_fail或者 exit的 got 表项，在 check canary 失败之后会调用这两个 libc 函数

4. 通过某种方式泄露出栈上的 canary 值，然后覆盖

5. 如果程序在 canary 被覆盖之后不会崩溃，而是重新运行，那么可以考虑每次覆盖 canary 的一个字节，一个字节一个字节的猜测 canary 的值

   ​

## 四、NX
由 CPU 提供支持，在页表中有Non-executable flag, 来限制一块内存区域不可执行。

### 绕过方法

1. mprotect()

## 五、PIE

[Postion-Indenpendent executable](https://zh.wikipedia.org/wiki/地址无关代码)(地址无关可执行文件)