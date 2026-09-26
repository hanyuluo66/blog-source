---
title: "9.27学习周报"
date: 2026-09-26
draft: false
tags:
    - C语言
    - Hugo
    - 学习周报
categories:
    - 学习
summary: "本周学习C语言核心知识点，包括冒泡排序、二维数组、函数分文件编写、指针与数组与函数综合使用、结构体，以及用Hugo+GitHub搭建个人博客。"
---

# 9.27 学习周报

> 本周主要围绕 C 语言核心语法和工具链进行学习，涵盖排序算法、数组、函数工程化、指针综合运用、结构体，以及使用 Hugo + GitHub Pages 搭建个人博客的完整流程。

---

## 一、C 语言冒泡排序

### 1.1 算法原理

- 冒泡排序是一种简单的**交换排序**算法
- 每一轮比较相邻元素，如果前一个比后一个大，则交换它们
- 每一轮结束后，当前未排序部分的最大值会"冒泡"到末尾
- 对于 `n` 个元素，最多需要 `n-1` 轮比较

### 1.2 代码实现

```c
#include <stdio.h>

void bubbleSort(int arr[], int n) {
    int i, j, temp;
    // 外层循环：控制排序轮数
    for (i = 0; i < n - 1; i++) {
        int swapped = 0;  // 优化：标记本轮是否发生交换
        // 内层循环：每轮比较相邻元素
        for (j = 0; j < n - 1 - i; j++) {
            if (arr[j] > arr[j + 1]) {
                temp = arr[j];
                arr[j] = arr[j + 1];
                arr[j + 1] = temp;
                swapped = 1;
            }
        }
        // 如果本轮没有发生交换，说明已经有序，提前退出
        if (swapped == 0) {
            break;
        }
    }
}

int main() {
    int arr[] = {64, 34, 25, 12, 22, 11, 90};
    int n = sizeof(arr) / sizeof(arr[0]);

    printf("排序前：");
    for (int i = 0; i < n; i++) {
        printf("%d ", arr[i]);
    }

    bubbleSort(arr, n);

    printf("\n排序后：");
    for (int i = 0; i < n; i++) {
        printf("%d ", arr[i]);
    }
    printf("\n");

    return 0;
}
```

### 1.3 执行过程演示

以数组 `{5, 3, 8, 1, 2}` 为例：

```
第1轮：5 3 8 1 2 → 3 5 1 2 [8]  （8冒泡到末尾）
第2轮：3 5 1 2 8 → 3 1 2 [5] 8  （5冒泡到末尾）
第3轮：3 1 2 5 8 → 1 2 [3] 5 8  （3冒泡到末尾）
第4轮：1 2 3 5 8 → 1 [2] 3 5 8  （已经有序）
```

### 1.4 关键要点

- **时间复杂度**：最坏 O(n²)，最好 O(n)（加入优化标记后）
- **空间复杂度**：O(1)，原地排序
- **稳定性**：稳定排序（相等元素不交换，相对顺序不变）
- `sizeof(arr) / sizeof(arr[0])` 是求数组长度的常用技巧

---

## 二、二维数组

### 2.1 定义与初始化

```c
#include <stdio.h>

int main() {
    // 方式1：完全初始化
    int arr1[2][3] = {
        {1, 2, 3},
        {4, 5, 6}
    };

    // 方式2：部分初始化（未赋值的元素自动为0）
    int arr2[2][3] = {
        {1, 2},
        {4}
    };

    // 方式3：省略行数（必须指定列数）
    int arr3[][3] = {
        {1, 2, 3},
        {4, 5, 6},
        {7, 8, 9}
    };

    return 0;
}
```

### 2.2 遍历二维数组

```c
#include <stdio.h>

int main() {
    int arr[3][4] = {
        {1,  2,  3,  4},
        {5,  6,  7,  8},
        {9,  10, 11, 12}
    };

    // 使用嵌套循环遍历
    for (int i = 0; i < 3; i++) {         // 遍历行
        for (int j = 0; j < 4; j++) {     // 遍历列
            printf("%3d ", arr[i][j]);
        }
        printf("\n");
    }

    return 0;
}
```

**输出结果：**
```
  1   2   3   4
  5   6   7   8
  9  10  11  12
```

### 2.3 实用案例：矩阵转置

