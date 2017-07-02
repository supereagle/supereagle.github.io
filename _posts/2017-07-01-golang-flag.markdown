---
layout:     post
title:      "Golang命令行参数解析库源码分析：flag VS pflag"
subtitle:   "Golang Flag Library Source Code Reading: flag VS pflag"
date:       2017-07-01
author:     "Robin"
header-img: "img/post-bg-2015.jpg"
tags:
    - Golang
    - 源码分析
---

Golang官方的命令行参数解析库flag，已经用的比较少。而是Google的一位Golang大牛Steve Francia自己写的命令行参数解析库[pflag](https://github.com/spf13/pflag)更加优秀，从而得到广泛使用。下面从源码的角度对它们的工作原理进行分析，同时对它们的功能进行一个全面的比较。

# Category

- [Golang Flag](#golang-flag)
	- [Flag Types](#flag-types)
	- [Flag Data Structure](#flag-data-structure)
- [How to Work](#how-to-work)
	- [Flag Defination](#flag-defination)
	- [Flag Parse](#flag-parse)
- [pflag VS flag](#pflag-vs-flag)

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

## How to Work

### Flag Defination

CommandLine是一个FlagSet类型的全局变量，用来存储所有定义过的flag。FlagSet的name就是命令行的第一个参数，即可执行程序的名字。

[golang/src/flag/flag.go](https://github.com/golang/go/blob/release-branch.go1.8/src/flag/flag.go#L949-L975)
```go
// CommandLine is the default set of command-line flags, parsed from os.Args.
// The top-level functions such as BoolVar, Arg, and so on are wrappers for the
// methods of CommandLine.
var CommandLine = NewFlagSet(os.Args[0], ExitOnError)

func init() {
	// Override generic FlagSet default Usage with call to global Usage.
	// Note: This is not CommandLine.Usage = Usage,
	// because we want any eventual call to use any updated value of Usage,
	// not the value it has when this line is run.
	CommandLine.Usage = commandLineUsage
}

func commandLineUsage() {
	Usage()
}

// NewFlagSet returns a new, empty flag set with the specified name and
// error handling property.
func NewFlagSet(name string, errorHandling ErrorHandling) *FlagSet {
	f := &FlagSet{
		name:          name,
		errorHandling: errorHandling,
	}
	f.Usage = f.defaultUsage
	return f
}
```

在解析和使用命令行参数前，必须定义flag。所有命令行参数类型提供的定义flag接口非常类似，下面以string类型的命令行参数为例进行说明。

[golang/src/flag/flag.go](https://github.com/golang/go/blob/release-branch.go1.8/src/flag/flag.go#L693-L717)
```go
// StringVar defines a string flag with specified name, default value, and usage string.
// The argument p points to a string variable in which to store the value of the flag.
func (f *FlagSet) StringVar(p *string, name string, value string, usage string) {
	f.Var(newStringValue(value, p), name, usage)
}

// StringVar defines a string flag with specified name, default value, and usage string.
// The argument p points to a string variable in which to store the value of the flag.
func StringVar(p *string, name string, value string, usage string) {
	CommandLine.Var(newStringValue(value, p), name, usage)
}

// String defines a string flag with specified name, default value, and usage string.
// The return value is the address of a string variable that stores the value of the flag.
func (f *FlagSet) String(name string, value string, usage string) *string {
	p := new(string)
	f.StringVar(p, name, value, usage)
	return p
}

// String defines a string flag with specified name, default value, and usage string.
// The return value is the address of a string variable that stores the value of the flag.
func String(name string, value string, usage string) *string {
	return CommandLine.String(name, value, usage)
}
```

定义的每个flag，都会添加到CommandLine这个全局的FlagSet中。对于每种命令行参数类型，都提供两种定义flag的接口。例如：StringVar()能够将string参数的值存储到指定的变量中；String()不是将string参数的值存储到指定的变量中，而是返回存储该参数的值的指针。

### Flag Parse

定义命令行参数之后，和使用命令行参数之前，必须调用`flag.Parse()`来解析命令行参数。`flag.Parse()`使CommandLine解析除命令行除第一个参数外的所有参数，通过`FlagSet.parseOne()`一个一个地解析，从而获得每个flag相应的值。

`FlagSet.parseOne()`首先判断剩余args的数量，flag前面是'-'或者'--'，flag后面是否有'='，flag是否被注册过，以及flag的值是否为bool类型等各种情况，最后获得flag的value，然后通过该flag提供的Set()方法，将value设置到flag中。

[golang/src/flag/flag.go](https://github.com/golang/go/blob/release-branch.go1.8/src/flag/flag.go#L830-L947)
```go
// parseOne parses one flag. It reports whether a flag was seen.
func (f *FlagSet) parseOne() (bool, error) {
	if len(f.args) == 0 {
		return false, nil
	}
	s := f.args[0]
	if len(s) == 0 || s[0] != '-' || len(s) == 1 {
		return false, nil
	}
	numMinuses := 1
	if s[1] == '-' {
		numMinuses++
		if len(s) == 2 { // "--" terminates the flags
			f.args = f.args[1:]
			return false, nil
		}
	}
	name := s[numMinuses:]
	if len(name) == 0 || name[0] == '-' || name[0] == '=' {
		return false, f.failf("bad flag syntax: %s", s)
	}

	// it's a flag. does it have an argument?
	f.args = f.args[1:]
	hasValue := false
	value := ""
	for i := 1; i < len(name); i++ { // equals cannot be first
		if name[i] == '=' {
			value = name[i+1:]
			hasValue = true
			name = name[0:i]
			break
		}
	}
	m := f.formal
	flag, alreadythere := m[name] // BUG
	if !alreadythere {
		if name == "help" || name == "h" { // special case for nice help message.
			f.usage()
			return false, ErrHelp
		}
		return false, f.failf("flag provided but not defined: -%s", name)
	}

	if fv, ok := flag.Value.(boolFlag); ok && fv.IsBoolFlag() { // special case: doesn't need an arg
		if hasValue {
			if err := fv.Set(value); err != nil {
				return false, f.failf("invalid boolean value %q for -%s: %v", value, name, err)
			}
		} else {
			if err := fv.Set("true"); err != nil {
				return false, f.failf("invalid boolean flag %s: %v", name, err)
			}
		}
	} else {
		// It must have a value, which might be the next argument.
		if !hasValue && len(f.args) > 0 {
			// value is the next arg
			hasValue = true
			value, f.args = f.args[0], f.args[1:]
		}
		if !hasValue {
			return false, f.failf("flag needs an argument: -%s", name)
		}
		if err := flag.Value.Set(value); err != nil {
			return false, f.failf("invalid value %q for flag -%s: %v", value, name, err)
		}
	}
	if f.actual == nil {
		f.actual = make(map[string]*Flag)
	}
	f.actual[name] = flag
	return true, nil
}

// Parse parses flag definitions from the argument list, which should not
// include the command name. Must be called after all flags in the FlagSet
// are defined and before flags are accessed by the program.
// The return value will be ErrHelp if -help or -h were set but not defined.
func (f *FlagSet) Parse(arguments []string) error {
	f.parsed = true
	f.args = arguments
	for {
		seen, err := f.parseOne()
		if seen {
			continue
		}
		if err == nil {
			break
		}
		switch f.errorHandling {
		case ContinueOnError:
			return err
		case ExitOnError:
			os.Exit(2)
		case PanicOnError:
			panic(err)
		}
	}
	return nil
}

// Parsed reports whether f.Parse has been called.
func (f *FlagSet) Parsed() bool {
	return f.parsed
}

// Parse parses the command-line flags from os.Args[1:].  Must be called
// after all flags are defined and before flags are accessed by the program.
func Parse() {
	// Ignore errors; CommandLine is set for ExitOnError.
	CommandLine.Parse(os.Args[1:])
}

// Parsed reports whether the command-line flags have been parsed.
func Parsed() bool {
	return CommandLine.Parsed()
}
```

## pflag VS flag

flag和pflag都是源自于Google，工作原理甚至代码实现基本上都是一样的。
flag虽然是Golang官方的命令行参数解析库，但是pflag却得到更加广泛的应用。
因为pflag相对flag有如下优势：
* 支持更加精细的参数类型：例如，flag只支持uint和uint64，而pflag额外支持uint8、uint16、int32。
* 支持更多参数类型：ip、ip mask、ip net、count、以及所有类型的slice类型，例如string slice、int slice、ip slice等。
* 兼容标准flag库的Flag和FlagSet。
* 原生支持更丰富flag功能：shorthand、deprecated、hidden等高级功能。

flag虽然不是原生支持shorthand，但是可以通过两个flag共享同一变量来间接支持，可以参考官方提供的Example：

[golang/src/flag/example_test.go](https://github.com/golang/go/blob/master/src/flag/example_test.go#L19-L31)
```go
// Example 2: Two flags sharing a variable, so we can have a shorthand.
// The order of initialization is undefined, so make sure both use the
// same default value. They must be set up with an init function.
var gopherType string

func init() {
	const (
		defaultGopher = "pocket"
		usage         = "the variety of gopher"
	)
	flag.StringVar(&gopherType, "gopher_type", defaultGopher, usage)
	flag.StringVar(&gopherType, "g", defaultGopher, usage+" (shorthand)")
}
```

pflag原生支持shorthand，在定义flag的时候为其指定shorthand，实现起来更加方便。flag虽然能够通过间接方式实现shorthand，但是flag的数量要翻倍，同时不能避免这两个flag被同时使用的错误用法。
上面flag的shorthand example的pflag版如下：
```go
// Example for shorthand in pflag.
import flag "github.com/spf13/pflag"

var gopherType string

func init() {
	const (
		defaultGopher = "pocket"
		usage         = "the variety of gopher"
	)
	flag.StringVarP(&gopherType, "gopher_type", "g", defaultGopher, usage)
}
```
