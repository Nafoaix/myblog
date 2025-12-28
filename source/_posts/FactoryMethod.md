---
title: 大话设计模式C语言之工厂方法
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
abbrlink: c91fbaea
date: 2025-12-05 12:33:24
---

## 大话设计模式C语言之工厂方法

### 工厂方法

>工厂方法（Factory Method）模式是一种创建型设计模式，定义一个用于创建对象的接口，让子类决定实例化哪一个类。工厂方法使一个类的实例化延迟到其子类。

&emsp;&emsp;第一篇学习的简单工厂模式虽然简单但缺点也很明显，当需要新增产品时就需要修改工厂类在switch中增加新的分支，这违背了开闭原则。工厂方法模式则解决了这个问题，将工厂类抽象出来，当需要新增产品时，只需要新增一个工厂类即可，不需要修改原有的工厂类。但是具体应用时要选用什么简单工厂或是工厂方法还是取决于实际情况，比如产品类型固定（或极少变化）的场合且创建逻辑简单的小型项目，选择使用简单工厂即可，不用为了所谓的优雅而生搬硬套某种模式，合适的才是最好的。

### 案例代码

&emsp;&emsp;这里与原书不同举了一个简单日志系统的例子，LoggerFactory包含通用的Logger接口,后通过console_logger_factory和FileLoggerFactory实现具体两个日志产品实现从控制台或从文件输出日志的功能，如果要新增其他Logger只需要增加其他Logger类的文件即可，不需要使用判断或修改客户端，结合书中例子可以更好理解工厂方法。

#### 产品接口 logger.h

>产品（Product）将会对接口进行声明。对于所有由创建者及其子类构建的对象，这些接口都是通用的。也就是定义了工厂方法所创建的对象的接口。

```c
#ifndef LOGGER_H
#define LOGGER_H

typedef struct Logger Logger;

/* Logger 接口 */
struct Logger {
    void (*log)(Logger *self, const char *message);
    void (*destroy)(Logger *self);
};

#endif // LOGGER_H

```

#### 工厂接口 logger_factory.h

>工厂类声明返回产品对象的工厂方法。该方法的返回对象类型必须与产品接口相匹配。

```c
#ifndef LOGGER_FACTORY_H
#define LOGGER_FACTORY_H

#include "logger.h"

/* Factory Method 接口 */
typedef struct LoggerFactory LoggerFactory;

struct LoggerFactory {
    Logger *(*create_logger)(LoggerFactory *self);
    void (*destroy)(LoggerFactory *self);
};

#endif /* LOGGER_FACTORY_H */

```

#### 具体工厂与产品-控制台日志 console_logger_factory.c

```c
#include <stdio.h>
#include <stdlib.h>
#include "logger_factory.h"

/* ========= Concrete Product ========= */

typedef struct {
    Logger base;
} ConsoleLogger;

static void console_log(Logger *self, const char *message) {
    (void)self;
    printf("[Console] %s\n", message);
}

static void console_logger_destroy(Logger *self) {
    free(self);
}

static Logger *console_logger_create(void) {
    ConsoleLogger *logger = malloc(sizeof(ConsoleLogger));
    if (!logger) return NULL;

    logger->base.log = console_log;
    logger->base.destroy = console_logger_destroy;
    return (Logger *)logger;
}

/* ========= Concrete Factory ========= */

typedef struct {
    LoggerFactory base;
} ConsoleLoggerFactory;

static Logger *console_factory_create(LoggerFactory *self) {
    (void)self;
    return console_logger_create();
}

static void console_factory_destroy(LoggerFactory *self) {
    free(self);
}

LoggerFactory *console_logger_factory_create(void) {
    ConsoleLoggerFactory *factory = malloc(sizeof(ConsoleLoggerFactory));
    if (!factory) return NULL;

    factory->base.create_logger = console_factory_create;
    factory->base.destroy = console_factory_destroy;
    return (LoggerFactory *)factory;
}

```

#### 具体工厂与产品-文件日志 file_logger_factory.c

```c
#include <stdio.h>
#include <stdlib.h>
#include "logger_factory.h"

/* ========= Concrete Product ========= */

typedef struct {
    Logger base;
    FILE *file;
} FileLogger;

static void file_log(Logger *self, const char *message) {
    FileLogger *logger = (FileLogger *)self;
    fprintf(logger->file, "[File] %s\n", message);
    fflush(logger->file);
}

static void file_logger_destroy(Logger *self) {
    FileLogger *logger = (FileLogger *)self;
    fclose(logger->file);
    free(logger);
}

static Logger *file_logger_create(void) {
    FileLogger *logger = malloc(sizeof(FileLogger));
    if (!logger) return NULL;

    logger->file = fopen("app.log", "a");
    if (!logger->file) {
        free(logger);
        return NULL;
    }

    logger->base.log = file_log;
    logger->base.destroy = file_logger_destroy;
    return (Logger *)logger;
}

/* ========= Concrete Factory ========= */

typedef struct {
    LoggerFactory base;
} FileLoggerFactory;

static Logger *file_factory_create(LoggerFactory *self) {
    (void)self;
    return file_logger_create();
}

static void file_factory_destroy(LoggerFactory *self) {
    free(self);
}

LoggerFactory *file_logger_factory_create(void) {
    FileLoggerFactory *factory = malloc(sizeof(FileLoggerFactory));
    if (!factory) return NULL;

    factory->base.create_logger = file_factory_create;
    factory->base.destroy = file_factory_destroy;
    return (LoggerFactory *)factory;
}

```

#### 测试代码 main.c

```c
#include "logger_factory.h"

/* 外部工厂构造函数 */
LoggerFactory *console_logger_factory_create(void);
LoggerFactory *file_logger_factory_create(void);

static void run(LoggerFactory *factory) {
    Logger *logger = factory->create_logger(factory);
    if (!logger) return;

    logger->log(logger, "Hello Factory Method");
    logger->destroy(logger);
}

int main(void) {
    LoggerFactory *console_factory = console_logger_factory_create();
    LoggerFactory *file_factory = file_logger_factory_create();

    run(console_factory);
    run(file_factory);

    console_factory->destroy(console_factory);
    file_factory->destroy(file_factory);

    while (1);
    return 0;
}

```

### 总结

>当你在编写代码的过程中， 如果无法预知对象确切类别及其依赖关系时， 可使用工厂方法。工厂方法将创建产品的代码与实际使用产品的代码分离， 从而能在不影响其他代码的情况下扩展产品创建部分代码。例如， 如果需要向应用中添加一种新产品， 你只需要开发新的创建者子类， 然后重写其工厂方法即可。

&emsp;&emsp;这里举的工厂模式的例子主要是体现工厂方法的特点，将行为定义在了对象中并向外暴露出了结构体，没有针对模块封装。实际使用过程中更好的方法是将方法定义在模块中，使用不透明指针只暴露接口函数，不暴露结构体本身。要实现上面功能只需要在中间定义一个中间层文件来放实际的抽象基类结构体并不暴露给client，并将方法定义成虚函数表，通过具体实现类中动态注入依赖实现多态。具体实现可以参考这篇文章[<关于工厂方法的补充>](https://nafx.top/archives/34189825.html)。
