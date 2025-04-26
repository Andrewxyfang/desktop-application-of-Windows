# desktop-application-of-Windows
# C++窗体深造计划

## 讲解逻辑
窗口类函数（回调函数和窗口名字）--窗口类初始化--main函数--消息机制--初始化棋盘函数
--块的行为--重置功能--自定义摆放功能--判定机制
## 块的移动
1.重绘
2.局部重绘
```c
hwnd = CreateWindowW(
        szAppName, // 窗口类的名称
        TEXT("The Hello Program"), // 窗口的标题
        WS_OVERLAPPEDWINDOW, // 窗口样式，使用预定义的重叠窗口样式
        CW_USEDEFAULT, // 窗口左上角的 x 坐标，使用默认值
        0, // 窗口左上角的 y 坐标，设置为 0 表示从屏幕顶部开始
        CW_USEDEFAULT, // 窗口的宽度，使用默认值
        0, // 窗口的高度，系统会根据样式和内容确定高度
        nullptr, // 父窗口的句柄，设置为 nullptr 表示没有父窗口
        nullptr, // 窗口菜单的句柄，设置为 nullptr 表示没有菜单
        hInstance, // 应用程序实例的句柄
        nullptr // 传递给窗口过程的额外参数，设置为 nullptr 表示不传递任何数据
    );
```

## 华容道
对于桌面程序开发，我们首先需要清楚整个代码的运行流程，各种回调函数的调用条件，了解程序的消息传递机制，窗口控件的制作，布局等前置知识；其次，梳理项目的实现思路：对于华容道，首先我们需要制作布局，此处我利用按钮控件制作华容道块（实际我制作了16个块，始终隐藏0号块），采用大小为16的vector数组储存对应的华容道块的数值；其次完善华容道块的行为，初始化采用随机数交换vector内的值，渲染到按钮上，设计块的滑动行为；最后完善程序的其他功能，包含用户手动重置，成功还原华容道的判断函数，用户自定义布局华容道块摆放的功能

