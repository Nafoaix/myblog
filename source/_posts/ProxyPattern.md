---
title: 大话设计模式C语言之代理模式
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
abbrlink: 153619f2
date: 2025-11-30 14:43:12
---

## 大话设计模式C语言之代理模式

### 代理模式

>代理模式是一种结构型设计模式， 让你能够提供对象的替代品或其占位符。 代理控制着对于原对象的访问， 并允许在将请求提交给对象前后进行一些处理。

&emsp;&emsp;代理模式和上篇提到的同为结构型设计模式的装饰模式很像，尤其是用C实现结构几乎一致，但语义完全不同，这也是 C 里最容易被滥用的地方。为了阅读下面具体例子时不会混淆，这里先说下两者的本质区别，就是两者的`核心目的`不同：
- 上篇学习的装饰模式主要是为了给对象增加职责。在不改变接口的前提下叠加功能。
- 而代理模式主要是为了控制对对象的访问，决定能不能/何时/是否调用真实对象，下面结合具体例子很容易就可以理解。

### 案例代码

&emsp;&emsp;这里与原书不同举了一个简单的下载的例子，以便更好的体现代理可以控制对真实对象的访问，结合书中例子可以更好理解代理模式。

#### 服务接口 download_service.h

>服务接口（Service Interface）声明了服务接口。代理必须遵循该接口才能伪装成服务对象。

>该类定义了真实对象类和代理类的公用接口，这样就可以在任何使用真实对象的地方使用代理。

```c
#ifndef DOWNLOAD_SERVICE_H
#define DOWNLOAD_SERVICE_H

/* 前向声明 */
typedef struct DownloadService DownloadService;

/* 接口定义 */
struct DownloadService {
    void (*download)(DownloadService *self, const char *url);
    void (*destroy)(DownloadService *self);
};

#endif

```

#### 真正服务类 real_download.c

>服务（Service）类定义了proxy所代表的真实实体，提供了一些实用的业务逻辑。

```c
#include <stdio.h>
#include <stdlib.h>
#include "download_service.h"

/* 真实对象结构体 */
typedef struct {
    DownloadService base;
} RealDownloadService;

/* 真实下载逻辑 */
static void real_download(DownloadService *self, const char *url) {
    (void)self;
    printf("[Real] Downloading file from: %s\n", url);
}

/* 销毁真实对象 */
static void real_destroy(DownloadService *self) {
    free(self);
}

/* 工厂函数 */
DownloadService *real_download_service_create(void) {
    RealDownloadService *service = malloc(sizeof(RealDownloadService));
    if (!service) return NULL;

    service->base.download = real_download;
    service->base.destroy  = real_destroy;

    return (DownloadService *)service;
}

```

#### 代理类 proxy_download.c

>代理（Proxy）类包含一个指向服务对象的引用成员变量。代理完成其任务（例如延迟初始化、记录日志、访问控制和缓存等）后会将请求传递给服务对象。通常情况下，代理会对其服务对象的整个生命周期进行管理。

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include "download_service.h"

/* 代理对象结构体 */
typedef struct
{
    DownloadService base;
    DownloadService *real_service;
    int has_permission;
} DownloadServiceProxy;

/* 权限检查 */
static int check_permission(DownloadServiceProxy *proxy)
{
    return proxy->has_permission;
}

/* 代理下载 */
static void proxy_download(DownloadService *self, const char *url)
{
    DownloadServiceProxy *proxy = (DownloadServiceProxy *)self;

    printf("[Proxy] Request download: %s\n", url);

    if (!check_permission(proxy))
    {
        printf("[Proxy] Permission denied.\n");
        return;
    }

    proxy->real_service->download(proxy->real_service, url);
}

/* 销毁代理 */
static void proxy_destroy(DownloadService *self)
{
    DownloadServiceProxy *proxy = (DownloadServiceProxy *)self;

    if (proxy->real_service)
    {
        proxy->real_service->destroy(proxy->real_service);
    }

    free(proxy);
}

/* 代理工厂 */
DownloadService *download_service_proxy_create(
    DownloadService *real_service,
    int has_permission)
{
    DownloadServiceProxy *proxy = malloc(sizeof(DownloadServiceProxy));
    if (!proxy)
        return NULL;

    proxy->base.download = proxy_download;
    proxy->base.destroy = proxy_destroy;
    proxy->real_service = real_service;
    proxy->has_permission = has_permission;

    return (DownloadService *)proxy;
}

```

#### 测试代码 main.c

```c
#include <stdio.h>
#include "download_service.h"

/* 工厂函数声明 */
DownloadService *real_download_service_create(void);
DownloadService *download_service_proxy_create(
    DownloadService *real_service,
    int has_permission);

int main(void)
{

    int permission = 0;
    /* 创建真实服务 */
    DownloadService *real = real_download_service_create();
    printf("Enter 1 enable download permission, 0 disable download permission\n");
    scanf("%d", &permission);

    /* 用代理包装真实服务 */
    DownloadService *proxy = download_service_proxy_create(real, permission);

    /* 代理访问 */
    proxy->download(proxy, "https://example.com/file.zip");

    /* 统一销毁 */
    proxy->destroy(proxy);

    while (1);
    return 0;
}

```

### 总结

&emsp;&emsp;代理和装饰有着相似的结构，但是其意图却非常不同。这两个模式的构建都基于组合原则，也就是说一个对象应该将部分工作委派给另一个对象。看完上面具体例子后在这里再详细说下两者更多的区别。

|  | 代理模式 | 装饰模式 |
| :-----: | :------: | :-----: |
| 从结构上 | 代理模式通常只包一层 | 装饰模式通常都是多层嵌套 |
| 真实对象 | 代理可以不调用真实对象 | 每一层都会调用被装饰对象 |
| 生命周期 | 通常自行管理其服务对象的生命周期 | 装饰器通常伴随其装饰组件对象释放 |