---
title: 大话设计模式C语言之装饰模式
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
abbrlink: '88161489'
date: 2025-11-23 15:04:25
---

## 大话设计模式C语言之装饰模式

### 装饰模式

&emsp;&emsp;装饰模式可以动态地给给对象添加额外的功能，而不改变其原有的结构或接口。所以目的上装饰器模式属于结构型模式，这类模式介绍如何将对象和类组装成较大的结构，并同时保持结构的灵活和高效。

### 案例代码

&emsp;&emsp;这里与原书不同举了一个简单的咖啡点单例子，点咖啡时可以选择加牛奶、加糖、全加或者全不加等，这里就是通过装饰模式实现动态地给咖啡添加职责，结合书中例子可以更好理解装饰模式。

#### 部件接口 component.h

>部件（Component）声明封装器和被封装对象的公用接口。

&emsp;&emsp;Component是定义一个对象接口，可以给这些对象动态地添加职责。在C中也就是结构体加函数指针的形式。这里定义了一个咖啡组件，具有显示价格，描述自己和销毁三个方法。

```c
#ifndef COMPONENT_H
#define COMPONENT_H

typedef struct Coffee Coffee;

struct Coffee {
    double (*cost)(Coffee *self);
    void   (*description)(Coffee *self);
    void   (*destroy)(Coffee *self);
};

#endif //COMPONENT_H

```

#### 具体实现部件 component.c

>具体部件 Concrete Component）类是被封装对象所属的类。 它定义了基础行为， 但装饰类可以改变这些行为。

&emsp;&emsp;这里对咖啡组件进行了具体实现，下面SimpleCoffee结构体定义可以看到，只是实现了部件接口中的三个方法，没有增加其他职责。

```c
#include <stdio.h>
#include <stdlib.h>
#include "component.h"

//ConcreteComponent SimpleCoffee 实现

typedef struct {
    Coffee base;
} SimpleCoffee;

static double simple_cost(Coffee *self) {
    (void)self;
    return 10.0;
}

static void simple_description(Coffee *self) {
    (void)self;
    printf("Simple Coffee");
}

static void simple_destroy(Coffee *self) {
    free(self);
}

Coffee *simple_coffee_create(void) {

    SimpleCoffee *coffee = malloc(sizeof(SimpleCoffee));
    if (!coffee) return NULL;

    coffee->base.cost        = simple_cost;
    coffee->base.description = simple_description;
    coffee->base.destroy     = simple_destroy;

    return (Coffee *)coffee;
}

```

#### 抽象装饰类 decorator.h

>基础装饰（Base Decorator）类拥有一个指向被封装对象的引用成员变量。该变量的类型应当被声明为通用部件接口，这样它就可以引用具体的部件和装饰。装饰基类会将所有操作委派给被封装的对象。

&emsp;&emsp;装饰抽象类，继承了component这里也就是咖啡组件，同时包含一个指向咖啡组件的指针，这样装饰类就可以动态地给咖啡组件添加职责。但对于component来说，它并不知道也无需知道装饰类的存在，它只知道自己的接口，所以component对于装饰类来说是透明的。这里也就是装饰模式的特点，就像是套娃一样将装饰类一层一层地套在部件上。

```c
#ifndef DECORATOR_H
#define DECORATOR_H

#include "component.h"

// 装饰器抽象层 

typedef struct {
    Coffee base;
    Coffee *component;
} CoffeeDecorator;

#endif // DECORATOR_H


```

#### 具体装饰类-牛奶 milkdecorator.c

>具体装饰类（Concrete Decorators）定义了可动态添加到部件的额外行为。具体装饰类会重写装饰基类的方法，并在调用父类方法之前或之后进行额外的行为。

&emsp;&emsp;具体装饰类，也就是抽象装饰的具体对象。这里具体实现了牛奶装饰类，在原有咖啡组件的基础上增加了牛奶的职责，也就是在原有咖啡组件的基础上增加了牛奶的价格和描述。

