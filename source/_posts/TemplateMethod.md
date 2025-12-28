---
title: 大话设计模式C语言之模板模式
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
abbrlink: 
date: 2025-12-13 11:22:03
---

## 大话设计模式C语言之模板模式

### 模板模式

>模板方法 (Template Method) 模式：在基类中定义算法骨架，将某些步骤延迟到子类实现，使子类在不改变算法结构的情况下重新定义算法的某些步骤。

### 案例代码

&emsp;&emsp;这里会举了两个简单例子来帮助理解原型模式，第一个例子帮助我们理解原型模式的基本结构，具体原型如何深拷贝实现clone接口来实现原型模式的，以及在符合开闭原则下通过函数指针的多态性对新原型的拓展。第二个例子主要是带我们了解原型注册表这一概念，注册表提供了一种访问常用原型的简单方法，其中存储了一系列可供随时复制的预生成对象，我们可以提供过注册表集中管理对象模板，这样我们就可以根据字符串、配置、网络消息等方式动态创建对象。

#### 案例代码一

#### 原型接口 prototype.h

>原型 （Prototype）接口将对克隆方法进行声明。在绝大多数情况下，其中只会有一个名为 clone克隆的方法。

&emsp;&emsp;原型模式需要注意的是在实现克隆接口的时候要注意浅拷贝问题，这里使用结构体指针作为参数，返回值也是结构体指针，这样就可以实现深拷贝。

```c
#ifndef PROTOTYPE_H
#define PROTOTYPE_H

typedef struct Prototype Prototype;

struct Prototype {
    Prototype* (*clone)(const Prototype* self);
    void (*print)(const Prototype* self);
    void (*destroy)(Prototype* self);
};

#endif /* prototype.h */

```

#### 具体原型A接口 concrete_a.h

```c
#ifndef CONCRETE_A_H
#define CONCRETE_A_H

#include "prototype.h"

Prototype* ConcreteA_create(const char* name, int value);

#endif // CONCRETE_A_H

```

#### 具体原型A实现 concrete_a.c

```c
#include "concrete_a.h"
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

typedef struct {
    Prototype base;
    char* name;
    int value;
} ConcreteA;

static Prototype* ConcreteA_clone(const Prototype* self) {
    const ConcreteA* src = (const ConcreteA*)self;

    ConcreteA* copy = malloc(sizeof(ConcreteA));
    if (!copy) return NULL;

    copy->base = src->base;
    copy->value = src->value;

    copy->name = malloc(strlen(src->name) + 1);
    if (!copy->name) {
        free(copy);
        return NULL;
    }
    strcpy(copy->name, src->name);

    return (Prototype*)copy;
}

static void ConcreteA_print(const Prototype* self) {
    const ConcreteA* obj = (const ConcreteA*)self;
    printf("ConcreteA { name=%s, value=%d }\n", obj->name, obj->value);
}

static void ConcreteA_destroy(Prototype* self) {
    ConcreteA* obj = (ConcreteA*)self;
    free(obj->name);
    free(obj);
}

Prototype* ConcreteA_create(const char* name, int value) {
    ConcreteA* obj = malloc(sizeof(ConcreteA));
    if (!obj) return NULL;

    obj->base.clone   = ConcreteA_clone;
    obj->base.print   = ConcreteA_print;
    obj->base.destroy = ConcreteA_destroy;

    obj->name = malloc(strlen(name) + 1);
    if (!obj->name) {
        free(obj);
        return NULL;
    }
    strcpy(obj->name, name);

    obj->value = value;
    return (Prototype*)obj;
}

```

#### 具体原型B接口 concrete_b.h

&emsp;&emsp;这里原型B与A的结构不同，体现原型的扩展性。

```c
#ifndef CONCRETE_B_H
#define CONCRETE_B_H

#include "prototype.h"

Prototype* ConcreteB_create(double factor);

#endif

```

#### 具体原型B实现 concrete_b.c

```c
#include "concrete_b.h"
#include <stdio.h>
#include <stdlib.h>

typedef struct {
    Prototype base;
    double factor;
} ConcreteB;

static Prototype* ConcreteB_clone(const Prototype* self) {
    ConcreteB* copy = malloc(sizeof(ConcreteB));
    if (!copy) return NULL;

    *copy = *(const ConcreteB*)self;
    return (Prototype*)copy;
}

static void ConcreteB_print(const Prototype* self) {
    const ConcreteB* obj = (const ConcreteB*)self;
    printf("ConcreteB { factor=%.2f }\n", obj->factor);
}

static void ConcreteB_destroy(Prototype* self) {
    free(self);
}

Prototype* ConcreteB_create(double factor) {
    ConcreteB* obj = malloc(sizeof(ConcreteB));
    if (!obj) return NULL;

    obj->base.clone   = ConcreteB_clone;
    obj->base.print   = ConcreteB_print;
    obj->base.destroy = ConcreteB_destroy;

    obj->factor = factor;
    return (Prototype*)obj;
}

```

#### 测试代码 main.c

```c
#include "concrete_a.h"
#include "concrete_b.h"

int main(void) {
    Prototype* p1 = ConcreteA_create("TemplateA", 42);
    Prototype* p2 = p1->clone(p1);

    Prototype* p3 = ConcreteB_create(3.14);
    Prototype* p4 = p3->clone(p3);

    p1->print(p1);
    p2->print(p2);
    p3->print(p3);
    p4->print(p4);

    p1->destroy(p1);
    p2->destroy(p2);
    p3->destroy(p3);
    p4->destroy(p4);
    
    while (1);
    return 0;
}

```