```c
#include <stdio.h>

int main() {
    int matrix[2][3] = {
        {1, 2, 3},
        {4, 5, 6}
    };
    int transposed[3][2];

    // 转置操作：行列互换
    for (int i = 0; i < 2; i++) {
        for (int j = 0; j < 3; j++) {
            transposed[j][i] = matrix[i][j];
        }
    }

    printf("原矩阵：\n");
    for (int i = 0; i < 2; i++) {
        for (int j = 0; j < 3; j++) {
            printf("%d ", matrix[i][j]);
        }
        printf("\n");
    }

    printf("转置后：\n");
    for (int i = 0; i < 3; i++) {
        for (int j = 0; j < 2; j++) {
            printf("%d ", transposed[i][j]);
        }
        printf("\n");
    }

    return 0;
}
```

### 2.4 关键要点

- 二维数组在内存中是**按行连续存储**的
- 定义时可以省略行数，但**不能省略列数**
- 传递给函数时，形参**必须指定列数**：`void func(int arr[][4], int rows)`

---

## 三、函数的分文件编写

### 3.1 为什么要分文件

- 代码量增大后，单文件难以维护
- 将功能模块化，便于团队协作和代码复用
- 头文件（`.h`）声明接口，源文件（`.c`）实现逻辑

### 3.2 项目结构

```
project/
├── main.c          # 主程序入口
├── calculator.h    # 函数声明（头文件）
└── calculator.c    # 函数实现（源文件）
```

### 3.3 代码示例

**calculator.h**（头文件 - 函数声明）：

```c
#ifndef CALCULATOR_H
#define CALCULATOR_H

// 加法
int add(int a, int b);

// 减法
int subtract(int a, int b);

// 乘法
int multiply(int a, int b);

// 除法（返回0表示除数为0的错误）
int divide(int a, int b, int *result);

#endif
```

**calculator.c**（源文件 - 函数实现）：

```c
#include "calculator.h"

int add(int a, int b) {
    return a + b;
}

int subtract(int a, int b) {
    return a - b;
}

int multiply(int a, int b) {
    return a * b;
}

int divide(int a, int b, int *result) {
    if (b == 0) {
        return 0;  // 失败，除数为0
    }
    *result = a / b;
    return 1;  // 成功
}
```

**main.c**（主程序）：

```c
#include <stdio.h>
#include "calculator.h"

int main() {
    int a = 10, b = 3;
    int result;

    printf("%d + %d = %d\n", a, b, add(a, b));
    printf("%d - %d = %d\n", a, b, subtract(a, b));
    printf("%d * %d = %d\n", a, b, multiply(a, b));

    if (divide(a, b, &result)) {
        printf("%d / %d = %d\n", a, b, result);
    }

    if (!divide(a, 0, &result)) {
        printf("错误：除数不能为0\n");
    }

    return 0;
}
```

### 3.4 编译方式

```bash
# 同时编译多个源文件
gcc main.c calculator.c -o calculator

# 运行
./calculator
```

### 3.5 关键要点

- 头文件使用 `#ifndef / #define / #endif` 防止重复包含（头文件守卫）
- `#include "xxx.h"` 用双引号包含自定义头文件，`#include <xxx.h>` 用尖括号包含系统头文件
- 编译时需要将所有 `.c` 文件一起编译

---

## 四、指针、数组、函数的综合使用

### 4.1 指针与数组的关系

```c
#include <stdio.h>

int main() {
    int arr[] = {10, 20, 30, 40, 50};
    int *p = arr;  // 数组名就是首元素的地址

    // 三种等价的访问方式
    for (int i = 0; i < 5; i++) {
        printf("arr[%d] = %d\n", i, arr[i]);        // 下标法
        printf("*(p+%d) = %d\n", i, *(p + i));      // 指针法
        printf("*(arr+%d) = %d\n", i, *(arr + i));   // 数组名法
    }

    return 0;
}
```

### 4.2 指针作为函数参数（实现交换函数）