```c
#include <stdio.h>
#include <stdlib.h>
#include "decorator.h"

// ConcreteDecorator 具体装饰器 MilkDecorator 实现

static double milk_cost(Coffee *self) {
    CoffeeDecorator *dec = (CoffeeDecorator *)self;
    return dec->component->cost(dec->component) + 2.0;
}

static void milk_description(Coffee *self) {
    CoffeeDecorator *dec = (CoffeeDecorator *)self;
    dec->component->description(dec->component);
    printf(" + Milk");
}

static void milk_destroy(Coffee *self) {
    CoffeeDecorator *dec = (CoffeeDecorator *)self;

    /* 先销毁被装饰对象 */
    if (dec->component) {
        dec->component->destroy(dec->component);
    }

    /* 再释放自己 */
    free(dec);
}

Coffee *milk_decorator_create(Coffee *coffee) {
    CoffeeDecorator *dec = malloc(sizeof(CoffeeDecorator));
    if (!dec) return NULL;

    dec->component          = coffee;
    dec->base.cost          = milk_cost;
    dec->base.description   = milk_description;
    dec->base.destroy       = milk_destroy;

    return (Coffee *)dec;
}

```

#### 具体装饰类-糖 sugardecorator.c

&emsp;&emsp;每个装饰器（如 MilkDecorator、SugarDecorator）都覆盖了 operation 方法，以增加自己的行为，同时调用底层对象的 operation。

```c
#include <stdio.h>
#include <stdlib.h>
#include "decorator.h"

// ConcreteDecorator 具体装饰器 SugarDecorator 实现

static double sugar_cost(Coffee *self) {
    CoffeeDecorator *dec = (CoffeeDecorator *)self;
    return dec->component->cost(dec->component) + 1.0;
}

static void sugar_description(Coffee *self) {
    CoffeeDecorator *dec = (CoffeeDecorator *)self;
    dec->component->description(dec->component);
    printf(" + Sugar");
}

static void sugar_destroy(Coffee *self) {
    CoffeeDecorator *dec = (CoffeeDecorator *)self;

    if (dec->component) {
        dec->component->destroy(dec->component);
    }

    free(dec);
}

Coffee *sugar_decorator_create(Coffee *coffee) {
    CoffeeDecorator *dec = malloc(sizeof(CoffeeDecorator));
    if (!dec) return NULL;

    dec->component        = coffee;
    dec->base.cost        = sugar_cost;
    dec->base.description = sugar_description;
    dec->base.destroy     = sugar_destroy;

    return (Coffee *)dec;
}

```

#### 测试代码 main.c

>客户端（Client）可以使用多层装饰来封装部件，只要它能使用通用接口与所有对象互动即可。

&emsp;&emsp;当创建了对咖啡的装饰后，只需要对基础咖啡组件的description、cost、destroy方法进行调用，就会一层一层调用到装饰类中的方法，从而实现了对基础咖啡组件的扩展,通过这个特点同时也实现了装饰器链的递归释放。同时通过测试例子可以注意到组合顺序会影响最终结果。

```c
#include <stdio.h>
#include "component.h"

Coffee *simple_coffee_create(void);
Coffee *milk_decorator_create(Coffee *coffee);
Coffee *sugar_decorator_create(Coffee *coffee);

int main(void)
{
    int num;
    printf(" Please enter the num of coffee you wish to order.\n");
    printf(" 1. Simple Coffee  -----------  $10.0\n");
    scanf("%d", &num);
    if (num != 1)
    {
        printf("Sorry, we only have simple coffee now.\n");
        return 0;
    }
    Coffee *coffee = simple_coffee_create();
    printf(" Enter the number of the add-ons you want in your coffee .\n");
    printf(" Use space as separators , Enter 0 to end the order .\n");
    printf(" 0. None  ---------------------  $0.0\n");
    printf(" 1. Sugar ---------------------  $1.0\n");
    printf(" 2. Milk  ---------------------  $2.0\n");
    while (scanf("%d", &num) == 1)
    {
        if (0 == num)
        {
            break;
        }
        else if (1 == num)
        {
            coffee = sugar_decorator_create(coffee);
        }
        else if (2 == num)
        {
            coffee = milk_decorator_create(coffee);
        }
    }
    coffee->description(coffee);
    printf("\nTotal Cost: %.2f\n", coffee->cost(coffee));
    /* 只需要一次 destroy */
    coffee->destroy(coffee);

    while (1)
        ;
    return 0;
}

```
### 总结
&emsp;&emsp;装饰模式是继承关系的一种替代方案，它通过组合而非继承来实现对象的功能扩展。装饰模式允许我们动态地添加或删除对象的职责，而不需要修改对象的代码。这使得装饰模式非常灵活和可扩展。