#### 案例代码二

#### 原型注册表接口 prototype_registry.h



```c
#ifndef PROTOTYPE_REGISTRY_H
#define PROTOTYPE_REGISTRY_H

#include "prototype.h"

typedef struct PrototypeRegistry PrototypeRegistry;

struct PrototypeRegistry {
    Prototype* (*create)(const PrototypeRegistry* self, const char* name);
    void (*destroy)(PrototypeRegistry* self);
};

PrototypeRegistry* PrototypeRegistry_create(void);
void PrototypeRegistry_destroy(PrototypeRegistry* self);

#endif /* prototype_registry.h */

```

#### 原型注册表接口设计 prototype_registry.h

```c
#ifndef PROTOTYPE_REGISTRY_H
#define PROTOTYPE_REGISTRY_H

#include "prototype.h"

/* 注册表句柄（不暴露内部结构） */
typedef struct PrototypeRegistry PrototypeRegistry;

/* 生命周期 */
PrototypeRegistry* PrototypeRegistry_create(void);
void PrototypeRegistry_destroy(PrototypeRegistry* registry);

/* 注册 / 反注册 */
int PrototypeRegistry_register(
    PrototypeRegistry* registry,
    const char* key,
    Prototype* prototype);

int PrototypeRegistry_unregister(
    PrototypeRegistry* registry,
    const char* key);

/* 核心能力：通过 key 克隆对象 */
Prototype* PrototypeRegistry_clone(
    const PrototypeRegistry* registry,
    const char* key);

#endif // PROTOTYPE_REGISTRY_H

```

#### 原型注册表实现 prototype_registry.c

```c
#include "prototype_registry.h"
#include <stdlib.h>
#include <string.h>

/* 简单链表实现（工程中可替换为 hash map） */

typedef struct RegistryNode {
    char* key;
    Prototype* prototype;
    struct RegistryNode* next;
} RegistryNode;

struct PrototypeRegistry {
    RegistryNode* head;
};

PrototypeRegistry* PrototypeRegistry_create(void) {
    PrototypeRegistry* registry = malloc(sizeof(PrototypeRegistry));
    if (!registry) return NULL;
    registry->head = NULL;
    return registry;
}

static void RegistryNode_destroy(RegistryNode* node) {
    free(node->key);
    node->prototype->destroy(node->prototype);
    free(node);
}

void PrototypeRegistry_destroy(PrototypeRegistry* registry) {
    RegistryNode* cur = registry->head;
    while (cur) {
        RegistryNode* next = cur->next;
        RegistryNode_destroy(cur);
        cur = next;
    }
    free(registry);
}

int PrototypeRegistry_register(
    PrototypeRegistry* registry,
    const char* key,
    Prototype* prototype)
{
    /* 防止重复注册 */
    for (RegistryNode* n = registry->head; n; n = n->next) {
        if (strcmp(n->key, key) == 0) {
            return -1;
        }
    }

    RegistryNode* node = malloc(sizeof(RegistryNode));
    if (!node) return -1;

    node->key = strdup(key);
    if (!node->key) {
        free(node);
        return -1;
    }

    node->prototype = prototype;
    node->next = registry->head;
    registry->head = node;

    return 0;
}

int PrototypeRegistry_unregister(
    PrototypeRegistry* registry,
    const char* key)
{
    RegistryNode** cur = &registry->head;
    while (*cur) {
        if (strcmp((*cur)->key, key) == 0) {
            RegistryNode* target = *cur;
            *cur = target->next;
            RegistryNode_destroy(target);
            return 0;
        }
        cur = &(*cur)->next;
    }
    return -1;
}

Prototype* PrototypeRegistry_clone(
    const PrototypeRegistry* registry,
    const char* key)
{
    for (RegistryNode* n = registry->head; n; n = n->next) {
        if (strcmp(n->key, key) == 0) {
            return n->prototype->clone(n->prototype);
        }
    }
    return NULL;
}

```

#### 测试代码 main.c

```c
#include "prototype_registry.h"
#include "concrete_a.h"
#include "concrete_b.h"
#include <stdio.h>

int main(void) {
    PrototypeRegistry* registry = PrototypeRegistry_create();

    PrototypeRegistry_register(
        registry,
        "configA",
        ConcreteA_create("DefaultA", 100));

    PrototypeRegistry_register(
        registry,
        "factorB",
        ConcreteB_create(2.5));

    Prototype* obj1 = PrototypeRegistry_clone(registry, "configA");
    Prototype* obj2 = PrototypeRegistry_clone(registry, "factorB");

    obj1->print(obj1);
    obj2->print(obj2);

    obj1->destroy(obj1);
    obj2->destroy(obj2);

    PrototypeRegistry_destroy(registry);

    while (1);
    return 0;
}

```
### 总结
&emsp;&emsp;真实项目里几乎不会单独使用某个模式，而是使用几种模式共同组合成一套结构，还是那句话，设计模式不是最终目的，只是一套经验的总结来帮助我们组织代码解决问题，到这里对于已经学过的几种模式其实已经有种原型保存状态 + 工厂控制流程 + 策略注入变化的基础结构形态了，但这里优先以了解每种模式为主，等所有模式讲完后我们最后再来了解这些。