第一步--首先初始化一个4*4的数组
第二步--布局我们16个块的位置和内容（块是用Windows API中提供按钮类来实现的）
第三步--修改块的字体
代码：
```c++
#include "framework.h"
#include "game2.h"
#include <vector>
#include <ctime>
#include <cwchar>
#include <utility>

#define MAX_LOADSTRING 100
#define block_size 100
#define margin 20
#define numsize 4


// 全局变量:
HINSTANCE hInst;                                // 当前实例
WCHAR szTitle[MAX_LOADSTRING];                  // 标题栏文本
WCHAR szWindowClass[MAX_LOADSTRING];            // 主窗口类名
int num[numsize][numsize];
HWND boardButtons[numsize][numsize];
int zero_r, zero_c;
int margintop;

HFONT hFont = CreateFont(
    40,                        // 字体高度
    0,                         // 字体宽度（0表示默认）
    0,                         // 文本倾斜度
    0,                         // 字体倾斜度
    FW_BOLD,                 // 字体粗细（FW_NORMAL 为正常）
    FALSE,                     // 是否斜体
    FALSE,                     // 是否下划线
    FALSE,                     // 是否删除线
    DEFAULT_CHARSET,           // 字符集
    OUT_OUTLINE_PRECIS,        // 输出精度
    CLIP_DEFAULT_PRECIS,       // 剪裁精度
    CLEARTYPE_QUALITY,         // 输出质量
    VARIABLE_PITCH,            // 字间距和族
    TEXT("Arial")              // 字体名称
);

// 此代码模块中包含的函数的前向声明:
ATOM                MyRegisterClass(HINSTANCE hInstance);
BOOL                InitInstance(HINSTANCE, int);
LRESULT CALLBACK    WndProc(HWND, UINT, WPARAM, LPARAM);
INT_PTR CALLBACK    About(HWND, UINT, WPARAM, LPARAM);
INT_PTR CALLBACK next(HWND, UINT , WPARAM, LPARAM);
INT_PTR CALLBACK set(HWND hDlg, UINT message, WPARAM wParam, LPARAM lParam);
void random_num();
void InitBoard(HWND hwnd);
void moveboard(int id);
bool succeed();
void rebegin(HWND hWnd);


int APIENTRY wWinMain(_In_ HINSTANCE hInstance,
                     _In_opt_ HINSTANCE hPrevInstance,
                     _In_ LPWSTR    lpCmdLine,
                     _In_ int       nCmdShow)
{
    UNREFERENCED_PARAMETER(hPrevInstance);
    UNREFERENCED_PARAMETER(lpCmdLine);

    // TODO: 在此处放置代码。
    srand(static_cast<unsigned int>(time(nullptr)));

    // 初始化全局字符串
    LoadStringW(hInstance, IDS_APP_TITLE, szTitle, MAX_LOADSTRING);
    LoadStringW(hInstance, IDC_GAME2, szWindowClass, MAX_LOADSTRING);
    MyRegisterClass(hInstance);

    // 执行应用程序初始化:
    if (!InitInstance (hInstance, nCmdShow))
    {
        return FALSE;
    }

    HACCEL hAccelTable = LoadAccelerators(hInstance, MAKEINTRESOURCE(IDC_GAME2));

    MSG msg;

    // 主消息循环:
    while (GetMessage(&msg, nullptr, 0, 0))
    {
        if (!TranslateAccelerator(msg.hwnd, hAccelTable, &msg))
        {
            TranslateMessage(&msg);
            DispatchMessage(&msg);
        }
    }

    return (int) msg.wParam;
}



//
//  函数: MyRegisterClass()
//
//  目标: 注册窗口类。
//
ATOM MyRegisterClass(HINSTANCE hInstance)
{
    WNDCLASSEXW wcex;

    wcex.cbSize = sizeof(WNDCLASSEX);

    wcex.style          = CS_HREDRAW | CS_VREDRAW;
    wcex.lpfnWndProc    = WndProc;
    wcex.cbClsExtra     = 0;
    wcex.cbWndExtra     = 0;
    wcex.hInstance      = hInstance;
    wcex.hIcon          = LoadIcon(hInstance, MAKEINTRESOURCE(IDI_GAME2));
    wcex.hCursor        = LoadCursor(nullptr, IDC_ARROW);
    wcex.hbrBackground  = (HBRUSH)(COLOR_WINDOW+1);
    wcex.lpszMenuName   = MAKEINTRESOURCEW(IDC_GAME2);
    wcex.lpszClassName  = szWindowClass;
    wcex.hIconSm        = LoadIcon(wcex.hInstance, MAKEINTRESOURCE(IDI_SMALL));

    return RegisterClassExW(&wcex);
}

//
//   函数: InitInstance(HINSTANCE, int)
//
//   目标: 保存实例句柄并创建主窗口
//
//   注释:
//
//        在此函数中，我们在全局变量中保存实例句柄并
//        创建和显示主程序窗口。
//
BOOL InitInstance(HINSTANCE hInstance, int nCmdShow)
{
   hInst = hInstance; // 将实例句柄存储在全局变量中

   int windowWidth = margin * 2 + block_size * numsize;
   int windowHeight = margin * 2 + block_size * numsize + GetSystemMetrics(SM_CYMENU); // 考虑菜单高度

   HWND hWnd = CreateWindowW(szWindowClass, szTitle, WS_OVERLAPPEDWINDOW,
       CW_USEDEFAULT, 0, windowWidth, windowHeight, nullptr, nullptr, hInstance, nullptr);

   if(!hWnd)
   {
      return FALSE;
   }
   random_num();
   InitBoard(hWnd);

   ShowWindow(hWnd, nCmdShow);
   UpdateWindow(hWnd);

   return TRUE;
}

//
//  函数: WndProc(HWND, UINT, WPARAM, LPARAM)
//
//  目标: 处理主窗口的消息。
//
//  WM_COMMAND  - 处理应用程序菜单
//  WM_PAINT    - 绘制主窗口
//  WM_DESTROY  - 发送退出消息并返回
//
//
LRESULT CALLBACK WndProc(HWND hWnd, UINT message, WPARAM wParam, LPARAM lParam)
{
    switch (message)
    {
    case WM_COMMAND:
        {
            int wmId = LOWORD(wParam);
            // 分析菜单选择:
            switch (wmId)
            {
            case IDM_ABOUT:
                DialogBox(hInst, MAKEINTRESOURCE(IDD_ABOUTBOX), hWnd, About);
                break;
            case IDM_EXIT:
                DestroyWindow(hWnd);
                break;
            case restart:
                random_num();
                rebegin(hWnd);
                break;
            case reset:
                DialogBox(hInst, MAKEINTRESOURCE(IDD_DIALOG1), hWnd, set);
                break;
            default:
                moveboard(wmId);
                if(succeed()){ 
                    EnableWindow(hWnd, FALSE);
                    DialogBox(hInst, MAKEINTRESOURCE(IDD_succeed), hWnd, next);
                    EnableWindow(hWnd, TRUE);
                }
                return DefWindowProc(hWnd, message, wParam, lParam);
            }
        }
        break;
    case WM_PAINT:
        {
            PAINTSTRUCT ps;
            HDC hdc = BeginPaint(hWnd, &ps);
            // TODO: 在此处添加使用 hdc 的任何绘图代码...
            EndPaint(hWnd, &ps);
        }
        break;
    case WM_DESTROY:
        if (hFont) {
            DeleteObject(hFont);  // 关键释放
            hFont = NULL;         // 避免重复释放
        }
        PostQuitMessage(0);
        break;
    default:
        return DefWindowProc(hWnd, message, wParam, lParam);
    }
    return 0;
}

// “关于”框的消息处理程序。
INT_PTR CALLBACK set(HWND hDlg, UINT message, WPARAM wParam, LPARAM lParam) {
    switch (message) {
    case WM_INITDIALOG:
        SetDlgItemText(hDlg, IDC_EDIT1, L"1 2 3 4 5 6 7 8 9 10 11 12 13 14 0 15");
        return (INT_PTR)TRUE;

    case WM_COMMAND:
        if (LOWORD(wParam) == IDOK) {
            wchar_t inputText[256];
            GetDlgItemText(hDlg, IDC_EDIT1, inputText, 256);

            // 按空格分割字符串（使用符合标准的wcstok_s）
            std::vector<int> numbers;
            wchar_t* context = nullptr;
            wchar_t* token = wcstok_s(inputText, L" ", &context);  // 安全的版本

            while (token != nullptr) {
                numbers.push_back(_wtoi(token));
                token = wcstok_s(nullptr, L" ", &context);
            }

            // 验证是否为16个数字
            if (numbers.size() != numsize * numsize) {
                MessageBox(hDlg,
                    L"请输入16个用空格分隔的数字！\n例如：1 2 3 4 5 6 7 8 9 10 11 12 13 14 15 0",
                    L"输入错误",
                    MB_ICONERROR);
                return (INT_PTR)TRUE;
            }

            // 应用到游戏板
            for (int r = 0; r < numsize; r++) {
                for (int c = 0; c < numsize; c++) {
                    int index = r * numsize + c;
                    num[r][c] = numbers[index];
                    if (num[r][c] == 0) {
                        zero_r = r;
                        zero_c = c;
                    }
                }
            }

            // 刷新界面
            HWND hParent = GetParent(hDlg);
            rebegin(hParent);  // 使用rebegin而不是InitBoard以正确清理旧按钮

            EndDialog(hDlg, LOWORD(wParam));
            return (INT_PTR)TRUE;
        }
        else if (LOWORD(wParam) == IDCANCEL) {
            EndDialog(hDlg, LOWORD(wParam));
            return (INT_PTR)TRUE;
        }
        break;
    }
    return (INT_PTR)FALSE;
}

INT_PTR CALLBACK About(HWND hDlg, UINT message, WPARAM wParam, LPARAM lParam)
{
    UNREFERENCED_PARAMETER(lParam);
    switch (message)
    {
    case WM_INITDIALOG:
        return (INT_PTR)TRUE;

    case WM_COMMAND:
        if (LOWORD(wParam) == IDOK || LOWORD(wParam) == IDCANCEL)
        {
            EndDialog(hDlg, LOWORD(wParam));
            return (INT_PTR)TRUE;
        }
        break;
    }
    return (INT_PTR)FALSE;
}
INT_PTR CALLBACK next(HWND hDlg, UINT message, WPARAM wParam, LPARAM lParam) {
    UNREFERENCED_PARAMETER(lParam);
    switch (message)
    {
    case WM_INITDIALOG:
        return (INT_PTR)TRUE;

    case WM_COMMAND:
        if(LOWORD(wParam) == IDCANCEL){
            EndDialog(hDlg, LOWORD(wParam));
            return (INT_PTR)TRUE;
        }
        else if (LOWORD(wParam) == IDOK) {
            EndDialog(hDlg, LOWORD(wParam));
            random_num();
            rebegin(GetParent(hDlg));
            return (INT_PTR)TRUE;
        }
        break;
    }
    return (INT_PTR)FALSE;
}


void random_num() {
    for (int r = 0; r < numsize; r++) {
        for (int c = 0; c < numsize; c++) {
            num[r][c] = r * numsize + c;
        }
    }
    for (int i = 0; i < 10; i++) {
        int a = rand() % 4;
        int b = rand() % 4;
        int c = rand() % 4;
        int d = rand() % 4;
        int one = num[a][ b];
        num[a][b] = num[c][d];
        num[c][d] = one;
    }
}
void InitBoard(HWND hwnd) {
    RECT clientRect;
    GetClientRect(hwnd, &clientRect); // hWndParent 是父窗口句柄
    int clientWidth = clientRect.right - clientRect.left;  // 客户区宽度
    int clientHeight = clientRect.bottom - clientRect.top; // 客户区高度
    margintop = clientHeight / 5;
    int marginleft = clientHeight / 5;

    for (int r = 0; r < numsize; r++) {
        for (int c = 0; c < numsize; c++) {
            if (num[r][c] != 0) {
                int x = marginleft /2 + c * margintop;
                int y = margintop/2 + r * margintop;
                char str[10];
                snprintf(str, sizeof(str), "%d", num[r][c]);
                boardButtons[r][c] = CreateWindowA(
                    "BUTTON",  // 按钮类名
                    str,            // 按钮显示的文本
                    WS_VISIBLE | WS_CHILD | BS_PUSHBUTTON| WS_BORDER,  // 按钮样式
                    x, y, margintop, margintop,  // 按钮位置和大小
                    hwnd, (HMENU)(r * numsize + c),  // 父窗口句柄和按钮 ID
                    NULL, NULL
                );
                SendMessage(boardButtons[r][c], WM_SETFONT, (WPARAM)hFont, TRUE);
                if (boardButtons[r][c] == NULL) {
                    // 处理按钮创建失败的情况，例如输出错误信息
                    MessageBoxA(hwnd, "按钮创建失败", "错误", MB_OK);
                }
                else{
                    ShowWindow(boardButtons[r][c], SW_SHOW);
                }
            }
            else {
                zero_r = r;
                zero_c = c;
                int x = marginleft / 2 + c * margintop;
                int y = margintop / 2 + r * margintop;
                char str[10];
                snprintf(str, sizeof(str), "%d", num[r][c]);
                boardButtons[r][c] = CreateWindowA(
                    "BUTTON",  // 按钮类名
                    str,            // 按钮显示的文本
                    WS_VISIBLE | WS_CHILD | BS_PUSHBUTTON | WS_BORDER ,  // 按钮样式
                    x, y, margintop, margintop,  // 按钮位置和大小
                    hwnd, (HMENU)(r * numsize + c),  // 父窗口句柄和按钮 ID
                    NULL, NULL
                );

                SendMessage(boardButtons[r][c], WM_SETFONT, (WPARAM)hFont, TRUE);
                ShowWindow(boardButtons[r][c], SW_HIDE);
            }
        }
    }
}
void moveboard(int id) {
    int r = id / numsize;
    int c = id % numsize;
    if (((r - zero_r == 1 || r - zero_r == -1)&& c == zero_c) || ((c - zero_c == 1 || c - zero_c == -1)&& r == zero_r) ){
        wchar_t str[10];
        swprintf_s(str, _countof(str), L"%d", num[r][c]);
        SetWindowText(boardButtons[zero_r][zero_c], str);
        num[zero_r][zero_c] = num[r][c];
        ShowWindow(boardButtons[zero_r][zero_c], SW_SHOW);
        swprintf_s(str, _countof(str), L"%d", 0);
        SetWindowText(boardButtons[r][c], str);
        ShowWindow(boardButtons[r][c], SW_HIDE);
        num[r][c] = 0;
        zero_c = c;
        zero_r = r;
    }
}
bool succeed() {
    for (int i = 0; i < numsize * numsize - 2; i++) {
        std::pair<int, int>  one, two;
        one.first = i / numsize;
        one.second = i % numsize;
        two.first = (i+1) / numsize;
        two.second = (i+1) % numsize;
        if (num[two.first][two.second] < num[one.first][one.second]) { return false; }
    }
    return true;
}

void rebegin(HWND hWnd) {
    for (int r = 0; r < numsize; r++) {
        for (int c = 0; c < numsize; c++) {
            if (boardButtons[r][c] != NULL) {
                DestroyWindow(boardButtons[r][c]);
                boardButtons[r][c] = NULL;
            }
        }
    }

    // 重新初始化游戏板
    InitBoard(hWnd);
}
```


