---
id: languagesgolang
title: Golang
description: 这个文档讲述了 openRuyi 的 Golang 打包指南。
slug: /guide/packaging-guidelines/languages/Golang
---

# Golang 打包指南

这个文档讲述了 openRuyi 的 Golang 打包指南。对于相关的构建系统，请参阅 [Golang](/docs/guide/packaging-guidelines/BuildSystems/golang).

## Versioning

许多 Go 库 (以及部分 Go 程序) 是直接存在于版本控制系统中的，而从未发布过版本。关于版本号的编写，请参阅[版本号](/docs/guide/packaging-guidelines/Versioning)。

## 仅包含二进制的软件包

仅包含二进制的软件包是指包含由 Go 编写的程序，但不包含源代码的软件包。例如 `ollama`，它由 Go 编写，但不提供 API，因此不提供源码包供其它 Go 包引用。

### 命名

这类软件包的名称应与上游项目一致，例如 `ollama`。命名为 `go-ollama` 是错误的。同样，生成的二进制软件包名称中也禁止包含 `go-` 前缀。

## 仅包含库 (或二进制+库的) 软件包

在 openRuyi 内，打包 Go 编写的库的唯一目的是为了构建 openRuyi 系统中其它 Go 程序，不应该被用户在日常开发流程中使用。对于日常开发的情形，用户应该使用 `go get`。

### 命名

所有的 Go 库都有自己的导入路径 (import path)，我们将其的所有斜杠替换为横杠，并使用规范的标识符代替主机名来给软件包命名。

在理想的情况下，软件包最终的名称通常都是导入路径。以下是一些例子:

| 导入路径                     | openRuyi 软件包名称          |
| --------------------------- | -------------------------- |
| golang.org/x/oauth2         | go-golang-x-oauth2         |
| google.golang.org/protobuf  | go-google-protobuf         |
| github.com/jpillora/backoff | go-github-jpillora-backoff |
| git.sr.ht/\~sbinet/cmpimg   | go-sourcehut-sbinet-cmpimg |

### 文件存放位置

所有文件必须安装至 `/usr/share/gocode/src`，这对应于 `$GOPATH/src`。

例如，对于 `github.com/boombuler/barcode` 来说 (即 `go-github-boombuler-barcode`)，其中一个文件的路径应该为 `/usr/share/gocode/src/github.com/boombuler/barcode/code93/encoder.go`。

### 多版本的情况处理

Go 库在不同的大版本之间经常会有破坏性变更。通常来说，这能从 Go 库的导入路径看出来。如果出现需要打包相同 Go 库但不同版本的情况，需要在软件包名称后面跟导入路径一样添加大版本号。

例如，`github.com/cespare/xxhash` 对于 v1.x 和 v2.x 的 Go 库导入路径就不同:

* 前者在打包时可命名为 `go-github-cespare-xxhash` (即 `go(github.com/cespare/xxhash)`)。

* 后者在打包时可命名为 `go-github-cespare-xxhash-v2` (即 `go(github.com/cespare/xxhash/v2)`)。

当然，也有一些例外情况。这通常是上游在发布具有破坏性变更的大版本时，没有变更导入路径。

例如，`github.com/olekukonko/tablewriter` 存在 `v0.0.5` 的遗留 (legacy) 和最新 (latest) 两个版本，但它们共用一个导入路径。这种情况下，应该让最新的包使用导入名称，而遗留的包添加大版本号。

* 遗留版本在打包时可命名为 `go-github-olekukonko-tablewriter-v0`。

* 最新版本在打包时可命名为 `go-github-olekukonko-tablewriter`。

同时，因为导入路径相同，需要在 RPM Spec 内声明两者互斥，不可同时安装，例如最新版本需要加入:

```specfile
Conflicts:      go(github.com/olekukonko/tablewriter) = 0.0.5
```

而遗留版本需要加入:

```specfile
Conflicts:      go(github.com/olekukonko/tablewriter) >= 1.0.0
```

## 依赖关系

自动生成 `Provides` 不会生成 `BuildRequires` 或 `Requires`。库软件包必须继续在 RPM Spec 的 `BuildRequires` 和 `Requires` 中列出其依赖的所有其它 Go 库。这些依赖项在构建时需要可用以运行测试，在安装时也需要可用以便其它软件包进行构建。

声明 Go 库依赖时，必须使用规定的虚拟依赖格式: `go(<import path>)`。

