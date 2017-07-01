---
layout:     post
title:      "Golang命令行参数解析库源码分析：flag"
subtitle:   "Golang Flag Library Source Code Reading: flag"
date:       2017-07-01
author:     "Robin"
header-img: "img/post-bg-2015.jpg"
tags:
    - Golang
    - 源码分析
---

Golang官方的命令行参数解析库flag，已经用的比较少。而是Google的一位Golang大牛Steve Francia自己写的命令行参数解析库[pflag](https://github.com/spf13/pflag)更加优秀，从而得到广泛使用。

# Category

- [Golang Flag](#golang-flag)
	- [Flag Types](#flag-types)
	- [Flag Data Structure](#flag-data-structure)

## Golang Flag

### Flag Types

Flag原生支持8种命令行参数类型：bool、int、int64、uint、uint64、string、float64和duration。
这些类型都需要实现`Value`接口：

[golang/src/flag/flag.go](https://github.com/golang/go/blob/release-branch.go1.8/src/flag/flag.go#L233-L246)
```go
// Value is the interface to the dynamic value stored in a flag.
// (The default value is represented as a string.)
//
// If a Value has an IsBoolFlag() bool method returning true,
// the command-line parser makes -name equivalent to -name=true
// rather than using the next command-line argument.
//
// Set is called once, in command line order, for each flag present.
// The flag package may call the String method with a zero-valued receiver,
// such as a nil pointer.
type Value interface {
	String() string
	Set(string) error
}
```

下面以int和int64类型为例，进行对比分析如何支持这两种类型的参数。首先要为基本类型自定义一个新的类型，新类型必须实现`Value`接口：`Set(string) error`方法能够将string类型的value解析出来，并赋值给该类型的变量；`String() string`方法能够将该类型变量的值，转化为string。

[golang/src/flag/flag.go](https://github.com/golang/go/blob/release-branch.go1.8/src/flag/flag.go#L233-L246)
```go
// -- int Value
type intValue int

func newIntValue(val int, p *int) *intValue {
	*p = val
	return (*intValue)(p)
}

func (i *intValue) Set(s string) error {
	v, err := strconv.ParseInt(s, 0, 64)
	*i = intValue(v)
	return err
}

func (i *intValue) Get() interface{} { return int(*i) }

func (i *intValue) String() string { return strconv.Itoa(int(*i)) }

// -- int64 Value
type int64Value int64

func newInt64Value(val int64, p *int64) *int64Value {
	*p = val
	return (*int64Value)(p)
}

func (i *int64Value) Set(s string) error {
	v, err := strconv.ParseInt(s, 0, 64)
	*i = int64Value(v)
	return err
}

func (i *int64Value) Get() interface{} { return int64(*i) }

func (i *int64Value) String() string { return strconv.FormatInt(int64(*i), 10) }
```

添加自定义的命令行参数类型也非常简单，只需要采用类似的方法实现`Value`接口。可以参考官方提供的example：

[golang/src/flag/example_test.go](https://github.com/golang/go/blob/master/src/flag/example_test.go#L33-L74)
```go
// Example 3: A user-defined flag type, a slice of durations.
type interval []time.Duration

// String is the method to format the flag's value, part of the flag.Value interface.
// The String method's output will be used in diagnostics.
func (i *interval) String() string {
	return fmt.Sprint(*i)
}

// Set is the method to set the flag value, part of the flag.Value interface.
// Set's argument is a string to be parsed to set the flag.
// It's a comma-separated list, so we split it.
func (i *interval) Set(value string) error {
	// If we wanted to allow the flag to be set multiple times,
	// accumulating values, we would delete this if statement.
	// That would permit usages such as
	//	-deltaT 10s -deltaT 15s
	// and other combinations.
	if len(*i) > 0 {
		return errors.New("interval flag already set")
	}
	for _, dt := range strings.Split(value, ",") {
		duration, err := time.ParseDuration(dt)
		if err != nil {
			return err
		}
		*i = append(*i, duration)
	}
	return nil
}

// Define a flag to accumulate durations. Because it has a special type,
// we need to use the Var function and therefore create the flag during
// init.

var intervalFlag interval

func init() {
	// Tie the command-line flag to the intervalFlag variable and
	// set a usage message.
	flag.Var(&intervalFlag, "deltaT", "comma-separated list of intervals to use between events")
}
```

### Flag Data Structure

有两重要的数据结构来表示flag：Flag是描述一个flag的数据结构，包括该flag的name、usage、value以及default value信息；FlagSet是描述所有Flag的集合，同时提供一系列的方法对集合中各种类型的所有Flag进行解析。

[golang/src/flag/flag.go](https://github.com/golang/go/blob/release-branch.go1.8/src/flag/flag.go#L267-L290)
```go
// A FlagSet represents a set of defined flags. The zero value of a FlagSet
// has no name and has ContinueOnError error handling.
type FlagSet struct {
	// Usage is the function called when an error occurs while parsing flags.
	// The field is a function (not a method) that may be changed to point to
	// a custom error handler.
	Usage func()

	name          string
	parsed        bool
	actual        map[string]*Flag
	formal        map[string]*Flag
	args          []string // arguments after flags
	errorHandling ErrorHandling
	output        io.Writer // nil means stderr; use out() accessor
}

// A Flag represents the state of a flag.
type Flag struct {
	Name     string // name as it appears on command line
	Usage    string // help message
	Value    Value  // value as set
	DefValue string // default value (as text); for usage message
}
```






