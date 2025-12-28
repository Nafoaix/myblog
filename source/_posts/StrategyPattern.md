---
title: 大话设计模式C语言之策略模式
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
abbrlink: 1c84a193
date: 2025-11-16 12:01:22
---

## 大话设计模式C语言之策略模式

### 策略模式

&emsp;&emsp;根据上篇提到的分类，策略模式（Strategy Pattern）属于对象行为模式，它通过对算法进行封装，把使用算法的责任和算法的实现分割开来，并委派给不同的对象对这些算法进行管理。策略模式定义了一系列算法，并将每一个算法封装起来，使它们可以互相替换，且算法的变化不会影响到使用算法的客户。

### 案例代码

#### 策略接口 payment_strategy.h

>策略模式建议找出负责用许多不同方式完成特定任务的类，然后将其中的算法抽取到一组被称为策略的独立类中。

&emsp;&emsp;根据`大话设计模式`书中举的例子，商场搞活动，有打折、满减、返利等不同策略，都可以抽象为付款行为中的计算折扣动作于是付款策略接口这里需要定义一个计算折扣的算法，最后通过简单工厂来在创建时动态注入不同的算法。

```c
#ifndef PAYMENT_STRATEGY_H
#define PAYMENT_STRATEGY_H

/* 策略接口 */
typedef struct PaymentStrategy
{
        double (*calculate_discount)(double);

} PaymentStrategy;

#endif // PAYMENT_STRATEGY_H
```

#### 策略上下文接口 payment_context.h

>名为上下文的原始类必须包含一个成员变量来存储对于每种策略的引用。 上下文并不执行任务， 而是将工作委派给已连接的策略对象。

&emsp;&emsp;这里是支付策略的上下文定义，包含一个指向策略接口的指针，通过这个指针来调用策略算法。

```c
#ifndef PAYMENT_CONTEXT_H
#define PAYMENT_CONTEXT_H

#include "payment_strategy.h"

typedef struct PaymentContext PaymentContext;

PaymentContext *paymentContextCreate(void);

void paymentContextDestroy(PaymentContext *context);

/* 设置策略 */
bool paymentContextSetStrategy(PaymentContext *context,char strategy_type);

/* 执行支付 */
double paymentContextPay(PaymentContext *context,double amount);

#endif //PAYMENT_CONTEXT_H

```

#### 策略上下文类 payment_context.c

&emsp;&emsp;策略上下文接口的具体实现，举例方便这里将具体算法实现和策略上下文类的实现放在一起。在创建策略上下文时动态注入需要的算法。

```c
#include <stdio.h>
#include <stdlib.h>
#include <stdbool.h>
#include "payment_context.h"

double CashNormal(double amount)
{
    return amount;
}

double CashRebate(double amount)
{
    return amount * 0.9;
}

double CashReturn(double amount)
{
    if (amount >= 300)
    {
        int times = (int)(amount / 300);
        return amount - times * 30;
    }
    return amount;
}

typedef struct PaymentContext
{
    const PaymentStrategy *strategy;
    long OperationNumber;
} PaymentContext;

static const PaymentStrategy CashNormalFunction = {
    .calculate_discount = CashNormal,
};

static const PaymentStrategy CashRebateFunction = {
    .calculate_discount = CashRebate,
};

static const PaymentStrategy CashReturnFunction = {
    .calculate_discount = CashReturn,
};

PaymentContext *paymentContextCreate(void)
{
    PaymentContext *context = (PaymentContext *)malloc(sizeof(PaymentContext));
    if (context == NULL)
    {
        return NULL;
    }
    context->strategy = NULL;
    context->OperationNumber = 0;
    return context;
}

void paymentContextDestroy(PaymentContext *context)
{
    if (context != NULL)
    {
        free(context);
    }
}

bool paymentContextSetStrategy(PaymentContext *context, char strategy_type)
{
    if (NULL == context)
    {
        return false;
    }

    switch (strategy_type)
    {
    case 'N':
        context->strategy = &CashNormalFunction;
        break;
    case 'A':
        context->strategy = &CashRebateFunction;
        break;
    case 'B':
        context->strategy = &CashReturnFunction;
        break;
    default:
        return false;
    }
    return true;
}
double paymentContextPay(PaymentContext *context, double amount)
{
    if (NULL == context)
    {
        return -1;
    }
    if (context->strategy != NULL)
    {
        return context->strategy->calculate_discount(amount);
    }
}

```

#### 测试代码 main.c

```c
#include <stdio.h>
#include "payment_context.h"

int main(void)
{
    double total_amount;
    char discount_method;
    PaymentContext *context = paymentContextCreate();
    printf("Please input the total amount of goods. \n");
    scanf("%lf", &total_amount);
    printf("Please enter the discount method: \n - N for no discount \n - A for 10%% off \n - B for 30 off for every 300 spent \n");
    scanf(" %c", &discount_method);
    if (!paymentContextSetStrategy(context, discount_method))
    {
        printf("Invalid discount method.\n");
        paymentContextDestroy(context);
        return 1;
    }
    printf("The final amount after discount is: %.2lf\n", paymentContextPay(context, total_amount));
    paymentContextDestroy(context);
    
    while (1);
    return 0;
}

```

### 总结
&emsp;&emsp;策略模式就是用来封装算法的，但在事件中，我们发现可以用它来发封装几乎任何类型的变化，只要在分析过程中听到需要在不同时间应用不同业务规则，就可以考虑使用策略模式处理这种变化的可能性。