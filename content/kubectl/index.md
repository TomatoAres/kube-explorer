# kubectl

> 本文基于 `kubernetes 1.17`

## 简介

`kubectl` 这个是使用 kubernetes 的伙伴几乎都知道的命令行工具，我们就简单介绍一下功能：

1. 命令行工具；
2. 和 Apiserver 通信，由于 apiserver 是一个极其规范的 `restful server`，kubectl 可以看作一个 `http client`
3. 使用最多，所有组件中代码量最少，最简单的组件，所以先学这个吧~~

## 入口

kubernetes 各种命令行工具入口都在 `cmd\` 下，kubectl 同理：`cmd\kubectl\kubectl.go`，这个代码量不大，但是启动核心逻辑都在其中，我就直接粘代码了

```go

package main

import (
    goflag "flag"
    "math/rand"
    "os"
    "time"

    "github.com/spf13/pflag"

    cliflag "k8s.io/component-base/cli/flag"
    // 日志包，需看一下
    "k8s.io/kubectl/pkg/util/logs"
    // 核心代码，我们的任务
    "k8s.io/kubernetes/pkg/kubectl/cmd"

    // 认证初始化——
    // 关键词：1.golang 的init 机制；2.k8s client-go
    _ "k8s.io/client-go/plugin/pkg/client/auth"
)

func main() {
    // 初始化随机生成器，由于底层随机生成器毕竟是伪随机，需要使用时间戳作为使其更合理
    rand.Seed(time.Now().UnixNano())

    // new kubectl 命令工具
    command := cmd.NewDefaultKubectlCommand()

    // 各种命令行包的爱恨交织，不关键，先跳过——挖坑待填
    // TODO: once we switch everything over to Cobra commands, we can go back to calling
    // cliflag.InitFlags() (by removing its pflag.Parse() call). For now, we have to set the
    // normalize func and add the go flag set by hand.
    pflag.CommandLine.SetNormalizeFunc(cliflag.WordSepNormalizeFunc)
    pflag.CommandLine.AddGoFlagSet(goflag.CommandLine)
    // cliflag.InitFlags()

    // 日志很关键，是一个合格程序必不可少的部分
    logs.InitLogs()
    defer logs.FlushLogs()

    // 启动了 ~~
    if err := command.Execute(); err != nil {
        os.Exit(1)
    }
}
```

流程：

1. 认证操作：带着账号进行操作
2. new 一个 kubectl
3. 启动

## 真·核心函数

kubernetes 真正核心代码基本都在 `pkg/` 目录下，我们需要研究的就是 `pkg/kubectl/`，tree 一下，你会发现：哇 怎么就这么点东西！？

```sh
cmd
├── auth
│   ├── auth.go                 // kubectl auth 命令定义
│   ├── cani.go                 // "Can I...", 检测是否具有执行某个动作的权限（鉴权
│   ├── cani_test.go
│   └── reconcile.go            // 协调RBAC角色、角色绑定、ClusterRole和ClusterRole绑定对象的规则
├── cmd.go                      // kubectl 主命令定义；基本 help，completion 命令定义
├── cmd_test.go
├── convert
│   ├── convert.go              // kubectl convert 命令定义
│   ├── convert_test.go
│   └── import_known_versions.go    // client 需要支持的 api 组
├── cp
│   ├── cp.go                       // kubectl cp 命令行定义
│   └── cp_test.go
└── profiling.go                    // 性能分析相关
```

我隐藏了部分不必要的文件，加上单元测试 总共也就这么些代码，是不是感觉很简单。既然这么简单我们从哪入手呢？~~

我个人习惯从之前的入口调用的 new 看起

由于代码量不大，我们其实可以。。
