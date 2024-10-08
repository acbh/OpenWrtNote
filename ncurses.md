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

简单菜单选项

```cpp
#include <ncurses.h>
#include <string>

using namespace std;

int main(int argc, char* argv[])
{
    initscr();
    noecho();
    cbreak();

    int yMax, xMax;
    getmaxyx(stdscr, yMax, xMax);

    WINDOW * inputwin = newwin(6, xMax-12, yMax-8, 5);
    box(inputwin, 0, 0);
    refresh();
    wrefresh(inputwin);

    keypad(inputwin, true);

    string choices[3] = {"walk", "Jog", "Run"};
    int choice;
    int highlight = 0;

    while (1)
    {
        for(int i = 0; i < 3; i ++)
        {
            if(i == highlight)
                wattron(inputwin, A_REVERSE);
            mvwprintw(inputwin, i + 1, 1, choices[i].c_str());
            wattroff(inputwin, A_REVERSE);
        }
        choice = wgetch(inputwin);

        switch(choice)
        {
            case KEY_UP:
                highlight --;
                if(highlight == -1) highlight = 2;
                break;
            case KEY_DOWN:
                highlight ++;
                if(highlight == 3) highlight = 0;
                break;
            default:
                break;
        }
        if(choice == 10)
            break;
    }
    printw("Your choice was: %s", choices[highlight].c_str());

    getch();
    endwin();

    return 0;
}
```

效果

```sh
     ┌─────────────────────────────────────────────────────────────────────────────────┐
     │walk                                                                             │
     │Jog                                                                              │
     │Run                                                                              │
     │                                                                                 │
     └─────────────────────────────────────────────────────────────────────────────────┘
```

添加鼠标支持

```c
#include <ncurses.h>
#include <stdlib.h>

typedef enum
{
	MODE_UP,
	MODE_DOWN,
	MODE_DOUBLE
} Mode;

void draw_mode(WINDOW *win, Mode mode)
{
	werase(win);	// 清除窗口内容
	box(win, 0, 0); // 绘制边框
	mvwprintw(win, 1, 1, "Current Mode: ");
	switch (mode)
	{
	case MODE_UP:
		wprintw(win, "UP");
		break;
	case MODE_DOWN:
		wprintw(win, "DOWN");
		break;
	case MODE_DOUBLE:
		wprintw(win, "DOUBLE");
		break;
	}
	wrefresh(win); // 刷新窗口内容
}

Mode handle_click(Mode mode)
{
	// 循环切换模式
	switch (mode)
	{
	case MODE_UP:
		return MODE_DOWN;
	case MODE_DOWN:
		return MODE_DOUBLE;
	case MODE_DOUBLE:
		return MODE_UP;
	}
	return mode;
}

int main()
{
	initscr();
	cbreak();
	noecho();
	keypad(stdscr, TRUE);
	mousemask(BUTTON1_CLICKED, NULL);

	// 检查终端是否支持鼠标
	if (!has_mouse())
	{
		endwin();
		printf("Mouse is not supported by this terminal.\n");
		exit(1);
	}

	int yMax, xMax;
	getmaxyx(stdscr, yMax, xMax);

	// 调整窗口大小并居中显示
	WINDOW *win = newwin(10, 50, (yMax / 2) - 3, (xMax / 2) - 20);
	box(win, 0, 0);

	Mode current_mode = MODE_UP;
	draw_mode(win, current_mode);

	while (1)
	{
		int ch = getch();
		if (ch == 'q')
			break;

		if (ch == KEY_MOUSE)
		{
			MEVENT event;
			if (getmouse(&event) == OK)
			{
				// 输出鼠标点击位置
				mvprintw(yMax - 2, 0, "Mouse clicked at (%d, %d)", event.x, event.y);
				refresh(); // 刷新主窗口以显示点击位置

				// 检查点击是否在窗口内
				if (event.x >= (xMax / 2) - 20 && event.x <= (xMax / 2) + 20 &&
					event.y >= (yMax / 2) - 3 && event.y <= (yMax / 2) + 1)
				{
					current_mode = handle_click(current_mode);
					draw_mode(win, current_mode);
				}
			}
		}
	}

	delwin(win);
	endwin();
	return 0;
}
```

![image-20240903162210087](/home/bhhh/snap/typora/90/.config/Typora/typora-user-images/image-20240903162210087.png)
