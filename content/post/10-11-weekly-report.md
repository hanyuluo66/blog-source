---
title: "10.11学习周报"
date: 2026-10-11
draft: false
slug: "10.11-weekly-report"
tags:
    - C语言
    - C++
    - 链表
    - 学习周报
categories:
    - 学习
summary: "本周完成C语言收尾（共用体、枚举、动态内存、文件操作、链表），并正式进入C++（命名空间、string、vector、引用与函数重载）。"
---

# 10.11 学习周报

> 本周是 **C 语言收尾 → 进入 C++** 的过渡周。前四天把 C 语言剩下的硬骨头啃完（共用体/枚举、动态内存、文件操作、链表），后三天正式跨进 C++，学完了命名空间、`string`、`vector`、引用和函数重载。
>
> 其中 **链表** 和 **引用** 是本周两个最大的卡点，代码都手写跑通了。

---

## 一、共用体、枚举与 typedef

### 1.1 共用体（union）的内存特性

- 所有成员**共用同一块内存**，起始地址相同
- `sizeof(union)` 等于**最大成员**的大小（还要考虑内存对齐）
- 同一时刻**只能有一个成员有效**，写入新成员会覆盖旧成员

```c
#include <stdio.h>

union Data {
    int   i;    // 4 字节
    char  c;    // 1 字节
    double d;   // 8 字节
};

int main() {
    union Data u;

    printf("sizeof(union Data) = %lu\n", sizeof(u));  // 输出 8

    u.i = 65;
    printf("u.i = %d, u.c = %c\n", u.i, u.c);  // 65, 'A'（65 的 ASCII 是 A）

    u.d = 3.14;
    // 此时 u.i 已被覆盖，值不再是 65
    printf("写入 u.d 后，u.i = %d（已被破坏）\n", u.i);

    return 0;
}
```

### 1.2 实用案例：判断机器的大小端

利用共用体成员共享地址的特性，可以判断当前机器的字节序：

```c
#include <stdio.h>

union Endian {
    int  i;
    char c[sizeof(int)];
};

int main() {
    union Endian e;
    e.i = 1;

    // 如果最低地址字节是 1，说明低位存低地址 → 小端
    if (e.c[0] == 1) {
        printf("本机为 小端存储 (Little-Endian)\n");
    } else {
        printf("本机为 大端存储 (Big-Endian)\n");
    }

    return 0;
}
```

**内存示意**（小端机器，`int` 占 4 字节）：

```
地址：  0x100  0x101  0x102  0x103
内容：   01     00     00     00      ← 低字节放在低地址
```

### 1.3 枚举（enum）

- 默认从 `0` 开始递增，可以手动指定某个值，**后面的值跟着顺延**
- 本质是**整型常量**，让代码可读性远高于写死的数字

```c
#include <stdio.h>

// 指定 MON = 1，后面自动 2、3、4...
enum Weekday { MON = 1, TUE, WED, THU, FRI, SAT, SUN };

// 跳号也可以
enum Level { LOW = 10, MID = 20, HIGH = 100 };

int main() {
    enum Weekday today = SAT;
    printf("SAT 的值 = %d\n", today);   // 6
    printf("HIGH = %d\n", HIGH);        // 100

    switch (today) {
        case MON: case TUE: case WED: case THU: case FRI:
            printf("工作日\n");
            break;
        case SAT: case SUN:
            printf("周末\n");
            break;
    }

    return 0;
}
```

### 1.4 typedef 的四种常见用法

```c
#include <stdio.h>

// 1. 给结构体起别名（最常用）
typedef struct {
    int x;
    int y;
} Point;

// 2. 给数组类型起别名
typedef int Arr10[10];

// 3. 给指针类型起别名
typedef int *IntPtr;

// 4. 给已有类型换个名字
typedef unsigned int uint;

int main() {
    Point p = {3, 4};           // 不用再写 struct Point
    Arr10 a = {0};              // 等价于 int a[10] = {0};
    int value = 42;
    IntPtr ip = &value;         // 等价于 int *ip = &value;
    uint count = 100;

    printf("p = (%d, %d), *ip = %d, count = %u\n", p.x, p.y, *ip, count);
    return 0;
}
```

### 1.5 关键要点

- `union` 和 `struct` 的区别：`struct` 各成员**独立占内存**（大小是总和 + 对齐），`union` 成员**共享内存**（大小取最大）
- 共用体最大的用途是**节省内存**和**类型双关**（同一块数据按不同类型解读）
- `enum` 变量的值可以超出枚举列表范围，因为它本质就是 `int`
- `typedef` 不创建新类型，只是**起别名**

---

## 二、动态内存分配与内存四区模型

### 2.1 内存四区（本周最重要的概念）

