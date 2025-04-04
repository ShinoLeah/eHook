# eHook

## ✨ 介绍

- 快速、方便地构建你的 uprobe hook 模块。
- 提供一些方便的封装。

## 🚀 运行环境

- 目前仅支持 ARM64 架构的 Android 系统，需要 ROOT 权限，推荐搭配 [KernelSU](https://github.com/tiann/KernelSU) 使用
- 系统内核版本5.10+ （可执行`uname -r`查看）

## 💕 使用

在 `/user` 目录下工作

- 在 `config.go` 里设置目标信息，如：

  ```go
  const PackageName = "com.android.myapplication"
  const LibraryName = "libc.so"
  // The offset to which the onEnter function will be attached. 0 if not used.
  const Enter_Offset = 0xAFF44
  // The offset to which the onLeave function will be attached. 0 if not used.
  const Leave_Offset = 0x9C158
  ```

  - 可以在`LibraryName`中指定库的绝对路径。

- 在 `user.c` 中编写你的 eBPF 模块。可以使用给定的封装（详见“API”节），也可以用任意的 eBPF API 来做操作，详见 [eBPF Docs](https://docs.ebpf.io/)。

  ```c++
  struct data_t {
      int a;
      char b;
  };
  VARIABLES_POOL(data_t);
  
  static __always_inline void onEnter(struct pt_regs* ctx) {
      SET(a, 1);
      SET(b, 'c');
  }
  
  static __always_inline void onLeave(struct pt_regs* ctx) {
      char s = GET(b);
      LOG(&s, 1);
  }
  ```

- 如果需要，在 `listener.go` 中编写数据处理函数。

  ```go
  func OnEvent(cpu int, data []byte, perfmap *manager.PerfMap, manager *manager.Manager) {
  	// Write your data handler here
      fmt.Printf("%s\n", data)
  }
  ```

## 💭 API

- `VARIABLES_POOL` 定义全局变量池，可以将需要全局共享的变量或太大的变量放入这个池子。

  ```c++
  struct data_t {
      int a;
      char b;
  };
  VARIABLES_POOL(data_t);
  ```

- `GET(name)`：从全局变量池获取变量，等同于  data->name。

  ```c++
  int a = GET(a);
  __builtin_memcpy(GET(b), "xxxxx", 5);
  ```

- `SET(name, var)`：设置变量，等同与 data->name = var; 字符串变量请使用 GET 设置。

- `READ_KERN(x)`：读取非用户空间变量，如 `READ_KERN(ctx->regs[0])`。

- `READ(ptr, len)`：读指定的用户空间地址。

- `WRITE(ptr, content)`：写指定的用户空间地址（目标地址必须可写）。

- `LOG(char*, len)`：输出到控制台。

- `SUBMIT(void*, len)`：提交数据到 `listen.go` 的 `OnEvent` 函数。

## 🧑‍💻 示例

以下是一个绕过 adb 的 property 检查的简单示例：

```go
// config.go
const PackageName = "com.android.myapplication"
const LibraryName = "libc.so"
const Enter_Offset = 0xAFF44 //__system_property_get
const Leave_Offset = 0x9C158
```

```c++
// user.c
#include "include/eHook.h"

struct data_t {
    __u64 X1;
};
VARIABLES_POOL(data_t);

static __always_inline void onEnter(struct pt_regs* ctx) {
    // Do not modify the name of 'onEnter' 'onLeave' or 'ctx'
    __u64 prop_name_addr = READ_KERN(ctx->regs[0]);
    char* s = READ(prop_name_addr, 14);
    if(!__builtin_memcmp(s, "init.svc.adbd", 14)) {
        SET(X1, READ_KERN(ctx->regs[1]));
    } else {
        SET(X1, 0);
    }

}
static __always_inline void onLeave(struct pt_regs* ctx) {
    if(GET(X1) != 0) {
        LOG("modified.\n", 10);
        WRITE(GET(X1), "stopped");
    }
}
```

## ⚠️ 注意

- eBPF 函数编写有一些严格的限制，如不能使用 libc 等，请自行了解。
- `Packagename` 参数只用于定位库，如果 hook libc 等系统库将会对所有进程生效。

## 🛫 构建

1. 环境准备

   本项目在 x86 Linux 下交叉编译

   ```shell
   sudo apt-get update
   sudo apt-get install golang-1.18
   sudo apt-get install clang-15
   export GOPROXY=https://goproxy.cn,direct
   export GO111MODULE=on
   
   git clone --recursive https://github.com/ShinoLeah/eHook.git
   ./build_env.sh
   ```

2. 编译运行

   ```shell
   make
   ```

   可以在 Makefile 中指定项目名称，产物在 `bin/` 目录下.
   ```shell
   adb push bin/eHook_Untitled /data/local/tmp
   adb shell
   su
   cd /data/local/tmp
   chmod +x eHook_Untitled
   ./eHook_Untitled
   ```

## ❤️‍🩹 其他

- 喜欢的话可以点点右上角 Star 🌟
- 欢迎提出 Issue 或 PR！