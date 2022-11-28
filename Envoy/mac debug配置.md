## 前需
编译器 cmake (envoy的一些依赖需要用cmake)
编译器 clang (envoy自身需要用clang)
编译组织者 bazel 
调试器 lldb
官网流程 https://github.com/envoyproxy/envoy/blob/main/bazel/README.md

## debug需要
### 编译了 dbg信息的二进制文件
`envoy`使用`bazel`系列工具构建,加上`-c dbg`可以使生成的文件里带有被debug工具识别的信息
```sh
bazel build envoy -c dbg --spawn_strategy=standalone --copt=-Wno-inconsistent-missing-override --genrule_strategy=standalone
```
>> 注: '--spawn_strategy=standalone --copt=-Wno-inconsistent-missing-override --genrule_strategy=standalone' 这三个参数是官方文档里没有的,但我不加这几个参数,无法实现编辑器到lldb的精确断点设置.
TODO 推敲这几个参数的具体作用

### debug工具
lldb

### vscode扩展
CodeLLDB 实现编辑器(vscode)交互与lldb的扩展(比如交互点击某一行,把这个事件转换成lldb里的断点设置)

#### 配置
.vscode/launch.json
```json
{
    "version": "0.2.0",
    "configurations": [
        {
            "name": "lldb //source/exe:envoy-static",
            "program": "/private/var/tmp/_bazel_fanhao/66f7c76c46cf5d6865a898d087a954c2/execroot/envoy/bazel-out/darwin-dbg/bin/source/exe/envoy-static", // 生成的带dbg的二进制文件的位置
            "sourceMap": {
                ".": "${workspaceFolder}", // 重要!尝试了各种sourceMap设置后,只有这个是简单直观且有效的将编辑器里的断点设置正确的传进了lldb工具
            },
            "cwd": "${workspaceFolder}",
            "args": [
                "--config-path",
                "${workspaceFolder}/configs/envoy-demo.yaml"
            ],
            "type": "lldb",
            "request": "launch"
        }
    ]
}
```