---
title: 关于工厂方法的补充
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
abbrlink: '34189825'
date: 2025-12-09 22:52:56
---

#### 产品接口 logger.h

&emsp;&emsp;在产品接口中将方法封装到模块中，外面看不到Logger的具体细节。

```c
#ifndef LOGGER_H
#define LOGGER_H

typedef struct Logger Logger;

void logger_log(Logger *logger, const char *message);
void logger_destroy(Logger *logger);


#endif //LOGGER_H

```

#### 产品中间层 logger_internal.h

&emsp;&emsp;内部对象模型，Logger的真实定义在这里，这个文件不对外提供。

```c
#ifndef LOGGER_INTERNAL_H
#define LOGGER_INTERNAL_H

#include "logger.h"

/* 内部虚函数表 */
typedef struct LoggerVTable {
    void (*log)(Logger *, const char *);
    void (*destroy)(Logger *);
} LoggerVTable;

/* Logger 的真实定义 */
struct Logger {
    const LoggerVTable *vtable;
};

#endif

```

#### 产品外部接口实现 logger.c

```c
#include "logger_internal.h"

void logger_log(Logger *logger, const char *message) {
    if (logger && logger->vtable && logger->vtable->log) {
        logger->vtable->log(logger, message);
    }
}

void logger_destroy(Logger *logger) {
    if (logger && logger->vtable && logger->vtable->destroy) {
        logger->vtable->destroy(logger);
    }
}

```

#### 工厂接口 logger_factory.h

```c
#ifndef LOGGER_FACTORY_H
#define LOGGER_FACTORY_H

#include "logger.h"

typedef struct LoggerFactory LoggerFactory;

/* Factory Method */
Logger *logger_factory_create(LoggerFactory *factory);
void logger_factory_destroy(LoggerFactory *factory);

/* 工厂构造 */
LoggerFactory *console_logger_factory_create(void);
LoggerFactory *file_logger_factory_create(const char *path);

#endif/* LOGGER_FACTORY_H */

```

#### 工厂中间层 logger_factory_internal.h

```c
#ifndef LOGGER_FACTORY_INTERNAL_H
#define LOGGER_FACTORY_INTERNAL_H

#include "logger_factory.h"
/* 抽象 Factory 基类 */
struct LoggerFactory {
    Logger *(*create)(struct LoggerFactory *);
    void (*destroy)(struct LoggerFactory *);
};

#endif

```

#### 工厂外部接口实现 logger_factory.c

```c
#include <stdio.h>
#include "logger_factory_internal.h"

Logger *logger_factory_create(LoggerFactory *factory) {
    return factory ? factory->create(factory) : NULL;
}

void logger_factory_destroy(LoggerFactory *factory) {
    if (factory && factory->destroy) {
        factory->destroy(factory);
    }
}

```

#### 具体产品实现 console_logger.c

```c
#include <stdio.h>
#include <stdlib.h>
#include "logger_internal.h"
#include "logger_factory_internal.h"

/* ================= Concrete Product ================= */

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

static const LoggerVTable CONSOLE_VTABLE = {
    .log = console_log,
    .destroy = console_logger_destroy
};

static Logger *console_logger_create(void) {
    ConsoleLogger *logger = malloc(sizeof(*logger));
    if (!logger) return NULL;

    logger->base.vtable = &CONSOLE_VTABLE;
    return (Logger *)logger;
}

/* ================= Concrete Factory ================= */

typedef struct {
    struct LoggerFactory base;
} ConsoleLoggerFactory;

static Logger *console_factory_create(LoggerFactory *self) {
    (void)self;
    return console_logger_create();
}

static void console_factory_destroy(LoggerFactory *self) {
    free(self);
}

LoggerFactory *console_logger_factory_create(void) {
    ConsoleLoggerFactory *factory = malloc(sizeof(*factory));
    if (!factory) return NULL;

    factory->base.create  = console_factory_create;
    factory->base.destroy = console_factory_destroy;
    return (LoggerFactory *)factory;
}

```

#### 具体产品实现 file_logger.c

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include "logger_internal.h"
#include "logger_factory_internal.h"

/* ================= Concrete Product ================= */

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

static const LoggerVTable FILE_VTABLE = {
    .log = file_log,
    .destroy = file_logger_destroy
};

static Logger *file_logger_create(const char *path) {
    FileLogger *logger = malloc(sizeof(*logger));
    if (!logger) return NULL;

    logger->file = fopen(path, "a");
    if (!logger->file) {
        free(logger);
        return NULL;
    }

    logger->base.vtable = &FILE_VTABLE;
    return (Logger *)logger;
}

/* ================= Concrete Factory ================= */

typedef struct {
    struct LoggerFactory base;
    char *path;
} FileLoggerFactory;

static Logger *file_factory_create(LoggerFactory *self) {
    FileLoggerFactory *factory = (FileLoggerFactory *)self;
    return file_logger_create(factory->path);
}

static void file_factory_destroy(LoggerFactory *self) {
    FileLoggerFactory *factory = (FileLoggerFactory *)self;
    free(factory->path);
    free(factory);
}

LoggerFactory *file_logger_factory_create(const char *path) {
    FileLoggerFactory *factory = malloc(sizeof(*factory));
    if (!factory) return NULL;

    factory->path = strdup(path);
    if (!factory->path) {
        free(factory);
        return NULL;
    }

    factory->base.create  = file_factory_create;
    factory->base.destroy = file_factory_destroy;
    return (LoggerFactory *)factory;
}

```

#### 测试代码 main.c

```c
#include "logger_factory.h"

int main(void) {
    LoggerFactory *console_factory = console_logger_factory_create();
    LoggerFactory *file_factory    = file_logger_factory_create("app.log");

    Logger *console = logger_factory_create(console_factory);
    Logger *file    = logger_factory_create(file_factory);

    logger_log(console, "Hello Console Logger");
    logger_log(file,    "Hello File Logger");

    logger_destroy(console);
    logger_destroy(file);

    logger_factory_destroy(console_factory);
    logger_factory_destroy(file_factory);
    
    while (1);
    return 0;
}

```
