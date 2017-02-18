---
title: 基于consul构建golang系统分布式服务发现机制
layout: post
tags:
  - 服务治理
category: golang
---

------------


在分布式架构中，服务治理是一个重要的问题。在没有服务治理的分布式集群中，各个服务之间通过手工或者配置的方式进行服务关系管理，遇到服务关系变化或者增加服务的时候，人肉配置极其麻烦且容易出错。

之前在一个C/C++项目中，采用ZooKeeper进行服务治理，可以很好的维护服务之间的关系，但是使用起来较为麻烦。现在越来越多新的项目采用consul进行服务治理，各方面的评价都优于ZooKeeper，经过几天的研究，这里做一个总结。

#### zookeeper和consul比较

- 开发语言方面，zookeeper采用java开发，安装的时候需要部署java环境；consul采用golang开发，所有依赖都编译到了可执行程序中，即插即用。
- 部署方面，zookeeper一般部署奇数个节点方便做简单多数的选举机制。consul部署的时候分server节点和client节点（通过不同的启动参数区分），server节点做leader选举和数据一致性维护，client节点部署在服务机器上，作为服务程序访问consul的接口。
- 一致性协议方面，zookeeper使用自定义的zab协议，consul的一致性协议采用更流行的Raft。
- zookeeper不支持多数据中心，consul可以跨机房支持多数据中心部署，有效避免了单数据中心故障不能访问的情况。
- 链接方式上，zookeeper client api和服务器保持长连接，需要服务程序自行管理和维护链接有效性，服务程序注册回调函数处理zookeeper事件，并自己维护在zookeeper上建立的目录结构有效性（如临时节点维护）；consul 采用DNS或者http获取服务信息，没有主动通知，需要自己轮训获取。
- 工具方面，zookeeper自带一个cli_mt工具，可以通过命令行登录zookeeper服务器，手动管理目录结构。consul自带一个Web UI管理系统， 可以通过参数启动并在浏览器中直接查看信息。

#### consul相关资源

可执行程序下载地址：https://www.consul.io/downloads.html
官方说明文档: https://www.consul.io/docs/index.html
api说明文档： https://godoc.org/github.com/hashicorp/consul/api#pkg-index
golang api代码位置：github.com/hashicorp/consul/api

linux系统中，下载consul可执行程序后直接拷贝到/usr/local/bin就可以使用了，无需其他额外配置。

服务节点启动方式： 

 	consul agent -server -bootstrap-expect 1 -data-dir /tmp/consul -node=service-center -bind=192.168.0.2 -client 0.0.0.0 -ui -config-dir /etc/consul.d/

参数说明：
	
	-server 表示以server节点模式启动consul
	-bootstrap-expect 1 表示期待的server节点一共有几个，如3个server集群模式
	-data-dir consul存储数据的目录
	-node 节点的名字
	-bind 绑定的服务ip
	-client 0.0.0.0 -ui 启动Web UI管理工具
	-config-dir 指定服务配置文件的目录（这个目录下的所有.json文件，作为服务配置文件读取）

#### consul服务发现机制测试
为了测试consul服务治理方式，设定如下场景：
	
	一个manager类型的服务，需要根据负载来管理若干worker类型的服务并进行业务通信；而worker服务也需要知道manager提供的内部服务接口地址做业务交互。即manger和worker都需要互相知道对方的通信地址。
	
做如下规则设定准备：

	manager和worker都需要向consul注册自己的服务，让对方发现自己的服务地址（ip和端口）
	采用consul的key-value存储机制，worker周期性更新自己的负载信息到相应的key；manger从worker的key中获取负载信息，并同步更新到本地。
	服务类型规则： manager的服务类型用字符串"manager"表示，各个worker的服务类型采用字符串"worker"表示。
	服务注册ID规则： 服务类型-服务IP，如 manager-192.168.0.2
	key的构建规则: 服务类型/IP:Port, 如 worker/192.168.0.2:5400
	存储的数据采用json格式：{"load":100,"ts":1482828232}

#### golang测试程序

