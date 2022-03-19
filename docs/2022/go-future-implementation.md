---
title: Go Future Implementation
---

## 1. 前言

最近因工作需要接触Uber的Golang项目比较多，闲暇之余学习下Uber的开源项目工具库中一些小巧的代码实现细节。

## 2. 代码版权先行声明

```golang
// Copyright (c) 2021 Uber Technologies, Inc.
//
// Permission is hereby granted, free of charge, to any person obtaining a copy
// of this software and associated documentation files (the "Software"), to deal
// in the Software without restriction, including without limitation the rights
// to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
// copies of the Software, and to permit persons to whom the Software is
// furnished to do so, subject to the following conditions:
//
// The above copyright notice and this permission notice shall be included in
// all copies or substantial portions of the Software.
//
// THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
// IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
// FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
// AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
// LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
// OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN
// THE SOFTWARE.
```

## 3. Future & Settable 接口

`Future`和`Settable`是future实现的两个接口，前者提供read方法，后者提供write方法。

```golang
type (
    Future interface {
        Get(ctx context.Context, valuePtr interface{}) error
        IsReady() bool
    }

    Settable interface {
        Set(interface{}, error)
    }
)
```

* 对于reader获取Future值的两种方式：
    1. 使用`IsReady`（非阻塞）判断future是否完成，然后调用`Get`获取值
    2. 使用`Get`（阻塞）获取future值
* 对于writer写入future值有两种方式：
    1. `Set(value, nil)`, 设置future值
    2. `Set(nil, err)`，设置error

## 4. 实现细节

### 4.1 futureImpl

```golang
type futureImpl struct {
    value   interface{}
    err     error
    readyCh chan struct{}
    status  int32
}
```

结构体`futureImpl`中字段：

* `value` 存放future值
* `err` 存放error值，`err`和`value`两者必须至少一个为`nil`
* `readyCh` 标志future是否完成channel
* `status` 设置future的原子锁, 值为`valueNotSet = 0`或者`valueSet = 1`

### 4.2 构造器`NewFuture`

`NewFuture`同时返回作为`Future`, `Settable`接口的值。构造器隐藏内部具体实现不应该直接返回`*futureImpl`，应该返回抽象。

```golang
func NewFuture() (Future, Settable) {
    future := &futureImpl {
        readyCh: make(chan struct{}),
        status: valueNotSet,
    }
    return future, future
}
```

### 4.3 实现Future接口方法

`IsReady`实现是判断`readCh` channel是否接收过值，通过`select`和`default`防止阻塞。写入信号到channel `readCh`的逻辑有`Set`负责实现 。

```golang hl_lines="3 5"
func (f *futureImpl) IsReady() bool {
    select {
    case <- f.readCh:
        return true
    default:
        return false
    }
}
```

`Get`阻塞获取值写入到`valuePtr`中，同时需要适配`Context`参数。写入到`valuePtr`是由`popluateValue`实现

```golang hl_lines="3 10 12"
// var _ Future = (*futureImpl)(nil)
func (f *futureImpl) Get(ctx context.Context, valuePtr interface{}) error {
    if err := ctx.Err(); err != nil {
        // if the given context is invalid,
		// guarantee to return an error
		return err
    }

    select {
    case <- f.readyCh:
        return f.popluateValue(valuePtr)
    case <- ctx.Done():
        return ctx.Err()
    }
}
```

`popluateValue`具体是通过反射实现值写入，同时还需要考虑`panic`情形

```golang hl_lines="8 12 17 19"
func (f *futureImpl) populateValue(valuePtr interface{}) (err error) {
	defer func() {
		if p := recover(); p != nil {
			err = fmt.Errorf("failed to populate valuePtr: %v", p)
		}
	}()

	if f.err != nil || f.value == nil || valuePtr == nil { // 优先返回error
		return f.err
	}

	rf := reflect.ValueOf(valuePtr)
	if rf.Type().Kind() != reflect.Ptr {  // valuePtr必须是指针
		return errors.New("valuePtr parameter is not a pointer")
	}

	fv := reflect.ValueOf(f.value)
	if fv.IsValid() {
		rf.Elem().Set(fv)  // 值写入
	}
	return nil
}

```

### 4.4 实现Settable接口方法

`Set`接口的实现是相当直白的，把value和err写入到futureImpl中即可。

```golang hl_lines="2 9"
// var _ Settable = (*futureImpl)(nil)
func (f *futureImpl) Set(value interface{}, err error) {
	if !atomic.CompareAndSwapInt32(&f.status, valueNotSet, valueSet) { // atomic lock
		panic("future has already been set")
	}

	f.value = value
	f.err = err
	close(f.readyCh)  // 这里需要注意关闭channel，防止泄漏
}
```

需要注意到`Set`需要同时负责关闭channel `readyCh`，即使是对于`Get`/`IsReady`，即使是关闭的channel，也不会有问题。在Golang中
```golang
value, ok <- f.readyCh // ok标识channel是否关闭
```