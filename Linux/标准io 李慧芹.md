## ✅ 本节课核心内容

### 1. 标准 IO 是什么

标准 IO 是**C 标准库（stdio.h）**提供的一套 IO 函数，**底层封装 Linux 系统调用（open/read/write/close）**。

- 系统调用：内核提供，跨平台不通用；
- 标准 IO：C 库封装，**自带用户态缓冲区，可移植性强，减少系统调用次数，提升效率**。

> 一句话：标准 IO = 用户空间缓冲区 + 系统调用封装。

### 2. 流（stream）概念

标准 IO 把所有读写对象抽象成**流**，用`FILE *`指针表示流。

- `FILE`：是一个结构体，内部包含：文件描述符 fd、缓冲区起始地址、缓冲区大小、当前读写位置、错误标记、EOF 标记等。

> 注意：**FILE 结构体对应用户层；fd 是内核层句柄**。

### 3. 程序启动自带 3 个标准流（进程一创建就自动打开）

```
stdin  → 标准输入，FILE*，对应fd=0，默认键盘
stdout → 标准输出，FILE*，对应fd=1，默认显示器
stderr → 标准错误，FILE*，对应fd=2，默认显示器
```

- `stdin`：读；`stdout/stderr`：写
- 面试考点：**stderr 无缓冲**，用来打印错误，保证出错信息立刻输出；stdout 是行缓冲。

### 4. 三大缓冲类型（本节课重点，后面大量坑都来自缓冲）

1. **全缓冲（fully buffered）** 缓冲区满了，才会调用系统调用写到内核。普通文件默认是全缓冲。
2. **行缓冲（line buffered）** 遇到`\n`换行，或者缓冲区满，才刷新。终端 stdout 默认行缓冲。
    
    > 经典例子：`printf("hello");`不加`\n`，不会立刻打印。
    
3. **无缓冲（unbuffered）** 只要调用写函数，立刻下发内核。stderr 默认无缓冲。

> 刷新缓冲区的触发条件（视频讲的）

4. 缓冲区满；
5. 遇到换行（仅行缓冲）；
6. 调用`fflush(fp)`强制刷新；
7. 程序正常退出`exit()`，会自动刷新所有标准 IO 流；
8. `fclose(fp)`关闭流，自动先刷新缓冲区。

### 5. 标准 IO 优缺点

✅优点

1. 可移植（Windows/Linux 都能用 stdio）
2. 用户层缓冲，减少系统调用，磁盘 IO 性能更好

❌缺点

1. 多了一层用户缓冲区，**和底层 fd 混用会发生数据错位（大坑）**
2. 缓冲区会带来数据延迟问题（没刷新前数据只在用户内存，没进内核）

### 6. 本节课没有复杂 API，主要建立认知

本节课**没有复杂代码 demo**，以原理讲解为主； 视频演示小实验：`printf`不加换行，程序 sleep，观察输出行为，直观理解**stdout 行缓冲**。

#### 👉 本节课配套实验代码（自己敲，验证视频缓冲现象）

```
#include <stdio.h>
#include <unistd.h>

int main(void)
{
    printf("Hello, p124 标准IO"); // 没有 \n
    sleep(3);
    printf("\n");
    return 0;
}
```

编译运行：

```
gcc buf_test.c -o buf_test
./buf_test
```

现象：**程序先 sleep 3 秒，然后一次性打印文字**。 原因：stdout 行缓冲，没有`\n`，数据停留在用户缓冲区，sleep 期间不会输出；等到`\n`才刷新缓冲区输出。

> 对比：换成 stderr 测试

```
fprintf(stderr, "stderr test");
sleep(3);
```

现象：**直接立刻打印，不会等 sleep 结束**，因为 stderr 无缓冲。

### 📌 本节课思考题（学完自测）

1. stdin/stdout/stderr 对应的 FILE * 和 fd 分别是？
2. 行缓冲、全缓冲、无缓冲分别适用场景？
3. printf 不加 \n 为什么不会马上输出？
4. 程序 return 0 退出的时候，标准 IO 缓冲区会不会自动刷新？


# p125｜fopen 函数 1