| 分区 | 存放内容 | 生命周期 | 由谁管理 |
|------|---------|---------|---------|
| **栈区 stack** | 局部变量、函数参数、返回地址 | 函数结束自动释放 | 编译器 |
| **堆区 heap** | `malloc` / `new` 申请的内存 | 手动释放前一直存在 | **程序员** |
| **全局/静态区** | 全局变量、`static` 变量 | 程序运行期间 | 系统 |
| **常量区** | 字符串字面量、`const` 全局常量 | 程序运行期间 | 系统 |
| **代码区** | 编译后的机器指令 | 程序运行期间 | 系统 |

```c
#include <stdio.h>
#include <stdlib.h>

int g_value = 10;              // 全局区

void demo() {
    int local = 5;             // 栈区
    static int s_count = 0;    // 静态区
    char *str = "hello";       // "hello" 在常量区，str 指针本身在栈区
    int *heap = malloc(sizeof(int));  // 指向的内存在堆区

    *heap = 100;
    printf("%d %d %s %d\n", local, g_value, str, *heap);

    free(heap);                // 必须手动释放堆内存
    heap = NULL;               // 释放后置空，防止野指针
}
```

### 2.2 四个内存函数的区别

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

int main() {
    // malloc：只申请，不初始化（内容是随机值）
    int *p1 = (int *)malloc(5 * sizeof(int));

    // calloc：申请并全部初始化为 0，参数是 (个数, 单个大小)
    int *p2 = (int *)calloc(5, sizeof(int));

    printf("malloc 后 p1[0] = %d（随机值）\n", p1[0]);
    printf("calloc 后 p2[0] = %d（一定是0）\n", p2[0]);

    // realloc：扩容/缩容，可能原地扩展，也可能搬家返回新地址
    int *p3 = (int *)realloc(p1, 10 * sizeof(int));
    if (p3 != NULL) {
        p1 = p3;   // 必须用返回值更新指针
    }

    // free：释放
    free(p1);
    free(p2);
    p1 = NULL;
    p2 = NULL;

    return 0;
}
```

**`realloc` 的关键陷阱：**

```c
// 错误写法：如果 realloc 失败返回 NULL，原来的 p 就丢了 → 内存泄漏
p = realloc(p, newSize);

// 正确写法：先用临时指针接收
int *tmp = realloc(p, newSize);
if (tmp != NULL) {
    p = tmp;
}
```

### 2.3 实战案例：动态学生数组

```c
#include <stdio.h>
#include <stdlib.h>

typedef struct {
    char name[20];
    int  score;
} Student;

// 按成绩降序排序
void sortByScore(Student *arr, int n) {
    for (int i = 0; i < n - 1; i++) {
        for (int j = 0; j < n - 1 - i; j++) {
            if (arr[j].score < arr[j + 1].score) {
                Student tmp = arr[j];
                arr[j] = arr[j + 1];
                arr[j + 1] = tmp;
            }
        }
    }
}

int main() {
    int n;
    printf("请输入学生人数：");
    scanf("%d", &n);

    // 人数由用户决定，编译期无法确定 → 必须用动态内存
    Student *students = (Student *)malloc(n * sizeof(Student));
    if (students == NULL) {
        printf("内存分配失败\n");
        return 1;
    }

    for (int i = 0; i < n; i++) {
        printf("第 %d 个学生（姓名 成绩）：", i + 1);
        scanf("%19s %d", students[i].name, &students[i].score);
    }

    sortByScore(students, n);

    printf("\n=== 成绩排名 ===\n");
    for (int i = 0; i < n; i++) {
        printf("第%d名：%s  %d分\n", i + 1, students[i].name, students[i].score);
    }

    free(students);      // 用完必须释放
    students = NULL;

    return 0;
}
```

### 2.4 三类典型内存错误

```c
// 错误1：内存泄漏 —— 申请了不释放
void leak() {
    int *p = malloc(100 * sizeof(int));
    // 函数结束，p 消失，但那 400 字节再也找不回来了
}

// 错误2：野指针 —— 释放后继续使用
void dangling() {
    int *p = malloc(sizeof(int));
    *p = 10;
    free(p);
    printf("%d\n", *p);   // 未定义行为！p 指向的内存已经不属于你了
}

// 错误3：重复释放
void doubleFree() {
    int *p = malloc(sizeof(int));
    free(p);
    free(p);              // 崩溃
}
```

**防范手段**：`free` 之后立刻 `p = NULL;`，这样后续误用会立刻崩溃暴露问题，而不是悄悄地破坏数据。

### 2.5 关键要点

- 栈空间很小（Windows 默认 1MB），**大数组和不定长数据必须放堆上**
- `malloc` 返回值必须**判空**，分配失败会返回 `NULL`
- `malloc` 返回 `void *`，C 语言中可以隐式转换，但**加上强制类型转换是习惯**（C++ 中必须加）
- `free` 和 `malloc` 必须**一一配对**

---

## 三、文件操作

### 3.1 打开与关闭

```c
#include <stdio.h>

