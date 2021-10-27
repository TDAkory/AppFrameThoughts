普罗米修斯内部包含很多组件，其server端结构如下：

![](https://raw.githubusercontent.com/TDAkory/ImageResources/main/img/20210926152404.png)

## main()
[`main()`](https://github.com/prometheus/prometheus/blob/v2.3.1/cmd/prometheus/main.go#L77-L600)函数初始化并运行所有Prometheus的组件:

1.  定义并解析服务进程的命令行参数，将其存为一个本地配置结构。

    这些配置参数与从配置文件获取的参数是互相独立的。命令行参数简单，仅在进程重启时可以更新。配置文件提供的参数可以让进程在运行时`reload`并`update`
```golang
    cfg := struct {
		configFile string

		localStoragePath    string
		notifier            notifier.Options
		notifierTimeout     model.Duration
		web                 web.Options
		tsdb                tsdb.Options
		lookbackDelta       model.Duration
		webTimeout          model.Duration
		queryTimeout        model.Duration
		queryConcurrency    int
		RemoteFlushDeadline model.Duration

		prometheusURL string

		logLevel promlog.AllowedLevel
	}
```

2. 实例化所有主要组件，并将它们通过`channel`、`reference`、`passing in context`等方式关联起来，这些组件包括：`service discovery`、`target scraping`、`storage`, etc

3. server用[actor-like model](https://www.brianstorti.com/the-actor-model/)来运行所有组件，使用`github.com/oklog/oklog/pkg/group`来协调所有组件的启停，多个`channel`用来协调组件间的行为顺序。

## Configuration

配置子系统负责读取、校验、使生效，由基于yaml的配置文件提供的配。[Documentation](https://prometheus.io/docs/prometheus/latest/configuration/configuration/)

### Configuration reading and parsing

### Reload handler

## Treminatin handler

## Scrape discovery manager
本质上是一个服务发现组件[`discovery.Manager`](https://github.com/prometheus/prometheus/blob/v2.3.1/discovery/manager.go#L73-L89)，利用`Prometheus`的服务发现能力，持续发现并更新需要抓取指标的目标列表。该组件独立于`scrape manager`运行，并通过一个`同步channel`向`scrape manager`投递目标组。

## Scrape manager
[scrape.Manager](https://github.com/prometheus/prometheus/blob/v2.3.1/scrape/manager.go#L47-L62)负责抓取目标的metrics，并将这些指标投递到`storage`子系统。

### Target updates and overall architecture

### Target labels and target relabeling

### Target hashing and scrape timing

### Target scrapes


## Storage

### Fanout storage

### Local storage

### Remote storage
