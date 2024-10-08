#### GCC编译器

分步骤使用 GCC 编译 `main.cpp` 和 `header.cpp` 的过程：

1. 假设 `header.cpp` 中定义了一些函数和变量，`main.cpp` 中调用了这些函数。

2. 首先，分别编译这两个源文件为目标文件（`.o` 文件）：

   `gcc -c main.cpp`

   `gcc -c header.cpp`

   这将生成 `main.o` 和 `header.o` 两个目标文件。

3. 然后，将这两个目标文件链接成一个可执行文件：

   `gcc main.o header.o -o program`

   这里的`program`可以任取名字。

```shell
gcc -c main.cpp
gcc -c header.cpp
gcc mian.o header.o program
```

4. 以上三步可以直接一步执行

```shell
gcc main.cpp header.cpp -o program
```

对于大型项目，使用单纯的gcc命令会非常繁琐，因此引入了Makefile来管理编译过程。

#### Makefile

解决vscode编辑器编写makefile总是缺失分隔符的问题，原因是vscode默认使用空格来缩进，点击右下角, `空格：4`更改成制符表缩进即可。![img](https://cdn.nlark.com/yuque/0/2024/png/40475032/1723685136398-46184391-ee90-4dfb-9910-387b957fedc9.png)