```c
#include <stdio.h>

// 通过指针修改原始变量的值
void swap(int *a, int *b) {
    int temp = *a;
    *a = *b;
    *b = temp;
}

// 通过数组指针操作数组
void printArray(int *arr, int size) {
    for (int i = 0; i < size; i++) {
        printf("%d ", *(arr + i));
    }
    printf("\n");
}

// 返回数组中的最大值和最小值
void findMaxMin(int *arr, int size, int *max, int *min) {
    *max = arr[0];
    *min = arr[0];
    for (int i = 1; i < size; i++) {
        if (arr[i] > *max) *max = arr[i];
        if (arr[i] < *min) *min = arr[i];
    }
}

int main() {
    // 测试 swap
    int x = 5, y = 10;
    printf("交换前：x=%d, y=%d\n", x, y);
    swap(&x, &y);
    printf("交换后：x=%d, y=%d\n", x, y);

    // 测试 printArray
    int arr[] = {3, 1, 4, 1, 5, 9, 2, 6};
    int size = sizeof(arr) / sizeof(arr[0]);
    printf("数组元素：");
    printArray(arr, size);

    // 测试 findMaxMin
    int max, min;
    findMaxMin(arr, size, &max, &min);
    printf("最大值：%d，最小值：%d\n", max, min);

    return 0;
}
```

### 4.3 函数指针

```c
#include <stdio.h>

int add(int a, int b) { return a + b; }
int multiply(int a, int b) { return a * b; }

// 函数指针作为参数
int compute(int (*op)(int, int), int a, int b) {
    return op(a, b);
}

int main() {
    // 定义函数指针
    int (*funcPtr)(int, int);

    funcPtr = add;
    printf("add(3, 4) = %d\n", funcPtr(3, 4));  // 输出 7

    funcPtr = multiply;
    printf("multiply(3, 4) = %d\n", funcPtr(3, 4));  // 输出 12

    // 通过函数指针传参
    printf("compute(add, 5, 6) = %d\n", compute(add, 5, 6));       // 11
    printf("compute(multiply, 5, 6) = %d\n", compute(multiply, 5, 6)); // 30

    return 0;
}
```

### 4.4 关键要点

- **数组名**本质上是指向首元素的常量指针
- 函数传参时，传数组实际传的是**地址**（退化为指针）
- 使用指针可以在函数内**修改调用者的变量**
- 函数指针可以实现**回调机制**，提高代码灵活性

---

## 五、结构体的使用

### 5.1 基本定义与使用

```c
#include <stdio.h>
#include <string.h>

// 定义结构体
struct Student {
    char name[50];
    int age;
    float score;
};

int main() {
    // 方式1：逐个赋值
    struct Student s1;
    strcpy(s1.name, "张三");
    s1.age = 20;
    s1.score = 95.5;

    // 方式2：初始化列表
    struct Student s2 = {"李四", 21, 88.0};

    printf("学生1：%s, %d岁, %.1f分\n", s1.name, s1.age, s1.score);
    printf("学生2：%s, %d岁, %.1f分\n", s2.name, s2.age, s2.score);

    return 0;
}
```

### 5.2 typedef 简化类型名

```c
#include <stdio.h>
#include <string.h>

// 使用 typedef 简化结构体类型名
typedef struct {
    char title[100];
    char author[50];
    int year;
    float price;
} Book;

void printBook(Book b) {
    printf("《%s》 作者：%s, 出版年份：%d, 价格：%.2f\n",
           b.title, b.author, b.year, b.price);
}

int main() {
    Book book1 = {"C程序设计", "谭浩强", 2010, 45.00};
    Book book2 = {
        .title = "数据结构",
        .author = "严蔚敏",
        .year = 2007,
        .price = 38.50
    };

    printBook(book1);
    printBook(book2);

    return 0;
}
```

### 5.3 结构体数组与指针

```c
#include <stdio.h>
#include <string.h>

typedef struct {
    char name[50];
    int score;
} Student;

// 结构体指针作为函数参数
void sortByScore(Student *students, int n) {
    Student temp;
    for (int i = 0; i < n - 1; i++) {
        for (int j = 0; j < n - 1 - i; j++) {
            if (students[j].score < students[j + 1].score) {
                temp = students[j];
                students[j] = students[j + 1];
                students[j + 1] = temp;
            }
        }
    }
}

int main() {
    Student class[] = {
        {"Alice", 92},
        {"Bob", 78},
        {"Charlie", 85},
        {"David", 96},
        {"Eve", 88}
    };
    int n = sizeof(class) / sizeof(class[0]);

    printf("排序前：\n");
    for (int i = 0; i < n; i++) {
        printf("  %s: %d分\n", class[i].name, class[i].score);
    }

    sortByScore(class, n);

    printf("\n按成绩降序排列：\n");
    for (int i = 0; i < n; i++) {
        printf("  %s: %d分\n", class[i].name, class[i].score);
    }

    return 0;
}
```

