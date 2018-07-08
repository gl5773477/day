
[test](./技术大纲)
[引](https://skyao.gitbooks.io/learning-etcd3/content/documentation/dev-guide/interacting_v3.html)
[很不错很全集群配置+注释](http://blog.csdn.net/yasonan/article/details/54945066)

# 用途及优点
1. 服务注册与发现
> 随着近来微服务概念的提出，服务的分解必然引入大量的服务之间的相互调用，而传统的服务调用一般通过配置文件读取ip进行调用，这里有诸多限制，如不灵活，无法感知服务的状态，实现服务调用负载均衡复杂等缺点，而引入etcd后，问题将大大化简，这里划分为几个步骤
- 服务启动后向etcd注册，并上报自己的监听的端口以及当前的权重因子等信息，且对该信息设置ttl值。
服务在ttl的时间内周期性上报权重因子等信息。
- client端调用服务时向etcd获取信息，进行调用，同时监听该服务是否变化(通过watch方法实现)。
- 当新增服务时watch方法监听到变化，将服务加入代用列表，当服务挂掉时ttl失效，client端检测到变化，将服务踢出调用列表，从而实现服务的动态扩展。
- 另一方面，client端通过每次变化获取到的权重因子来进行client端的加权调用策略，从而保证后端服务的负载均衡
2. 共享配置
> 一般服务启动时需要加载一些配置信息，如数据库访问地址，连接配置，这些配置信息每个服务都差不多，如果通过读取配置文件进行配置会存在要写多份配置文件，且每次更改时这些配置文件都要更改，且更改配置后，需要重启服务后才能生效，这些无疑让配置极不灵活，如果将配置信息放入到etcd中，程序启动时进行加载并运行，同时监听配置文件的更改，当配置文件发生更改时，自动将旧值替换新值，这样无疑简化程序配置，更方便于服务部署。
3. 优点
> 相比于zookeeper内部使用paxos算法保证一致性，而etcd则使用更简单的raft算法。另一方面，相比zookeeper, etcd算是新兴项目，因而在知名度及使用量来说和zookeeper有一定差距，不过,随着越来越多的知名项目在逐渐使用etcd(如docker),相信etcd将会在未来更多的出现在各种服务体系中。

## 版本
版本不同，命令也不相同。切换API版本到3：`export ETCDCTL_API=3`
## 存取kv
- put、get
- 指定历史版本：--rev=1
- 查看所有：`etcdctl get --prefix=true ""`
- put，即Add操作不会覆盖之前的信息，因为roundRobin会检测是否已存在，若存在则绕过。
## 监控
- watch
- 前缀匹配：--prefix
- 从历史版本监控所有变化：--rev=1
## 压缩历史版本
## 租约TTL
租约的意义：一旦程序不能维持ttl心跳，即租约到期不能续约，则服务节点摘除。
- 授予
- 撤销
- 维持续约
## 分布式锁
它是一个grpc接口，可以用来确保只有持有锁时才能更新etcd。


# 集群
### 启动方式
- 静态配置
- etcd发现
- dns发现

### 基本命令
- 查看集群成员：`./etcdctl  member list`
- 集群健康状态：`./etcdctl cluster-health`

etcd --name infra0 --initial-advertise-peer-urls http://192.168.0.105:2380 \
  --listen-peer-urls http://192.168.0.105:2380 \
  --listen-client-urls http://192.168.0.105:2379,http://127.0.0.1:2379 \
  --advertise-client-urls http://192.168.0.105:2379 \
  --discovery http://localhost:2379/v2/keys/discovery/6c007a14875d53d9bf0ef5a6fc0257c817f0fb83

etcd --name infra0 --initial-advertise-peer-urls http://192.168.0.105:2380 \
  --listen-peer-urls http://192.168.0.105:2380 \
  --listen-client-urls http://192.168.0.105:2379,http://127.0.0.1:2379 \
  --advertise-client-urls http://192.168.0.105:2379 \
  --initial-cluster-token etcd-cluster-1 \
  --initial-cluster infra0=http://192.168.0.105:2380,infra1=http://192.168.0.105:12380,infra2=http://192.168.0.105:22380 \
  --initial-cluster-state new


  #Human-readable name for this member.
--name: 'etcd1'    // 建议起名字

# Path to the data directory.
data-dir: "/var/lib/etcd/etcd0.etcd"   //一般是这个，可也以使用默认 "/var/lib/etcd/default.etcd"

# Path to the dedicated wal directory.
wal-dir:

# Number of committed transactions to trigger a snapshot to disk.
snapshot-count: 10000

# Time (in milliseconds) of a heartbeat interval.
heartbeat-interval: 100

# Time (in milliseconds) for an election to timeout.
election-timeout: 1000

# Raise alarms when backend size exceeds the given quota. 0 means use the
# default quota.
quota-backend-bytes: 0

# List of comma separated URLs to listen on for peer traffic.
listen-peer-urls: http://192.168.0.105:2380  //操作电脑ip

# List of comma separated URLs to listen on for client traffic.
listen-client-urls: http://192.168.0.105:2379   //操作电脑ip

# Maximum number of snapshot files to retain (0 is unlimited).
max-snapshots: 5

# Maximum number of wal files to retain (0 is unlimited).
max-wals: 5

# Comma-separated white list of origins for CORS (cross-origin resource sharing).
cors:

# List of this member's peer URLs to advertise to the rest of the cluster.
# The URLs needed to be a comma-separated list.
initial-advertise-peer-urls: http://192.168.0.105:2380

# List of this member's client URLs to advertise to the public.//操作电脑ip
# The URLs needed to be a comma-separated list.
advertise-client-urls: http://192.168.0.105:2379  //操作电脑ip

# Discovery URL used to bootstrap the cluster.
discovery:

# Valid values include 'exit', 'proxy'
discovery-fallback: 'proxy'

# HTTP proxy to use for traffic to discovery service.
discovery-proxy:

# DNS domain used to bootstrap initial cluster.
discovery-srv:

# Initial cluster configuration for bootstrapping.
initial-cluster: "etcd1=http://192.168.0.105:2380,etcd2=http://192.168.0.105:12380,etcd3=http://192.168.0.105:22380"                  //要集群的机子的IP，与前面起的名字一致

# Initial cluster token for the etcd cluster during bootstrap.
initial-cluster-token: 'etcd-cluster'     //集群token ，三台机子一样

# Initial cluster state ('new' or 'existing').
initial-cluster-state: 'new'    //集群状态为  new/existing  第一次new

# Reject reconfiguration requests that would cause quorum loss.
strict-reconfig-check: false

# Accept etcd V2 client requests
enable-v2: true

# Valid values include 'on', 'readonly', 'off'
proxy: 'off'

# Time (in milliseconds) an endpoint will be held in a failed state.
proxy-failure-wait: 5000

# Time (in milliseconds) of the endpoints refresh interval.
proxy-refresh-interval: 30000

# Time (in milliseconds) for a dial to timeout.
proxy-dial-timeout: 1000

# Time (in milliseconds) for a write to timeout.
proxy-write-timeout: 5000

# Time (in milliseconds) for a read to timeout.
proxy-read-timeout: 0

client-transport-security:
  # DEPRECATED: Path to the client server TLS CA file.
  ca-file:

  # Path to the client server TLS cert file.
  cert-file:

  # Path to the client server TLS key file.
  key-file:

  # Enable client cert authentication.
  client-cert-auth: false

  # Path to the client server TLS trusted CA key file.
  trusted-ca-file:

  # Client TLS using generated certificates
  auto-tls: false

peer-transport-security:
  # DEPRECATED: Path to the peer server TLS CA file.
  ca-file:

  # Path to the peer server TLS cert file.
  cert-file:

  # Path to the peer server TLS key file.
  key-file:

  # Enable peer client cert authentication.
  client-cert-auth: false

  # Path to the peer server TLS trusted CA key file.
   trusted-ca-file:

  # Peer TLS using generated certificates.
  auto-tls: false

# Enable debug-level logging for etcd.
debug: false

# Specify a particular log level for each etcd package (eg: 'etcdmain=CRITICAL,etcdserver=DEBUG'.
log-package-levels:

# Force to create a new one member cluster.
force-new-cluster: false



TOKEN=token-03
CLUSTER_STATE=new
NAME_1=machine-1
NAME_2=machine-2
NAME_3=machine-3
PORT_1=12379
PORT_2=22379
PORT_3=32379
HOST_1=192.168.0.105
HOST_2=192.168.0.105
HOST_3=192.168.0.105
CLUSTER=${NAME_1}=http://${HOST_1}:12380,${NAME_2}=http://${HOST_2}:22380,${NAME_3}=http://${HOST_3}:32380

THIS_NAME=${NAME_1}
THIS_IP=${HOST_1}
etcd --data-dir=/opt/data1.etcd --name ${THIS_NAME} \
	--initial-advertise-peer-urls http://${THIS_IP}:12380 --listen-peer-urls http://${THIS_IP}:12380 \
	--advertise-client-urls http://${THIS_IP}:${PORT_1} --listen-client-urls http://${THIS_IP}:${PORT_1} \
	--initial-cluster ${CLUSTER} \
	--initial-cluster-state ${CLUSTER_STATE} --initial-cluster-token ${TOKEN}

THIS_NAME=${NAME_2}
THIS_IP=${HOST_2}
etcd --data-dir=/opt/data2.etcd --name ${THIS_NAME} \
	--initial-advertise-peer-urls http://${THIS_IP}:22380 --listen-peer-urls http://${THIS_IP}:22380 \
	--advertise-client-urls http://${THIS_IP}:${PORT_2} --listen-client-urls http://${THIS_IP}:${PORT_2} \
	--initial-cluster ${CLUSTER} \
	--initial-cluster-state ${CLUSTER_STATE} --initial-cluster-token ${TOKEN}

THIS_NAME=${NAME_3}
THIS_IP=${HOST_3}
etcd --data-dir=/opt/data3.etcd --name ${THIS_NAME} \
	--initial-advertise-peer-urls http://${THIS_IP}:32380 --listen-peer-urls http://${THIS_IP}:32380 \
	--advertise-client-urls http://${THIS_IP}:${PORT_3} --listen-client-urls http://${THIS_IP}:${PORT_3} \
	--initial-cluster ${CLUSTER} \
	--initial-cluster-state ${CLUSTER_STATE} --initial-cluster-token ${TOKEN}

export ETCDCTL_API=3
ENDPOINTS=$HOST_1:${PORT_1},$HOST_2:${PORT_2},$HOST_3:${PORT_3}


#### 使用已有etcd服务创建用于etcd发现的url：
`curl -X PUT http://localhost:2379/v2/keys/discovery/6c007a14875d53d9bf0ef5a6fc0257c817f0fb85/_config/size -d value=3`
`curl http://localhost:2379/v2/members`

TOKEN=token-05
SERVICE_TOKEN=6c007a14875d53d9bf0ef5a6fc0257c817f0fb85
rm -fr /opt/data1.etcd
<!-- DISCOVERY=http://localhost:2379/v2/keys/discovery/${SERVICE_TOKEN} -->
DISCOVERY=https://discovery.etcd.io/5de752c0aa47cda470aad2d52ebd7e40
etcd --data-dir=/opt/data1.etcd --name ${THIS_NAME} \
	--initial-advertise-peer-urls http://${THIS_IP}:12380 --listen-peer-urls http://${THIS_IP}:12380 \
	--advertise-client-urls http://${THIS_IP}:${PORT_1} --listen-client-urls http://${THIS_IP}:${PORT_1} \
	--discovery ${DISCOVERY} \
	--initial-cluster-state ${CLUSTER_STATE} --initial-cluster-token ${TOKEN}

TOKEN=token-05
SERVICE_TOKEN=6c007a14875d53d9bf0ef5a6fc0257c817f0fb85
rm -fr /opt/data2.etcd
<!-- DISCOVERY=http://localhost:2379/v2/keys/discovery/${SERVICE_TOKEN} -->
DISCOVERY=https://discovery.etcd.io/5de752c0aa47cda470aad2d52ebd7e40
etcd --data-dir=/opt/data2.etcd --name ${THIS_NAME} \
	--initial-advertise-peer-urls http://${THIS_IP}:22380 --listen-peer-urls http://${THIS_IP}:22380 \
	--advertise-client-urls http://${THIS_IP}:${PORT_2} --listen-client-urls http://${THIS_IP}:${PORT_2} \
	--discovery ${DISCOVERY} \
	--initial-cluster-state ${CLUSTER_STATE} --initial-cluster-token ${TOKEN}

TOKEN=token-05
SERVICE_TOKEN=6c007a14875d53d9bf0ef5a6fc0257c817f0fb85
rm -fr /opt/data3.etcd
<!-- DISCOVERY=http://localhost:2379/v2/keys/discovery/${SERVICE_TOKEN} -->
DISCOVERY=https://discovery.etcd.io/5de752c0aa47cda470aad2d52ebd7e40
etcd --data-dir=/opt/data3.etcd --name ${THIS_NAME} \
	--initial-advertise-peer-urls http://${THIS_IP}:32380 --listen-peer-urls http://${THIS_IP}:32380 \
	--advertise-client-urls http://${THIS_IP}:${PORT_3} --listen-client-urls http://${THIS_IP}:${PORT_3} \
	--discovery ${DISCOVERY} \
	--initial-cluster-state ${CLUSTER_STATE} --initial-cluster-token ${TOKEN}


#### member异常恢复拉起
etcd --name NAME_1 \
--data-dir=/opt/data1.etcd \
--listen-peer-urls http://${THIS_IP}:12380 \
--listen-client-urls http://${THIS_IP}:${PORT_1} \
--advertise-client-urls http://${THIS_IP}:${PORT_1}

etcd --name NAME_2 \
--listen-peer-urls http://${THIS_IP}:22380 \
--listen-client-urls http://${THIS_IP}:${PORT_2} \
--advertise-client-urls http://${THIS_IP}:${PORT_2}

etcd --name NAME_3 \
--listen-peer-urls http://${THIS_IP}:32380 \
--listen-client-urls http://${THIS_IP}:${PORT_3} \
--advertise-client-urls http://${THIS_IP}:${PORT_3}

#### 添加、删除、更新
- get member ID
etcdctl --endpoints=${ENDPOINTS} member list

假设old memberid:a8266ecf031671f3，新的url：http://10.0.1.10:2380
- `etcdctl --endpoints=${ENDPOINTS} member update 5ac410fee041431b http://${THIS_IP}:2380`

如果删除的是 leader 节点，则需要耗费额外的时间重新选举 leader。
- `etcdctl --endpoints=${ENDPOINTS} member remove 5ac410fee041431b`


- `etcdctl member add ${NAME_3} http://${THIS_IP}:32380`
```shell
# etcdctl 在注册完新节点后，会返回一段提示，包含3个环境变量。
# 然后在第二部启动新节点的时候，带上这3个环境变量即可。实际上如果使用发现的方式则不必。
export ETCD_NAME="machine-3"
export ETCD_INITIAL_CLUSTER="machine-3=http://192.168.0.105:32380,default=http://localhost:2380"
export ETCD_INITIAL_CLUSTER_STATE="existing"

etcd --data-dir=/opt/data3.etcd \
--listen-client-urls http://${THIS_IP}:${PORT_3} \
--advertise-client-urls http://${THIS_IP}:${PORT_3} \
--listen-peer-urls http://${THIS_IP}:32380 \
--initial-advertise-peer-urls http://${THIS_IP}:32380 \
```

etcdctl --endpoints=${ENDPOINTS} \
	member add ${NAME_2} \
	--peer-urls=http://${HOST_2}:22380

#### etcd集群运行过程中的改配
使用 --force-new-cluster 参数启动Etcd服务。这个参数会重置集群ID和集群的所有成员信息，其中节点的监听地址会被重置为localhost:2379, 表示集群中只有一个节点。

主要用于故障节点替换，集群扩容需求。

集群运行过程中的改配不区分static和discovery方式。

1、替换的步骤：

比如集群中某一member重启后仍不能恢复时，就需要替换一个新member进来。

a、从集群中删除老member

etcdctl member remove 1609b5a3a078c227

b、向集群中新增新member

etcdctl member add <name> <peerURL> 
例子：敲命令 etcdctl member add etcd2 http://192.168.2.56:2380  后返回如下信息
Added member named etcd2 with ID b7d510356ee2e68b to cluster

ETCD_NAME="etcd2"
ETCD_INITIAL_CLUSTER="etcd0=http://192.168.2.55:2380,etcd2=http://192.168.2.56:2380,etcd1=http://192.168.2.54:2380"
ETCD_INITIAL_CLUSTER_STATE="existing"

c、删除老节点data目录
如果不删除，启动后节点仍旧沿用之前的老id， 其他正常节点不认，不能建立联系

d、在新member上启动etcd进程

#### 注意：
1. 当采用发现的方式时，只有当集群的节点数真的达到了size个数量，即有size个etcd节点在服务时，集群才是健康的，否则，虽然能用，但是会报告不健康。这个直接用静态配置方式启动是类似的，好处仅仅是不用指定ip而已。且不推荐后续扩容时使用discovery的方式，有很多问题，官方已指明。
2. 尽量使用节点更新，即新起一个新的etcd，并更新到某个老的上面去。而不是直接扩容（之所以说扩容是因为集群健康时已经是满size了）。或者每一个添加最好对应一个删除。