> 视频地址：[https://www.bilibili.com/video/BV18p4y167Md/?p=125](https://www.bilibili.com/video/BV18p4y167Md/?p=125) 主题：`fopen` 打开文件、6 种打开模式（r/w/a/r+/w+/a+），FILE * 返回值

## ✅ 本节课核心内容

### 1. fopen 函数原型

```
#include <stdio.h>
FILE *fopen(const char *pathname, const char *mode);
```

- 参数 1：`pathname` 文件路径字符串（相对路径 / 绝对路径）
- 参数 2：`mode` 打开模式，字符串，**必须双引号**，不是字符！
- 返回值：成功返回 FILE * 流指针；失败返回`NULL`

> 重点：**打开失败返回 NULL，一定要判断，否则野指针！**

### 2. 6 种打开 mode（本节课核心，面试必考）

表格

|mode|读写权限|文件不存在|文件已存在|文件位置|
|---|---|---|---|---|
|`"r"`|只读 read|打开失败|正常打开|文件开头|
|`"w"`|只写 write|创建新文件|**清空原有内容（截断）**|文件开头|
|`"a"`|追加 append|创建新文件|保留原有内容|**文件末尾**|
|`"r+"`|读 + 写|打开失败|正常打开|文件开头|
|`"w+"`|读 + 写|创建新文件|**清空原有内容**|文件开头|
|`"a+"`|读 + 写|创建新文件|保留原有内容|读在开头，**写永远追加到末尾**|

> 视频强调的关键点：

1. `w / w+` 只要文件存在，**直接清空内容**，小心误删数据！
2. `a / a+` 无论读写指针在哪，**所有写操作一定追加到文件尾部（原子追加）**。
3. `r` 和 `r+` 文件必须事先存在，否则返回 NULL 打开失败。

### 3. 文件权限（新建文件时）

当 fopen 的 mode 是 w/a/w+/a+，需要新建文件时： 新建文件的权限 = `0666 & ~umask`

> 注意：fopen 不能直接指定权限，由系统 umask 掩码控制，后面 p127 会细讲 umask。

### 4. 简单示例代码（视频演示 demo）

```
#include <stdio.h>

int main(void)
{
    FILE *fp;
    fp = fopen("test.txt", "r"); //只读打开
    
    if (fp == NULL)
    {
        perror("fopen test.txt"); //打印错误原因
        return 1;
    }

    printf("open success\n");

    fclose(fp); // 打开的流必须关闭！
    return 0;
}
```

编译运行

```
gcc fopen_demo.c -o fopen_demo
./fopen_demo
```

- 如果当前目录没有 test.txt：`fopen`返回 NULL，perror 打印 `fopen test.txt: No such file or directory`
- 创建 test.txt 后再运行：打印 open success

#### 测试其他模式，修改 mode 参数测试：

```
fp = fopen("test_w.txt", "w"); // w模式，存在就清空
fp = fopen("test_a.txt", "a"); // a追加模式
```

### 5. 易错点（视频重点提醒）

1. mode 写成 `'r'`（单引号）❌ 错误！必须是字符串 `"r"`
2. 忘记判断返回值 NULL，直接读写，程序段错误
3. `"w"` 模式误打开已有文件，直接清空原有数据，生产事故来源
4. 打开文件用完**必须 fclose**，否则资源泄露

## 📌 本节课自测题

1. `"r+"` 和 `"a+"` 的核心区别是什么？
2. `"w+"` 和 `"r+"` 区别？
3. fopen 打开失败返回什么，怎么打印错误信息？


# p126｜fopen 函数 2

> 视频地址：[https://www.bilibili.com/video/BV18p4y167Md/?p=126](https://www.bilibili.com/video/BV18p4y167Md/?p=126) 主题：errno 全局错误号、perror、strerror，fopen 打开失败的完整错误处理

## ✅ 本节课核心内容

### 1. errno 介绍

`errno`：**全局整型变量**，系统调用 / 库函数出错时，会把错误编号存入`errno`。

- 头文件：`#include <errno.h>`
- 注意重点（视频反复强调）：
    1. 不会自动清零：上次错误的编号会保留，直到下一次函数失败改写它。。
    2. 必须**在函数失败后立刻读取 errno**，后面别的函数调用会覆盖 errno 的值。
    3. errno 只是一个数字，人看不懂，需要工具转成文字描述。

### 2. 两个打印错误信息的函数

#### ① perror () 【最常用，视频示例首选】

```
#include <stdio.h>
void perror(const char *s);
```

- 作用：打印自定义字符串`s` + 冒号 + errno 对应的错误描述 + 换行
- 原理：内部自动读取 errno，翻译成字符串输出到 stderr（无缓冲）

> 例：`perror("fopen file");` 输出：`fopen file: No such file or directory`

#### ② strerror()

```
#include <string.h>
char *strerror(int errnum);
```

- 参数：传入 errno 值，返回对应错误描述字符串
- 适合：想把错误信息存变量、写入日志文件，而不是直接打印到终端 示例：`printf("err: %s\n", strerror(errno));`

### 3. 完整健壮版 fopen 代码（本节课示例）

```
#include <stdio.h>
#include <errno.h>
#include <string.h>

int main(void)
{
    FILE *fp;
    fp = fopen("test.txt", "r");

    if (fp == NULL)
    {
        // 两种方式打印错误，二选一即可
        perror("fopen test.txt");
        // printf("fopen error: %s\n", strerror(errno));
        return 1;
    }

    printf("文件打开成功\n");
    fclose(fp);
    return 0;
}
```

> 编译运行：

```
gcc fopen_err.c -o fopen_err
./fopen_err
```

没有 test.txt 时输出：`fopen test.txt: No such file or directory`

### 4. 视频演示的几个常见 errno 场景

1. ENOENT：文件不存在，`"r"`模式打开不存在文件就触发
2. EACCES：权限不足，文件存在但没有读 / 写权限
3. EISDIR：你传入的路径是一个目录，不是普通文件

### 5. 高频坑点（视频重点）

1. ❌ 错误写法：先 printf，再 perror

```
fp = fopen("test.txt","r");
printf("open failed\n");   // printf会修改errno！
perror("fopen");
```

> 问题：一旦 printf 内部出错，会覆盖原来 fopen 产生的 errno，错误信息错乱！ ✅ 原则：**函数失败后，第一时间处理 errno/perror**，中间不要插入其他库函数调用。

2. errno 不是 0 代表出错：不能用`if(errno !=0)`判断函数是否失败！

> 正确逻辑：**判断函数返回值**（fopen 返回 NULL 才代表失败），再看 errno。 错误示范：

```
fopen(xxx,"r");
if(errno !=0){ ... } // 大错！成功时errno可能残留上次旧值
```

## 📌 本节课自测题

1. errno 成功的时候会自动置 0 吗？
2. perror 输出到哪个流（stdout 还是 stderr）？
3. 为什么不能用 errno !=0 判断函数是否出错？
4. perror 和 strerror 的区别，各自适用场景？


## p127：fclose 和文件权限问题，umask

跑完代码，理解 errno 的坑，没问题我们继续 p127？

今天 20:06

下一个

今天 20:15

# p127｜fclose 和文件权限问题、umask

> 视频地址：[https://www.bilibili.com/video/BV18p4y167Md/?p=127](https://www.bilibili.com/video/BV18p4y167Md/?p=127) 主题：`fclose`函数、流关闭时缓冲区行为、新建文件权限、umask 掩码

## ✅ 本节课核心内容

### 1. fclose 函数原型

```
#include <stdio.h>
int fclose(FILE *fp);
```

- 参数：已经打开的 FILE * 文件流指针
- 返回值：成功返回 0；失败返回`EOF`（-1）

> 重点：

1. **fclose 会自动刷新用户缓冲区**，把缓冲区数据写入内核；再关闭底层 fd。
2. 如果不调用 fclose，程序正常退出时，标准 IO 库会自动帮你刷新并关闭；**但长期运行的服务程序一定要手动 fclose，不然文件描述符泄露**。
3. 传空指针`fclose(NULL)`会直接段错误！

### 2. fclose 失败场景

什么时候 fclose 返回错误？ 缓冲区在 fclose 刷盘的时候，发生 IO 错误（磁盘满、IO 硬件错误），此时 fclose 返回 EOF。

> 坑：很多人只检查 fopen 返回值，**不检查 fclose 返回值**，写的数据可能因为磁盘满，在 fclose 阶段才失败，数据丢失！

示例判断 fclose：

```
if(fclose(fp) == EOF)
{
    perror("fclose");
    return 1;
}
```

### 3. fopen 创建新文件时的权限规则（本节课核心）

当使用`w / a / w+ / a+`模式打开**不存在**的文件，会新建文件。 fopen 内部调用 open 系统调用时，默认 mode 是 **0666**

> 新建文件实际权限 = `0666 & (~umask)`

#### umask 是什么

umask：权限掩码，**用来剥夺权限**，不是赋予权限。 `umask`是进程属性，默认继承 shell 的 umask。 查看 shell 当前 umask：

```
umask
```

常见默认值：`0022`

- 0666 & ~0022 = 0644 → rw-r--r--

举例：

- umask=0022：新文件权限 0644
- umask=0002：新文件权限 0664

> 注意两点：

1. umask**只影响新建文件 / 文件夹**，对已经存在的文件无任何作用。
2. fopen 没有参数直接指定新建文件权限；如果需要精确控制权限，后面要用底层 open 系统调用。

### 4. 完整 demo（本节课示例代码）

```
#include <stdio.h>

int main(void)
{
    FILE *fp;
    fp = fopen("test_umask.txt", "w"); //w模式，不存在就创建

    if (fp == NULL)
    {
        perror("fopen");
        return 1;
    }
    fputs("hello umask\n", fp);

    // 检查fclose返回值
    if(fclose(fp) == EOF)
    {
        perror("fclose");
        return 1;
    }
    printf("done\n");
    return 0;
}
```

编译运行：

```
gcc fclose_umask.c -o fclose_umask
./fclose_umask
ls -l test_umask.txt
```

你可以看到文件权限，由 umask 算出。

### 5. 修改进程内 umask（代码里设置）

```
#include <stdio.h>
#include <sys/stat.h> //umask头文件
int main(void)
{
    umask(0000); //修改当前进程的umask，只对本进程生效！
    FILE *fp = fopen("umask0.txt","w");
    if(fp == NULL){ perror("fopen");return 1; }
    fputs("test\n",fp);
    fclose(fp);
    return 0;
}
```

> 注意：umask 参数是八进制，前面必须写 0。这个修改**只影响当前程序进程，不会改 shell 全局 umask**。

### 6. 视频强调易错点

1. 忘记 fclose，长时间循环打开文件，耗尽文件描述符（资源泄露）。
2. 误以为 fclose 一定成功，忽略 fclose 返回值，丢失数据。
3. umask 是**屏蔽权限**，不是直接设置文件权限，很多人搞反。
4. umask 对已经存在的文件无效，只有新建文件才生效。

## 📌 本节课自测题

1. fclose 做了哪两件核心事情？
2. 新建文件权限计算公式是什么？
3. umask (0000)，fopen w 新建文件，权限是多少？
4. 为什么长期运行服务器程序必须 fclose？

# p128 fgetc 和 fputc 单字符读写

> 视频地址：[https://www.bilibili.com/video/BV18p4y167Md/?p=128](https://www.bilibili.com/video/BV18p4y167Md/?p=128) 主题：`fgetc()`、`fputc()` 单个字符读写，EOF，文件结束判断

## ✅ 本节课核心内容

### 1. 函数原型

```
#include <stdio.h>
// 从流读取一个字符
int fgetc(FILE *stream);

// 向流写入一个字符
int fputc(int c, FILE *stream);
```

### 2. fgetc 重点（面试高频坑）

- 返回值是 **int**，不是 char！
- 成功：返回读到字符的**ASCII 码（unsigned char 转 int）**
- 失败 / 读到文件末尾：返回 `EOF`（宏，值为 `-1`）

> 为什么返回 int？ char 范围：0~255；EOF=-1。 如果返回 char，当文件里读到字节 0xff (255)，会和 EOF (-1) 混淆，无法区分到底是读到 0xff 还是文件结束。 所以必须用 int 接收返回值！

✅正确写法：`int ch = fgetc(fp);` ❌错误写法：`char ch = fgetc(fp);`

### 3. fputc 函数

- 参数`c`虽然写 int，实际只会取低 8bit 作为字符写入。
- 返回值：成功返回写入的字符；失败返回`EOF`。

### 4. EOF 说明

EOF 不是文件里真实存在的字符，**只是库函数返回的标记**。 两种情况 fgetc 返回 EOF：

1. 读到文件末尾（正常结束）
2. 读取出错（IO 错误）

> 如果要区分到底是到末尾还是出错： `feof(fp)`：判断是否到达文件末尾标记 `ferror(fp)`：判断流是否发生读写错误

```
if (feof(fp)){
    printf("到达文件末尾\n");
}
if (ferror(fp)){
    perror("read error");
}
```

## 5. 视频示例 1：读取文件，逐个字符打印

```
#include <stdio.h>

int main(void)
{
    FILE *fp;
    int ch;   // 必须int！

    fp = fopen("test.txt", "r");
    if (fp == NULL)
    {
        perror("fopen");
        return 1;
    }

    // 循环读，直到EOF
    while( (ch = fgetc(fp)) != EOF )
    {
        putchar(ch); // 输出到终端，等价于fputc(ch, stdout);
    }

    // 循环退出后，区分是读完还是读取出错
    if(ferror(fp))
    {
        perror("fgetc error");
    }
    else if(feof(fp))
    {
        printf("\nread file end\n");
    }

    fclose(fp);
    return 0;
}
```

编译运行：

```
gcc fgetc_demo.c -o fgetc_demo
./fgetc_demo
```

## 6. 视频示例 2：用 fgetc+fputc 实现文件拷贝（单字符版本 mycopy）

逐个字符读源文件，写入目标文件

```
#include <stdio.h>

int main(void)
{
    FILE *fpsrc, *fpdst;
    int ch;

    fpsrc = fopen("src.txt", "r");
    if(fpsrc == NULL){ perror("open src"); return 1; }

    fpdst = fopen("dst.txt", "w");
    if(fpdst == NULL){ perror("open dst"); fclose(fpsrc); return 1; }

    while( (ch = fgetc(fpsrc)) != EOF )
    {
        fputc(ch, fpdst);
    }

    if(ferror(fpsrc) || ferror(fpdst))
    {
        perror("copy error");
    }

    fclose(fpsrc);
    fclose(fpdst);
    return 0;
}
```

> 编译运行后：src.txt 内容复制到 dst.txt。 原理：标准 IO 自带缓冲区，虽然代码上是一个字符读写，但底层不是一次一个字节系统调用，缓冲区攒满才调用 write。

## 7. 本节课重点坑点（视频强调）

1. 用 char 接收 fgetc 返回值 → 经典 bug！二进制文件 0xff 会误判为 EOF。
2. 以为 EOF 是文件中的一个字符。
3. while 判断顺序：`(ch = fgetc(fp)) != EOF` 括号不能丢，赋值优先级低于不等号。
4. 循环结束直接认为一定读到文件末尾，忽略读写错误。

## 📌 自测题

1. fgetc 返回值为什么是 int，不能用 char？
2. fgetc 返回 EOF 代表哪两种情况，如何区分？
3. fputc 的参数是 int，写入文件保存的是多少 bit？

## ⏭️ 下一节 p129：fread 和 fwrite 块读写函数。

把上面拷贝代码跑一遍，验证文件复制功能，没问题我们进入 p129？

今天 20:29

继续

# p129｜fread 和 fwrite 块读写函数

> 视频地址：[https://www.bilibili.com/video/BV18p4y167Md/?p=129](https://www.bilibili.com/video/BV18p4y167Md/?p=129) 主题：二进制块读写，`fread` / `fwrite`，返回值判断，二进制文件读写

## ✅ 本节课核心内容

### 1. 函数原型

```
#include <stdio.h>
// 读：从流读取块数据
size_t fread(void *ptr, size_t size, size_t nmemb, FILE *stream);

// 写：向流写入块数据
size_t fwrite(const void *ptr, size_t size, size_t nmemb, FILE *stream);
```

参数统一说明：

- `ptr`：内存缓冲区首地址（存放读 / 写数据）
- `size`：**单个成员的字节大小**
- `nmemb`：希望读写多少个成员
- `stream`：文件流 `FILE*`

**返回值：成功读写到的成员个数（不是字节数！）**

- 正常读完：返回读到的成员数量，**小于 nmemb 不一定代表出错**（读到文件末尾）
- 写失败：返回小于 nmemb

> 总字节 = 返回值 × size

> ⚠️ 重点：`size_t` 是无符号类型！做减法判断的时候容易踩坑。

### 2. fread 特点

适合批量读，多用于**二进制文件**（图片、结构体数据）； 文本文件也能用，但文本一般用 fgets/fgetc。 当剩余数据不足一个`size`大小时，返回的成员数会小于 nmemb。

### 3. fwrite 特点

二进制写入，**不会自动追加换行，不做任何文本转换**。

> 注意：Windows 下标准 IO 文本模式会自动转换`\n<->\r\n`；Linux 下文本模式和二进制模式无差别。

### 4. 示例 1：基础读写 demo（视频演示）

```
#include <stdio.h>

int main(void)
{
    FILE *fp;
    char buf[128] = "hello fread fwrite test\n";
    size_t ret;

    // 写入
    fp = fopen("block.txt", "w");
    if(fp == NULL){ perror("fopen w"); return 1; }
    ret = fwrite(buf, sizeof(char), sizeof(buf), fp);
    printf("fwrite return members: %zu\n", ret);
    fclose(fp);

    // 读出
    char read_buf[128] = {0};
    fp = fopen("block.txt", "r");
    if(fp == NULL){ perror("fopen r"); return 1; }
    ret = fread(read_buf, sizeof(char), sizeof(read_buf), fp);
    printf("fread return members: %zu\n", ret);
    printf("content: %s\n", read_buf);
    fclose(fp);
    return 0;
}
```

### 5. 示例 2：结构体读写（二进制保存结构体，视频重点例子）

> 常用于把结构体直接存入文件，属于二进制持久化

```
#include <stdio.h>

struct Student
{
    char name[20];
    int age;
};

int main(void)
{
    FILE *fp;
    struct Student s1 = {"zhangsan", 20};
    struct Student s2;

    // 写结构体
    fp = fopen("stu.dat", "w");
    if(fp == NULL){ perror("fopen"); return 1; }
    fwrite(&s1, sizeof(struct Student), 1, fp);
    fclose(fp);

    // 读结构体
    fp = fopen("stu.dat", "r");
    fread(&s2, sizeof(struct Student), 1, fp);
    printf("name:%s, age:%d\n", s2.name, s2.age);
    fclose(fp);
    return 0;
}
```

> ⚠️ 缺点：这种结构体二进制存储，**跨平台、不同编译器、内存对齐不同，文件会不兼容**。

### 6. 示例 3：使用 fread+fwrite 实现文件拷贝（块拷贝版本 mycopy）

```
#include <stdio.h>
#define BUF_SIZE 1024

int main(void)
{
    FILE *fpsrc, *fpdst;
    char buf[BUF_SIZE];
    size_t ret;

    fpsrc = fopen("src.bin", "r");
    if(fpsrc == NULL){ perror("open src"); return 1; }
    fpdst = fopen("dst.bin", "w");
    if(fpdst == NULL){ perror("open dst"); fclose(fpsrc); return 1; }

    // 循环读取块
    while( (ret = fread(buf, sizeof(char), BUF_SIZE, fpsrc)) > 0 )
    {
        fwrite(buf, sizeof(char), ret, fpdst);
    }

    if(ferror(fpsrc))
    {
        perror("read error");
    }

    fclose(fpsrc);
    fclose(fpdst);
    return 0;
}
```

> 对比上一节 fgetc 单字符拷贝：代码层面批量读写，底层同样走标准 IO 缓冲区，性能更高。

## 7. 本节课坑点（视频重点强调）

1. **返回值是成员个数，不是字节数！** 最常见错误。 比如`fread(buf, 5, 2, fp)`，一次想读 2 个、每个 5 字节。如果只读到 5 字节，返回 1，不是 5。
2. `fread返回小于nmemb`，**不能直接判定出错**，必须用`ferror`区分是到文件末尾还是 IO 错误。
3. 二进制存储结构体：内存对齐问题，不适合跨平台。
4. size_t 无符号，`if(ret - 100 <0)` 这种写法会出 bug（无符号永远 >=0）。

## 📌 自测题

1. fread/fwrite 返回值含义？返回值代表字节数吗？
2. fread 返回值小于 nmemb，一定是读取出错吗？怎么区分？
3. 使用 fwrite 保存结构体到文件，存在什么隐患？


## # p130｜printf /scanf 函数族，fprintf /fscanf/sprintf

> 视频地址：[https://www.bilibili.com/video/BV18p4y167Md/?p=130](https://www.bilibili.com/video/BV18p4y167Md/?p=130) 主题：标准 IO 格式化输入输出家族，屏幕 / 文件 / 内存三类格式化函数

## ✅ 本节课核心内容

### 1. 函数家族总览（视频核心表格）

表格

|函数|作用|输出目标|
|---|---|---|
|`printf()`|格式化输出|stdout（终端屏幕）|
|`fprintf()`|格式化输出|指定 FILE * 流（文件 /stdout/stderr）|
|`sprintf()`|格式化输出|**内存字符数组（缓冲区）**|
|`scanf()`|格式化输入|stdin（键盘）|
|`fscanf()`|格式化输入|指定 FILE * 流|
|`sscanf()`|格式化输入|**内存字符数组**|

> 一句话总结：

- `f`开头 → 操作文件流 FILE*
- `s`开头 → 操作内存字符串
- 不带前缀 → 默认 stdin/stdout

### 2. 原型

```
#include <stdio.h>

int printf(const char *format, ...);
int fprintf(FILE *stream, const char *format, ...);
int sprintf(char *str, const char *format, ...);

int scanf(const char *format, ...);
int fscanf(FILE *stream, const char *format, ...);
int sscanf(const char *str, const char *format, ...);
```

**返回值**

- printf/fprintf/sprintf：**成功返回打印的字符个数；失败返回负数**
- scanf/fscanf/sscanf：**成功返回成功匹配到的数据项个数；读到末尾返回 EOF**

### 3. fprintf 示例（写入文件格式化文本，视频演示）

```
#include <stdio.h>

int main(void)
{
    FILE *fp = fopen("log.txt", "w");
    if(fp == NULL){
        perror("fopen");
        return 1;
    }

    int id = 1001;
    char name[] = "zhangsan";
    float score = 92.5;

    // 格式化写入文件
    fprintf(fp, "id=%d, name=%s, score=%.2f\n", id, name, score);
    // fprintf(stdout, ...) 等价于 printf
    // fprintf(stderr, ...) 打印错误信息到标准错误

    fclose(fp);
    return 0;
}
```

运行后 log.txt 里面内容：

```
id=1001, name=zhangsan, score=92.50
```

### 4. fscanf 示例：从文件读取格式化数据

```
#include <stdio.h>

int main(void)
{
    FILE *fp = fopen("log.txt", "r");
    if(fp == NULL){
        perror("fopen");
        return 1;
    }
    int id;
    char name[32];
    float score;

    // 按照格式读取
    int ret = fscanf(fp, "id=%d, name=%s, score=%f", &id, name, &score);
    printf("匹配到 %d 个数据\n", ret);
    printf("id:%d name:%s score:%.2f\n",id,name,score);

    fclose(fp);
    return 0;
}
```

> fscanf 遇到空白（空格、换行、tab）自动分隔，**不推荐用来读带空格的字符串**。

### 5. sprintf：把格式化内容输出到内存字符串（重点 + 坑）

```
#include <stdio.h>

int main(void)
{
    char buf[64];
    int a = 100;
    sprintf(buf, "number = %d", a);
    printf("%s\n", buf);
    return 0;
}
```

⚠️ **巨大安全坑（视频重点强调）** `sprintf`不检查缓冲区长度！如果格式化后的字符串长度超过 buf 大小 → **缓冲区溢出，内存越界，高危漏洞**。 ✅安全替代：`snprintf(buf, sizeof(buf), "xxx", ...);` 第二个参数指定最多写入字节数，防止溢出。

```
// snprintf安全版本
snprintf(buf, sizeof(buf), "number = %d", a);
```

### 6. sscanf：从内存字符串中提取数据

```
#include <stdio.h>
int main(void)
{
    char str[] = "id:1002 age:25";
    int id,age;
    int ret = sscanf(str, "id:%d age:%d", &id, &age);
    printf("id=%d, age=%d\n",id,age);
    return 0;
}
```

用处：字符串解析，协议解析，提取数字，嵌入式很常用。

## 7. 视频强调重点坑点

1. `scanf/fscanf` 参数变量**必须传地址**，忘记 & 直接段错误。
2. sprintf 无边界检查，极易缓冲区溢出，开发优先用 snprintf。
3. 返回值：scanf 系列返回**成功匹配项数量**，不是字符数！ 例：`fscanf`期待读 3 个变量，只读到 2 个，返回 2。读到文件末尾返回 EOF。
4. `%s`在 scanf/fscanf 遇到空格就截断，不能读取带空格的一整行。

## 📌 自测题

1. printf /fprintf/sprintf 的输出目标分别是什么？
2. sprintf 有什么安全问题，用哪个函数替代？
3. fscanf 返回值含义是什么？


# p130｜printf /scanf 函数族，fprintf /fscanf/sprintf

> 视频地址：[https://www.bilibili.com/video/BV18p4y167Md/?p=130](https://www.bilibili.com/video/BV18p4y167Md/?p=130) 主题：标准 IO 格式化输入输出家族，屏幕 / 文件 / 内存三类格式化函数

## ✅ 本节课核心内容

### 1. 函数家族总览（视频核心表格）

表格

|函数|作用|输出目标|
|---|---|---|
|`printf()`|格式化输出|stdout（终端屏幕）|
|`fprintf()`|格式化输出|指定 FILE * 流（文件 /stdout/stderr）|
|`sprintf()`|格式化输出|**内存字符数组（缓冲区）**|
|`scanf()`|格式化输入|stdin（键盘）|
|`fscanf()`|格式化输入|指定 FILE * 流|
|`sscanf()`|格式化输入|**内存字符数组**|

> 一句话总结：

- `f`开头 → 操作文件流 FILE*
- `s`开头 → 操作内存字符串
- 不带前缀 → 默认 stdin/stdout

### 2. 原型

```
#include <stdio.h>

int printf(const char *format, ...);
int fprintf(FILE *stream, const char *format, ...);
int sprintf(char *str, const char *format, ...);

int scanf(const char *format, ...);
int fscanf(FILE *stream, const char *format, ...);
int sscanf(const char *str, const char *format, ...);
```

**返回值**

- printf/fprintf/sprintf：**成功返回打印的字符个数；失败返回负数**
- scanf/fscanf/sscanf：**成功返回成功匹配到的数据项个数；读到末尾返回 EOF**

### 3. fprintf 示例（写入文件格式化文本，视频演示）

```
#include <stdio.h>

int main(void)
{
    FILE *fp = fopen("log.txt", "w");
    if(fp == NULL){
        perror("fopen");
        return 1;
    }

    int id = 1001;
    char name[] = "zhangsan";
    float score = 92.5;

    // 格式化写入文件
    fprintf(fp, "id=%d, name=%s, score=%.2f\n", id, name, score);
    // fprintf(stdout, ...) 等价于 printf
    // fprintf(stderr, ...) 打印错误信息到标准错误

    fclose(fp);
    return 0;
}
```

运行后 log.txt 里面内容：

```
id=1001, name=zhangsan, score=92.50
```

### 4. fscanf 示例：从文件读取格式化数据

```
#include <stdio.h>

int main(void)
{
    FILE *fp = fopen("log.txt", "r");
    if(fp == NULL){
        perror("fopen");
        return 1;
    }
    int id;
    char name[32];
    float score;

    // 按照格式读取
    int ret = fscanf(fp, "id=%d, name=%s, score=%f", &id, name, &score);
    printf("匹配到 %d 个数据\n", ret);
    printf("id:%d name:%s score:%.2f\n",id,name,score);

    fclose(fp);
    return 0;
}
```

> fscanf 遇到空白（空格、换行、tab）自动分隔，**不推荐用来读带空格的字符串**。

### 5. sprintf：把格式化内容输出到内存字符串（重点 + 坑）

```
#include <stdio.h>

int main(void)
{
    char buf[64];
    int a = 100;
    sprintf(buf, "number = %d", a);
    printf("%s\n", buf);
    return 0;
}
```

⚠️ **巨大安全坑（视频重点强调）** `sprintf`不检查缓冲区长度！如果格式化后的字符串长度超过 buf 大小 → **缓冲区溢出，内存越界，高危漏洞**。 ✅安全替代：`snprintf(buf, sizeof(buf), "xxx", ...);` 第二个参数指定最多写入字节数，防止溢出。

```
// snprintf安全版本
snprintf(buf, sizeof(buf), "number = %d", a);
```

### 6. sscanf：从内存字符串中提取数据

```
#include <stdio.h>
int main(void)
{
    char str[] = "id:1002 age:25";
    int id,age;
    int ret = sscanf(str, "id:%d age:%d", &id, &age);
    printf("id=%d, age=%d\n",id,age);
    return 0;
}
```

用处：字符串解析，协议解析，提取数字，嵌入式很常用。

## 7. 视频强调重点坑点

1. `scanf/fscanf` 参数变量**必须传地址**，忘记 & 直接段错误。
2. sprintf 无边界检查，极易缓冲区溢出，开发优先用 snprintf。
3. 返回值：scanf 系列返回**成功匹配项数量**，不是字符数！ 例：`fscanf`期待读 3 个变量，只读到 2 个，返回 2。读到文件末尾返回 EOF。
4. `%s`在 scanf/fscanf 遇到空格就截断，不能读取带空格的一整行。

## 📌 自测题

1. printf /fprintf/sprintf 的输出目标分别是什么？
2. sprintf 有什么安全问题，用哪个函数替代？
3. fscanf 返回值含义是什么？

# 131｜fseeko、ftello 文件偏移，大文件支持

> 视频地址：[https://www.bilibili.com/video/BV18p4y167Md/?p=131](https://www.bilibili.com/video/BV18p4y167Md/?p=131) 主题：文件读写位置偏移，`fseeko` / `ftello`，对比旧版`fseek/ftell`，大文件（超过 2G）支持

## ✅ 本节课核心内容

### 1. 旧版函数 fseek /ftell（先了解，有缺陷）

```
#include <stdio.h>
int fseek(FILE *stream, long offset, int whence);
long ftell(FILE *stream);
```

- `fseek`：移动文件读写位置指针
- `ftell`：获取当前文件位置偏移量（返回从文件开头算起的字节数）
- `whence` 三个基准位置：
    1. `SEEK_SET`：文件开头
    2. `SEEK_CUR`：当前位置
    3. `SEEK_END`：文件末尾

> ❌ 缺陷：参数类型是`long`，在 32 位系统 long 只有 32bit，**最大只能寻址 2GB 文件**；超过 2G 的大文件会溢出失效。

### 2. 本节课重点：fseeko /ftello（推荐，支持大文件）

```
#include <stdio.h>
int fseeko(FILE *stream, off_t offset, int whence);
off_t ftello(FILE *stream);
```

- 参数：
    - `stream`：文件流
    - `offset`：偏移量，类型`off_t`（64 位长整型，支持超大文件）
    - `whence` 和 fseek 完全一样：`SEEK_SET / SEEK_CUR / SEEK_END`
- 返回值
    - fseeko：成功返回 0；失败返回非 0
    - ftello：成功返回当前偏移；失败返回`(off_t)-1`

> 简单一句话：**fseeko/ftello = fseek/ftell 的 64 位大文件版本**。

> 编译注意：有些系统需要定义宏 `_FILE_OFFSET_BITS=64` 来开启 64 位 off_t，Linux 下一般默认支持。

### 3. 视频基础示例代码

```
#include <stdio.h>

int main(void)
{
    FILE *fp = fopen("seektest.txt", "r+");
    if(fp == NULL){
        perror("fopen");
        return 1;
    }

    off_t pos;
    pos = ftello(fp);
    printf("初始位置 pos = %lld\n", (long long)pos);

    // 移动到文件第10字节位置 SEEK_SET
    fseeko(fp, 10, SEEK_SET);
    pos = ftello(fp);
    printf("移动后 pos = %lld\n", (long long)pos);

    // 从当前位置向后偏移3字节
    fseeko(fp, 3, SEEK_CUR);
    pos = ftello(fp);
    printf("当前 pos = %lld\n", (long long)pos);

    // 定位到文件末尾
    fseeko(fp, 0, SEEK_END);
    pos = ftello(fp);
    printf("文件总字节数 pos = %lld\n", (long long)pos);

    fclose(fp);
    return 0;
}
```

> 注意：`off_t`打印不能直接用`%d`，要强转`long long`用`%lld`输出。

### 4. 利用 fseeko+ftello 获取文件总大小（视频演示）

原理：偏移到文件末尾，ftello 拿到的值就是文件字节大小

```
off_t get_file_size(FILE *fp)
{
    fseeko(fp, 0, SEEK_END);
    off_t size = ftello(fp);
    // 用完最好把文件指针恢复到开头，避免后续读写位置错乱
    fseeko(fp, 0, SEEK_SET);
    return size;
}
```

### 5. 关键特性 & 坑点（视频重点强调）

1. 文件偏移是**流的属性（用户层 FILE 结构体维护）**，不是代码变量；读写操作会自动向后移动偏移。
2. `r+`读写模式下，**读后面直接写、写后面直接读，中间必须重新定位（fseeko）或者 fflush**，否则行为未定义！（后面重点大坑）
3. 偏移可以设置到文件末尾之后，此时写入会形成**空洞文件**（p144 会专门讲空洞文件）。空洞位置不占用磁盘块，读出来是 0。
4. 文本文件慎用 fseek/fseeko！Windows 文本模式`\n<->\r\n`转换，字节偏移不准；**二进制文件才推荐使用随机定位**。Linux 中文本和二进制模式无差别。
5. ftello 失败返回`(off_t)-1`，判断的时候要小心，off_t 是无符号的情况会踩坑。

### 6. rewind 顺带一提（视频简单带过）

```
void rewind(FILE *stream);
```

等价于：`fseeko(fp,0,SEEK_SET);`，并且清除流的错误标记。直接把文件指针跳回文件开头。

## 📌 本节课自测题

1. fseeko 对比 fseek 最大优势是什么？类型差别在哪？
2. whence 三个参数 SEEK_SET SEEK_CUR SEEK_END 含义？
3. 获取文件大小的原理是什么？
4. 文本文件为什么不推荐用 fseeko 随机偏移？

# 132｜文件位置函数、缓冲区刷新 fflush ()，缓冲区刷新条件，用户态缓冲与内核缓冲区别

> 视频地址：[https://www.bilibili.com/video/BV18p4y167Md/?p=132](https://www.bilibili.com/video/BV18p4y167Md/?p=132) 主题：fflush、缓冲层级（用户缓冲区 vs 内核页缓存）、标准 IO 读写切换规则

## ✅ 本节课核心内容

### 1. fflush 函数原型

```
#include <stdio.h>
int fflush(FILE *stream);
```

- 功能：**强制刷新【用户空间缓冲区】的数据写入内核缓冲区（页缓存）**

> ⚠️ 重点：fflush **不能保证数据落地磁盘！** 仅仅把用户层数据推到内核。内核什么时候刷到物理磁盘由操作系统调度。 如果需要强制刷到硬件磁盘，后面要用 `fsync()`（系统调用，不是标准 IO）。

- 参数：
    - `fp`：指定流，`fflush(fp)` 只刷新这个流
    - `fflush(NULL)`：刷新**当前进程所有打开的输出流**（输入流不会刷新）
- 返回值：成功返回 0；失败返回 EOF。

> 注意：**fflush 对输入流（读的流）行为是未定义的！不要对读文件的 fp 调用 fflush**，只用于输出流。

### 2. 复习：缓冲区自动刷新条件（本节课汇总，p124 提到过）

触发用户缓冲区刷新（调用底层 write 系统调用）的场景：

1. 缓冲区写满；
2. 行缓冲遇到 `\n`（stdout 终端场景）；
3. 手动调用`fflush(fp)`强制刷新；
4. `fclose(fp)`关闭流，自动先 fflush；
5. 程序正常`exit()`退出，自动刷新所有标准 IO 输出流；
6. 主函数`return 0`本质会调用 exit，也会自动刷新。

> 异常崩溃（段错误、kill -9）：**不会自动刷新缓冲区，用户缓冲区数据丢失！**

### 3. 两层缓冲（视频核心概念，面试高频）

1. **用户态缓冲区（标准 IO 库维护，FILE 结构体内部）** fputc/fprintf/fwrite 先写到这里；fflush 把这层数据交给内核。**属于 C 库，不是内核**。
2. **内核页缓存（操作系统内核维护）** write 系统调用写入这里；内核在合适时机异步写入磁盘。
    - fflush：用户缓冲区 → 内核缓存
    - fsync：内核缓存 → 物理磁盘（系统调用，不属于 stdio）

> 一句话区分： fflush：清用户缓冲；fsync：强制内核把缓存刷进硬盘。

### 4. 读写切换的强制规则（本节课大坑！重点）

> 在 `r+ / w+ / a+` 读写模式下： **读操作后面直接写操作，或者写后面直接读，中间必须二选一：fflush 或者 fseeko/rewind** 否则行为未定义，数据错乱！

原因：FILE 内部维护一套用户缓冲区 + 读写位置，读写切换不刷新会造成缓冲和文件偏移不同步。

✅合法示例：读 → fflush → 写；或者读 → fseeko → 写。

### 5. 视频演示代码示例

```
#include <stdio.h>
#include <unistd.h>

int main(void)
{
    FILE *fp = fopen("flush_test.txt", "w");
    if(fp == NULL){
        perror("fopen");
        return 1;
    }

    fprintf(fp, "test fflush data");
    // 此时数据还停留在【用户缓冲区】，没有进内核
    printf("sleep 5秒，此时还没进内核\n");
    sleep(5);

    fflush(fp); // 强制把用户缓冲写入内核页缓存
    printf("已经fflush，数据进入内核缓存，但不一定落盘\n");
    sleep(5);

    fclose(fp); // fclose内部自动fflush
    return 0;
}
```

> 测试现象：fflush 执行之后，你可以在文件里看到内容；但如果程序在 fflush 之前被 kill -9 杀死，文件是空的。

### 6. stdin 的 fflush（常见误区）

很多人写`fflush(stdin);`清空输入缓冲区。

> ❌ **C 标准中 fflush 对输入流是未定义行为！** Windows 编译器（MSVC）支持 fflush (stdin)，Linux gcc 下无效，不要跨平台依赖这个写法。Linux 清空 stdin 缓冲区要用别的方案。

## 📌 本节课自测题

1. fflush 作用是什么？fflush 之后数据一定写到磁盘上吗？
2. 用户缓冲区和内核页缓存的区别？
3. r + 模式，读完文件直接写，为什么需要 fflush 或者 fseeko？
4. kill -9 杀死程序，缓冲区数据会自动刷新吗？

# 133｜getline，按行读取，动态分配缓冲区

> 视频地址：[https://www.bilibili.com/video/BV18p4y167Md/?p=133](https://www.bilibili.com/video/BV18p4y167Md/?p=133) 主题：`getline()` 按行读取文件，**自动动态 malloc 缓冲区**，替代 fgets，解决固定数组长度限制

## ✅ 本节课核心内容

### 1. 函数原型

```
#include <stdio.h>
ssize_t getline(char **lineptr, size_t *n, FILE *stream);
```

> ⚠️ 注意：`getline` 是**GNU 扩展函数**（Linux 有，标准 C 没有，Windows 不自带）。

参数解释：

1. `lineptr`：**char**指针的地址。
    - 如果`*lineptr=NULL`：getline 内部自动`malloc`分配内存；
    - 如果`*lineptr`已经指向一块 malloc 好的内存，就复用这块内存；不够大时内部自动`realloc`扩容。
2. `n`：size_t 类型指针，存放`*lineptr`指向缓冲区的**总字节容量**。扩容后 getline 会自动更新`*n`的值。
3. `stream`：FILE * 文件流。

返回值：

- 成功：返回**读到的字符数量（包含末尾`\n`换行符）**
- 返回`-1`：读到文件末尾 EOF，或者读取出错

> 重点特性： ✅ 读到的行**会保留末尾的`\n`换行符**（和 fgets 一样） ✅ 缓冲区自动扩容，**不用预先定义固定大小数组**，不会出现 fgets 那种行太长截断的问题 ✅ 用完必须手动`free(*lineptr)`！哪怕读到 EOF，分配的内存依然存在，不 free 就内存泄漏

### 2. 和 fgets 的对比（视频重点对比）

- fgets：缓冲区固定大小，超长行会被截断，需要自己循环拼接
- getline：动态内存，一行多长就读多长；Linux 下读文本文件首选。

### 3. 视频示例代码 1（最常用：lineptr 传 NULL，自动分配内存）

```
#include <stdio.h>
#include <stdlib.h>

int main(void)
{
    FILE *fp = fopen("test_line.txt", "r");
    if(fp == NULL)
    {
        perror("fopen");
        return 1;
    }

    char *buf = NULL;
    size_t buf_len = 0;
    ssize_t read_cnt;

    // 循环逐行读取
    while( (read_cnt = getline(&buf, &buf_len, fp)) != -1 )
    {
        printf("读到字符数：%zd\n", read_cnt);
        printf("该行内容：%s", buf); // buf自带\n，不用额外加换行
    }

    free(buf); // 必须释放！！
    fclose(fp);
    return 0;
}
```

编译运行：

```
gcc getline_demo.c -o getline_demo
./getline_demo
```

### 4. 示例 2：自己预先分配缓冲区（复用内存，减少 realloc）

```
#include <stdio.h>
#include <stdlib.h>

int main(void)
{
    FILE *fp = fopen("test_line.txt", "r");
    if(fp == NULL){ perror("fopen"); return 1; }

    char *buf = malloc(128);
    size_t buf_len = 128;
    ssize_t ret;

    while( (ret = getline(&buf, &buf_len, fp)) != -1 )
    {
        printf("%s", buf);
    }

    free(buf);
    fclose(fp);
    return 0;
}
```

> 如果一行超过 128 字节，getline 内部自动 realloc 扩大 buf，更新 buf_len。

### 5. 坑点（视频反复强调）

1. **内存必须 free！** 只要 getline 分配过内存，不管成功读到行还是 EOF，最后一定要 free (*lineptr)，否则内存泄漏。
2. 返回值类型`ssize_t`是有符号整数，打印用`%zd`，不能用`%zu`（% zu 是 size_t 无符号）。
3. 读到的字符串包含`\n`，如果业务不需要换行，需要手动把`\n`替换成`\0`。

```
// 去掉末尾换行符示例
if(read_cnt > 0 && buf[read_cnt-1] == '\n')
{
    buf[read_cnt-1] = '\0';
}
```

4. getline 不是标准 C 库函数，**跨平台不通用，Windows MSVC 没有**，只适合 Linux 开发。
5. `lineptr`必须是 malloc 出来的堆内存，**不能传栈数组地址**！❌错误示范：

```
char arr[256];
getline(&arr, &n, fp); // 错误！栈内存不能realloc
```

## 📌 本节课自测题

1. getline 和 fgets 最大区别是什么？
2. getline 读取成功，返回值包含`\n`吗？
3. getline 使用完毕，为什么必须 free 缓冲区？
4. getline 是否属于 C 标准库函数？Windows 能不能直接用？

# p134｜临时文件 tmpfile、临时文件安全

> 视频地址：[https://www.bilibili.com/video/BV18p4y167Md/?p=134](https://www.bilibili.com/video/BV18p4y167Md/?p=134) 主题：`tmpfile()` 创建匿名临时文件，自动销毁特性，临时文件安全风险，顺带对比`tmpnam`（不推荐使用）

## ✅ 本节课核心内容

### 1. tmpfile () 函数原型

```
#include <stdio.h>
FILE *tmpfile(void);
```

- 功能：创建一个**匿名临时文件**，以`wb+`模式打开（二进制读写）
- 返回值：成功返回`FILE*`流指针；失败返回`NULL`

> 核心特性（视频重点）

1. 文件放在系统临时目录（Linux 默认`/tmp`）
2. **匿名，无文件名**，外部程序无法直接通过文件名访问这个文件，安全性更好
3. 自动销毁：
    - 调用`fclose(fp)`关闭流，文件立刻删除
    - 进程正常退出，会自动关闭并删除临时文件
4. 打开模式`wb+`：可读可写；二进制模式，Linux 文本 / 二进制无差别

> ⚠️ 注意：进程被`kill -9`强制杀死，临时文件**不会自动删除**，会残留在 /tmp。

### 2. tmpfile 示例代码（视频 demo）

```
#include <stdio.h>

int main(void)
{
    FILE *fp = tmpfile();
    if(fp == NULL)
    {
        perror("tmpfile");
        return 1;
    }

    // 写入临时文件
    fprintf(fp, "this is temp file data\n");

    // 把文件偏移移回文件开头，准备读取
    rewind(fp);

    char buf[128];
    fgets(buf, sizeof(buf), fp);
    printf("read from temp file: %s", buf);

    fclose(fp); // fclose之后，临时文件直接被系统删除
    return 0;
}
```

编译运行：

```
gcc tmpfile_demo.c -o tmpfile_demo
./tmpfile_demo
```

运行现象：程序运行期间，你在`/tmp`目录看不到这个文件名字；fclose 之后文件消失。

### 3. tmpnam 函数（视频重点：**不推荐，存在安全漏洞**）

```
#include <stdio.h>
char *tmpnam(char *s);
```

- 作用：生成一个唯一临时文件名字字符串
- 致命安全缺陷（竞争条件 TOCTOU 漏洞）：

> tmpnam 生成文件名 → 程序调用 fopen 创建文件。 在这两个操作**中间的极短时间窗口**，别的进程可以抢先创建同名文件 / 软链接，进行文件注入攻击。 文件名生成和文件创建不是原子操作！

✅结论：**禁止在生产代码使用 tmpnam**。

### 4. 安全的替代方案（视频提及）

- `tmpfile()`：推荐，匿名文件，创建 + 打开原子完成，无文件名竞争漏洞。缺点：拿不到文件路径，无法多进程共享。
- `mkstemp()`：系统调用（后面课程会见到），生成文件名并原子创建文件，返回 fd；可以拿到文件路径，适合需要文件名的场景。

### 5. 临时文件安全要点总结

1. 尽量使用`tmpfile()`，匿名文件，没有文件名竞争漏洞；
2. `tmpnam` 存在 TOCTOU 时间竞态漏洞，废弃不要使用；
3. `/tmp`是公共目录，所有用户都可读可写，自己创建带名字的临时文件一定要小心权限；
4. tmpfile 是标准 IO 函数，跨平台；mkstemp 是 POSIX 系统调用。

## 📌 本节课自测题

1. tmpfile 创建的临时文件什么时候自动删除？
2. tmpfile 的打开模式是什么？
3. tmpnam 为什么不安全？
4. kill -9 进程，tmpfile 生成的临时文件会被删除吗？

## ✅ 到此为止！【标准 IO 章节全部学完】p124~p134 结束