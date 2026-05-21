---
categories: [""]
tags: ["Linux", "TrustZone" , "Qemu" , "OP-TEE"]
title: "None"
date: 2026-05-12
draft: True
---
# 动手编写你的第一个 OP-TEE 可信应用：以 hello_world 为例

在[上一篇文章](https://yoursite.com/qemu-optee-setup)中，我们一起在 QEMU 上搭建了 OP-TEE 的 TrustZone 仿真环境，并成功运行了 `xtest` 测试套件。这次我们将更进一步，学习如何编写自己的 Client Application（CA，普通世界应用）和 Trusted Application（TA，安全世界可信应用）。我们会以最经典的 `hello_world` 示例为蓝本，分析它的代码结构、编译流程和运行过程，并总结开发一个完整 TA/CA 需要遵循的模式。

## 1. CA 与 TA 的职责划分

回顾一下 TrustZone 的双世界模型：

- **Normal World（REE）**：运行 Linux（Buildroot），我们在这里执行 CA。CA 通过 GlobalPlatform TEE Client API（`TEEC_*` 函数）与安全世界通信。
- **Secure World（TEE）**：运行 OP-TEE OS，TA 在这里执行。TA 实现具体的敏感逻辑，通过 TEE Internal API（`TEE_*` 函数）访问安全资源。

一个典型的交互过程是：**CA 打开会话 → 发送命令 → TA 处理并返回结果 → 关闭会话**。`hello_world` 演示了最简化的流程：CA 传递一个整数，TA 将其加 1 后返回。

## 2. hello_world 源码解析

OP-TEE 的所有示例都放在 `<optee_root>/optee_examples/` 下，每个示例包含 `host/`（CA）和 `ta/`（TA）两个子目录，以及各自的 Makefile。

### 2.1 CA 端：`host/main.c`

```c
#include <tee_client_api.h>
#include <hello_world_ta.h>   // 定义 UUID 和命令号

int main(void)
{
    TEEC_Result res;
    TEEC_Context ctx;
    TEEC_Session sess;
    TEEC_Operation op;
    uint32_t err_origin;

    /* 1. 初始化 TEE 上下文 */
    res = TEEC_InitializeContext(NULL, &ctx);
    if (res != TEEC_SUCCESS)
        errx(1, "TEEC_InitializeContext failed");

    /* 2. 打开与 TA 的会话 */
    res = TEEC_OpenSession(&ctx, &sess, &TA_HELLO_WORLD_UUID,
                           TEEC_LOGIN_PUBLIC, NULL, NULL, &err_origin);
    if (res != TEEC_SUCCESS)
        errx(1, "TEEC_OpenSession failed");

    /* 3. 准备操作参数 */
    memset(&op, 0, sizeof(op));
    op.paramTypes = TEEC_PARAM_TYPES(TEEC_VALUE_INOUT, TEEC_NONE,
                                     TEEC_NONE, TEEC_NONE);
    op.params[0].value.a = 42;  // 传入 42，TA 会返回 43

    /* 4. 调用命令 */
    res = TEEC_InvokeCommand(&sess, TA_HELLO_WORLD_CMD_INC_VALUE, &op,
                             &err_origin);
    if (res != TEEC_SUCCESS)
        errx(1, "TEEC_InvokeCommand failed");

    printf("TA incremented value to %u\n", op.params[0].value.a);

    /* 5. 关闭会话并清理 */
    TEEC_CloseSession(&sess);
    TEEC_FinalizeContext(&ctx);
    return 0;
}
```

**关键点：**
- 必须包含 `tee_client_api.h`（提供 `TEEC_*` 函数）和自定义的 TA 头文件（提供 UUID 和命令号）。
- **UUID 唯一标识一个 TA**，CA 通过它找到对应的 TA。UUID 必须在 CA 和 TA 两端完全一致。
- 参数通过 `TEEC_Operation` 传递，可以是值（`TEEC_VALUE_INPUT/OUTPUT/INOUT`）或内存引用（`TEEC_MEMREF_TEMP_INPUT` 等）。
- 命令号（如 `TA_HELLO_WORLD_CMD_INC_VALUE`）用于告知 TA 执行哪个功能。

### 2.2 TA 端：`ta/hello_world_ta.c`

```c
#include <tee_internal_api.h>
#include <hello_world_ta.h>

/* 被 CA 调用时触发的命令处理函数 */
static TEE_Result inc_value(uint32_t param_types, TEE_Param params[4])
{
    uint32_t val = params[0].value.a;
    IMSG("Got value: %u from NW", val);
    val++;
    IMSG("Increase value to: %u", val);
    params[0].value.a = val;   // 写回 CA
    return TEE_SUCCESS;
}

TEE_Result TA_CreateEntryPoint(void)
{
    IMSG("TA has been created");  // TA 第一次实例化时打印
    return TEE_SUCCESS;
}

void TA_DestroyEntryPoint(void)
{
    IMSG("TA destroyed");        // TA 实例销毁时打印
}

TEE_Result TA_OpenSessionEntryPoint(uint32_t param_types,
                                    TEE_Param params[4], void **sess_ctx)
{
    IMSG("Hello World!");        // CA 打开会话时打印
    return TEE_SUCCESS;
}

void TA_CloseSessionEntryPoint(void *sess_ctx)
{
    IMSG("Goodbye!");            // CA 关闭会话时打印
}

TEE_Result TA_InvokeCommandEntryPoint(void *sess_ctx, uint32_t cmd_id,
                                      uint32_t param_types, TEE_Param params[4])
{
    switch (cmd_id) {
    case TA_HELLO_WORLD_CMD_INC_VALUE:
        return inc_value(param_types, params);
    default:
        return TEE_ERROR_BAD_PARAMETERS;
    }
}
```

**关键点：**
- TA 必须实现 5 个入口函数：`TA_CreateEntryPoint`、`TA_DestroyEntryPoint`、`TA_OpenSessionEntryPoint`、`TA_CloseSessionEntryPoint`、`TA_InvokeCommandEntryPoint`。OP-TEE OS 会在对应的生命周期阶段调用它们。
- `TA_InvokeCommandEntryPoint` 通过 `cmd_id` 分发到具体的业务函数（如 `inc_value`）。
- `IMSG` 是 TA 的打印宏，输出会显示在 Secure World 串口日志中，对调试非常有用。
- `params[0].value.a` 在这里既是输入也是输出，因为 CA 设置的是 `TEEC_VALUE_INOUT`。

### 2.3 头文件与 UUID：`ta/include/hello_world_ta.h`

```c
#define TA_HELLO_WORLD_UUID \
    { 0x8aaaf200, 0x2450, 0x11e4, \
      { 0xab, 0xe2, 0x00, 0x02, 0xa5, 0xd5, 0xc5, 0x1b } }

#define TA_HELLO_WORLD_CMD_INC_VALUE  0
```

- UUID 必须全局唯一，可以用 `uuidgen` 命令生成。
- 这个头文件被 CA 和 TA 共同包含，保证 UUID 和命令号一致。

## 3. 编译流程

在 `optee_examples` 的顶层 CMakeLists.txt 中已经包含了 `hello_world` 子目录，因此整个系统编译（`make run` 或 `make buildroot`）时会自动编译 CA 和 TA。

编译产物位置（主机上）：
- CA 可执行文件：`out-br/target/usr/bin/optee_example_hello_world`
- TA 二进制文件：`out-br/target/lib/optee_armtz/8aaaf200-....ta`

Buildroot 会将它们打包进根文件系统，最终在 QEMU 中看到：
- `/usr/bin/optee_example_hello_world`（CA）
- `/lib/optee_armtz/8aaaf200-....ta`（TA）

## 4. 运行与观察

启动 QEMU（`make run-only`），在 Normal World 登录 root 后执行：
```bash
# optee_example_hello_world
Invoking TA to increment 42
TA incremented value to 43
```

同时在 Secure World 串口可以看到 TA 的生命周期日志：
```
D/TA:  TA_CreateEntryPoint:18 has been called
D/TA:  __GP11_TA_OpenSessionEntryPoint:47 has been called
I/TA:  Hello World!
D/TA:  inc_value:78 has been called
I/TA:  Got value: 42 from NW
I/TA:  Increase value to: 43
D/TA:  TA_DestroyEntryPoint:29 has been called
I/TA:  Goodbye!
```

这清晰地展示了：CA 连接 → TA 创建并打开会话 → 处理命令 → 关闭会话并销毁 TA 的完整过程。

## 5. 开发自己的 TA：以 cal_add 为例

基于 `hello_world` 创建新项目 `cal_add` 的步骤可以总结为：

1. **复制模板**：`cp -r hello_world cal_add`
2. **修改 `host/main.c`**：替换 UUID 和命令号，实现自己的业务逻辑（例如接收两个数并返回它们的和）。
3. **修改 `ta/cal_add_ta.c`**：实现相加功能，并修改入口函数的打印信息。
4. **修改头文件**（`ta/include/cal_add_ta.h`）：定义新的 UUID 和命令号。
5. **修改 `user_ta_header_defines.h`**：将 `TA_UUID` 改为新 UUID，并更新描述字符串。
6. **在顶层 CMakeLists.txt 中加入新项目**：确保 `cal_add` 参与整体编译。
7. **重新编译 Buildroot**（`make buildroot`），然后启动 QEMU 测试。

过程中常见的错误：
- 头文件路径错误或文件名不匹配 → 编译时 `fatal error: xxx.h: No such file or directory`
- UUID 不一致 → 运行时找不到 TA 或会话打开失败
- 参数类型不匹配 → 导致 TA 收到的数据异常
- 头文件语法错误（如少写引号、反斜杠） → 编译失败，查看具体错误行修正

## 6. 小结

通过分析 `hello_world`，我们掌握了 CA 与 TA 编程的基本框架：

- **CA 侧**：使用 `TEEC_InitializeContext` → `TEEC_OpenSession` → `TEEC_InvokeCommand` → `TEEC_CloseSession` → `TEEC_FinalizeContext` 的标准流程。
- **TA 侧**：通过实现 5 个标准入口函数，在 `TA_InvokeCommandEntryPoint` 中分发命令。
- **编译集成**：利用 Buildroot 的 OP-TEE 示例包自动打包 CA/TA 到根文件系统。

TrustZone 的安全隔离使得普通世界无法直接访问 TA 的内存或代码，这种硬件级的保护正是 TEE 的核心价值。接下来你可以尝试实现更复杂的功能，例如安全存储、加密运算或指纹比对，继续探索安全世界的无限可能。

---

**参考资料**
- [OP-TEE Documentation - How to write a TA](https://optee.readthedocs.io/en/latest/building/trusted_applications.html)
- [GlobalPlatform TEE Client API Specification](https://globalplatform.org/specs-library/tee-client-api-specification/)
- [Hello World Example Source Code](https://github.com/OP-TEE/optee_examples/tree/master/hello_world)