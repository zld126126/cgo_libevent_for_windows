# cgo-libevent-for-windows
- [cgo-libevent-for-windows](#cgo-libevent-for-windows)
	- [参考文章：](#参考文章)
	- [1.环境（windows）](#1环境windows)
	- [2.下载libevent源码 并制作dll动态库 利用mingw-utils生成.a .def](#2下载libevent源码-并制作dll动态库-利用mingw-utils生成a-def)
	- [3.测试代码：](#3测试代码)
	- [4.go build：](#4go-build)

## 参考文章：
https://www.kancloud.cn/idzqj/customer/2026575
https://www.cnblogs.com/pandamohist/p/14227440.html

## 1.环境（windows）
安装文件和具体文档都在setup文件夹中

    go 1.13
    libevent 2.1.10
    cmake 3.22
    mingw && mingw-utils
    visualstudio 2013/2017
    openssl

## 2.下载libevent源码 并制作dll动态库 利用mingw-utils生成.a .def
    根据libevent编译手册操作即可
    
mingw-utils命令：

        pexports libevent.dll > libevent.def
        dlltool.exe -D libevent.dll -d libevent.def -l libevent.a -k
        
        pexports libevent_core.dll > libevent_core.def
        dlltool.exe -D libevent_core.dll -d libevent_core.def -l libevent_core.a -k
        
        pexports libevent_extra.dll > libevent_extra.def
        dlltool.exe -D libevent_extra.dll -d libevent_extra.def -l libevent_extra.a -k
        
## 3.测试代码：
```
package main

/*
#cgo CFLAGS: -I ${SRCDIR}/include
// 不把lib的资源放lib文件夹的原因是启动的时候，会判断dll是否存在
#cgo LDFLAGS: -Wl,--allow-multiple-definition -L${SRCDIR} -levent -levent_core -levent_extra
#include <Windows.h>
#include <stdio.h> // for c.puts
#include <string.h>
#include <errno.h>
#include <stdio.h>
#include <signal.h>
#include <WinSock2.h>
#include <stdint.h>
#include "event.h"
#include "event2/bufferevent.h"
#include "event2/buffer.h"
#include "event2/listener.h"
#include "event2/util.h"

typedef void(*cb)(evutil_socket_t, short, void *);

struct event evg;
struct timeval tvg;

struct event* getEventPtr()  {
	return &evg;
}

struct timeval* getTvPtr()  {
	tvg.tv_sec = 1;
	tvg.tv_usec = 0;
	return &tvg;
}

void timer_cb(evutil_socket_t fd, short event , void* argc) {
	struct timeval tv;
	tv.tv_sec = 2;
	tv.tv_usec = 0;
	printf("timer wakeup\n");
	struct event* ev = (struct event*)argc;
	event_add(ev, &tv); // reschedule timer
}

void ev_set_timer(struct event* evt, cb c)
{
 	evtimer_set(evt, c, (void*)evt);
}
*/
import "C"
import (
	"fmt"
	"unsafe"
)

func main() {
	C.puts(C.CString("cgo-libevent-for-windows"))

	var pev *C.struct_event = C.getEventPtr()
	var ptv *C.struct_timeval = C.getTvPtr()
	var base *C.struct_event_base

	base   = C.event_base_new()  //*C.struct_event_base
	var pbase *C.struct_event_base  = (*C.struct_event_base)(unsafe.Pointer(base))

	C.ev_set_timer(pev, (C.cb)(unsafe.Pointer(C.timer_cb)))
	C.event_base_set(pbase, pev)
	C.event_add(pev, ptv)
	C.event_base_dispatch(pbase)

	C.event_base_free(pbase)

	Pause()
}

func Pause() {
	var str string
	fmt.Println("")
	fmt.Print("请按任意键继续...")
	fmt.Scanln(&str)
	fmt.Print("程序退出...")
}

```

## 4.go build：
常见错误：cgo_libevent_for_windows cc1.exe: sorry, unimplemented: 64-bit mode not compiled in

    windows命令行：
        SET CGO_ENABLED=1
        SET GOOS=windows
        SET GOARCH=386
        SET GO111MODULE=off
        go build -x
    
    goland编译器：
        goland-environment：CGO_ENABLED=1;GOOS=windows;GOARCH=386 