### 开发中遇到的问题
```c++
void moveboard(int id) {
    int r = id / numsize;
    int c = id % numsize;
    if (((r - zero_r == 1 || r - zero_r == -1)&& c == zero_c) || ((c - zero_c == 1 || c - zero_c == -1)&& r == zero_r) ){
        wchar_t str[10];
        swprintf_s(str, _countof(str), L"%d", num[r][c]);
        SetWindowText(boardButtons[zero_r][zero_c], str);
        num[zero_r][zero_c] = num[r][c];
        ShowWindow(boardButtons[zero_r][zero_c], SW_SHOW);
        swprintf_s(str, _countof(str), L"%d", 0);
        SetWindowText(boardButtons[r][c], str);
        ShowWindow(boardButtons[r][c], SW_HIDE);
        num[r][c] = 0;
        zero_c = c;
        zero_r = r;
    }
}
```
这是块的移动代码，实际上我做的是块的序号的交换。第一次制作时我采用了15个块，使用moveWindow()函数实现移动，值得一提的是该函数的位置参数是相对于父窗口的位置，此处注意坐标的转换，如此做存在一个问题：当我的块移动时，我块的id并没有发生变化，如此我vector数组难以对块进行定位，容易失去对应关系，所以之后改为了块不做移动，交换块的内容，实现移动的形式。