int main() {
    // "r" 只读（文件必须存在）
    // "w" 只写（不存在则创建，存在则清空！）
    // "a" 追加（不存在则创建，存在则在末尾续写）
    // "r+" 读写    "w+" 读写清空    "a+" 读和追加
    // 加 "b" 表示二进制模式，如 "rb"、"wb"

    FILE *fp = fopen("test.txt", "w");
    if (fp == NULL) {
        printf("文件打开失败\n");
        return 1;
    }

    // ... 读写操作 ...

    fclose(fp);   // 必须关闭，否则缓冲区数据可能没写入磁盘
    return 0;
}
```

### 3.2 三种文本读写方式

```c
#include <stdio.h>

int main() {
    // === 写入 ===
    FILE *fp = fopen("data.txt", "w");

    fputc('A', fp);                    // 字符读写
    fputs("Hello World\n", fp);        // 字符串读写
    fprintf(fp, "%s %d\n", "张三", 90); // 格式化读写

    fclose(fp);

    // === 读取 ===
    fp = fopen("data.txt", "r");

    int ch;
    while ((ch = fgetc(fp)) != EOF) {  // 逐字符读，EOF 表示读到文件尾
        putchar(ch);
    }

    fclose(fp);
    return 0;
}
```

**按行读取（更常用）：**

```c
#include <stdio.h>

int main() {
    FILE *fp = fopen("data.txt", "r");
    if (fp == NULL) return 1;

    char line[256];
    int lineNo = 1;
    // fgets 会保留行尾的 '\n'，读到文件尾或出错返回 NULL
    while (fgets(line, sizeof(line), fp) != NULL) {
        printf("第%d行：%s", lineNo++, line);
    }

    fclose(fp);
    return 0;
}
```

### 3.3 二进制块读写（效率更高）

```c
#include <stdio.h>
#include <stdlib.h>

typedef struct {
    char name[20];
    int  age;
    int  score;
} Student;

int main() {
    Student list[3] = {
        {"Alice", 20, 92},
        {"Bob",   21, 78},
        {"Carol", 19, 88}
    };

    // 写入：一次性把整个数组写进文件
    FILE *fp = fopen("students.dat", "wb");
    fwrite(list, sizeof(Student), 3, fp);
    //        ↑数据首地址  ↑单块大小  ↑块数  ↑文件指针
    fclose(fp);

    // 读取：直接还原到内存结构体
    Student *buf = (Student *)malloc(3 * sizeof(Student));
    fp = fopen("students.dat", "rb");
    int count = fread(buf, sizeof(Student), 3, fp);  // 返回实际读到的块数
    fclose(fp);

    printf("成功读取 %d 条记录：\n", count);
    for (int i = 0; i < count; i++) {
        printf("  %s, %d岁, %d分\n", buf[i].name, buf[i].age, buf[i].score);
    }

    free(buf);
    return 0;
}
```

### 3.4 文件指针定位

```c
#include <stdio.h>

int main() {
    FILE *fp = fopen("data.txt", "r");

    // fseek(文件指针, 偏移量, 起始位置)
    // 起始位置：SEEK_SET(文件头) / SEEK_CUR(当前) / SEEK_END(文件尾)

    fseek(fp, 0, SEEK_END);       // 跳到文件末尾
    long size = ftell(fp);        // 获取当前位置 = 文件总字节数
    printf("文件大小：%ld 字节\n", size);

    rewind(fp);                   // 回到文件开头

    fseek(fp, 10, SEEK_SET);      // 跳到第 11 个字节
    char ch = fgetc(fp);
    printf("第11个字节是：%c\n", ch);

    fclose(fp);
    return 0;
}
```

### 3.5 实战案例：学生通讯录（数据持久化）

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

#define MAX_STU 100
#define DATA_FILE "contacts.txt"

typedef struct {
    char name[20];
    char phone[15];
    int  age;
} Contact;

// 保存到文件
int saveToFile(Contact *list, int n) {
    FILE *fp = fopen(DATA_FILE, "w");
    if (fp == NULL) return 0;
    for (int i = 0; i < n; i++) {
        fprintf(fp, "%s %s %d\n", list[i].name, list[i].phone, list[i].age);
    }
    fclose(fp);
    return 1;
}

// 从文件加载
int loadFromFile(Contact *list) {
    FILE *fp = fopen(DATA_FILE, "r");
    if (fp == NULL) return 0;   // 文件不存在 = 第一次运行

    int n = 0;
    while (n < MAX_STU &&
           fscanf(fp, "%19s %14s %d", list[n].name, list[n].phone, &list[n].age) == 3) {
        n++;
    }
    fclose(fp);
    return n;
}

int main() {
    Contact list[MAX_STU];
    int n = loadFromFile(list);
    printf("已加载 %d 条历史记录\n", n);

    // 新增一条
    if (n < MAX_STU) {
        strcpy(list[n].name, "David");
        strcpy(list[n].phone, "13800138000");
        list[n].age = 22;
        n++;
    }

    saveToFile(list, n);
    printf("保存成功，当前共 %d 条\n", n);

    // 打印全部
    for (int i = 0; i < n; i++) {
        printf("%-8s %-13s %d岁\n", list[i].name, list[i].phone, list[i].age);
    }

    return 0;
}
```

