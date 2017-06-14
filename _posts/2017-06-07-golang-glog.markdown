---
layout:     post
title:      "Golang日志库源码分析：Glog"
subtitle:   "Golang Log Library Source Code Reading: Glog"
date:       2017-06-07
author:     "Robin"
header-img: "img/post-bg-2015.jpg"
tags:
    - Golang
    - 源码分析
---

[Glog](https://github.com/golang/glog)是著名[google开源C++日志库glog](https://github.com/google/glog)的golang版本，具有轻量级、简单、稳定和高效等特性。
目前被用在大型的容器云开源项目[Kubernetes](https://github.com/kubernetes/kubernetes)中。

# Category

- [Overview](#overview)
- [Usage](#usage)
- [Source Code Reading](#source-code-reading)
- [Reference](#reference)

## Overview

Glog主要有以下特点：
* 支持四种日志等级INFO < WARING < ERROR < FATAL，不支持DEBUG等级。
* 每个日志等级对应一个日志文件，低等级的日志文件中除了包含该等级的日志，还会包含高等级的日志。
* 日志文件可以根据大小切割，但是不能根据日期切割。
* 日志文件名称格式：program.host.userName.log.log_level.date-time.pid，不可自定义。
* 固定日志输出格式：Lmmdd hh:mm:ss.uuuuuu threadid file:line] msg…，不可自定义。
* 程序开始时必须调用flag.Parse()解析命令行参数，退出时必须调用glog.Flush()确保将缓存区日志输出。

> Glog虽然简单，但是功能还存在一定的局限性。例如，不支持DEBUG等级，不支持根据日期切割等。如果需要更多Glog功能，可以参考[定制化Glog](https://github.com/sdbaiguanghe/glog)。

## Usage

**Example：**

```go
package main

import (
	"flag"
	"fmt"
	"os"

	"github.com/golang/glog"
)

func main() {
	//Init the command-line flags.
	flag.Parse()

	// Will be ignored as the program has exited in Fatal().
	defer func() {
		fmt.Println("Message in defer")
	}()

	// Flushes all pending log I/O.
	defer glog.Flush()

	// The temp folder for log files when --log_dir is not set.
	fmt.Printf("Temp folder for log files: %s\n", os.TempDir())

	glog.Info("Info")
	glog.V(1).Info("L1 info")
	glog.Error("Error")
	glog.Fatal("Fatal")

	// Will be ignored as the program has exited in Fatal().
	glog.Error("Error after Fatal")
}
```

**Usage**
* 参数列表

```
$ ./glog.exe -h
Usage of E:\gocode\src\test\glog\glog.exe:
  -alsologtostderr
        log to standard error as well as files (同时输出到os.Stderr和log files)
  -log_backtrace_at value
        when logging hits line file:N, emit a stack trace
  -log_dir string
        If non-empty, write log files in this directory （指定log files的目录，默认是os.TempDir()）
  -logtostderr
        log to standard error instead of files （只输出到os.Stderr）
  -stderrthreshold value
        logs at or above this threshold go to stderr (大于等于该severity的log，会输出到os.Stderr)
  -v value
        log level for V logs （设置vLog的等级）
  -vmodule value
        comma-separated list of pattern=N settings for file-filtered logging
```

* Output

```
$ ./glog.exe
Temp folder for log files: C:\Users\robin\AppData\Local\Temp
E0608 13:09:38.304553    2584 main.go:28] Error
F0608 13:09:38.305553    2584 main.go:29] Fatal
goroutine 1 [running]:
github.com/golang/glog.stacks(0x544d00, 0x0, 0x30, 0x40)
        E:/gocode/src/github.com/golang/glog/glog.go:769 +0x8b
github.com/golang/glog.(*loggingT).output(0x535100, 0x3, 0x116980a0, 0x52206c, 0                                                                                                    x7, 0x1d, 0x0)
        E:/gocode/src/github.com/golang/glog/glog.go:720 +0x2d1
github.com/golang/glog.(*loggingT).printDepth(0x535100, 0x3, 0x1, 0x116b7f64, 0x                                                                                                    1, 0x1)
        E:/gocode/src/github.com/golang/glog/glog.go:646 +0xe7
github.com/golang/glog.(*loggingT).print(0x535100, 0x3, 0x116b7f64, 0x1, 0x1)
        E:/gocode/src/github.com/golang/glog/glog.go:637 +0x49
github.com/golang/glog.Fatal(0x116b7f64, 0x1, 0x1)
        E:/gocode/src/github.com/golang/glog/glog.go:1128 +0x43
main.main()
        E:/gocode/src/test/glog/main.go:29 +0x28d

$ ls /C/Users/robin/AppData/Local/Temp | grep glog
glog.exe.robin-PC.robin-PC_robin.log.ERROR.20170608-130938.2584
glog.exe.robin-PC.robin-PC_robin.log.FATAL.20170608-130938.2584
glog.exe.robin-PC.robin-PC_robin.log.INFO.20170608-130938.2584
glog.exe.robin-PC.robin-PC_robin.log.WARNING.20170608-130938.2584
```

Log files输出到默认的临时目录（C:\Users\robin\AppData\Local\Temp）下。在标准输出上只能看到Error和Fatal log，是因为默认的`--stderrthreshold`为ERROR。不过所有log都会输出到log files，除非设置`--logtostderr=true`。设置`--alsologtostderr=true`，所有log除了输出到log files，还会输出到标准输出，此选项会覆盖`--stderrthreshold`的设置。

需要注意的是，调用`glog.Fatal("Fatal")`，程序会输出所有goroutine的堆栈信息，然后调用`os.Exit()`退出程序。所以，其前面的defer代码以及后面的代码，都不会执行。

## Source Code Reading

Glog的代码非常简单，总共代码行数1.4K左右，分两个文件：
* **glog.go**：主要实现log等级定义、输出以及vlog。
* **glog_file.go**：主要实现日志文件目录和各等级日志文件的创建。

### Log Levels Definition

```go
// severity identifies the sort of log: info, warning etc. It also implements
// the flag.Value interface. The -stderrthreshold flag is of type severity and
// should be modified only through the flag.Value interface. The values match
// the corresponding constants in C++.
type severity int32 // sync/atomic int32

// These constants identify the log levels in order of increasing severity.
// A message written to a high-severity log file is also written to each
// lower-severity log file.
const (
	infoLog severity = iota
	warningLog
	errorLog
	fatalLog
	numSeverity = 4
)

const severityChar = "IWEF"

var severityName = []string{
	infoLog:    "INFO",
	warningLog: "WARNING",
	errorLog:   "ERROR",
	fatalLog:   "FATAL",
}
```

### Flush Daemon

Glog在初始化的时候，会定义一些命令行参数，同时启动flush守护进程。Flush守护进程会间隔30s周期性地flush缓冲区中的log。

```go
func init() {
	flag.BoolVar(&logging.toStderr, "logtostderr", false, "log to standard error instead of files")
	flag.BoolVar(&logging.alsoToStderr, "alsologtostderr", false, "log to standard error as well as files")
	flag.Var(&logging.verbosity, "v", "log level for V logs")
	flag.Var(&logging.stderrThreshold, "stderrthreshold", "logs at or above this threshold go to stderr")
	flag.Var(&logging.vmodule, "vmodule", "comma-separated list of pattern=N settings for file-filtered logging")
	flag.Var(&logging.traceLocation, "log_backtrace_at", "when logging hits line file:N, emit a stack trace")

	// Default stderrThreshold is ERROR.
	logging.stderrThreshold = errorLog

	logging.setVState(0, nil, false)
	go logging.flushDaemon()
}

// Flush flushes all pending log I/O.
func Flush() {
	logging.lockAndFlushAll()
}

...

const flushInterval = 30 * time.Second

// flushDaemon periodically flushes the log file buffers.
func (l *loggingT) flushDaemon() {
	for _ = range time.NewTicker(flushInterval).C {
		l.lockAndFlushAll()
	}
}
```

### Log输出原理

所有的log都支持多种输出模式，例如，Info\[f\|ln\|Depth\]()，最终都是调用`output()`来输出到log files或者Stderr。

`output()`在输出log的整个过程中都会加锁，以防止写冲突。
每次都会check是否已调用`flag.Parse()`，否则就不会输出log信息。
在输出过程中，会根据命令行参数来决定输出行为。如果设置`--toStderr=true`，就只会输出到Stderr。
如果设置`--alsoToStderr=true`，且输出log level大于或等于`--stderrThreshold`的话，
输出到log file的同时也会输出到Stderr。`stderrThreshold`的默认值是**ERROR**。

Glog还有另外一个很赞的功能就是，遇到Fatal log会在自动退出程序，并在退出前将所有goroutine的堆栈信息输出，方便排错。因此，在程序需要异常退出的时候，直接调用Fatal\[f\|ln\|Depth\]()，应该成为一种标准做法。K8s都是采用这种方式处理程序异常退出的。

```go
// output writes the data to the log files and releases the buffer.
func (l *loggingT) output(s severity, buf *buffer, file string, line int, alsoToStderr bool) {
	l.mu.Lock()
	if l.traceLocation.isSet() {
		if l.traceLocation.match(file, line) {
			buf.Write(stacks(false))
		}
	}
	data := buf.Bytes()
	if !flag.Parsed() {
		os.Stderr.Write([]byte("ERROR: logging before flag.Parse: "))
		os.Stderr.Write(data)
	} else if l.toStderr {
		os.Stderr.Write(data)
	} else {
		if alsoToStderr || l.alsoToStderr || s >= l.stderrThreshold.get() {
			os.Stderr.Write(data)
		}
		if l.file[s] == nil {
			if err := l.createFiles(s); err != nil {
				os.Stderr.Write(data) // Make sure the message appears somewhere.
				l.exit(err)
			}
		}
		switch s {
		case fatalLog:
			l.file[fatalLog].Write(data)
			fallthrough
		case errorLog:
			l.file[errorLog].Write(data)
			fallthrough
		case warningLog:
			l.file[warningLog].Write(data)
			fallthrough
		case infoLog:
			l.file[infoLog].Write(data)
		}
	}
	if s == fatalLog {
		// If we got here via Exit rather than Fatal, print no stacks.
		if atomic.LoadUint32(&fatalNoStacks) > 0 {
			l.mu.Unlock()
			timeoutFlush(10 * time.Second)
			os.Exit(1)
		}
		// Dump all goroutine stacks before exiting.
		// First, make sure we see the trace for the current goroutine on standard error.
		// If -logtostderr has been specified, the loop below will do that anyway
		// as the first stack in the full dump.
		if !l.toStderr {
			os.Stderr.Write(stacks(false))
		}
		// Write the stack trace for all goroutines to the files.
		trace := stacks(true)
		logExitFunc = func(error) {} // If we get a write error, we'll still exit below.
		for log := fatalLog; log >= infoLog; log-- {
			if f := l.file[log]; f != nil { // Can be nil if -logtostderr is set.
				f.Write(trace)
			}
		}
		l.mu.Unlock()
		timeoutFlush(10 * time.Second)
		os.Exit(255) // C++ uses -1, which is silly because it's anded with 255 anyway.
	}
	l.putBuffer(buf)
	l.mu.Unlock()
	if stats := severityStats[s]; stats != nil {
		atomic.AddInt64(&stats.lines, 1)
		atomic.AddInt64(&stats.bytes, int64(len(data)))
	}
}
```

### vLog

vLog是用户自定义的log级别，与glog自带的log级别完全独立，使log等级控制更加丰富，灵活。
不过只提供Info()，Infof()和Infoln()三个方法，因此只能对infoLog进行更细粒度的等级划分，可以认为是补充DEBUG等级。

使用方法很简单，有如下两种等价形式。虽然第二种简短，不过第一种代价更低。
```go
if glog.V(2) {
	glog.Info("log this")
}
// Equals
glog.V(2).Info("log this")
```

vLog的实现原理也很简单：
```go
// Verbose is a boolean type that implements Infof (like Printf) etc.
// See the documentation of V for more information.
type Verbose bool

func V(level Level) Verbose {
	// Here is a cheap but safe test to see if V logging is enabled globally.
	if logging.verbosity.get() >= level {
		return Verbose(true)
	}

	if atomic.LoadInt32(&logging.filterLength) > 0 {
		logging.mu.Lock()
		defer logging.mu.Unlock()
		if runtime.Callers(2, logging.pcs[:]) == 0 {
			return Verbose(false)
		}
		v, ok := logging.vmap[logging.pcs[0]]
		if !ok {
			v = logging.setV(logging.pcs[0])
		}
		return Verbose(v >= level)
	}
	return Verbose(false)
}

// Info is equivalent to the global Info function, guarded by the value of v.
// See the documentation of V for usage.
func (v Verbose) Info(args ...interface{}) {
	if v {
		logging.print(infoLog, args...)
	}
}

// Infoln is equivalent to the global Infoln function, guarded by the value of v.
// See the documentation of V for usage.
func (v Verbose) Infoln(args ...interface{}) {
	if v {
		logging.println(infoLog, args...)
	}
}

// Infof is equivalent to the global Infof function, guarded by the value of v.
// See the documentation of V for usage.
func (v Verbose) Infof(format string, args ...interface{}) {
	if v {
		logging.printf(infoLog, format, args...)
	}
}
```

# Reference
- [Kubernetes Logging Conventions](https://github.com/kubernetes/community/blob/master/contributors/devel/logging.md)
- [Go 语言日志指南](https://linux.cn/article-8543-1.html)
- [golang日志库glog解析](http://shanks.leanote.com/post/Untitled-55ca439338f41148cd000759-18)