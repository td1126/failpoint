# failpoint
[![LICENSE](https://img.shields.io/github/license/pingcap/failpoint.svg)](https://github.com/pingcap/failpoint/blob/master/LICENSE)
[![Language](https://img.shields.io/badge/Language-Go-blue.svg)](https://golang.org/)
[![Go Report Card](https://goreportcard.com/badge/github.com/pingcap/failpoint)](https://goreportcard.com/report/github.com/pingcap/failpoint)
[![Build Status](https://github.com/pingcap/failpoint/actions/workflows/suite.yml/badge.svg?branch=master)](https://github.com/pingcap/failpoint/actions/workflows/suite.yml?query=event%3Apush+branch%3Amaster)
[![Coverage Status](https://codecov.io/gh/pingcap/failpoint/branch/master/graph/badge.svg)](https://codecov.io/gh/pingcap/failpoint)
[![Mentioned in Awesome Go](https://awesome.re/mentioned-badge.svg)](https://github.com/avelino/awesome-go)  

An implementation of [failpoints][failpoint] for Golang. Fail points are used to add code points where errors may be injected in a user controlled fashion. Fail point is a code snippet that is only executed when the corresponding failpoint is active.
Golang 的 [failpoints][failpoint] 的实现。故障点用于添加可能以用户控制的方式注入错误的代码点。故障点是仅在相应故障点处于活动状态时才执行的代码片段。

[failpoint]: http://www.freebsd.org/cgi/man.cgi?query=fail

## Quick Start

1.  Build `failpoint-ctl` from source

    ``` bash
    git clone https://github.com/pingcap/failpoint.git
    cd failpoint
    make
    ls bin/failpoint-ctl
    ```

2.  Inject failpoints to your program, eg:

    ``` go
    package main

    import "github.com/pingcap/failpoint"

    func main() {
        failpoint.Inject("testPanic", func() {
            panic("failpoint triggerd")
        })
    }
    ```

3.  Transfrom your code with `failpoint-ctl enable`

4.  Build with `go build`

5.  Enable failpoints with `GO_FAILPOINTS` environment variable

    ``` bash
    GO_FAILPOINTS="main/testPanic=return(true)" ./your-program
    ```

6.  If you use `go run` to run the test, don't forget to add the generated `binding__failpoint_binding__.go` in your command, like:

    ```bash
    GO_FAILPOINTS="main/testPanic=return(true)" go run your-program.go binding__failpoint_binding__.go
    ```

## Design principles
## 设计原则

- Define failpoint in valid Golang code, not comments or anything else
- Failpoint does not have any extra cost
- 在有效的 Golang 代码中定义故障点，而不是注释或其他任何内容
- Failpoint 没有任何额外费用

    - Will not take effect on regular logic
    - Will not cause regular code performance regression
    - Failpoint code will not appear in the final binary
    - 不会对常规逻辑生效
    - 不会导致常规代码性能降低
    - 故障点代码不会出现在最终的二进制文件中

- Failpoint routine is writable/readable and should be checked by a compiler
- Generated code by failpoint definition is easy to read
- Keep the line numbers same with the injecting codes(easier to debug)
- Support parallel tests with context.Context
- 故障点例程是可写/可读的，应该由编译器检查
- 由故障点定义生成的代码易于阅读
- 保持与注入代码相同的行号（更易于调试）
- 支持带有 context.Context 的并行测试

## Key concepts
## 关键概念

- Failpoint
- 故障点

    Faillpoint is a code snippet that is only executed when the corresponding failpoint is active.
    The closure will never be executed if `failpoint.Disable("failpoint-name-for-demo")` is executed.
    故障点是一个代码片段，只有在相应的故障点处于活动状态时才会执行。
    如果执行了 `failpoint.Disable("failpoint-name-for-demo")`，则永远不会执行闭包。

    ```go
    var outerVar = "declare in outer scope"
    failpoint.Inject("failpoint-name-for-demo", func(val failpoint.Value) {
        fmt.Println("unit-test", val, outerVar)
    })
    ```

- Marker functions
- 标记功能

    - It is just an empty function
    - 它只是一个空函数

        - To hint the rewriter to rewrite with an equality statement
        - To receive some parameters as the rewrite rule
        - It will be inline in the compiling time and emit nothing to binary (zero cost)
        - The variables in external scope can be accessed in closure by capturing, and the converted code is still legal
        because all the captured-variables location in outer scope of IF statement.
        - 提示重写器用相等语句重写
        - 接收一些参数作为重写规则
        - 它将在编译时内联，并且不会向二进制发送任何内容（零成本）
        - 外部作用域中的变量可以通过捕获在闭包中访问，并且转换后的代码仍然是合法的，因为所有捕获的变量都位于 IF 语句的外部作用域中。

    - It is easy to write/read 
    - Introduce a compiler check for failpoints which cannot compile in the regular mode if failpoint code is invalid
    - 易于写/读
    - 如果故障点代码无效，则对无法在常规模式下编译的故障点引入编译器检查

- Marker funtion list
- 标记功能列表

    - `func Inject(fpname string, fpblock func(val Value)) {}`
    - `func InjectContext(fpname string, ctx context.Context, fpblock func(val Value)) {}`
    - `func Break(label ...string) {}`
    - `func Goto(label string) {}`
    - `func Continue(label ...string) {}`
    - `func Fallthrough() {}`
    - `func Return(results ...interface{}) {}`
    - `func Label(label string) {}`

- Supported failpoint environment variable
- 支持的故障点环境变量

    failpoint can be enabled by export environment variables with the following patten, which is quite similar to [freebsd failpoint SYSCTL VARIABLES](https://www.freebsd.org/cgi/man.cgi?query=fail)
    failpoint 可以通过导出环境变量启用，模式如下，与 [freebsd failpoint SYSCTL VARIABLES](https://www.freebsd.org/cgi/man.cgi?query=fail) 非常相似

    ```regexp
    [<percent>%][<count>*]<type>[(args...)][-><more terms>]
    ```

    The <type> argument specifies which action to take; it can be one of:
    <type> 参数指定要采取的行动；它可以是以下之一：

    - off: Take no action (does not trigger failpoint code)
    - return: Trigger failpoint with specified argument
    - sleep: Sleep the specified number of milliseconds
    - panic: Panic
    - break: Execute gdb and break into debugger
    - print: Print failpoint path for inject variable
    - pause: Pause will pause until the failpoint is disabled
    - off：不采取任何行动（不触发故障点代码）
    - return：使用指定参数触发故障点
    - sleep：休眠指定的毫秒数
    - panic：恐慌
    - break：执行 gdb 并进入调试器
    - print：打印注入变量的故障点路径
    - pause：暂停将暂停，直到故障点被禁用
  
## How to inject a failpoint to your program
## 如何向程序注入故障点

- You can call `failpoint.Inject` to inject a failpoint to the call site, where `failpoint-name` is
used to trigger the failpoint and `failpoint-closure` will be expanded as the body of the IF statement.
- 您可以调用 `failpoint.Inject` 向调用站点注入故障点，其中 `failpoint-name` 用于触发故障点，而 `failpoint-closure` 将扩展为 IF 语句的主体。

    ```go
    failpoint.Inject("failpoint-name", func(val failpoint.Value) {
        failpoint.Return("unit-test", val)
    })
    ```

    The converted code looks like:
    转换后的代码如下所示：

    ```go
    if val, _err_ := failpoint.Eval(_curpkg_("failpoint-name")); _err_ == nil {
        return "unit-test", val
    }
    ```

- `failpoint.Value` is the value that passes by `failpoint.Enable("failpoint-name", "return(5)")`
which can be ignored.
- `failpoint.Value` 是通过 `failpoint.Enable("failpoint-name", "return(5)")` 传递的值，可以忽略。

    ```go
    failpoint.Inject("failpoint-name", func(_ failpoint.Value) {
        fmt.Println("unit-test")
    })
    ```

    OR

    ```go
    failpoint.Inject("failpoint-name", func() {
        fmt.Println("unit-test")
    })
    ```

    And the converted code looks like:

    ```go
    if _, _err_ := failpoint.Eval(_curpkg_("failpoint-name")); _err_ == nil {
        fmt.Println("unit-test")
    }
    ```

- Also, the failpoint closure can be a function which takes `context.Context`. You can
do some customized things with `context.Context` like controlling whether a failpoint is
active in parallel tests or other cases. For example,
- 此外，故障点闭包可以是一个采用 `context.Context` 的函数。您可以使用 `context.Context` 做一些自定义的事情，
例如控制故障点在并行测试或其他情况下是否处于活动状态。例如，

    ```go
    failpoint.InjectContext(ctx, "failpoint-name", func(val failpoint.Value) {
        fmt.Println("unit-test", val)
    })
    ```

    The converted code looks like:

    ```go
    if val, _err_ := failpoint.EvalContext(ctx, _curpkg_("failpoint-name")); _err_ == nil {
        fmt.Println("unit-test", val)
    }
    ```

- You can ignore `context.Context`, and this will generate the same code as above non-context version. For example,
- 您可以忽略`context.Context`，这将生成与上述非上下文版本相同的代码。例如，

    ```go
    failpoint.InjectContext(nil, "failpoint-name", func(val failpoint.Value) {
        fmt.Println("unit-test", val)
    })
    ```

    Becomes

    ```go
    if val, _err_ := failpoint.EvalContext(nil, _curpkg_("failpoint-name")); _err_ == nil {
        fmt.Println("unit-test", val)
    }
    ```

- You can control a failpoint by failpoint.WithHook
- 您可以通过 failpoint.WithHook 控制故障点

    ```go
    func (s *dmlSuite) TestCRUDParallel() {
        sctx := failpoint.WithHook(context.Backgroud(), func(ctx context.Context, fpname string) bool {
            return ctx.Value(fpname) != nil // Determine by ctx key
        })
        insertFailpoints = map[string]struct{} {
            "insert-record-fp": {},
            "insert-index-fp": {},
            "on-duplicate-fp": {},
        }
        ictx := failpoint.WithHook(context.Backgroud(), func(ctx context.Context, fpname string) bool {
            _, found := insertFailpoints[fpname] // Only enables some failpoints.
            return found
        })
        deleteFailpoints = map[string]struct{} {
            "tikv-is-busy-fp": {},
            "fetch-tso-timeout": {},
        }
        dctx := failpoint.WithHook(context.Backgroud(), func(ctx context.Context, fpname string) bool {
            _, found := deleteFailpoints[fpname] // Only disables failpoints. 
            return !found
        })
        // other DML parallel test cases.
        s.RunParallel(buildSelectTests(sctx))
        s.RunParallel(buildInsertTests(ictx))
        s.RunParallel(buildDeleteTests(dctx))
    }
    ```

- If you use a failpoint in the loop context, maybe you will use other marker functions.
- 如果您在循环上下文中使用故障点，也许您将使用其他标记函数。

    ```go
    failpoint.Label("outer")
    for i := 0; i < 100; i++ {
        inner:
            for j := 0; j < 1000; j++ {
                switch rand.Intn(j) + i {
                case j / 5:
                    failpoint.Break()
                case j / 7:
                    failpoint.Continue("outer")
                case j / 9:
                    failpoint.Fallthrough()
                case j / 10:
                    failpoint.Goto("outer")
                default:
                    failpoint.Inject("failpoint-name", func(val failpoint.Value) {
                        fmt.Println("unit-test", val.(int))
                        if val == j/11 {
                            failpoint.Break("inner")
                        } else {
                            failpoint.Goto("outer")
                        }
                    })
            }
        }
    }
    ```

    The above code block will generate the following code:
    上述代码块将生成以下代码：

    ```go
    outer:
        for i := 0; i < 100; i++ {
        inner:
            for j := 0; j < 1000; j++ {
                switch rand.Intn(j) + i {
                case j / 5:
                    break
                case j / 7:
                    continue outer
                case j / 9:
                    fallthrough
                case j / 10:
                    goto outer
                default:
                    if val, _err_ := failpoint.Eval(_curpkg_("failpoint-name")); _err_ == nil {
                        fmt.Println("unit-test", val.(int))
                        if val == j/11 {
                            break inner
                        } else {
                            goto outer
                        }
                    }
                }
            }
        }
    ```

- You may doubt why we do not use `label`, `break`, `continue`, and `fallthrough` directly
instead of using failpoint marker functions. 
- 你可能会怀疑为什么我们不直接使用 `label`、`break`、`continue` 和 `fallthrough` 而不是使用故障点标记函数。

    - Any unused symbol like an ident or a label is not permitted in Golang. It will be invalid if some
    label is only used in the failpoint closure. For example,
    - Golang 中不允许使用任何未使用的符号，如标识或标签。如果某个标签仅在故障点闭包中使用，它将是无效的。例如，
    
        ```go
        label1: // compiler error: unused label1
            failpoint.Inject("failpoint-name", func(val failpoint.Value) {
                if val.(int) == 1000 {
                    goto label1 // illegal to use goto here
                }
                fmt.Println("unit-test", val)
            })
        ```

    - `break` and `continue` can only be used in the loop context, which is not legal in the Golang code 
    if we use them in closure directly.
    - `break` 和 `continue` 只能在循环上下文中使用，如果我们直接在闭包中使用它们，这在 Golang 代码中是不合法的。

### Some complicated failpoints demo
### 一些复杂的故障点演示

- Inject a failpoint to the IF INITIAL statement or CONDITIONAL expression

    ```go
    if a, b := func() {
        failpoint.Inject("failpoint-name", func(val failpoint.Value) {
            fmt.Println("unit-test", val)
        })
    }, func() int { return rand.Intn(200) }(); b > func() int {
        failpoint.Inject("failpoint-name", func(val failpoint.Value) int {
            return val.(int)
        })
        return rand.Intn(3000)
    }() && b < func() int {
        failpoint.Inject("failpoint-name-2", func(val failpoint.Value) {
            return rand.Intn(val.(int))
        })
        return rand.Intn(6000)
    }() {
        a()
        failpoint.Inject("failpoint-name-3", func(val failpoint.Value) {
            fmt.Println("unit-test", val)
        })
    }
    ```

    The above code block will generate something like this:

    ```go
    if a, b := func() {
        if val, _err_ := failpoint.Eval(_curpkg_("failpoint-name")); _err_ == nil {
            fmt.Println("unit-test", val)
        }
    }, func() int { return rand.Intn(200) }(); b > func() int {
        if val, _err_ := failpoint.Eval(_curpkg_("failpoint-name")); _err_ == nil {
            return val.(int)
        }
        return rand.Intn(3000)
    }() && b < func() int {
        if val, ok := failpoint.Eval(_curpkg_("failpoint-name-2")); ok {
            return rand.Intn(val.(int))
        }
        return rand.Intn(6000)
    }() {
        a()
        if val, ok := failpoint.Eval(_curpkg_("failpoint-name-3")); ok {
            fmt.Println("unit-test", val)
        }
    }
    ```

- Inject a failpoint to the SELECT statement to make it block one CASE if the failpoint is active

    ```go
    func (s *StoreService) ExecuteStoreTask() {
        select {
        case <-func() chan *StoreTask {
            failpoint.Inject("priority-fp", func(_ failpoint.Value) {
                return make(chan *StoreTask)
            })
            return s.priorityHighCh
        }():
            fmt.Println("execute high priority task")

        case <- s.priorityNormalCh:
            fmt.Println("execute normal priority task")

        case <- s.priorityLowCh:
            fmt.Println("execute normal low task")
        }
    }
    ```

    The above code block will generate something like this:

    ```go
    func (s *StoreService) ExecuteStoreTask() {
        select {
        case <-func() chan *StoreTask {
            if _, ok := failpoint.Eval(_curpkg_("priority-fp")); ok {
                return make(chan *StoreTask)
            })
            return s.priorityHighCh
        }():
            fmt.Println("execute high priority task")

        case <- s.priorityNormalCh:
            fmt.Println("execute normal priority task")

        case <- s.priorityLowCh:
            fmt.Println("execute normal low task")
        }
    }
    ```

- Inject a failpoint to dynamically extend SWITCH CASE arms

    ```go
    switch opType := operator.Type(); {
    case opType == "balance-leader":
        fmt.Println("create balance leader steps")

    case opType == "balance-region":
        fmt.Println("create balance region steps")

    case opType == "scatter-region":
        fmt.Println("create scatter region steps")

    case func() bool {
        failpoint.Inject("dynamic-op-type", func(val failpoint.Value) bool {
            return strings.Contains(val.(string), opType)
        })
        return false
    }():
        fmt.Println("do something")

    default:
        panic("unsupported operator type")
    }
    ```

    The above code block will generate something like this:

    ```go
    switch opType := operator.Type(); {
    case opType == "balance-leader":
        fmt.Println("create balance leader steps")

    case opType == "balance-region":
        fmt.Println("create balance region steps")

    case opType == "scatter-region":
        fmt.Println("create scatter region steps")

    case func() bool {
        if val, ok := failpoint.Eval(_curpkg_("dynamic-op-type")); ok {
            return strings.Contains(val.(string), opType)
        }
        return false
    }():
        fmt.Println("do something")

    default:
        panic("unsupported operator type")
    }
    ```

- More complicated failpoints

    - There are more complicated failpoint sites that can be injected to
        - for the loop INITIAL statement, CONDITIONAL expression and POST statement
        - for the RANGE statement
        - SWITCH INITIAL statement
        - …
    - Anywhere you can call a function

## Failpoint name best practice

As you see above, `_curpkg_` will automatically wrap the original failpoint name in `failpoint.Eval` call.
You can think of `_curpkg_` as a macro that automatically prepends the current package path to the failpoint name. For example,

```go
package ddl // which parent package is `github.com/pingcap/tidb`

func demo() {
	// _curpkg_("the-original-failpoint-name") will be expanded as `github.com/pingcap/tidb/ddl/the-original-failpoint-name`
	if val, ok := failpoint.Eval(_curpkg_("the-original-failpoint-name")); ok {...}
}
```

You do not need to care about `_curpkg_` in your application. It is automatically generated after running `failpoint-ctl enable`
and is deleted with `failpoint-ctl disable`.

Because all failpoints in a package share the same namespace, we need to be careful to
avoid name conflict. There are some recommended naming rules to improve this situation.

- Keep name unique in current subpackage
- Use a self-explanatory name for the failpoint
    
    You can enable failpoints by environment variables
    ```shell
    GO_FAILPOINTS="github.com/pingcap/tidb/ddl/renameTableErr=return(100);github.com/pingcap/tidb/planner/core/illegalPushDown=return(true);github.com/pingcap/pd/server/schedulers/balanceLeaderFailed=return(true)"
    ```
    
## Implementation details

1. Define a group of marker functions
2. Parse imports and prune a source file which does not import a failpoint
3. Traverse AST to find marker function calls
4. Marker function calls will be rewritten with an IF statement, which calls `failpoint.Eval` to determine whether a
failpoint is active and executes failpoint code if the failpoint is enabled

![rewrite-demo](./media/rewrite-demo.png)

## Acknowledgments

- Thanks [gofail](https://github.com/etcd-io/gofail) to provide initial implementation.
