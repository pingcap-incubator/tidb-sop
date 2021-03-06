# 06 临时关机维护某线上主机
> 周跃跃  2020 年 2 月 28 日

## 一、背景
> 有些时候，由于硬件维护、搬移机架位等需求，需要主动关闭线上主机，时间间隔可能在几分钟到数小时之间。如果主机上运行着线上 TiDB 集群的某个组件，用户需要如何提前做好准备，减少对线上业务的影响呢。本文描述了临时维护的主机的处理方式，包含 PD，TiKV，TiDB 等各组件。

## 二、PD 组件
> 通常大多数的线上集群有 3 或 5 个 PD 节点，如果维护的主机上有 PD 组件，需要具体考虑节点是 leader 还是 follower（1 和 2 两部分），关闭 follower 对集群运行没有任何影响，关闭 leader 需要先切换，并在切换时有 3 s 左右的性能抖动。特殊情况下如果集群总共只有 1 个 PD，可以参考第 3 部分。

### 1、当前机器包括一个 PD follower 节点且集群 PD 总数 >= 3

* 检查当前待操作 PD 集群节点信息

```
pd-ctl -i -u http://127.0.0.1:2379 >> member //显示当前所有成员
```
* 在待维护机器上执行

```
${deploy_dir}/scripts/stop_pd.sh
```

* 关机维护

* 启动机器

* 登陆机器执行

```
${deploy_dir}/scripts/start_pd.sh
```
* 检查当前待操作 PD 集群节点信息

### 2、当前机器包括一个 PD leader 节点且集群 PD 总数 >= 3
* 检查当前待操作 PD 集群节点信息

```
pd-ctl -i -u http://127.0.0.1:2379 >> member //显示当前所有成员
```

* 检查当前待操作 PD 节点角色

```
pd-ctl -i -u http://127.0.0.1:2379 >> mebmer leader show //显示 leader 的信息
```

* 迁移 leader 节点

```
pd-ctl -i -u http://127.0.0.1:2379 >> member leader transfer pd2 // 将 leader 迁移到指定成员
```

* 检查迁移结果

```
pd-ctl -i -u http://127.0.0.1:2379 >> member //显示当前所有成员，迁移成功进行下一步，否则等待
```
* 在待维护机器上执行

```
${deploy_dir}/scripts/stop_pd.sh
```

* 关机维护

* 启动机器

* 在维护后机器上执行

```
${deploy_dir}/scripts/start_pd.sh
```

* leader 迁回（可选）

```
pd-ctl -i -u http://127.0.0.1:2379 >> member leader transfer pd1 // 将 leader 迁移到指定成员
检查当前待操作 PD 集群节点信息
```

### 3、当前机器包括一个 PD leader 节点且集群 PD 总数 = 1

在临时关机维护时，针对集群不同情况的需求，可分为三种处理方式

