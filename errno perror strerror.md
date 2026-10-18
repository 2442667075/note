## 1. errno

- 头文件：`<errno.h>`（很多场景 stdio.h 间接包含，规范代码建议手动引入）
- 本质：**全局整型变量**，保存最近一次系统调用 / 库函数失败对应的错误编号。
- 核心规则
    1. **只有函数返回失败时，errno 才有意义；函数调用成功，errno 值不可信，不能用来判断是否出错。**
    2. 一旦调用其他库 / 系统函数，errno 会被覆盖。**出错后尽快使用 errno**。
    3. 不会自动清零：上次错误的编号会保留，直到下一次函数失败改写它。

示例：

```
FILE *fp = fopen("test.txt","r");
if(fp == NULL){
    // fopen失败，此时errno有效
}
```

## 2. perror()

- 头文件：`<stdio.h>`
- 原型：`void perror(const char *s);`
- 功能：
    1. 自动读取全局变量`errno`
    2. 拼接字符串：`s : 错误描述文字`，末尾自带换行
    3. 输出到 **stderr（标准错误流，fd=2）**
- 特点 ✅ 简单，一行打印错误 ✅ 输出 stderr，stdout 重定向时错误信息依旧打印屏幕 ❌ 格式固定，不能自定义输出内容
- 示例

```
perror("fopen test.txt");
// 输出：fopen test.txt: No such file or directory
```

## 3. strerror()

- 头文件：`<string.h>`
- 原型：`char *strerror(int errnum);`
- 功能：传入错误号 errnum，**返回对应的错误描述字符串，不会打印任何内容**。
- 特点 ✅ 格式高度自定义，可以拼接 pid、时间等信息 ✅ 可以把错误信息存到变量，不直接输出 ❌ 需要配合`fprintf(stderr)`手动打印，需要自己加换行
- 等价替换：

```
perror("fopen test.txt");
// 等价于
fprintf(stderr, "fopen test.txt: %s\n", strerror(errno));
```

## 4. 三者关系一句话总结

`fopen`这类函数失败 → 设置`errno`； `perror`自动读`errno`并直接打印错误信息； `strerror`接收错误号，只返回错误字符串，打印需要自己写。

## 5. 易错坑点（必背）

1. ❌ 错误：先调用 printf，再读取 errno

```
if(fp == NULL)
{
    printf("open fail\n"); // printf可能改写errno！
    fprintf(stderr,"%s\n",strerror(errno));
}
```

✅ 正确：出错立刻保存 errno

```
if(fp == NULL)
{
    int save_err = errno; // 保存错误号
    printf("open fail\n");
    fprintf(stderr,"%s\n",strerror(save_err));
}
/*
- rintf 底层要执行**write 系统调用**，操作 stdout 缓冲区
- 如果 printf 内部系统调用过程中出现临时错误（哪怕 printf 最终执行成功！），**内核也会更新 errno**
- ⚠️重点：哪怕 printf 正常打印成功，也有可能把原来的 errno 覆盖成别的值
- */
```

2. stdout 与 stderr 重定向区别

- `printf` → stdout（fd=1），`> log.txt` 重定向 stdout
- perror / fprintf(stderr) → stderr（fd=2），`2> err.log`重定向 stderr

```
./app > log.txt        # stdout进文件，stderr屏幕输出
./app > all.log 2>&1   # stdout、stderr全部写入同一个文件
```

## 6. 使用场景选择

- 简单调试打印错误：**perror**
- 需要自定义日志格式（打印进程号、时间）：**strerror + fprintf(stderr)**

## 7. 面试简答背诵版

> errno 是全局错误编号，系统调用失败时赋值；perror 读取 errno，拼接提示字符串输出到 stderr；strerror 接收错误号，返回错误描述字符串，本身不输出。stderr 默认不跟随 stdout 重定向，用来区分正常输出和错误信息。

要不要我再给你出 2 道选择题自测巩固这个知识点？