### 3.6 关键要点

- `"w"` 模式会**清空原有内容**，想追加必须用 `"a"`，这是最容易踩的坑
- 文本模式会做换行符转换（Windows 下 `\n` ↔ `\r\n`），**二进制模式不做任何转换**
- 结构体数据用 `fread` / `fwrite` 读写，比 `fprintf` / `fscanf` 快得多，但**换行、空格会丢失**，只适合程序自己读写
- `scanf` 的 `%19s` 限制读取长度，防止数组越界（缓冲区溢出）

---

## 四、链表（本周重点）

### 4.1 为什么需要链表

数组的两个硬伤：

- 长度**编译期就固定**，运行时改不了（除非用动态内存，但扩容要整体搬家）
- 插入/删除要**移动大量元素**，效率 O(n)

链表用**离散的内存块 + 指针串联**解决了这两个问题：

```
数组（内存连续）：  [10][20][30][40]     插入要挪位置
链表（内存离散）：  [10|●]→[20|●]→[30|∅]  插入只改指针
```

### 4.2 节点定义

```c
typedef struct Node {
    int data;              // 数据域
    struct Node *next;     // 指针域，指向下一个节点
} Node;
```

> **注意**：结构体内部必须写 `struct Node *next`，不能写 `Node *next`，因为此时 `Node` 这个别名还没定义完。

### 4.3 完整实现：增删改查

```c
#include <stdio.h>
#include <stdlib.h>

typedef struct Node {
    int data;
    struct Node *next;
} Node;

// ---------- 创建节点 ----------
Node *createNode(int data) {
    Node *p = (Node *)malloc(sizeof(Node));
    if (p == NULL) {
        printf("内存分配失败\n");
        exit(1);
    }
    p->data = data;
    p->next = NULL;
    return p;
}

// ---------- 头部插入 O(1) ----------
void pushFront(Node **head, int data) {
    Node *p = createNode(data);
    p->next = *head;    // 新节点指向原来的头
    *head = p;          // 头指针指向新节点
}

// ---------- 尾部插入 O(n) ----------
void pushBack(Node **head, int data) {
    Node *p = createNode(data);
    if (*head == NULL) {     // 空链表，直接作为头
        *head = p;
        return;
    }
    Node *cur = *head;
    while (cur->next != NULL) {   // 走到最后一个节点
        cur = cur->next;
    }
    cur->next = p;
}

// ---------- 遍历打印 ----------
void printList(Node *head) {
    Node *cur = head;
    while (cur != NULL) {
        printf("%d -> ", cur->data);
        cur = cur->next;
    }
    printf("NULL\n");
}

// ---------- 查找 ----------
Node *find(Node *head, int data) {
    Node *cur = head;
    while (cur != NULL) {
        if (cur->data == data) return cur;
        cur = cur->next;
    }
    return NULL;   // 没找到
}

// ---------- 删除第一个匹配的节点 ----------
int deleteNode(Node **head, int data) {
    Node *cur = *head, *prev = NULL;

    while (cur != NULL) {
        if (cur->data == data) {
            if (prev == NULL) {
                *head = cur->next;    // 删的是头节点
            } else {
                prev->next = cur->next;  // 前一个节点跳过它
            }
            free(cur);
            return 1;
        }
        prev = cur;
        cur = cur->next;
    }
    return 0;
}

// ---------- 求长度 ----------
int length(Node *head) {
    int len = 0;
    Node *cur = head;
    while (cur != NULL) {
        len++;
        cur = cur->next;
    }
    return len;
}

// ---------- 释放整个链表（防内存泄漏）----------
void freeList(Node **head) {
    Node *cur = *head;
    while (cur != NULL) {
        Node *next = cur->next;   // 必须先存下一个，否则 free 后就找不到了
        free(cur);
        cur = next;
    }
    *head = NULL;
}

// ---------- 测试 ----------
int main() {
    Node *head = NULL;

    pushBack(&head, 10);
    pushBack(&head, 20);
    pushBack(&head, 30);
    pushFront(&head, 5);
    printf("插入后：");
    printList(head);        // 5 -> 10 -> 20 -> 30 -> NULL

    printf("链表长度：%d\n", length(head));

    Node *p = find(head, 20);
    if (p != NULL) {
        p->data = 99;       // 通过指针直接修改节点
        printf("把 20 改成 99：");
        printList(head);
    }

    deleteNode(&head, 10);
    printf("删除 10 后：");
    printList(head);

    freeList(&head);
    printf("释放后 head = %p\n", (void *)head);

    return 0;
}
```