```go
package main

import (
	"encoding/json"
	"flag"
	"fmt"
	"github.com/hashicorp/consul/api"
	"log"
	"math/rand"
	"net/http"
	"os"
	"os/signal"
	"strconv"
	"strings"
	"sync"
	"time"
)

type ServiceInfo struct {
	ServiceID string
	IP        string
	Port      int
	Load      int
	Timestamp int //load updated ts
}
type ServiceList []ServiceInfo

type KVData struct {
	Load      int `json:"load"`
	Timestamp int `json:"ts"`
}

var (
	servics_map     = make(map[string]ServiceList)
	service_locker  = new(sync.Mutex)
	consul_client   *api.Client
	my_service_id   string
	my_service_name string
	my_kv_key       string
)

func CheckErr(err error) {
	if err != nil {
		log.Printf("error: %v", err)
		os.Exit(1)
	}
}
func StatusHandler(w http.ResponseWriter, r *http.Request) {
	fmt.Println("check status.")
	fmt.Fprint(w, "status ok!")
}

func StartService(addr string) {
	http.HandleFunc("/status", StatusHandler)
	fmt.Println("start listen...")
	err := http.ListenAndServe(addr, nil)
	CheckErr(err)
}

func main() {
	var status_monitor_addr, service_name, service_ip, consul_addr, found_service string
	var service_port int
	flag.StringVar(&consul_addr, "consul_addr", "localhost:8500", "host:port of the service stuats monitor interface")
	flag.StringVar(&status_monitor_addr, "monitor_addr", "127.0.0.1:54321", "host:port of the service stuats monitor interface")
	flag.StringVar(&service_name, "service_name", "worker", "name of the service")
	flag.StringVar(&service_ip, "ip", "127.0.0.1", "service serve ip")
	flag.StringVar(&found_service, "found_service", "worker", "found the target service")
	flag.IntVar(&service_port, "port", 4300, "service serve port")
	flag.Parse()

	my_service_name = service_name

	DoRegistService(consul_addr, status_monitor_addr, service_name, service_ip, service_port)

	go DoDiscover(consul_addr, found_service)

	go StartService(status_monitor_addr)

	go WaitToUnRegistService()

	go DoUpdateKeyValue(consul_addr, service_name, service_ip, service_port)

	select {}
}

func DoRegistService(consul_addr string, monitor_addr string, service_name string, ip string, port int) {
	my_service_id = service_name + "-" + ip
	var tags []string
	service := &api.AgentServiceRegistration{
		ID:      my_service_id,
		Name:    service_name,
		Port:    port,
		Address: ip,
		Tags:    tags,
		Check: &api.AgentServiceCheck{
			HTTP:     "http://" + monitor_addr + "/status",
			Interval: "5s",
			Timeout:  "1s",
		},
	}

	client, err := api.NewClient(api.DefaultConfig())
	if err != nil {
		log.Fatal(err)
	}
	consul_client = client
	if err := consul_client.Agent().ServiceRegister(service); err != nil {
		log.Fatal(err)
	}
	log.Printf("Registered service %q in consul with tags %q", service_name, strings.Join(tags, ","))
}

func WaitToUnRegistService() {
	quit := make(chan os.Signal, 1)
	signal.Notify(quit, os.Interrupt, os.Kill)
	<-quit

	if consul_client == nil {
		return
	}
	if err := consul_client.Agent().ServiceDeregister(my_service_id); err != nil {
		log.Fatal(err)
	}
}

func DoDiscover(consul_addr string, found_service string) {
	t := time.NewTicker(time.Second * 5)
	for {
		select {
		case <-t.C:
			DiscoverServices(consul_addr, true, found_service)
		}
	}
}

func DiscoverServices(addr string, healthyOnly bool, service_name string) {
	consulConf := api.DefaultConfig()
	consulConf.Address = addr
	client, err := api.NewClient(consulConf)
	CheckErr(err)

	services, _, err := client.Catalog().Services(&api.QueryOptions{})
	CheckErr(err)

	fmt.Println("--do discover ---:", addr)

	var sers ServiceList
	for name := range services {
		servicesData, _, err := client.Health().Service(name, "", healthyOnly,
			&api.QueryOptions{})
		CheckErr(err)
		for _, entry := range servicesData {
			if service_name != entry.Service.Service {
				continue
			}
			for _, health := range entry.Checks {
				if health.ServiceName != service_name {
					continue
				}
				fmt.Println("  health nodeid:", health.Node, " service_name:", health.ServiceName, " service_id:", health.ServiceID, " status:", health.Status, " ip:", entry.Service.Address, " port:", entry.Service.Port)

				var node ServiceInfo
				node.IP = entry.Service.Address
				node.Port = entry.Service.Port
				node.ServiceID = health.ServiceID

				//get data from kv store
				s := GetKeyValue(service_name, node.IP, node.Port)
				if len(s) > 0 {
					var data KVData
					err = json.Unmarshal([]byte(s), &data)
					if err == nil {
						node.Load = data.Load
						node.Timestamp = data.Timestamp
					}
				}
				fmt.Println("service node updated ip:", node.IP, " port:", node.Port, " serviceid:", node.ServiceID, " load:", node.Load, " ts:", node.Timestamp)
				sers = append(sers, node)
			}
		}
	}

	service_locker.Lock()
	servics_map[service_name] = sers
	service_locker.Unlock()
}

func DoUpdateKeyValue(consul_addr string, service_name string, ip string, port int) {
	t := time.NewTicker(time.Second * 10)
	for {
		select {
		case <-t.C:
			StoreKeyValue(consul_addr, service_name, ip, port)
		}
	}
}

func StoreKeyValue(consul_addr string, service_name string, ip string, port int) {

	my_kv_key = my_service_name + "/" + ip + ":" + strconv.Itoa(port)

	var data KVData
	data.Load = rand.Intn(100)
	data.Timestamp = int(time.Now().Unix())
	bys, _ := json.Marshal(&data)

	kv := &api.KVPair{
		Key:   my_kv_key,
		Flags: 0,
		Value: bys,
	}

	_, err := consul_client.KV().Put(kv, nil)
	CheckErr(err)
	fmt.Println(" store data key:", kv.Key, " value:", string(bys))
}

func GetKeyValue(service_name string, ip string, port int) string {
	key := service_name + "/" + ip + ":" + strconv.Itoa(port)

	kv, _, err := consul_client.KV().Get(key, nil)
	if kv == nil {
		return ""
	}
	CheckErr(err)

	return string(kv.Value)
}
```
程序通过参数控制自己启动的服务角色类型和需要发现的服务类型。传入的consul_addr是本机consul client agent的地址，一般是loacalhost:8500 。 由于consul集成了服务健康检查，所以服务需要启动一个检查接口，这里启动一个http服务来做响应。

