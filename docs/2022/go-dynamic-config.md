---
title: Go Dynamic Config
---

## 1. 版权声明

```golang title="Copyright"
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

## 2. 客户端接口

```golang title="Client"

type Filter int

// Client allows fetching values from a dynamic configuration system NOTE: This does not have async
// options right now. In the interest of keeping it minimal, we can add when requirement arises.
type Client interface {
	GetValue(name Key, defaultValue interface{}) (interface{}, error)
	GetValueWithFilters(name Key, filters map[Filter]interface{}, defaultValue interface{}) (interface{}, error)

	GetIntValue(name Key, filters map[Filter]interface{}, defaultValue int) (int, error)
	GetFloatValue(name Key, filters map[Filter]interface{}, defaultValue float64) (float64, error)
	GetBoolValue(name Key, filters map[Filter]interface{}, defaultValue bool) (bool, error)
	GetStringValue(name Key, filters map[Filter]interface{}, defaultValue string) (string, error)
	GetMapValue(
		name Key, filters map[Filter]interface{}, defaultValue map[string]interface{},
	) (map[string]interface{}, error)
	GetDurationValue(
		name Key, filters map[Filter]interface{}, defaultValue time.Duration,
	) (time.Duration, error)
	// UpdateValue takes value as map and updates by overriding. It doesn't support update with filters.
	UpdateValue(name Key, value interface{}) error
	RestoreValue(name Key, filters map[Filter]interface{}) error
	ListValue(name Key) ([]*types.DynamicConfigEntry, error)
}
```

## 3. File Based Client

```golang title="fileBasedClient"
// FileBasedClientConfig is the config for the file based dynamic config client.
// It specifies where the config file is stored and how often the config should be
// updated by checking the config file again.
type FileBasedClientConfig struct {
	Filepath     string        `yaml:"filepath"`
	PollInterval time.Duration `yaml:"pollInterval"`
}

type fileBasedClient struct {
	values          atomic.Value            // 原子存储
	lastUpdatedTime time.Time               // 最后更新时间
	config          *FileBasedClientConfig
	doneCh          chan struct{}           // 关闭channel
	logger          log.Logger
}
```

### 3.2 构造器

`NewFileBasedClient`构造器集成相关的配置、logger以及接收关闭信号的channel `doneCh`。实现中尝试首次`update`加载数据，如果报错直接error返回，如果正常设定定时器定时`update`。

```golang hl_lines="12 15-29" title="NewFileBasedClient"
// NewFileBasedClient creates a file based client.
func NewFileBasedClient(config *FileBasedClientConfig, logger log.Logger, doneCh chan struct{}) (Client, error) {
	if err := validateConfig(config); err != nil {
		return nil, err
	}

	client := &fileBasedClient{
		config: config,
		doneCh: doneCh,
		logger: logger,
	}
	if err := client.update(); err != nil {  // first update check not failed
		return nil, err
	}
	go func() { // tick update
		ticker := time.NewTicker(client.config.PollInterval)
		for {
			select {
			case <-ticker.C:
				err := client.update()
				if err != nil { // if err happened, only log and do nothing
					client.logger.Error("Failed to update dynamic config", tag.Error(err))
				}
			case <-client.doneCh:
				ticker.Stop()
				return
			}
		}
	}()
	return client, nil
}
```

`update`函数通过获取文件的最后修改时间判定时候需要重新load，具体实现如下：

```golang hl_lines="8 12" title="update"
func (fc *fileBasedClient) update() error {
	defer func() {
		fc.lastUpdatedTime = time.Now()
	}()

	newValues := make(map[string][]*constrainedValue)

	info, err := os.Stat(fc.config.Filepath)    // 获取文件信息
	if err != nil {
		return fmt.Errorf("failed to get status of dynamic config file: %v", err)
	}
	if !info.ModTime().After(fc.lastUpdatedTime) {  // 对比文件最后修改时间
		return nil
	}

	confContent, err := ioutil.ReadFile(fc.config.Filepath)
	if err != nil {
		return fmt.Errorf("failed to read dynamic config file %v: %v", fc.config.Filepath, err)
	}

	if err = yaml.Unmarshal(confContent, newValues); err != nil {
		return fmt.Errorf("failed to decode dynamic config %v", err)
	}

	return fc.storeValues(newValues)
}
```

## 4. Collection

使用Client过程中，如果用户使用`value := client.GetValue("key", nil)`保存了配置值，有可能导致无法获取更新后的配置。需要对配置进行一层封装，这里使用的是`Collection`。

```golang title="Collection"
// Collection wraps dynamic config client with a closure so that across the code, the config values
// can be directly accessed by calling the function without propagating the client everywhere in
// code
type Collection struct {
	client        Client        // wrapper of Client
	logger        log.Logger
	logKeys       *sync.Map // map of config Keys for logging to capture changes
	errCount      int64
	filterOptions []FilterOption
}
```

### 4.1 `NewCollection`

构造器的实现是`trivial`的：

```golang title="NewCollection"
// NewCollection creates a new collection
func NewCollection(
	client Client,
	logger log.Logger,
	filterOptions ...FilterOption,
) *Collection {

	return &Collection{
		client:        client,
		logger:        logger,
		logKeys:       &sync.Map{},
		errCount:      -1,
		filterOptions: filterOptions,
	}
}
```

### 4.2 PropertyFn

这里使用了`thunk`方式的wrapper，不是直接获取value，而是返回value的function:

```golang title="PropertyFn" hl_lines="2 5"
// PropertyFn is a wrapper to get property from dynamic config
type PropertyFn func() interface{}

// IntPropertyFn is a wrapper to get int property from dynamic config
type IntPropertyFn func(opts ...FilterOption) int

