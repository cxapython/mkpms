# KernelPatch Modules (KPM) 开发项目

基于 KernelPatch 框架的 Android 内核模块开发项目，用于开发可动态加载的内核模块 (KPM)。

## 项目特性

- 支持 ARM64 架构
- 动态加载/卸载内核模块
- 提供多种 Hook 机制（系统调用、内联 Hook、函数指针 Hook）
- 包含多个实用示例模块

## 目录结构

```
mkpms/
├── kernel/                  # KernelPatch 框架代码 (只读参考)
│   ├── base/                # hook、内存管理等基础功能
│   ├── include/             # 公共头文件
│   ├── patch/               # 补丁和模块加载
│   └── linux/               # Linux 内核头文件适配版本
├── kpms/                    # KPM 模块开发目录
│   ├── common/              # 公共辅助代码
│   ├── demo-hello/          # 基础示例模块
│   ├── demo-inlinehook/     # 内联 Hook 示例
│   ├── demo-syscallhook/    # 系统调用 Hook 示例
│   ├── hide-maps/           # 进程内存映射隐藏
│   ├── anti-detect/         # 模拟器检测绕过
│   └── wxshadow/            # W^X Shadow 隐藏断点模块
├── tools/kpatch/            # kpatch 工具
├── CMakeLists.txt           # 构建配置
└── hello.lds                # 链接脚本
```

## 环境要求

- **操作系统**: Linux / macOS
- **编译器**: `aarch64-linux-gnu-gcc` (ARM64 交叉编译器)
- **构建工具**: CMake 3.10+
- **目标设备**: 已安装 KernelPatch 或 APatch 的 Android 设备

### 安装交叉编译器

**Ubuntu/Debian:**
```bash
sudo apt install gcc-aarch64-linux-gnu
```

**macOS:**
```bash
brew install aarch64-elf-gcc
# 或使用 Android NDK 中的编译器
```

## 编译

### 完整编译

```bash
mkdir build && cd build
cmake -DCMAKE_C_COMPILER=aarch64-linux-gnu-gcc ..
make
```

### 编译单个模块

```bash
make wxshadow.kpm           # 编译 wxshadow KPM 模块
make wxshadow_client        # 编译 wxshadow 用户态客户端
make anti-detect.kpm        # 编译 anti-detect 模块
make hide-maps.kpm          # 编译 hide-maps 模块
```

### 清理

```bash
make clean                  # 清理所有
make clean-wxshadow         # 清理单个模块
```

## 模块说明

### 1. demo-hello

最基础的 KPM 模块示例，展示模块的基本结构。

**功能:**
- 模块加载/卸载日志
- 控制接口示例

**加载:**
```bash
kpatch <superkey> kpm load /data/local/tmp/hello.kpm
```

### 2. demo-inlinehook

内联 Hook 示例，展示如何 Hook 内核函数。

**功能:**
- Hook 内核函数 `add()`
- 在函数调用前后执行自定义代码
- 修改函数返回值

**关键代码:**
```c
hook_wrap2((void *)add, before_add, after_add, 0);
```

### 3. demo-syscallhook

系统调用 Hook 示例，展示如何拦截和监控系统调用。

**功能:**
- Hook `openat` 系统调用
- 支持函数指针 Hook 和内联 Hook 两种方式
- 记录进程打开文件的详细信息

**加载:**
```bash
# 函数指针方式 Hook
kpatch <superkey> kpm load /data/local/tmp/syscallhook.kpm "function_pointer_hook"

# 内联方式 Hook
kpatch <superkey> kpm load /data/local/tmp/syscallhook.kpm "inline_hook"
```

### 4. hide-maps

隐藏进程内存映射，用于隐藏特定内存区域。

**功能:**
- Hook `show_map_vma` 函数
- 过滤 `/proc/pid/maps` 中包含特定字符串的条目
- 隐藏指定内存区域不被检测

**使用场景:**
- 隐藏注入的内存区域
- 绕过内存扫描检测

### 5. anti-detect

模拟器检测绕过模块，隐藏模拟器特征文件。

**功能:**
- 拦截 `faccessat`、`fstatat`、`statx`、`readlinkat` 等系统调用
- 过滤 `getdents64` 返回的目录列表
- 隐藏包含 `goldfish_` 等模拟器特征的文件
- 保护 KernelPatch supercall 接口

**Hook 列表:**
| 系统调用 | 处理方式 |
|---------|---------|
| `faccessat` | 返回 ENOENT |
| `faccessat2` | 返回 ENOENT |
| `fstatat` | 返回 ENOENT |
| `statx` | 返回 ENOENT |
| `readlinkat` | 返回 ENOENT |
| `getdents64` | 过滤隐藏条目 |

**加载:**
```bash
kpatch <superkey> kpm load /data/local/tmp/anti-detect.kpm "<superkey>"
```

### 6. wxshadow

**W^X Shadow Memory** - 基于 shadow page 技术的隐藏断点/Patch 模块。

**核心原理:**
- 使用 shadow 页面技术实现代码隐藏断点
- 进程读取代码时看到原始内容
- 执行时触发断点或自定义 patch
- 支持 CRC 校验绕过

**页面状态机:**
```
NONE → SHADOW_X(--x) ↔ ORIGINAL(r--) ↔ STEPPING(r-x)
                    ↘ DORMANT (hook 退休，保留 shadow)
```