#### consul集群启动

启动3个consul server :

	consul agent -server -bootstrap-expect 3 -data-dir /tmp/consul -node=server001 -bind=10.2.1.54
	consul agent -server  -data-dir /tmp/consul -node=server002 -bind=10.2.1.83 -join 10.2.1.54
	consul agent -server  -data-dir /tmp/consul -node=server003 -bind=10.2.1.80 -join 10.2.1.54

server001-003构成了一个3个server node的consul集群。先启动server001，并指定需要3个server node构成集群，server002和server003启动的时候指定加入（-join）server001.

启动一个manger：

	consul agent -data-dir /tmp/consul -node=mangaer -bind=10.2.1.92  -join 10.2.1.54
	./service -consul_addr=127.0.0.1:8500 -monitor_addr=127.0.0.1:54321 -service_name=manager -ip=10.2.1.92 -port=4300 -found_service=worker

启动2个worker:

	consul agent -data-dir /tmp/consul -node=worker001 -bind=10.2.1.93  -join 10.2.1.54
	./service -consul_addr=127.0.0.1:8500 -monitor_addr=127.0.0.1:54321 -service_name=worker -ip=10.2.1.93 -port=4300 -found_service=manager
	consul agent -data-dir /tmp/consul -node=worker002 -bind=10.2.1.94  -join 10.2.1.54
	./service -consul_addr=127.0.0.1:8500 -monitor_addr=127.0.0.1:54321 -service_name=worker -ip=10.2.1.94 -port=4300 -found_service=manager

service程序是前面部分代码编译后的测试程序。

这样就构建了3个server node的consul集群，以及1各manager和2个worker的分布式服务程序，他们可以互相发现对方，并且manager可以获取到worker的负载情况，实现了互通。

#### 结束

通过使用consul的服务注册发现机制和key-value存储机制，实现了服务发现以及manager获取worker服务负载数据的机制。由于consul的发现机制不能进行更多的数据交互，所以只能使用key-value机制配合进行数据共享（zookeeper中数据可以存储在节点上）。如果业务有进一步需求，可以方便的扩展存储的数据结构来实现。

以上的测试程序既有服务注册，存储数据更新，也有服务发现和数据获取，但是代码量比zookeeper机制少很多，因为zookeeper需要自己建立和维护目录树，注册和处理zookeeper event事件，监控zookeeper的链接并处理重连和信息重建等健康管理工作。

总的来说，consul比zookeeper使用简单易用很多。可以在新项目中尝试使用，特别是golang项目，技术栈也比较统一。