// GetProperty gets a interface property and returns defaultValue if property is not found
func (c *Collection) GetProperty(key Key, defaultValue interface{}) PropertyFn {
	return func() interface{} {
		val, err := c.client.GetValue(key, defaultValue)
		if err != nil {
			c.logError(key, nil, err)
		}
		c.logValue(key, nil, val, defaultValue, reflect.DeepEqual)
		return val
	}
}

// GetIntProperty gets property and asserts that it's an integer
func (c *Collection) GetIntProperty(key Key, defaultValue int) IntPropertyFn {
	return func(opts ...FilterOption) int {
		filters := c.toFilterMap(opts...)
		val, err := c.client.GetIntValue(
			key,
			filters,
			defaultValue,
		)
		if err != nil {
			c.logError(key, filters, err)
		}
		c.logValue(key, filters, val, defaultValue, intCompareEquals)
		return val
	}
}

```

具体使用上是先获取`PropertyFn`，之后对函数求值：

```golang title="Usage"
prop := c.GetProperty("key", nil)
value := prop()
```

最后，一个配置结构体的变成了如下情形：
```golang title="Config"
// Config represents configuration for cadence-frontend service
type Config struct {
	PersistenceMaxQPS               dynamicconfig.IntPropertyFn
	PersistenceGlobalMaxQPS         dynamicconfig.IntPropertyFn
	EnableVisibilitySampling        dynamicconfig.BoolPropertyFn
	EnableReadFromClosedExecutionV2 dynamicconfig.BoolPropertyFn
    // ...
}
```
所有的字段的值都是一个`PropertyFn`。

## 5. 持久化存储

### 5.1 configStoreClient

持久化配置存储的`client`实现是类似的：
```golang title="configStoreClient"

type configStoreClient struct {
	values             atomic.Value
	lastUpdatedTime    time.Time
	config             *csc.ClientConfig
	configStoreManager persistence.ConfigStoreManager
	doneCh             chan struct{}
	logger             log.Logger
}

type configStoreClient struct {
	values             atomic.Value
	lastUpdatedTime    time.Time
	config             *csc.ClientConfig
	configStoreManager persistence.ConfigStoreManager
	doneCh             chan struct{}
	logger             log.Logger
}
```

### 5.2 构造器

构造器实现类似`fileBasedClient`，同样通过实现了定时更新的策略。

```golang title="NewConfigStoreClient"  hl_lines="22"
// NewConfigStoreClient creates a config store client
func NewConfigStoreClient(clientCfg *csc.ClientConfig, persistenceCfg *config.Persistence, logger log.Logger, doneCh chan struct{}) (dc.Client, error) {
	if err := validateClientConfig(clientCfg); err != nil {
		logger.Error("Invalid Client Config Values, Using Default Values")
		clientCfg = defaultConfigValues
	}

	if persistenceCfg == nil {
		return nil, errors.New("persistence cfg is nil")
	} else if persistenceCfg.DefaultStore != "cass-default" {
		return nil, fmt.Errorf("persistence cfg default store is not Cassandra")
	} else if store, ok := persistenceCfg.DataStores[persistenceCfg.DefaultStore]; !ok {
		return nil, errors.New("persistence cfg datastores missing Cassandra")
	} else if store.NoSQL == nil {
		return nil, errors.New("NoSQL struct is nil")
	}

	client, err := newConfigStoreClient(clientCfg, persistenceCfg.DataStores[persistenceCfg.DefaultStore].NoSQL, logger, doneCh)
	if err != nil {
		return nil, err
	}
	err = client.startUpdate()
	if err != nil {
		return nil, err
	}
	return client, nil
}


func (csc *configStoreClient) startUpdate() error {
	if err := csc.update(); err != nil {
		return err
	}
	go func() {
		ticker := time.NewTicker(csc.config.PollInterval)
		for {
			select {
			case <-ticker.C:
				err := csc.update()
				if err != nil {
					csc.logger.Error("Failed to update dynamic config", tag.Error(err))
				}
			case <-csc.doneCh:
				ticker.Stop()
				return
			}
		}
	}()
	return nil
}
```

`update`的具体实现不过是通过`configStoreManager`从持久化存储中获取数据而已。

```golang title="update" hl_lines="5"
func (csc *configStoreClient) update() error {
	ctx, cancel := context.WithTimeout(context.Background(), csc.config.FetchTimeout)
	defer cancel()

	res, err := csc.configStoreManager.FetchDynamicConfig(ctx)

	select {
	case <-ctx.Done():
		return errors.New("timeout error on fetch")
	default:
		if err != nil {
			return fmt.Errorf("failed to fetch dynamic config snapshot %v", err)
		}

		if res != nil && res.Snapshot != nil {
			defer func() {
				csc.lastUpdatedTime = time.Now()
			}()

			return csc.storeValues(res.Snapshot)
		}
	}
	return nil
}
```
这里不再讨论`FetchDynamicConfig`的具体实现了，不过是对SQL的封装。

## 6. 总结

`Dynamic Config`的封装实现，可以做到对配置的修改而不用重启服务。抽象接口`Client`下可以使用`filedBaseClient`或者`configStoreClient`。两者的实现思路基本相同，都是设置了定时器的方式，定时更新配置。同时为了防止配置被保存在内存中，通过`PropertyFn`封装了对配置的访问。

`Client`接口实现的思路是的抽象非常干净，用户可以自行切换具体的实现，比如使用ETCD等服务真正实现实时响应的配置修改。

同时对于`fileBasedClient`的理论上不一定需要使用定时器的方式实现，通过文件监听的方式也是可行的。