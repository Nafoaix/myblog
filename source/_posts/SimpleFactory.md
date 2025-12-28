---
title: 大话设计模式C语言之简单工厂
tags:
  - 学习笔记
  - 设计模式
categories:
  - 学习笔记
  - 设计模式
  - C
toc: true
top_img: 'https://cdn.nafx.top/post/wizarding-world-legacy.jpg'
cover: 'https://cdn.nafx.top/post/blueprint.png'
abbrlink: fe9a5c11
date: 2025-11-09 16:37:09
---

## 大话设计模式C语言之简单工厂

### 关于设计模式

#### 是什么
&emsp;&emsp;设计模式简单来说就是软件开发人员通过在软件开发过程中对常见问题的解决方案总结出来的组织代码方式的抽象。

#### 做什么
&emsp;&emsp;设计模式目的就是为了是实现代码设计的`高内聚,低耦合`思想，实现可复用，可维护，可扩展，灵活性好，易于理解，易于协作，易于测试等特点。

#### 怎么用
&emsp;&emsp;大话设计模式这本书较好的总结了23种设计模式的具体使用方法，但原书例子是C#编写的，这里一边重读下这本书一边用C语言实现一遍，加深理解。

### 简单工厂模式

&emsp;&emsp;简单工厂模式主要是解决对象的创建问题，将对象的创建过程封装在一个工厂类中，通过调用工厂类的方法来创建对象，从而实现对象创建的解耦，并可以根据提供不同参数创建不同对象。

### 案例代码

#### 简单工厂接口 simplefactory.h

&emsp;&emsp;关于接口的定义有两种方法，第一种是行为在对象中，另一种是行为在模块中，这里采用的是将行为放在模块中的方法，因为这样可以真正做到不暴露结构体，更好的封装模块的功能。但是在需要运行期多态的情况下，像后面马上要介绍的策略、装饰、原型、状态模式等就需要将行为放在对象中。

```c
#ifndef __SIMPLEFACTORY_H__
#define __SIMPLEFACTORY_H__

#include <stdbool.h>

typedef struct NumOperation NumOperation; //前向声明

NumOperation* createNumOperation(char opt);
bool CalculateNum (NumOperation *operation , double num1, double num2);
double getResult(NumOperation *operation);
void destroyNumOperation(NumOperation *operation);

#endif //__SIMPLEFACTORY_H__
```

#### 简单工厂实现 simplefactory.c
```c
#include <stdio.h>
#include <stdlib.h>
#include "simplefactory.h"

double calculate_add(double num1, double num2)
{
    return num1 + num2;
}

double calculate_sub(double num1, double num2)
{
    return num1 - num2;
}

double calculate_mul(double num1, double num2)
{
    return num1 * num2;
}

double calculate_div(double num1, double num2)
{
    if (num2 != 0)
    {
        return num1 / num2;
    }
    else
    {
        printf("Error: Division by zero!\n");
        return 0.0;
    }
}

struct NumOperation
{
    double num1;
    double num2;
    double result;
    char symbols;
    double (*getResult)(double num1, double num2);
};

NumOperation *createNumOperation(char str)
{
    NumOperation *operation = (NumOperation *)malloc(sizeof(NumOperation));
    if (operation == NULL)
    {
        return NULL;
    }
    switch (str)
    {
    case '+':
        operation->getResult = calculate_add;
        break;
    case '-':
        operation->getResult = calculate_sub;
        break;
    case '*':
        operation->getResult = calculate_mul;
        break;
    case '/':
        operation->getResult = calculate_div;
        break;

    default:
        free(operation);
        return NULL;
        break;
    }
    operation->num1 = 0.0;
    operation->num2 = 0.0;
    operation->result = 0.0;
    operation->symbols = '\0';

    return operation;
}

bool CalculateNum(NumOperation *operation, double num1, double num2)
{
    if (operation == NULL || operation->getResult == NULL)
    {
        return false;
    }
    operation->result = operation->getResult(num1, num2);
    return true;
}

double getResult(NumOperation *operation)
{
    if (operation == NULL)
    {
        return 0.0;
    }
    return operation->result;
}

void destroyNumOperation(NumOperation *operation)
{
    if (operation != NULL)
    {
        free(operation);
    }
}

```

#### 测试代码 main.c
```c
#include <stdio.h>
#include "simplefactory.h"

int main (void)
{
    double num1;
    double num2;
    char symbols;

    
    printf("Please input two numbers with space.\n");
    scanf("%lf %lf", &num1, &num2);
    printf("Please input an operator (+, -, *, /):\n");
    scanf(" %c", &symbols);
    printf("The symbols is: %c\n",symbols);
    NumOperation* calc = createNumOperation(symbols);
    if (calc == NULL)
    {
        printf("Error: Failed to create NumOperation!\n");
        return -1;
    }
    if (CalculateNum(calc, num1, num2))
    {
        printf("Result: %f\n", getResult(calc));
    }
    else
    {
        printf("Error: Calculation failed!\n");
    }
    destroyNumOperation(calc);
   
    while (1);
    return 0;
}
```
### 总结
&emsp;&emsp;设计模式根据其意图可分为三类：创建型模式，结构型模式，行为型模式。简单工厂模式当然就属于创建型模式。
- 创建型模式提供创建对象的机制， 增加已有代码的灵活性和可复用性。
- 结构型模式提供将对象和类组装成较大的结构， 并同时保持结构的灵活和高效的方法。
- 行为模式负责对象间的高效沟通和职责委派。