```c++
INT_PTR CALLBACK set(HWND hDlg, UINT message, WPARAM wParam, LPARAM lParam) {
    switch (message) {
    case WM_INITDIALOG:
        SetDlgItemText(hDlg, IDC_EDIT1, L"1 2 3 4 5 6 7 8 9 10 11 12 13 14 0 15");
        return (INT_PTR)TRUE;

    case WM_COMMAND:
        if (LOWORD(wParam) == IDOK) {
            wchar_t inputText[256];
            GetDlgItemText(hDlg, IDC_EDIT1, inputText, 256);

            // 按空格分割字符串（使用符合标准的wcstok_s）
            std::vector<int> numbers;
            wchar_t* context = nullptr;
            wchar_t* token = wcstok_s(inputText, L" ", &context);  // 安全的版本

            while (token != nullptr) {
                numbers.push_back(_wtoi(token));
                token = wcstok_s(nullptr, L" ", &context);
            }

            // 验证是否为16个数字
            if (numbers.size() != numsize * numsize) {
                MessageBox(hDlg,
                    L"请输入16个用空格分隔的数字！\n例如：1 2 3 4 5 6 7 8 9 10 11 12 13 14 15 0",
                    L"输入错误",
                    MB_ICONERROR);
                return (INT_PTR)TRUE;
            }

            // 应用到游戏板
            for (int r = 0; r < numsize; r++) {
                for (int c = 0; c < numsize; c++) {
                    int index = r * numsize + c;
                    num[r][c] = numbers[index];
                    if (num[r][c] == 0) {
                        zero_r = r;
                        zero_c = c;
                    }
                }
            }

            // 刷新界面
            HWND hParent = GetParent(hDlg);
            rebegin(hParent);  // 使用rebegin而不是InitBoard以正确清理旧按钮

            EndDialog(hDlg, LOWORD(wParam));
            return (INT_PTR)TRUE;
        }
        else if (LOWORD(wParam) == IDCANCEL) {
            EndDialog(hDlg, LOWORD(wParam));
            return (INT_PTR)TRUE;
        }
        break;
    }
    return (INT_PTR)FALSE;
}
```
