---
title: "【转】使用GRafana监控Go应用"
date: 2018-08-07T12:17:15+08:00
weight: 170
keywords: ["monitor"]
description: "使用GRafana监控Go应用"
tags: ["monitor"]
categories: ["转载"]
author: "去去"
---
互联网企业背后，依靠的是成千上万台服务器日夜不停运转，以支撑其业务的运转，宕机对于互联网企业来说，代价是 沉重的，轻则影响用户体验，重则直接影响交易，特别给我们这些做电商的造成不可挽回的损失。对于这些机器队友的 开发和运维人员来说，依靠人力是不可能完成24小时不间断盯着服务器的。所以对于互联网公司，监控工作的地位 已经越来越重要了，作为Devops也是需要掌握各种技术方案的监控。

Go语言在过去的2016年成为了年度的最佳语言，这说明在国内甚至全球范围内，越来越多公司都在使用Go来开发他们的产品。 Go具备高并发，部署方便等特点，已经成为很多后端开发工程师的首选语言。

本文会以Go为入口，利用目前在监控工具中很火的Grafana和infuxDB作为例子，介绍下我们如何数据可视化的方式实时监控我们的 go应用。

环境准备
----

#### influxDB 和 grafana

Grafana一般是和一些时间序列数据库进行配合来展示数据的，例如：Graphite、OpenTSDB、InfluxDB等，主要特点：

*   grafana是用于可视化大型测量数据的开源程序，他提供了强大和优雅的方式去创建、共享、浏览数据。dashboard中显示了你不同metric数据源中的数据。
    
*   grafana最常用于因特网基础设施和应用分析，但在其他领域也有机会用到，比如：工业传感器、家庭自动化、过程控制等等。
    
*   grafana有热插拔控制面板和可扩展的数据源，目前已经支持Graphite、InfluxDB、OpenTSDB、Elasticsearch。
    

本次实验， 我们需要使用`influxDB`存储我们的指标数据，由于我们的指标数据是时序性的，所以influxDB比较合适，当然也可以使用 `prometheus` 作为时序数据持久化方案。

由于我想快速搭建环境，我就不拘束于常规的安装方式，这里直接使用Docker来构建我们的influxDB容器

    sudo docker run -d -p 8083:8083 -p 8086:8086 tutum/influxdb:latest
    

同样，我们也用Docker开构建我们的grafana容器

    sudo docker run -d -p 9090:3000/tcp --name=grafana grafana/grafana:latest
    

上面的容器构建并启动后，我们可以在浏览器完成一些相关的配置：

#### 初始化influxDB

我们在浏览器访问 8083端口地址，如下图

![go-monitor-1](http://og0usnhfv.bkt.clouddn.com/monitor-go-1.png)

接着我们在Query框输入以下命令，完成测试数据库的创建

    CREATE DATABASE "helloWorld"
    

![go-monitor-2](http://og0usnhfv.bkt.clouddn.com/monitor-go-2.png)

这样我们的测试数据库就创建完成了。

#### grafana设置

我们需要访问grafana,进行一些参数和环境的设置

用户名和密码默认为 admin / admin

![go-monitor-3](http://og0usnhfv.bkt.clouddn.com/monitor-go-3.png)

登陆成功后，进入主页面

![go-monitor-4](http://og0usnhfv.bkt.clouddn.com/monitor-go-4.png)

在上面的主页面，点击一下 “Add data source”的绿色按钮，点击后，进入创建数据源的信息填写表单：

![go-monitor-5](http://og0usnhfv.bkt.clouddn.com/monitor-go-5.png)

现在，我们填写我们influxDB的一些关键信息，如上图，然后点击“保存” ，这样我们在Grafana就完成了数据源的关联工作。

在进一步配置我们的监控面板之前，我们先实现我们指标监控测试代码

数据埋点
----

我们先实现一个用于测试的代码， 本代码是模拟在随机的时间内申请内存，用于参数内存和GC等监控指标。

这些指标会发送到influxDB中，最后，我们可以再Grafana中完成对指标数据的监控观察。

    package main
    
    import (
        "fmt"
        "math/rand"
        "time"
    
        "github.com/rcrowley/go-metrics"
        "github.com/vrischmann/go-metrics-influxdb"
    )
    
    //使用内存
    func useMemory() {
        a := rand.Intn(100)
        m := make([]int, a*1e6)
    
        d := time.Second * time.Duration(rand.Intn(15))
        fmt.Printf("Generated something with size %dMB, sleeping for %s\n", len(m)/1e6, d)
        time.Sleep(d)
        fmt.Printf("Done using %dMB\n", len(m)/1e6)
    }
    
    func main() {
    
        r := metrics.NewRegistry()
        metrics.RegisterDebugGCStats(r)
        metrics.RegisterRuntimeMemStats(r)
    
        go metrics.CaptureDebugGCStats(r, time.Second*5)
        go metrics.CaptureRuntimeMemStats(r, time.Second*5)
    
        go influxdb.InfluxDB(
            r,                             // metrics registry
            time.Second*5,                 // 时间间隔
            "http://192.168.139.140:8086", // InfluxDB url
            "helloWorld",                  // InfluxDB 数据库名
            "",                            // InfluxDB user
            "",                            // InfluxDB password
        )
    
        time.Sleep(time.Second * 10)
    
        for {
            go useMemory()
            time.Sleep(time.Duration(rand.Intn(15)) * time.Second)
        }
    }
    
    

我们在终端启动我们的测试代码：

    go run main.go
    

创建监控面板
------

一旦我们的DataSource完成配置后，在Grafana我们可以开始造一个监控面板，我们继续点击左上角的logo图案，在Dashboards中 选择`New`,来创建一个新的panel，如下图：

![go-monitor-6](http://og0usnhfv.bkt.clouddn.com/monitor-go-6.png)

然后我们选择“Graph”，如下图：

![go-monitor-7](http://og0usnhfv.bkt.clouddn.com/monitor-go-7.png)

当panel出现后，我们鼠标点击标题，当出现一个选择创建的时候，我们继续选择“Edit”, 如下图：

![go-monitor-8](http://og0usnhfv.bkt.clouddn.com/monitor-go-8.png)

接下来，我们需要创建查询Query的组合语句，如下图

![go-monitor-10](http://og0usnhfv.bkt.clouddn.com/monitor-go-10.png)

注意：如果我们想更多了解我们有哪些的指标项，我们还可以再浏览器进行相关查询，例如：

![go-monitor-9](http://og0usnhfv.bkt.clouddn.com/monitor-go-9.png)

再设置X轴和Y轴的数据展示单位

![go-monitor-11](http://og0usnhfv.bkt.clouddn.com/monitor-go-11.png)

上述主要的设置完成后，保存并刷新页面，可以看到我们的监控数据已经出现在我们的监控面板上：

![go-monitor-12](http://og0usnhfv.bkt.clouddn.com/monitor-go-12.png)
