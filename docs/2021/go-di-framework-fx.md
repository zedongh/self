# Go依赖注入框架Fx


```go title="main.go" hl_lines="2-8" linenums="1"
func main() {
    db := NewDB()
    aStore := NewAStore(db)
    bStore := NewBStore(db)
    aService := NewAService(aStore)
    bService := NewBService(bStore)
    controller := NewController(aService, bService)
    srv := NewServer(controller)
    srv.Run()
}
```

&emsp;&emsp;习惯了Spring的Java开发者切换到Go项目开发的时候最大的不适应便是Go的项目中总是充斥着各种手动的依赖注入。这些初始化代码不仅无聊且冗长，当依赖关系改变的时候，需要
开发者再次手动处理这些依赖初始化关系。这对于习惯了“自动档”的Javaer是一件相当难以忍受的事情......

## 依赖关系

&emsp;&emsp;开发者在开发过程中总是基于底层模块的提供的功能（依赖），加上功能的实现（业务逻辑），暴露出相应的接口给上层

## Uber Fx 来拯救

&emsp;&emsp;Fx是go的依赖注入框架，旨在方便依赖注入，消除全局变量和`func init()`

```go title="main.go" linenums="1"
func main() {
    app := fx.New(
        fx.Provide(
            NewDB,
            NewService,
            NewController,
            Server,
        ),
        fx.Invoke(func (srv *Server) {
            go srv.Run()
        }),
    )
}
```

## fx.Provide和fx.Invoke

## fx.Annotated