### 5.4 结构体嵌套

```c
#include <stdio.h>

typedef struct {
    int year;
    int month;
    int day;
} Date;

typedef struct {
    char name[50];
    Date birthday;    // 嵌套结构体
    Date enrollDate;
} Student;

int main() {
    Student s = {
        .name = "王五",
        .birthday = {2004, 6, 15},
        .enrollDate = {2022, 9, 1}
    };

    printf("姓名：%s\n", s.name);
    printf("生日：%d-%02d-%02d\n", s.birthday.year, s.birthday.month, s.birthday.day);
    printf("入学：%d-%02d-%02d\n", s.enrollDate.year, s.enrollDate.month, s.enrollDate.day);

    return 0;
}
```

### 5.5 关键要点

- `typedef` 可以给结构体起别名，简化书写
- 结构体作为函数参数时，传指针更高效（避免值拷贝）
- 通过 `.` 访问成员，通过 `->` 访问指针指向的结构体成员
- 结构体可以嵌套，实现复杂的数据组织

---

## 六、用 Hugo + GitHub 创建博客

### 6.1 环境准备

1. **安装 Git**：从 [git-scm.com](https://git-scm.com) 下载安装
2. **安装 Hugo**：从 [Hugo 官网](https://gohugo.io) 下载对应系统的版本
3. **注册 GitHub 账号**：在 [github.com](https://github.com) 注册

### 6.2 创建 Hugo 站点

```bash
# 创建新站点
hugo new site my-blog

# 进入站点目录
cd my-blog

# 初始化 Git 仓库
git init
```

### 6.3 安装主题

```bash
# 添加主题（以 hugo-theme-stack 为例）
git submodule add https://github.com/CaiJimmy/hugo-theme-stack.git themes/hugo-theme-stack
```

在 `hugo.toml` 中配置主题：

```toml
baseURL = "https://你的用户名.github.io/"
title = "我的博客"
theme = "hugo-theme-stack"
languageCode = "zh-cn"
```

### 6.4 创建文章

```bash
# 使用 hugo 命令创建新文章
hugo new post/my-first-post.md
```

### 6.5 本地预览

```bash
# 启动本地服务器
hugo server -D

# 浏览器访问 http://localhost:1313 预览效果
```

### 6.6 部署到 GitHub Pages

**步骤 1：在 GitHub 创建仓库**

- 仓库名必须为 `你的用户名.github.io`
- 勾选 Public（公开仓库才能使用 GitHub Pages）

**步骤 2：生成静态文件并推送**

```bash
# 生成静态文件
hugo

# 进入 public 目录
cd public

# 初始化 Git 并推送
git init
git add .
git commit -m "first commit"
git branch -M main
git remote add origin https://github.com/你的用户名/你的用户名.github.io.git
git push -u origin main
```

**步骤 3：启用 GitHub Pages**

- 进入仓库 → Settings → Pages
- Source 选择 `main` 分支，目录选 `/ (root)`
- 保存后等待部署完成

**步骤 4：访问博客**

- 打开 `https://你的用户名.github.io` 即可看到博客

### 6.7 日常更新流程

```bash
# 1. 写好文章后，生成静态文件
hugo

# 2. 进入 public 目录提交
cd public
git add .
git commit -m "update: 新增9.27学习周报"
git push

# 3. 等待 1-2 分钟后刷新博客页面
```

### 6.8 关键要点

- Hugo 是**静态站点生成器**，生成纯 HTML，无需服务器
- GitHub Pages 免费提供静态网站托管
- 仓库名 `用户名.github.io` 是约定格式，不能随意命名
- 每次更新文章后需要重新 `hugo` 生成并推送

---

## 本周学习总结

| 知识点 | 掌握程度 | 备注 |
|--------|---------|------|
| 冒泡排序 | 熟练 | 理解了优化标记的作用 |
| 二维数组 | 熟练 | 掌握了矩阵转置等应用 |
| 函数分文件编写 | 了解 | 需要多练习工程化项目 |
| 指针综合运用 | 熟练 | 函数指针还需深入 |
| 结构体 | 熟练 | 嵌套和指针操作已掌握 |
| Hugo 博客搭建 | 完成 | 已成功部署上线 |

> 下周计划：继续深入学习指针的高级用法、动态内存分配，以及链表等数据结构。
