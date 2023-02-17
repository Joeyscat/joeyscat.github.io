+++
title = "vscode 常用配置"
description = "文档中内嵌了数组时，如何更新数组中的元素"
date = 2022-03-13
+++

# C/C++项目配置

.vscode/c_cpp_properties.json

```json
{
    "configurations": [
        {
            "name": "Linux",        
            "includePath": [
                "${workspaceFolder}/**", // 头文件目录
                "am/include",
                "klib/include"
            ],
            "defines": [
                "__GUEST_ISA_=native", // 宏定义
                "__ISA__=native",
                "ITRACE_COND=true",
                "__DIFF_REF_QEMU__",
                "__ENGINE_interpreter__",
                "__ISA_native__"
            ],
            "compilerPath": "/usr/bin/gcc",
            "cStandard": "c17",
            "cppStandard": "c++17",
            "intelliSenseMode": "clang-x64"
        }
    ],
    "version": 4
}
```
