---
author: "李昌"
title: "静态代码检查: golangci-lint"
date: "2022-02-09"
tags: ["lint", "golang"]
categories: ["Golang"]
ShowToc: true
TocOpen: true
---

## 1. 简介
golangci-lint 是对golang进行静态代码检查的工具。其具有以下特性：
- 速度非常快：golangci-lint 是基于 gometalinter 开发的，但是平均速度要比 gometalinter 快 5 倍。golangci-lint 速度快的原因有三个：可以并行检查代码；可以复用 go build 缓存；会缓存分析结果。
- 可配置：支持 YAML 格式的配置文件，让检查更灵活，更可控。
- IDE 集成：可以集成进多个主流的 IDE，例如 VS Code、GNU Emacs、Sublime Text、Goland 等。
- linter 聚合器：1.41.1 版本的 golangci-lint 集成了 76 个 linter，不需要再单独安装这 76 个 linter。并且 golangci-lint 还支持自定义 linter。
- 最小的误报数：golangci-lint 调整了所集成 linter 的默认设置，大幅度减少了误报。
- 良好的输出：输出的结果带有颜色、代码行号和 linter 标识，易于查看和定位。

## 2. 安装
```bash
# 安装
go get github.com/golangci/golangci-lint/cmd/golangci-lint@v1.41.1

# 检查是否安装成功
golangci-lint version # 输出 golangci-lint 版本号，说明安装成功

golangci-lint has version v1.44.0 built from (unknown, mod sum: "h1:YJPouGNQEdK+x2KsCpWMIBy0q6MSuxHjkWMxJMNj/DU=") on (unknown)
```

## 3. 基本使用
对项目做静态检查，非常简单, 在项目跟目录：
```sh
golangci-lint run
```

等同于
```sh
golangci-lint run ./...
```

指定检查目录：
```sh
golangci-lint run dir1 dir2/... dir3/file1.go
```
这里需要注意的是，golangci-lint不会递归检查目录下的子目录，要想递归检查，可使用`dir/...`

## 4. linter
golangci-lint预定义了一些检查规则, 称为`linter`
```
golangci-lint help linters
```

```
Enabled by default linters:
deadcode: Finds unused code [fast: false, auto-fix: false]
errcheck: Errcheck is a program for checking for unchecked errors in go programs. These unchecked errors can be critical bugs in some cases [fast: false, auto-fix: false]
gosimple (megacheck): Linter for Go source code that specializes in simplifying a code [fast: false, auto-fix: false]
govet (vet, vetshadow): Vet examines Go source code and reports suspicious constructs, such as Printf calls whose arguments do not align with the format string [fast: false, auto-fix: false]
ineffassign: Detects when assignments to existing variables are not used [fast: true, auto-fix: false]
staticcheck (megacheck): Staticcheck is a go vet on steroids, applying a ton of static analysis checks [fast: false, auto-fix: false]
structcheck: Finds unused struct fields [fast: false, auto-fix: false]
...
```

你可以选择只使用某些linter(使用`-E/--enable`)或不使用某些linter(使用`-D/--disable`)

你还可以对这些linter进行更细粒度的配置，例如：
```yaml
linters-settings:
  bidichk:
    # The following configurations check for all mentioned invisible unicode runes.
    # All runes are enabled by default.
    left-to-right-embedding: false
    right-to-left-embedding: false
    pop-directional-formatting: false
    left-to-right-override: false
    right-to-left-override: false
    left-to-right-isolate: false
    right-to-left-isolate: false
    first-strong-isolate: false
    pop-directional-isolate: false
```
访问`https://golangci-lint.run/usage/linters/#errorlint`查看更多示例

## 5. 配置
配置文件的优先级低于命令行配置，使用文件配置后再次在命令行中指定配置，则命令行中指定的配置将覆盖文件中的配置。

配置文件名称：
- `.golangci.yml`
- `.golangci.yaml`
- `.golangci.toml`
- `.golangci.json`

可用的设置如下：
```yaml
# Options for analysis running.
run:
  # See the dedicated "run" documentation section.
  option: value
# output configuration options
output:
  # See the dedicated "output" documentation section.
  option: value
# All available settings of specific linters.
linters-settings:
  # See the dedicated "linters-settings" documentation section.
  option: value
linters:
  # See the dedicated "linters" documentation section.
  option: value
issues:
  # See the dedicated "issues" documentation section.
  option: value
severity:
  # See the dedicated "severity" documentation section.
  option: value
```

具体设置可查看文档`https://golangci-lint.run/usage/configuration`

## 6. 对某些代码忽略检查
1. 忽略某一行的检查
```golang
var bad_name int //nolint
```

2. 忽略某一行指定 linter 的检查，可以指定多个 linter，用逗号 , 隔开
```golang
var bad_name int //nolint:golint,unused
```

3. 忽略某个代码块的检查
```golang
//nolint
func allIssuesInThisFunctionAreExcluded() *string {
  // ...
}

//nolint:govet
var (
  a int
  b int
)
```

4. 忽略某个文件的指定linter检查
在 package xx 上面一行添加//nolint注释。
```golang
//nolint:unparam
package pkg
...
```

在使用 nolint 的过程中，有 3 个地方需要你注意。

首先，如果启用了 nolintlint，你就需要在//nolint后面添加 nolint 的原因// xxxx。

其次，你使用的应该是//nolint而不是// nolint。因为根据 Go 的规范，需要程序读取的注释 // 后面不应该有空格。

最后，如果要忽略所有 linter，可以用//nolint；如果要忽略某个指定的 linter，可以用//nolint:,。

## References  
1. https://golangci-lint.run/  
2. https://time.geekbang.org/column/article/390401