### 4.4 为什么函数参数要用 `Node **head`

```c
// 错误：只传了头指针的副本，函数内修改 head 影响不到外面
void pushFront(Node *head, int data) { ... }

// 正确：传头指针的地址，才能真正修改调用者的 head
void pushFront(Node **head, int data) { ... }
```

**判断标准**：函数内是否需要**改变头指针本身的指向**？
- 需要（插入头节点、删除头节点）→ 用 `Node **`
- 不需要（遍历、查找、打印）→ 用 `Node *` 就够了

### 4.5 链表 vs 数组

| 对比项 | 数组 | 链表 |
|--------|------|------|
| 内存布局 | 连续 | 离散 |
| 随机访问 | O(1)，`arr[i]` 直达 | O(n)，必须从头遍历 |
| 头部插入/删除 | O(n)，要挪元素 | **O(1)**，只改指针 |
| 长度 | 固定（或 realloc 搬家） | 动态，随时增减 |
| 额外开销 | 无 | 每个节点多存一个指针 |

### 4.6 关键要点

- **画图！画图！画图！** 链表的所有操作，先在纸上把指针的指向画清楚，再写代码
- 修改指针的**顺序不能错**：`p->next = *head;` 必须在 `*head = p;` 之前，否则原链表就断了
- `freeList` 里必须**先保存 `next` 再 `free`**，否则 `cur->next` 已经是野内存
- 删除节点要考虑**三种情况**：删头节点、删中间节点、节点不存在

---

## 五、C++ 入门：命名空间、IO、string、vector

### 5.1 命名空间（namespace）

解决的问题：不同库里有同名函数/变量时的**命名冲突**。

```cpp
#include <iostream>

// 定义两个命名空间，里面可以有同名的东西
namespace Chinese {
    int value = 100;
    void greet() { std::cout << "你好" << std::endl; }
}

namespace English {
    int value = 200;
    void greet() { std::cout << "Hello" << std::endl; }
}

int main() {
    // 方式1：作用域运算符 ::（最明确，推荐）
    Chinese::greet();
    std::cout << English::value << std::endl;

    // 方式2：using 声明单个名字
    using Chinese::value;
    std::cout << value << std::endl;   // 100

    // 方式3：using namespace（图省事，但容易冲突，头文件里禁止用）
    using namespace English;
    greet();

    return 0;
}
```

### 5.2 cin / cout 对比 scanf / printf

```cpp
#include <iostream>
#include <string>
using namespace std;

int main() {
    int age;
    string name;

    // C 写法：scanf("%d %s", &age, buf);  要记格式符、要取地址、要防溢出
    // C++ 写法：自动识别类型，不用格式符
    cout << "请输入年龄：";
    cin >> age;

    cout << "请输入姓名：";
    cin >> name;          // string 自动扩容，不用担心缓冲区溢出

    cout << "姓名：" << name << "，年龄：" << age << endl;

    // 连续输出用 << 串起来
    cout << "年龄的平方是 " << age * age << "，明年 " << age + 1 << " 岁" << endl;

    return 0;
}
```

**`cin >>` 和 `getline` 的区别（重要）：**

```cpp
// cin >> 遇到空格、Tab、换行就停止 → 读不了带空格的整行
// getline 读一整行，包括空格

string line;
cout << "请输入一行（可含空格）：";
getline(cin, line);
cout << "你输入了：" << line << endl;
```

### 5.3 string 常用操作

```cpp
#include <iostream>
#include <string>
using namespace std;

int main() {
    string s1 = "Hello";
    string s2 = "World";

    // 拼接
    string s3 = s1 + " " + s2;
    cout << s3 << endl;                    // Hello World

    // 长度
    cout << s3.size() << " " << s3.length() << endl;   // 都是 11

    // 访问字符（两种都行）
    cout << s3[0] << " " << s3.at(0) << endl;          // H H

    // 比较（直接用 == != < >，按字典序）
    cout << (s1 == "Hello") << endl;       // 1 (true)

    // 查找，找不到返回 string::npos
    size_t pos = s3.find("World");
    if (pos != string::npos) {
        cout << "找到了，位置 " << pos << endl;   // 6
    }

    // 截取 substr(起始位置, 长度)
    cout << s3.substr(0, 5) << endl;       // Hello
    cout << s3.substr(6) << endl;          // World（省略长度 = 到末尾）

    // 替换、插入、删除
    s3.replace(0, 5, "Hi");
    cout << s3 << endl;                    // Hi World

    // 转 C 风格字符串（要传给 C 函数时用）
    const char *cstr = s3.c_str();
    printf("%s\n", cstr);

    return 0;
}
```