**prctl 接口:**
| 命令 | 值 | 说明 |
|------|------|------|
| `PR_WXSHADOW_SET_BP` | `0x57580001` | 设置隐藏断点 |
| `PR_WXSHADOW_SET_REG` | `0x57580002` | 配置寄存器修改 |
| `PR_WXSHADOW_DEL_BP` | `0x57580003` | 删除断点 |
| `PR_WXSHADOW_PATCH` | `0x57580006` | 自定义 patch |
| `PR_WXSHADOW_RELEASE` | `0x57580008` | 释放 shadow |

**客户端使用:**
```bash
# 查看目标进程可执行区域
./wxshadow_client -p <pid> -m

# 设置断点
./wxshadow_client -p <pid> -a 0x7b5c001234

# 按库名+偏移设置断点
./wxshadow_client -p <pid> -b libc.so -o 0x12345

# 设置断点并修改寄存器
./wxshadow_client -p <pid> -a 0x7b5c001234 -r x0=0 -r x1=0x100

# 删除断点
./wxshadow_client -p <pid> -a 0x7b5c001234 -d

# 自定义 patch (NOP)
./wxshadow_client -p <pid> -a 0x7b5c001234 --patch d503201f

# 释放 shadow
./wxshadow_client -p <pid> -a 0x7b5c001234 --release
```

## 开发指南

### 模块模板

```c
#include <kpmodule.h>
#include <linux/printk.h>

KPM_MODULE_INFO("module-name", "1.0.0", "GPL v2", "author", "description");

static long module_init(const char *args, const char *event, void *reserved) {
    pr_info("module loaded\n");
    return 0;
}

static long module_exit(void *reserved) {
    pr_info("module unloaded\n");
    return 0;
}

KPM_INIT(module_init);
KPM_EXIT(module_exit);
```

### 常用 API

```c
// 符号查找
void *addr = kallsyms_lookup_name("symbol_name");

// 内联 Hook
hook_wrap2(func, before_callback, after_callback, user_data);
unhook(func);

// 系统调用 Hook
hook_syscalln(syscall_nr, narg, before, after, user_data);
unhook_syscalln(syscall_nr, before, after);

// 日志输出
pr_info("info message\n");
pr_err("error message\n");
pr_warn("warning message\n");

// 用户态内存访问
compat_strncpy_from_user(kernel_buf, user_ptr, size);
compat_copy_to_user(user_ptr, kernel_buf, size);
```

### CMakeLists.txt 配置

每个模块需要创建 `CMakeLists.txt`:

```cmake
add_kpm_module(module_name SOURCES source1.c source2.c)

# 如果需要编译用户态客户端
if(DEFINED CMAKE_C_COMPILER_NATIVE)
    add_executable(module_name_client client.c)
endif()
```

## 部署与使用

### 推送文件到设备

```bash
adb push build/kpms/module_name/module_name.kpm /data/local/tmp/
adb shell chmod 644 /data/local/tmp/module_name.kpm
```

### 加载模块

```bash
# KernelPatch
kpatch <superkey> kpm load /data/local/tmp/module_name.kpm

# APatch
# 通过 APatch 管理器加载
```

### 查看日志

```bash
adb shell dmesg | grep kpm
adb shell dmesg | grep module_name
```

### 卸载模块

```bash
kpatch <superkey> kpm unload module_name
```

## 注意事项

1. **内核版本兼容性**: 不同内核版本的 `task_struct`、`mm_struct` 等结构偏移量可能不同，部分模块需要动态扫描
2. **`kernel/` 目录只读**: 该目录为框架参考代码，请勿修改
3. **Hook 清理**: 模块卸载时必须 unhook 所有钩子，否则会导致内核崩溃
4. **内存安全**: 注意 `kmalloc`/`kfree` 配对使用，避免内存泄漏
5. **锁机制**: 多线程环境下需要使用自旋锁保护共享数据

## 技术架构

### Hook 机制

| 类型 | 说明 | 适用场景 |
|-----|------|---------|
| Inline Hook | 修改函数入口指令 | 内核函数 Hook |
| Function Pointer Hook | 替换函数指针表 | 系统调用 Hook |
| Syscall Hook | 封装的系统调用 Hook | 快速系统调用拦截 |

### 内存管理

```c
// 页面分配
unsigned long pages = __get_free_pages(GFP_KERNEL, order);
free_pages(pages, order);

// 内核内存
void *p = kzalloc(size, GFP_KERNEL);
kfree(p);
```

### 进程信息

```c
#include <asm/current.h>
#include <linux/sched.h>

struct task_struct *task = current;
pid_t pid = task_pid_vnr(task);
const char *comm = get_task_comm(task);
```

## 相关链接

- [KernelPatch 项目](https://github.com/bmax121/KernelPatch)
- [APatch 项目](https://github.com/bmax121/APatch)
- [配合 RustFrida 实现 Java/Native Hook](https://bbs.kanxue.com/thread-290304.htm)

## 交流群

![wxshadow 微信群](image.png)

## License

本项目采用 [GPL v3](LICENSE) 许可证。任何使用、修改或分发本项目代码的衍生作品必须在相同许可证下开源。

## 免责声明

本项目仅供安全研究与学习交流使用，**严禁用于任何非法用途**。使用者应遵守所在地区的法律法规，因使用本项目产生的一切后果由使用者自行承担，与项目作者无关。
