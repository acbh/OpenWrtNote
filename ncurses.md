#### ncurses界面库

用于设计界面显示，可以展示文本

安装

```bash
sudo apt-get install libncurses5
```

简单示例

```c
#include <ncurses.h>

int main()
{
    initscr(); // 初始化 ncurses 并创建终端窗口
    cbreak();  // 启用即时获取输入字符模式，无需等待回车键
    noecho();  // 禁止输入字符回显到屏幕

    mvaddch(0, 0, '+');  // 在 (0, 0) 位置添加字符 '+'
    mvaddch(LINES - 1, 0, '-');  // 在最后一行的 (0) 位置添加字符 '-'
    mvaddstr(10, 30, "press any key to quit");  // 在 (10, 30) 位置添加字符串 "press any key to quit"
    refresh();  // 刷新屏幕显示

    getch();  // 等待用户输入一个字符
    endwin();  // 结束 ncurses 并恢复终端原始状态
}
```

编译需要把`ncurses`库连接进来

```bash
gcc -o test test.c -lncurses
```

实现Up键Down键翻页效果

```c
#include <ncurses.h>
#include <stdlib.h>
#include <string.h>

#define max_items 50
#define items_per_page 10

void display(int page, char items[max_items][20], int total_pages)
{
	// 计算当前页码的起始和结束索引
	int start = page * items_per_page;
	int end = start + items_per_page;
	if (end > max_items)
	{
		end = max_items;
	}

	// 计算当前页码的标题
	char title[20];
	sprintf(title, "Page %d/%d", page + 1, total_pages);

	// 清除屏幕并显示标题
	erase();
	mvprintw(0, 0, title);

	// 显示当前页码的内容
	for (int i = start; i < end; i++)
	{
		mvprintw(i - start + 1, 0, items[i]);
	}

	// 刷新屏幕
	refresh();
}

int main()
{
	initscr();
	cbreak();
	noecho();
	keypad(stdscr, TRUE); // 启用方向键

	// 初始化一个字符串数组，每个元素存储一个格式化的字符串，表示一个项目
	char items[max_items][20];
	for (int i = 0; i < max_items; i++)
	{
		sprintf(items[i], "item %d", i + 1);
	}
	int total_pages = max_items / items_per_page + (max_items % items_per_page > 0); // 计算总页数
	int page = 0;																	 // 当前页码
	display(page, items, total_pages);												 // 显示当前页码的内容

	while (1)
	{
		int c = getch(); // 读取一个字符
		switch (c)
		{
		case KEY_UP: // 向上翻页
			if (page > 0)
			{
				page--;
				display(page, items, total_pages);
			}
			break;

		case KEY_DOWN: // 向下翻页
			if (page < total_pages - 1)
			{
				page++;
				display(page, items, total_pages);
			}
			break;

		case 'q': // 退出程序
			endwin();
			exit(0);
			break;
		}
	}

	endwin(); // 结束 ncurses 并恢复终端原始状态
	return 0;
}
```

更多`ncursesAPI`文档 [manpage](https://manpages.debian.org/bullseye/ncurses-doc/)