### 5.4 vector 容器

```cpp
#include <iostream>
#include <vector>
#include <string>
using namespace std;

int main() {
    // 创建
    vector<int> v1;                    // 空 vector
    vector<int> v2(5, 0);              // 5 个元素，都初始化为 0
    vector<int> v3 = {1, 2, 3};        // 列表初始化

    // 尾部添加（会自动扩容，这是 vector 最大的优势）
    v1.push_back(10);
    v1.push_back(20);
    v1.push_back(30);

    // 遍历方式1：下标
    for (int i = 0; i < v1.size(); i++) {
        cout << v1[i] << " ";
    }
    cout << endl;

    // 遍历方式2：迭代器
    for (vector<int>::iterator it = v1.begin(); it != v1.end(); it++) {
        cout << *it << " ";
    }
    cout << endl;

    // 遍历方式3：范围 for（C++11，最简洁）
    for (int x : v1) {
        cout << x << " ";
    }
    cout << endl;

    // 常用操作
    cout << "大小：" << v1.size() << endl;        // 3
    cout << "容量：" << v1.capacity() << endl;    // 实际分配的内存能装几个
    cout << "是否为空：" << v1.empty() << endl;    // 0 (false)
    cout << "首元素：" << v1.front() << endl;      // 10
    cout << "尾元素：" << v1.back() << endl;       // 30

    v1.pop_back();          // 删除最后一个
    v1.insert(v1.begin(), 5);  // 在开头插入 5
    v1.erase(v1.begin());      // 删除开头元素

    return 0;
}
```

### 5.5 实战：用 C++ 重写通讯录

```cpp
#include <iostream>
#include <string>
#include <vector>
using namespace std;

struct Student {
    string name;
    int score;
};

// 按成绩降序排序
void sortByScore(vector<Student> &v) {   // 引用传参，避免整体拷贝
    for (size_t i = 0; i < v.size(); i++) {
        for (size_t j = 0; j < v.size() - 1 - i; j++) {
            if (v[j].score < v[j + 1].score) {
                Student tmp = v[j];
                v[j] = v[j + 1];
                v[j + 1] = tmp;
            }
        }
    }
}

int main() {
    vector<Student> students;

    // C 版本要 malloc + 手动算长度，这里一行搞定
    students.push_back({"张三", 85});
    students.push_back({"李四", 92});
    students.push_back({"王五", 78});

    // 还能继续添加，自动扩容
    string name;
    int score;
    cout << "再输入一个学生（姓名 成绩）：";
    cin >> name >> score;
    students.push_back({name, score});

    sortByScore(students);

    cout << "\n=== 成绩排名 ===" << endl;
    for (size_t i = 0; i < students.size(); i++) {
        cout << "第" << i + 1 << "名："
             << students[i].name << "  "
             << students[i].score << "分" << endl;
    }

    return 0;
    // 不用 free！vector 离开作用域自动释放内存
}
```

### 5.6 C 与 C++ 写法对照

| 功能 | C 语言 | C++ |
|------|--------|-----|
| 字符串 | `char buf[50]` + `strcpy` | `string s` + `=` |
| 字符串拼接 | `strcat` | `s1 + s2` |
| 字符串比较 | `strcmp(a,b) == 0` | `a == b` |
| 动态数组 | `malloc` + 手动 `realloc` | `vector` + `push_back` |
| 输入输出 | `scanf` / `printf` | `cin` / `cout` |
| 申请内存 | `malloc` / `free` | `new` / `delete` |
| 内存释放 | **必须手动 free** | 容器自动管理 |

### 5.7 关键要点

- `vector` 内部本质就是**自动扩容的动态数组**，理解了 `malloc` / `realloc` 就理解了它
- `size()` 是**元素个数**，`capacity()` 是**已分配内存能装的个数**，两者通常不相等
- `vector<Student> &v` 用**引用传参**很关键，否则整个 vector 会被复制一份，性能差且改不到原数据
- 头文件里**不要写 `using namespace`**，会污染所有包含它的文件

---

## 六、C++ 引用与函数提高

### 6.1 引用是什么

引用就是**变量的别名**，和原变量指向同一块内存：

```cpp
#include <iostream>
using namespace std;

int main() {
    int a = 10;
    int &r = a;      // r 是 a 的别名，必须在定义时初始化

    r = 20;          // 改 r 就是改 a
    cout << "a = " << a << endl;   // 20

    a = 30;
    cout << "r = " << r << endl;   // 30

    cout << "&a = " << &a << ", &r = " << &r << endl;  // 地址完全相同

    return 0;
}
```

