---
title: Go Future Implementation
---

## 1. 代码版权先行声明
```golang
// Copyright (c) 2017 Uber Technologies, Inc.
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

## 2. Lock

golang的`sync`提供了Locker接口

```golang
// A Locker represents an object that can be locked and unlocked.
type Locker interface {
    Lock()
    Unlock()
}
```
但是Lock方法的第一个参数没有提供`context.Context`，Uber封装了一套适配的接口，可以方便实现cancelable，timeout lock等功能

```golang
// Mutex accepts a context in its Lock method.
// It blocks the goroutine until either the lock is acquired or the context
// is closed.
type Mutex interface {
    Lock(context.Context) error
    Unlock()
}
```

## 3. 实现细节

### 3.1 mutexImpl & NewMutex

`mutexImpl`是对`sync.Mutex`的封装，结构和构造器非常简单明了

```golang hl_lines="6"
type mutexImpl struct {
    sync.Mutex
}

func NewMutex() {
    return &mutexImpl{} // 默认sync.Mutex值就够用了
}
```

### 实现Mutex接口方法

`Lock`是由`lockInternal`实现的，由于`context.Context`可能会在lock过程中关闭，需要考虑释放。通过设置`state`原子锁确保锁的正确获取和释放。

```golang hl_lines="14-23 26 28 30"
const (
	acquiring = iota
	acquired
	bailed
)

func (m *mutexImpl) Lock(ctx context.Context) error {
	return m.lockInternal(ctx)
}

func (m *mutexImpl) lockInternal(ctx context.Context) error {
	var state int32 = acquiring

	acquiredCh := make(chan struct{})
	go func() {
		m.Mutex.Lock()
		if !atomic.CompareAndSwapInt32(&state, acquiring, acquired) {
			// already bailed due to context closing
			m.Unlock()
		}

		close(acquiredCh)
	}()

	select {
	case <-acquiredCh:  // race
		return nil
	case <-ctx.Done():  // race
		{
			if !atomic.CompareAndSwapInt32(&state, acquiring, bailed) { // 检测是否因为context过期导致lock失败
				return nil
			}
			return ctx.Err()
		}
	}
}
```

`Unlock`可以复用`sync.Mutex`的`Unlock`。