对于最终安装源码树中成功发现的每个合格 Go 包，`%go_generate_provides` 会创建一个 `.goipath` 标记。生成器会排除名称为 `main` 的 Go 包、没有普通 Go、Cgo 或汇编源码文件的 Go 包、位于 `example` 或 `examples` 目录下的 Go 包，以及 `go list` 报告错误的 Go 包。RPM Go 文件属性生成器会读取最终生成的标记，并自动为拥有相应标记的 RPM 二进制软件包生成一个带版本的 `go(<import path>)` 虚拟提供。对于自动生成机制覆盖的规范导入路径，库软件包不得手动编写 `Provides` 条目。

只有生成器无法从打包的 `.goipath` 标记推导出的有意虚拟提供，才仍需手动编写 `Provides`。典型情况包括仅通过兼容性符号链接保留的旧导入路径别名，以及没有可导入 Go 包目录但需要有意提供的虚拟模块根。打包者必须确认手写的虚拟提供不会被自动重复生成。

其它库软件包可以用相同方式引用这些自动生成的虚拟提供:

```specfile
BuildRequires:  go(github.com/clipperhouse/stringish)
# ...
Requires:       go(github.com/clipperhouse/stringish)
```

### 必须的 BuildRequires

以下两个编译依赖是不可或缺的:

* `go` - 提供 Go 程序本身。

* `go-rpm-macros` - 提供打包 Go 库所需的 RPM 宏和 Go 文件属性生成器。例如，对于 `/usr/share/gocode/src` 这个路径，可以直接使用 `%{go_sys_gopath}` 代替。

## 当上游迁移仓库地址

有时，上游软件包会将仓库从一个托管服务商迁移到另一个托管服务商。此时，软件包必须按照新的规范导入路径重命名。如果现有使用者仍依赖旧导入路径，`%install` 阶段必须创建从旧路径指向规范路径的兼容性符号链接。自动生成器会为规范路径下可导入的 Go 包生成虚拟提供，但不会根据兼容性符号链接推导别名。因此，RPM Spec 必须为现有使用者仍然需要的每个旧导入路径兼容性别名手动声明带版本的 `Provides` 条目:

```specfile
# Preserve the legacy import path for existing consumers.
Provides:       go(example.org/legacy/project) = %{version}
```

## 自动化引入

openRuyi 目前有一个将 Go 库/程序引入的小程序 go2spec。这个小工具是基于 Debian Go Package Team 的 dh-make-golang 修改的，以适应 openRuyi 的需求。对此表示感谢。

例如，要打包 Go 库 `github.com/fsnotify/fsnotify`:

```bash
./go2spec pack github.com/fsnotify/fsnotify
```

`go2spec` 将会自动处理相关的信息 (目前仅限 GitHub)，随后会产出对应的 Spec 文件。

## 用于 Go 模块的 RPM 宏

这里包含了一些常用的用于 Go 模块的 RPM 宏。

### %go_common

此宏会做两件事情:

* 告诉 Go 忽略 `go.mod` 文件，让 Go 回到旧版的 GOPATH 模式。

* 定义 Go 的工作空间 (GOPATH) 路径，让 Go 把当前正在构建的项目源码和系统级 Go 依赖包的默认位置当作 GOPATH 的一部分。

### %__go

此宏为系统中安装的 go 可执行文件的名称。

### %go_generate_provides

该宏扫描最终安装源码树中的一个 Go 导入根，并为成功发现的每个合格 Go 包写入 `.goipath` 标记。该宏本身不会输出 `Provides` 行。在 RPM 依赖生成阶段，RPM Go 文件属性生成器会将每个打包进 RPM 的标记转换为带版本的 `go(<import path>)` 虚拟提供。

`%buildsystem_golangmodules_install` 会在复制源码树后自动调用该宏。无论是直接调用 `%buildsystem_golangmodules_install`，还是保留 `golangmodules` 的默认安装阶段，RPM Spec 都不得再为同一导入根调用 `%go_generate_provides`。

如果完整自定义的 `%install` 没有调用 `%buildsystem_golangmodules_install`，则必须先完成 Go 源码树的复制、移动和裁剪，再调用生成器:

```specfile
%go_generate_provides -i %{go_import_path} -v %{version}
```

调用生成器后，不得再添加或删除可导入的 Go 包，否则标记将无法反映最终打包的源码树。此后可以继续执行非 Go 源码的安装操作。如果自定义安装过程处理多个独立导入根，则必须为每个导入根分别调用一次该宏，并显式传入 `-i` 和 `-v` 参数。

#### go_submodules