### 6.2 引用作函数参数（对比指针写法）

```cpp
#include <iostream>
using namespace std;

// 值传递：改不到外面的变量
void swapByValue(int a, int b) {
    int t = a; a = b; b = t;
}

// 指针传递（C 语言写法）：要取地址、要解引用
void swapByPointer(int *a, int *b) {
    int t = *a; *a = *b; *b = t;
}

// 引用传递（C++ 写法）：语法上就像直接操作原变量
void swapByReference(int &a, int &b) {
    int t = a; a = b; b = t;
}

int main() {
    int x = 5, y = 10;

    swapByValue(x, y);
    cout << "值传递后：" << x << ", " << y << endl;      // 5, 10（没变）

    swapByPointer(&x, &y);      // 要写 &
    cout << "指针交换后：" << x << ", " << y << endl;    // 10, 5

    swapByReference(x, y);      // 不用写 &，看着像值传递，其实是引用
    cout << "引用交换后：" << x << ", " << y << endl;    // 5, 10

    return 0;
}
```

### 6.3 引用作返回值

```cpp
#include <iostream>
using namespace std;

// 返回数组中最大元素的引用 → 可以当左值使用，直接修改原数组
int &getLargest(int arr[], int n) {
    int idx = 0;
    for (int i = 1; i < n; i++) {
        if (arr[i] > arr[idx]) idx = i;
    }
    return arr[idx];
}

// 危险示范：返回局部变量的引用
int &dangerous() {
    int local = 100;
    return local;    // local 在函数结束时已销毁，引用变成野引用！
}

int main() {
    int arr[] = {3, 9, 1, 7};

    cout << "最大值：" << getLargest(arr, 4) << endl;   // 9

    getLargest(arr, 4) = 100;    // 把最大值改成 100
    for (int i = 0; i < 4; i++) cout << arr[i] << " ";
    cout << endl;                // 3 100 1 7

    return 0;
}
```

### 6.4 引用 vs 指针（三个核心区别）

| 对比项 | 引用 `&` | 指针 `*` |
|--------|---------|---------|
| 是否可为空 | **不能**，必须绑定有效对象 | 可以是 `NULL` / `nullptr` |
| 是否必须初始化 | **必须**，定义时就要绑定 | 可以先定义后赋值 |
| 能否改指向 | **不能**，一旦绑定终身不变 | 可以随时指向别的对象 |
| 是否占内存 | 编译器层面通常是别名（底层仍用指针实现） | 明确占 4/8 字节 |
| 使用语法 | 直接用，无需解引用 | 要 `*` 解引用、`->` 访问成员 |

```cpp
int a = 1, b = 2;
int &r = a;
r = b;        // 这是【赋值】：把 b 的值给 a，r 仍然绑定 a！

int *p = &a;
p = &b;       // 这是【改指向】：p 现在指向 b
```

> **这个区别极其容易搞混**：`r = b` 不是让 r 变成 b 的别名，而是把 b 的值赋给 a。引用一旦绑定就**无法解绑**。

**选用原则**：
- 能用引用就用引用（语法干净、不会空指针）
- 需要"可为空"、"需要改指向"、"要做指针运算"时用指针
- 函数参数只是读取、且类型是内置类型（`int`、`double`）时，直接值传递即可

### 6.5 函数默认参数

```cpp
#include <iostream>
using namespace std;

// 默认值必须【从右往左】连续设置
void print(int a, int b = 10, int c = 20) {
    cout << a << ", " << b << ", " << c << endl;
}

int main() {
    print(1);          // 1, 10, 20
    print(1, 2);       // 1, 2, 20
    print(1, 2, 3);    // 1, 2, 3
    // print(1, , 3);  ← 不能跳过中间的参数
    return 0;
}
```

### 6.6 函数重载

**同一个函数名，参数列表不同**，编译器根据实参自动匹配：

```cpp
#include <iostream>
#include <string>
using namespace std;

// 参数【类型】不同
int add(int a, int b) {
    cout << "[int 版本] ";
    return a + b;
}

double add(double a, double b) {
    cout << "[double 版本] ";
    return a + b;
}

// 参数【个数】不同
int add(int a, int b, int c) {
    cout << "[三个 int] ";
    return a + b + c;
}

// 参数【顺序】不同
string add(string a, int b) {
    return a + to_string(b);
}

string add(int b, string a) {
    return to_string(b) + a;
}

int main() {
    cout << add(1, 2) << endl;           // int 版本 → 3
    cout << add(1.5, 2.5) << endl;       // double 版本 → 4
    cout << add(1, 2, 3) << endl;        // 三参数版本 → 6
    cout << add("num=", 42) << endl;     // num=42
    cout << add(42, "=num") << endl;     // 42=num
    return 0;
}
```

