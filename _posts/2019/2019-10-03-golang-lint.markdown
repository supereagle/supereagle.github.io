---
layout:     post
title:      "让你的 Golang 代码更规范"
subtitle:   "Lint for Your Golang Code"
date:       2019-10-03
author:     "Robin"
header-img: "img/post-bg-2015.jpg"
catalog: true
tags:
    - Golang
---

代码规范除了让多人协作的项目的代码风格一致，甚至能够发现一些潜在的缺陷。因此，代码规范可以认为是代码开发的最佳实践，能够大大提高研发效率。
针对 Golang 语言，继 [Gometalinter](https://github.com/alecthomas/gometalinter) 不再维护之后，[GolangCI-Lint](https://github.com/golangci/golangci-lint) 成为最佳的代码规范检查工具。
本文将基于实例详细介绍一些最佳实践，让你的 Golang 代码更规范。

## GolangCI-Lint 最佳实践

基于 GolangCI-Lint 的使用经验，总结出如下最佳实践：

* 集成到 CI：通过 CI 反复地自动检查，禁止不规范代码合入。
* 使用固定版本：由于 GolangCI-Lint 自身一直在不停迭代，使用固定版本能够保证结果的确定性。
* 使用配置文件：GolangCI-Lint 通过配置文件提供大量灵活的配置，支持的配置参考 [.golangci.example.yml](https://github.com/golangci/golangci-lint/blob/master/.golangci.example.yml)。使用配置文件而不是命令行参数，方便进行版本控制。
* 设置合理 `GOGC`：通过设置合理的 `GOGC` 达到内存与时间消耗的平衡，设置指导参考 [Memory Usage of Golangci-lint](https://github.com/golangci/golangci-lint#memory-usage-of-golangci-lint)。

## Lint 常见问题及解决方案

> 如下演示的例子，基于 [supereagle/go-example/lint](https://github.com/supereagle/go-example/tree/master/lint) 源代码。

### 依赖包顺序

**错误提示：**

```shell
main.go:4: File is not `gofmt`-ed with `-s` (gofmt)
        "github.com/supereagle/go-example/lint/students"
make: *** [lint] Error 1
```

**解法方案：**

依赖包的顺序为：标准库包、第三方依赖包、本项目自定义包。执行 gofmt -s -w ${xxx.go} 自动修复。

```diff
diff --git a/lint/main.go b/lint/main.go
index eccd170..ad8ecf7 100644
--- a/lint/main.go
+++ b/lint/main.go
@@ -1,8 +1,9 @@
package main

import (
-       "github.com/supereagle/go-example/lint/students"
        "fmt"
+
+       "github.com/supereagle/go-example/lint/students"
)

func main() {
```

### 没有使用的代码

**错误提示：**

无用函数：

```shell
main.go:15:6: `unusedFunc` is unused (deadcode)
func unusedFunc() {
        ^
make: *** [lint] Error 1
```

无用结构体字段：

```shell
students/students.go:6:2: `sex` is unused (structcheck)
        sex  string
        ^
make: *** [lint] Error 1
```

**解法方案：**

```diff
diff --git a/lint/main.go b/lint/main.go
index 7cfa2b1..ad8ecf7 100644
--- a/lint/main.go
+++ b/lint/main.go
@@ -11,7 +11,3 @@ func main() {

        fmt.Printf("Student: name: %s, age: %d\n", s.GetName(), s.GetAge())
}
-
-func unusedFunc() {
-       fmt.Println("Remove me!")
-}
diff --git a/lint/students/students.go b/lint/students/students.go
index 19818c5..1a0d28d 100644
--- a/lint/students/students.go
+++ b/lint/students/students.go
@@ -3,7 +3,6 @@ package students
type Students struct {
        name string
        age  int
-       sex  string
}

func New(name string, age int) *Students {
```

### 单词拼写错误

**错误提示：**

```shell
main.go:12:20: `infromation` is a misspelling of `information` (misspell)
        // Output student infromation.
                        ^
make: *** [lint] Error 1
```

**解法方案：**

纠正拼写错误的单词。

```diff
diff --git a/lint/main.go b/lint/main.go
index 8b7651f..5c9621a 100644
--- a/lint/main.go
+++ b/lint/main.go
@@ -9,6 +9,6 @@ import (
func main() {
        s := students.New("Robin", 30)

-       // Output student infromation.
+       // Output student information.
        fmt.Printf("Student: name: %s, age: %d\n", s.GetName(), s.GetAge())
}
```

### 变量申明时类型重复

**错误提示：**

```shell
students/students.go:9:8: should omit type *Students from declaration of var s; it will be inferred from the right-hand side (golint)
        var s *Students = &Students{
                ^
make: *** [lint] Error 1
```

**解法方案：**

删除赋值表达式左边的类型。

```diff
diff --git a/lint/students/students.go b/lint/students/students.go
index 14bde64..a1e2b68 100644
--- a/lint/students/students.go
+++ b/lint/students/students.go
@@ -6,7 +6,7 @@ type Students struct {
}

func New(name string, age int) *Students {
-       var s *Students = &Students{
+       var s = &Students{
                name: name,
                age:  age,
        }
```

### 命名中带下划线

**错误提示：**

```shell
func (s *Students) Get_Name() string {
                ^
make: *** [lint] Error 1
```

**解法方案：**

采用驼峰格式命名。

```diff
diff --git a/lint/students/students.go b/lint/students/students.go
index 9f21a82..1a0d28d 100644
--- a/lint/students/students.go
+++ b/lint/students/students.go
@@ -12,7 +12,7 @@ func New(name string, age int) *Students {
        }
}

-func (s *Students) Get_Name() string {
+func (s *Students) GetName() string {
        return s.name
}
```

### 注释格式错误

**错误提示：**

```shell
main.go:9:1: comment on exported var `Mack` should be of the form `Mack ...` (golint)
// A smart boy.
^
make: *** [lint] Error 1
```

**解法方案：**

注释的第一个单词，应该为被注释的变量、方法、属性等的名字。

```diff
diff --git a/lint/main.go b/lint/main.go
index 7ce6ebd..10a1533 100644
--- a/lint/main.go
+++ b/lint/main.go
@@ -6,7 +6,7 @@ import (
        "github.com/supereagle/go-example/lint/students"
)

-// A smart boy.
+// Mack is a smart boy.
var Mack = students.New("Mack", 20)

func main() {
```

### 导出的函数返回未被导出的类型

**错误提示：**

```shell
students/students.go:8:32: exported func New returns unexported type *students.students, which can be annoying to use (golint)
func New(name string, age int) *students {
                                ^
make: *** [lint] Error 1
```

**解法方案：**

导出返回的类型。

```diff
diff --git a/lint/students/students.go b/lint/students/students.go
index 5c6b37a..1a0d28d 100644
--- a/lint/students/students.go
+++ b/lint/students/students.go
@@ -1,21 +1,21 @@
package students

-type students struct {
+type Students struct {
        name string
        age  int
}

-func New(name string, age int) *students {
-       return &students{
+func New(name string, age int) *Students {
+       return &Students{
                name: name,
                age:  age,
        }
}

-func (s *students) GetName() string {
+func (s *Students) GetName() string {
        return s.name
}

-func (s *students) GetAge() int {
+func (s *Students) GetAge() int {
        return s.age
}
```

### 冗余代码

**错误提示：**

```shell
main.go:14:2: S1023: redundant `return` statement (gosimple)
        return
        ^
make: *** [lint] Error 1
```

**解法方案：**

删除冗余代码。

```diff
diff --git a/lint/main.go b/lint/main.go
index 9c50634..5c9621a 100644
--- a/lint/main.go
+++ b/lint/main.go
@@ -11,5 +11,4 @@ func main() {

        // Output student information.
        fmt.Printf("Student: name: %s, age: %d\n", s.GetName(), s.GetAge())
-       return
}
```

### If 判断返回 bool 值

**错误提示：**

```shell
students/students.go:24:2: S1008: should use 'return <expr>' instead of 'if <expr> { return <bool> }; return <bool>' (gosimple)
        if s.age < 18 {
        ^
make: *** [lint] Error 1
```

**解法方案：**

直接返回 bool 表达式，不用再 if 判断。

```diff
diff --git a/lint/students/students.go b/lint/students/students.go
index cfb7b24..d1118f9 100644
--- a/lint/students/students.go
+++ b/lint/students/students.go
@@ -21,9 +21,5 @@ func (s *Students) GetAge() int {
}

func (s *Students) IsYoung() bool {
-       if s.age < 18 {
-               return true
-       }
-
-       return false
+       return s.age < 18
}
```

### 包名跟导出类型名的前缀相同

**错误提示：**

```shell
students/students.go:23:6: type name will be used as students.StudentsScore by other packages, and that stutters; consider calling this Score (golint)
type StudentsScore struct {
        ^
make: *** [lint] Error 1
```

**解法方案：**

方案一：去掉类型名中跟包名相同的前缀

```diff
diff --git a/lint/students/students.go b/lint/students/students.go
index 8733913..91ec74b 100644
--- a/lint/students/students.go
+++ b/lint/students/students.go
@@ -20,13 +20,13 @@ func (s *Students) GetAge() int {
        return s.age
}

-type StudentsScore struct {
+type Score struct {
        cource string
        score  int
}

-func NewStudentsScore(cource string, score int) *StudentsScore {
-       return &StudentsScore{
+func NewScore(cource string, score int) *Score {
+       return &Score{
                cource: cource,
                score:  score,
        }
```

方案二：修改包名，使其跟类型名的前缀不同

```diff
diff --git a/lint/main.go b/lint/main.go
index 5c9621a..02bf513 100644
--- a/lint/main.go
+++ b/lint/main.go
@@ -3,11 +3,11 @@ package main
import (
        "fmt"

-       "github.com/supereagle/go-example/lint/students"
+       "github.com/supereagle/go-example/lint/student"
)

func main() {
-       s := students.New("Robin", 30)
+       s := student.New("Robin", 30)

        // Output student information.
        fmt.Printf("Student: name: %s, age: %d\n", s.GetName(), s.GetAge())
diff --git a/lint/students/students.go b/lint/student/students.go
similarity index 99%
rename from lint/students/students.go
rename to lint/student/students.go
index 9cf942f4..fbe5b6d7 100644
--- a/lint/students/students.go
+++ b/lint/student/students.go
@@ -1,4 +1,4 @@
-package students
+package student

type Students struct {
        name string
```

## Reference

- [GolangCI-Lint](https://github.com/golangci/golangci-lint)