默认情况下，每个自动生成的虚拟提供使用通过 `-v` 传入的版本。`%buildsystem_golangmodules_install` 会传入 `%{version}`。如果源码树包含独立发布且版本不同的嵌套 Go 模块，则必须在调用生成器前定义 `go_submodules`:

```specfile
# Record the independent versions of nested Go modules.
%global go_submodules %{expand:
# PATH VERSION
example.org/project/sub 2.3.4
example.org/project/sub/nested 3.4.5

# This module may be absent on some architectures.
example.org/project/archonly 4.5.6
}
```

对于每个自动生成的虚拟提供，路径上距离最近的已声明模块祖先决定其版本。当导入路径与 `PATH` 完全相同，或者以 `PATH/` 开头时，映射才会匹配。例如，`example.org/project/sub` 会匹配 `example.org/project/sub/child`，但不会匹配 `example.org/project/submarine`。表中各行的顺序不影响结果。没有匹配行的导入路径继续使用通过 `-v` 传入的版本。

该表必须符合以下要求:

* 每个非空、非注释行必须恰好包含 `PATH` 和 `VERSION` 两个字段，字段之间使用空格或制表符分隔。可以使用行首和行尾空白、空行，以及首个非空白字符为 `#` 的整行注释。不得使用行尾注释，因为注释内容会构成额外字段。
* 如果某个构建中不存在对应的 Go 包，有效映射可以不匹配任何路径。所有行仍必须在每个架构上保持有效。字段缺失或多余、展开后的路径重复、RPM 宏标记未解析，以及路径或版本无效，都会导致生成失败。
* `PATH` 支持 ASCII 字母、数字、`.`、`_`、`~`、`+`、`-` 和 `/`。路径不得以 `/` 开头或结尾，不得包含 `//`，也不得包含 `.` 或 `..` 路径组件。不支持通配符。
* `VERSION` 支持 ASCII 字母、数字、`.`、`_`、`+`、`~` 和 `^`，并且必须至少包含一个字母或数字。RPM Spec 必须仅填写 RPM `Version` 值，不得包含 `Epoch` 或 `Release` 后缀。例如，应使用 `2.3.4`，而不是 `2.3.4-1`。

### 定义宏

在使用以下宏的时候，我们希望能在定义上方编写一行注释说明意图。

#### go_test_exclude

如果需要在测试的时候屏蔽某个失败的库，可直接在 RPM Spec 顶部定义:

```specfile
%define go_test_exclude github.com/minio/c2goasm/cgocmp
```

如果需要屏蔽多个失败的库:

```specfile
%define go_test_exclude %{shrink:
    github.com/json-iterator/go/type_tests
    github.com/json-iterator/go/benchmarks
}
```

#### go_test_exclude_glob

如果需要在测试的时候通过正则屏蔽某类失败的库，可直接在 RPM Spec 顶部定义:

```specfile
%define go_test_exclude_glob github.com/pierrec/lz4/v4/cmd*
```

如果需要屏蔽多个失败的库:

```specfile
%define go_test_exclude_glob %{shrink:
    github.com/gogo/protobuf/protoc-gen-gogo
    github.com/gogo/protobuf/test/dashfilename
    github.com/gogo/protobuf/test/embedconflict
    github.com/gogo/protobuf/test/issue270
}
```

除此之外，还可以在测试 (`%check`) 配置内使用 `export GO_TEST_EXCLUDE_GLOB` 达到同样的效果。

#### go_test_include

如果需要在测试的时候只测试某个库，可直接在 RPM Spec 顶部定义:

```specfile
%define go_test_include github.com/cespare/xxhash/v2
```

不推荐使用本宏添加多个库用于测试，应考虑屏蔽测试失败的库。

#### go_test_include_glob

如果需要在测试的时候只测试某类库，可直接在 RPM Spec 顶部定义:

```specfile
%define go_test_include_glob github.com/cespare/xxhash/v2*
```

不推荐使用本宏添加多个库用于测试，应考虑屏蔽测试失败的库。

除此之外，还可以在测试 (`%check`) 配置内使用 `export GO_TEST_INCLUDE_GLOB` 达到同样的效果。

#### go_test_ignore_failure

如果出于某种原因，需要在测试不通过的情况下继续进行打包操作，可直接在 RPM Spec 顶部定义:

```specfile
%define go_test_ignore_failure 1
```

除此之外，还可以在测试 (`%check`) 配置内使用 `export GO_TEST_IGNORE_FAILURE` 达到同样的效果。