**重载的判定规则**：只看**参数列表**（类型、个数、顺序），**返回值类型不构成重载**。

```cpp
// 编译错误：只有返回值不同，不是重载
int    func(int a);
double func(int a);   // error: 重定义
```

### 6.7 重载与默认参数的冲突

```cpp
// 这两个声明会冲突！
int add(int a, int b = 10);
int add(int a);

int main() {
    add(5);   // error: 二义性 —— 编译器不知道该调用哪个
}
```

**原因**：`add(5)` 既能匹配第一个（b 用默认值 10），也能匹配第二个。实际项目中**不要同时使用重载和默认参数**，容易出这种歧义。

### 6.8 关键要点

- 引用的本质在编译器层面通常还是指针实现，但**语法上屏蔽了解引用**，更安全
- 引用**不能返回局部变量**，函数结束后栈内存已回收
- 默认参数**从右往左**设置；如果函数声明和定义分离，**默认值只能写在声明处**（`.h` 里）
- 函数重载靠**参数列表**区分，和返回值无关

---

## 七、常见坑点整理

这周踩到的和容易踩的坑，集中记一下：

| 坑 | 现象 | 解决办法 |
|----|------|---------|
| `free` 后继续用指针 | 程序随机崩溃或数据错乱 | `free(p)` 后立刻 `p = NULL` |
| `realloc` 直接覆盖原指针 | 扩容失败时原内存泄漏 | 用临时指针接收返回值并判空 |
| `fopen` 用 `"w"` 打开已有文件 | **文件内容被清空** | 想追加用 `"a"`，先确认再打开 |
| 忘记 `fclose` | 缓冲区数据没落盘，文件内容不完整 | 每个 `fopen` 配对一个 `fclose` |
| 链表 `freeList` 里先 free 再取 next | 访问已释放内存，崩溃 | 先 `next = cur->next` 再 `free(cur)` |
| 头插入时先改 head 再改 next | 原链表丢失，只剩一个节点 | 顺序必须是 `p->next = *head; *head = p;` |
| `r = b` 以为是让引用改绑 | 引用绑定的对象被赋值了 | 引用一旦绑定**无法解绑** |
| 返回局部变量的引用 | 未定义行为，可能拿到垃圾值 | 返回堆上的指针，或返回值拷贝 |
| 重载 + 默认参数同时用 | 编译报"二义性"错误 | 二选一，别混用 |
| `cin >>` 读带空格的输入 | 只读到第一个空格前 | 用 `getline(cin, s)` |

---

## 八、本周学习总结

| 知识点 | 掌握程度 | 备注 |
|--------|---------|------|
| 共用体 / 枚举 / typedef | 熟练 | 理解了共用体的内存共享机制 |
| 内存四区模型 | 熟练 | 能画出分区图，清楚各区生命周期 |
| malloc / free 家族 | 熟练 | 动态学生数组完整实现 |
| 文件操作 | 熟练 | 文本与二进制读写都跑通了 |
| 单链表 | 基本掌握 | 增删改查手写通过，复杂操作还需练 |
| C++ 命名空间 / IO | 了解 | 语法不难，主要是习惯转换 |
| string / vector | 基本掌握 | 体会到了比 C 字符串好用太多 |
| 引用与函数重载 | 基本掌握 | 引用 vs 指针的区别还需巩固 |

### 本周最大的收获

1. **内存四区模型打通了任督二脉**。之前学指针总有种"知道语法但不知道为什么"的感觉，理解了栈区和堆区的区别之后，`malloc`、链表、引用返回局部变量的坑，全都串起来了。

2. **链表是指针和结构体的终极考验**。手写增删改查的过程里，暴露了很多对指针理解的模糊点，尤其是 `Node **head` 为什么要用二级指针。画内存图是唯一有效的办法。

3. **从 C 到 C++ 的核心转变是"少管内存"**。同样是通讯录，C 版本要 `malloc` + 算长度 + `free`，C++ 用 `vector` + `push_back` 几行搞定，还不会泄漏。

### 下周计划

- **类和对象**：封装、`class` 与 `struct` 的区别、构造函数与析构函数、`this` 指针
- **对象的初始化和清理**：构造函数分类与调用方式、拷贝构造函数、深拷贝与浅拷贝（重点，和链表内存管理直接相关）
- **C++ 内存管理**：`new` / `delete`，对比 `malloc` / `free`，理解构造函数在 `new` 时被自动调用
- 把本周的通讯录用 **类** 重新封装一遍，作为综合练习

---

> 本周所有代码都单独编译运行验证过，源码已按模块归档。继续坚持"每个知识点配一个可运行的 demo"。