##### (1) 线上环境，服务需要高可用，不接受停服务
* 如确实需要高可用，需要在其他机器扩容至三个 PD ，参考 [PD 节点库容](https://pingcap.com/docs-cn/stable/how-to/scale/with-ansible/#%E6%89%A9%E5%AE%B9-tidbtikv-%E8%8A%82%E7%82%B9)
* 临时关机维护，参考 2 （当前机器包括一个 PD leader 节点且集群 PD 总数 >= 3 ）

##### (2) 线上或测试环境，不需要服务高可用或者资源紧张，可接受维护期间服务不可用

* 中控机执行

```
ansible-playbook stop.yml --tags=pd  （只停 PD，操作少，但客户端会报异常，不影响正确性）
```
或者

```
ansible-playbook stop.yml （停整个集群）
```
* 关机维护
* 启动机器
* 中控机执行

```
ansible-playbook start.yml --tags=pd  
```
或者 

```
ansible-playbook start.yml 
```

* 检查当前待操作 PD 集群节点信息

##### （3）线上或测试环境，不需要服务高可用或者资源紧张，不接受维护期间服务不可用

* 新增一个 PD 节点

 *   参考 [PD 节点扩容](https://pingcap.com/docs-cn/stable/how-to/scale/with-ansible/#%E6%89%A9%E5%AE%B9-tidbtikv-%E8%8A%82%E7%82%B9)

* 检查 PD 集群节点信息

```
pd-ctl -i -u http://127.0.0.1:2379 >> member //显示当前所有成员
```

* 迁移 PD leader 节点至新增 PD 节点

```
pd-ctl -i -u http://127.0.0.1:2379 >> member leader transfer pd2 // 将 leader 迁移到指定成员
```

* 检查迁移结果

```
pd-ctl -i -u http://127.0.0.1:2379 >> member //显示当前所有成员，迁移成功进行下一步，否则等待
```

* 删除待维护机器 PD 节点
	* 参考 [PD 节点缩容](https://pingcap.com/docs-cn/stable/how-to/scale/with-ansible/#%E6%89%A9%E5%AE%B9-tidbtikv-%E8%8A%82%E7%82%B9)

* 检查当前待操作 PD 集群节点信息

* 关机维护
* 启动机器
* 迁回 PD 节点（可选）
> 在维护结束的机器上新增一个 PD 节点，将之前新扩容 leader transfer 至该节点，并删除最开始新增的节点。具体步骤参考：01，02，03，04，05，06


## 三、TiKV 组件
>通常情况下，线上集群对 TiKV 的部署使用，一般是单机部署或者单机多实例，在对机器做临时维护时，需要根据部署情况来进行处理：单机单实例和单机多实例。节点下线过程中 leader 调度对集群的服务影响很小，可忽略。

#####（1）单机多实例临时关机维护步骤

* 检查是否有 label

```
pd-ctl -i -u http://127.0.0.1:2379 >> label //用于显示集群标签信息
```

* 单机多实例，并且有 label 

	* 迁移该机器上所有 store 的 leader
	
	```
	pd-ctl -i -u http://127.0.0.1:2379 >> scheduler add evict-leader-scheduler 1 // 把 store 1 上的所有 region 的 leader 从 store 1 调度出去
	```
	
	* 修改 max-store-down-time 超过服务器维护时间，默认 30 min，保证在服务器维护期间不发生补副本行为
	
	```
pd-ctl -i -u http://127.0.0.1:2379  >> config set max-store-down-time 3h
```

	* 检查 leader 情况

	```
pd-ctl -i -u http://127.0.0.1:2379  >> store store_id // 检查 该机器所有 tikv 节点上的 leader，leader 数量为 0 进行下一步，否则等待为 0
```

	* 在待维护机器上执行
	
	```
	${deploy_dir}/scripts/stop_tikv.sh
	```
	
	* 关机维护
	* 重启机器
	* 在维护后机器上执行
	
	```
	${deploy_dir}/scripts/start_tikv.sh
	```
	
	* 删除 leader 调度策略
	
	```
pd-ctl -i -u http://127.0.0.1:2379  >> scheduler remove grant-leader-scheduler-1
	```
	
* 单机多实例，并且无 label
	* 这种情况属于错误配置，单机多实例必须要有 label，否则可能出现一个 raft group 的 2 个副本出现在一台机器的多个 tikv。这时停机会导致 region 异常。需要先给所有 tikv 补设 label，等 region 调度完成后，再操作关机
	
	```
	pd-ctl -i -u http://127.0.0.1:2379 >> store label {store_id} host &tikvhost (对所有的 store 都要指定 host 这个 label, 确保相同主机的 tikv 相同的 &tikvhost, 另外 "host" 只是 label 的名字可以随意修改)
	pd-ctl -i -u http://127.0.0.1:2379 >> config set location-labels "" 
pd-ctl -i -u http://127.0.0.1:2379 >> region --jq=".regions[] | {id: .id, peer_stores: [.peers[].store_id] | select(any(.==store_id1)) | select(any(.==store_id2)) }"  // 等待调度完成，输出为空，确认没有任何region 在该机器上有 2 个副本，其中 store_id1 和 store_id2 是该机器多个 tikv 的 store id
	```
	* 开始执行下线，同 （02）操作

#####（2）单机单实例关机维护步骤
 > 可直接参考（单机多实例，并且有 label）操作

 
##### 风险点
 
> * 如果写入特点是集中少数一些 region，停机两小时，leader 上堆积的 raft 日志有限。启动的时候这些少数 region 大概率需要重新拿 snapshot。在这种情况下上面的方案没什么风险。
>* 如果写入特点是非常分散的，停机两小时，leader 上堆积的 raft 日志有可能会比较多，取决于写入的流量。这种情况上面的方案有一点点风险，raft log 堆积太多可能会稍微影响下性能。


## 四、TiDB
> 一般情况下，线上使用 TiDB 会搭配负载均衡使用，下面操作步骤基于此实现。

* 从 lvs 把要升级的 TiDB 节点摘除

* 停实例

```
${deploy_dir}/scripts/stop_tidb.sh
```

* 进程停止之后开始关机维护

* 启动实例

```
${deploy_dir}/scripts/start_tidb.sh
```

* 添加 IP 至 lvs

* 使用新添加的 ip & port 连接数据库观察是否有新链接，确认 tidb-server 可正常提供服务

风险点
>在进行 stop tidb  节点时，如果当前节点为 owner 节点（curl http://{TiDBIP}:10080/info
）且正在进行 DDL 变更，直接下掉 tidb 节点会在 60s lease 之后在进行新的 owner 选举，DDL 变更会变慢。另外如果当前节点非 owner 节点，在下掉之后有 DDL 操作时，每个状态变更时也会去访问该下线的节点，会对集群 DDL 操作有影响，该问题 10min 后会消除，因此尽量避免在临时下掉 tidb 时以及期间进行 DDL 操作。


## 五、Binlog
> 在使用 Binlog 组件时，根据部署文档，一般情况下可部署多个 pump ，一个或者多个drainer。

### 1、pump 组件
>在对有部分 pump 组件的主机临时停机维护时，由于在同步 binlog 过程中因为 pump 个数减少，同步操作会变慢，会出现数据延迟问题，特别是当在业务高峰期，停掉部分 pump 来进行主机维护时延迟现象会被放大。因此建议在业务低峰期进行停机维护。
>
* 如果希望操作简单，能接受维护期间可能出现的延迟情况，建议原地停止 pump，即 原地下线 pump
* 如果不能接受延迟，可以新增一台 pump，再停机维护，即新增 pump
 
##### (1) 原地下线 pump
* 查询所有 pump 状态以及分布，确保状态为 online

```
bin/binlogctl -pd-urls=http://127.0.0.1:2379 -cmd pumps
```

* 下线指定的 pump

```
bin/binlogctl -pd-urls=http://127.0.0.1:2379 -cmd offline-pump -node-id ${ip}:${port}
```

* 查询 pump 状态

	```
bin/binlogctl -pd-urls=http://127.0.0.1:2379 -cmd pumps 
	```
	
	* 当待维护主机上 pump 状态变为 offline 时，进行下一步，否则等待
	
* 停掉 pump 进程

```
ansible-playbook stop.yml -l pump pump_name
```

* 关机维护
* 重启机器
* 重新上线 pump
	* 在中控机执行	 

	``` 
 ansible-playbook start.yml --tags=pump -l pump_name
	```
 
	* 查询所有 pump 状态 为 online

	```
bin/binlogctl -pd-urls=http://127.0.0.1:2379 -cmd pumps
	```
##### (2) 新增 pump 
* 查询所有 pump 状态以及分布

```
bin/binlogctl -pd-urls=http://127.0.0.1:2379 -cmd pumps
```

* 新增 pump [参考文档](https://pingcap.com/docs-cn/stable/reference/tidb-binlog/deploy/)

* 查询所有 pump 状态

```
bin/binlogctl -pd-urls=http://127.0.0.1:2379 -cmd pumps
```

* 按照 (1) 原地下线 pump  完成维护主机
* 按需缩容掉任意一个 pump。编辑 inventory.ini 文件，移除 pump 节点信息

```
pump_name  ansible_host=xxx  ansible_port=xxx deploy_dir=/data/pump
```

* 更新 Prometheus 配置并重启

```
ansible-playbook rolling_update_monitor.yml 
```

* 查询所有 pump 状态 为 online

```
bin/binlogctl -pd-urls=http://127.0.0.1:2379 -cmd pumps
```

### 2、drainer
> Drainer 从各个 Pump 中收集 Binlog 进行归并，再将 Binlog 转化成 SQL 或者指定格式的数据，最终同步到下游。当包含 drainer 的机器需要关机维护，对当前的数据同步肯定会有影响。
>
* 如果希望操作简单，能接受维护期间暂停复制，建议原地停止 drainer，即原地停止 drainer。 
* 如果不能接受复制暂停，可以用一个新的 drainer 替换，再停机维护，即替换 drainer

#####（1）原地停止 drainer
* 查询所有 drainer 状态以及分布

```
bin/binlogctl -pd-urls=http://127.0.0.1:2379 -cmd drainer
```

* 暂停指定的 drainer

```
bin/binlogctl -pd-urls=http://127.0.0.1:2379 -cmd pause-drainer -node-id ${ip}:${port}
```

* 查询 drainer 状态

```
bin/binlogctl -pd-urls=http://127.0.0.1:2379 -cmd drainer 
```
>当待维护主机上 drainer 状态变为 paused 时，进行下一步，否则等待。
	
* 关机维护

* 重启机器

* 重新上线 drainer

	```
ansible-playbook start_drainer.yml 
	```

	* 查询所有 drainer 状态

	```
bin/binlogctl -pd-urls=http://127.0.0.1:2379 	-cmd drainer
```

#####（2）替换 drainer
* 下线待维护机器上 drainer，记录最新的 checkpoint ts

	```
bin/binlogctl -pd-urls=http://127.0.0.1:2379 -cmd offline-drainer -node-id ${ip}:${port}
	```

	* 停掉 drainer 进程
	
	```
	ansible-playbook stop.yml -l drainer  drainer_name
	```
	
	* 编辑 inventory.ini 文件，移除 drainer  节点信息
	
	```
drainer_mysql ansible_host=xxx initial_commit_ts="xxx"
	```
	
	* 更新 Prometheus 配置并重启

	```
ansible-playbook rolling_update_monitor.yml --tags=prometheus 
	```
	* 查询所有 drainer 状态

	```
bin/binlogctl -pd-urls=http://127.0.0.1:2379 -cmd drainer
	```

* 在其他机器部署 drainer ，参考文档[部署 drainer](https://pingcap.com/docs-cn/stable/reference/tidb-binlog/deploy/) ，使用之前暂停 drainer 的  checkpoint ts 作为新 drainer 的 initial_commit_ts  ，同时新 drainer 配置参考旧 drainer

* 关机维护

## 六、DM
> 使用 DM 在进行数据同步时，如果涉及到组件所在机器需要临时关机维护，根据不同组件不同情况来分别处理

>* 如果是维护 DM-master 的机器，不会影响同步正常进行，注意期间不要执行 DDL
* 如果是维护 DM-worker 的机器，原地停止会停止数据同步；如果不能接受停止同步，可以先把 worker 迁移走。

### 1、DM-worker
##### (1) 原地停止 dm-worker 

* 停掉待维护机器上 dm-worker 服务
	* 在中控机上执行
	```
	ansible-playbook stop.yml --tags=dm-worker -l dm_worker3
	```
	
* 关机维护
* 重启机器
* 启动 dm-worker 服务
	```
ansible-playbook stop.yml --tags=dm-worker -l dm_worker3
	```
	* 未启动 sharding DDL 同步功能
		* 手动启动同步任务
		
		```
		./dmctl  start-task task.yaml
		```
	* 启动 sharding DDL 同步功能
		* 当 DM-worker 重启发生在 sharding DDL 语句同步开始前或完成后，DM-worker 会根据断点信息和本地记录的子任务信息自动恢复数据同步
			* 手动启动同步任务
			
			```
			./dmctl  start-task task.yaml
			```
			
		* 当 DM-worker 重启发生在 sharding DDL 语句同步过程中，可能会出现作为 DDL lock owner 的 DM-worker 实例已执行了 DDL 语句并成功变更了下游数据库表结构，但其他 DM-worker 实例重启而无法跳过 DDL 语句也无法更新断点的情况。
			* 手动启动同步任务
			 
			```
			./dmctl  start-task task.yaml
			```
			* 报错处理[参考文档](https://pingcap.com/docs-cn/stable/reference/tools/data-migration/features/manually-handling-sharding-ddl-locks/#%E5%9C%BA%E6%99%AF%E4%BA%8Cunlock-%E8%BF%87%E7%A8%8B%E4%B8%AD%E9%83%A8%E5%88%86-dm-worker-%E9%87%8D%E5%90%AF)

#### (2) 迁移 dm-worker 
* 参考[迁移 dm-worker 文档](https://pingcap.com/docs-cn/stable/reference/tools/data-migration/cluster-operations/)

注意：尽量避免在 sharding DDL 同步过程中重启 DM-worker


### 2、DM-master
#### (1) 原地停止
* 停掉带维护机器 dm-master 服务
	* 在中控机执行
	```
	ansible-playbook stop.yml --tags=dm-master
	```
	
* 关机维护
* 重启机器
* 启动服务
	* 在中控机执行
	```
	ansible-playbook start.yml --tags=dm-master
	```

#### (2)迁移 DM-master
* 参考[迁移 dm-master 文档](https://pingcap.com/docs-cn/stable/reference/tools/data-migration/cluster-operations/#%E6%9B%BF%E6%8D%A2%E8%BF%81%E7%A7%BB-dm-master-%E5%AE%9E%E4%BE%8B)

##### 根据部署情况确定关机维护方案
* 数据同步不能长时间停止
	* 包含 DM-worker 组件时，参考 DM-worker 部分：02（迁移 dm-worker ）
	* 包含 DM-master 组件时，参考 DM-master 部分：02（迁移 dm-master）
* 数据同步能长时间停止
	* 包含 DM-worker 组件时，参考 DM-worker 部分：01（原地更新 dm-worker ）
	* 包含 DM-master 组件时，参考 DM-master 部分：01（原地更新 dm-master）












