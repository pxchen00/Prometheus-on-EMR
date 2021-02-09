# Prometheus在EMR上的实践



## 1. 什么是Prometheus
	Prometheus(普罗米修斯)是一款非常好用的开源监控工具，我们大数据team在进行ETL处理时使用Prometheus+Grafana来对整个系统进行监控。
	Prometheus支持多维度的指标数据模型，服务端通过HTTP协议定时拉取数据，然后给用户提供灵活的查询语言PromQL，实现监控的目的。客户端记录相关数据，
	并提供对外查询接口，服务端则通过服务器发现客户端，并定时抓取形成时序数据存储起来，最后通过可视化工具加以展现，其整体架构如下图：
<div padding="100px"><img src="./architecture.png" width="80%" height="60%" padding="1000"></div>

## 2. 什么是EMR集群
	Amazon EMR 是一个托管集群平台，可简化在 AWS 上运行大数据框架（如 Apache Hadoop 和 Apache Spark）以处理和分析海量数据的操作
	，Freewheel的Transformer team的Spark业务正是run 在EMR集群上的，由于我们核心服务都迁移到EMR上了，所以对EMR集群有一个实时的监控，
	是非常重要的，比如磁盘的状态，内存的状态，集群是否稳定等等，都会对我们的pipeline 造成很大的影响；另一方面，我们也是为了能够充分的利用集群资源，
	做好cost saving 也需要知道现在资源的利用率是否需要调整。
	
具体的可以看我同事写的一篇关于EMR的介绍https://github.com/pkexcellent/articles/blob/main/EMR%E5%AE%9E%E8%B7%B5/AWS%20EMR%E5%9C%A8Freewheel%E7%9A%84%E5%BA%94%E7%94%A8%E4%B8%8E%E5%AE%9E%E8%B7%B5.md


## 3. 什么是Exporter
	一般Prometheus提供监控数据的程序都可以成为一个exporter的，一个exporter的实例称为Target, 现有的Prometheus生态中Exporter可以在官网查看，非常丰富
https://prometheus.io/docs/instrumenting/exporters/


### 3.1 Graphite Exporter
	纯文本协议中导出的指标的导出器。 它通过TCP和UDP接收数据，并进行转换并将其公开以供Prometheus使用。该导出器对于从现有Graphite设置导出度量标准
	以及核心Prometheus导出器（例如Node Exporter）未涵盖的度量标准非常有用。https://github.com/prometheus/graphite_exporter, 下面是安装 
	Graphite Exporter 的流程:
**step 1: Download node_exporter release from original repo**

	curl -L -O  https://github.com/prometheus/graphite_exporter/releases/download/v0.9.0/graphite_exporter-0.9.0.linux-amd64.tar.gz
	tar -xzvf graphite_exporter-0.9.0.linux-amd64.tar.gz
	sudo cp graphite_exporter-0.9.0.linux-amd64/graphite_exporter /usr/local/bin/
	rm -rf graphite_exporter-0.9.0.linux-amd64.tar.gz
	rm -rf graphite_exporter-0.9.0.linux-amd64
**step 2: Add graphite_exporter's mapping file**

	sudo mkdir -p /etc/graphite_exporter
	sudo tee /etc/graphite_exporter/graphite_exporter_mapping << END
	mappings:
	- match: '*.*.executor.filesystem.*.*'
	  name: filesystem_usage
	  labels:
	    application: \$1
	    executor_id: \$2
	    fs_type: \$3
	    qty: \$4

	- match: '*.*.jvm.*.*'
	  name: jvm_memory_usage
	  labels:
	    application: \$1
	    executor_id: \$2
	    mem_type: \$3
	    qty: \$4

	- match: '*.*.executor.jvmGCTime.count'
	  name: jvm_gcTime_count
	  labels:
	    application: \$1
	    executor_id: \$2

	- match: '*.*.jvm.pools.*.*'
	  name: jvm_memory_pools
	  labels:
	    application: \$1
	    executor_id: \$2
	    mem_type: \$3
	    qty: \$4

	- match: '*.*.executor.threadpool.*'
	  name: executor_tasks
	  labels:
	    application: \$1
	    executor_id: \$2
	    qty: \$3

	- match: '*.*.BlockManager.*.*'
	  name: block_manager
	  labels:
	    application: \$1
	    executor_id: \$2
	    type: \$3
	    qty: \$4

	- match: DAGScheduler.*.*
	  name: DAG_scheduler
	  labels:
	    type: \$1
	    qty: \$2
	END

**step 3: Add graphite_exporter as systemd service**

	sudo tee /etc/systemd/system/graphite_exporter.service << END
	[Unit]
	Description=Graphite Exporter
	Wants=network-online.target
	After=network-online.target

	[Service]
	User=ec2-user
	Group=ec2-user
	ExecStart=/usr/local/bin/graphite_exporter --graphite.mapping-config=/etc/graphite_exporter/graphite_exporter_mapping.yaml

	Restart=always

	[Install]
	WantedBy=multi-user.target
	END
	sudo chown -R ec2-user:ec2-user /etc/graphite_exporter/
	sudo systemctl daemon-reload
	sudo systemctl start graphite_exporter
	sudo systemctl enable graphite_exporter
	

### 3.2 JMX Exporter
	JMX到Prometheus导出器：一个收集器，该收集器可以可配置地抓取和公开JMX目标的mBean。该导出程序旨在作为Java代理运行，公开HTTP服务器并提供本地JVM的度量。 
	它也可以作为独立的HTTP服务器运行，并scrape远程JMX目标，但这有许多缺点，例如难以配置和无法公开进程指标（例如，内存和CPU使用率）。 
	因此，强烈建议将导出程序作为Java代理运行。
https://github.com/prometheus/jmx_exporter

### 3.3 Node Exporter
	Prometheus导出程序，用于* NIX内核公开的硬件和操作系统指标，使用可插入的指标收集器用Go编写。
	建议Windows用户使用Windows导出程序。 要公开NVIDIA GPU指标，可以使用prometheus-dcgm。
https://github.com/prometheus/node_exporter


## 4. 如何收集EMR集群的指标
	    因为Prometheus只支持pull模式的方式去收集数据，所以我们需要通过声明Prometheus配置文件中的scrape_configs，
	来指定Prometheus在运行时需要拉取指标的目标，目标实例需要实现一个可以被Prometheus进行轮询的端点接口，而Exporter正是用来提供这样接口的，
	比如用来拉取操作系统指标的Node Exporter，它会从操作系统上收集硬件指标，供Prometheus来拉取。
	    Prometheus pull时，可以通过static_configs参数配置静态配置目标，也可以使用受支持的服务发现机制之一动态发现目标。下面是一个完整的Prometheus配置：
	
	global:
	  #How frequently to scrape targets
	  scrape_interval:     15s # By default, scrape targets every 15 seconds.

	  #How frequently to evaluate rules
	  evaluation_interval: 5s

	  #Attach these labels to any time series or alerts when communicating with
	  #external systems (federation, remote storage, Alertmanager).
	  external_labels:
	    monitor: 'emr'

	# Rules and alerts are read from the specified file(s)
	#rule_files:
	#  - rules.yml

	# Alerting specifies settings related to the Alertmanager
	#alerting:
	#  alertmanagers:
	#    - scheme: http
	#      static_configs:
	#        - targets: ['transformer-etl-emr.fw1.aws.fwmrm.net:9093']

	# A scrape configuration containing exactly one endpoint to scrape:
	# Here it's Prometheus itself.
	scrape_configs:

	  # Graphite_Exporter_Job is designed to collect Spark metrics
	  - job_name: 'Graphite_Exporter_Job'
	    # metrics_path defaults to '/metrics'
	    # scheme defaults to 'http'.

	    static_configs:
	      - targets: ['transformer-etl-emr.fw1.aws.fwmrm.net:9108']

	  # Node_Exporter_Job is designed to collect OS system level metrics
	  - job_name: 'Node_Exporter_Job'
	    # Override the global default and scrape targets from this job every 15 seconds.
	    scrape_interval: 15s

	    ec2_sd_configs:
	      - region: us-east-1
		profile: Role-PRD-Transformer-Aggregator-OptimusExecutor-emr
		port: 9100
		filters:
		  - name: tag:Project
		    values:
		      - Transformer

	    relabel_configs:
	      - source_labels: [__meta_ec2_tag_aws_elasticmapreduce_instance_group_role]
		target_label: cluster_role
	      #Use instance ID as the instance label instead of private ip:port
	      - source_labels: [__meta_ec2_instance_id]
		target_label: instance
	      - source_labels: [__meta_ec2_tag_aws_elasticmapreduce_job_flow_id]
		target_label: cluster_id
	      - source_labels: [__meta_ec2_tag_Service]
		target_label: service

	  # Jmx_Exporter_Job is designed to collect java application such HDFS metrics
	  - job_name: 'Jmx_Exporter_Job'
	    # Override the global default and scrape targets from this job every 15 seconds.
	    scrape_interval: 15s

	    ec2_sd_configs:
	      - region: us-east-1
		profile: Role-PRD-Transformer-Aggregator-OptimusExecutor-emr
		port: 7001
		filters:
		  - name: tag:Project
		    values:
		      - Transformer
		  - name: tag:Service
		    values:
		      - OptimusExecutor
	    relabel_configs:
	      #This job is for monitoring CORE and TASK nodes, so drop MASTER node.
	      - source_labels: [__meta_ec2_tag_aws_elasticmapreduce_instance_group_role]
		regex: MASTER
		action: drop
		#Use instance ID as the instance label instead of private ip:port
	      - source_labels: [__meta_ec2_instance_id]
		target_label: instance
	      - source_labels: [__meta_ec2_tag_aws_elasticmapreduce_job_flow_id]
		: cluster_id
	      - source_labels: [__meta_ec2_tag_Service]
		target_label: service
		
		

## 5. 数据的查询及可视化
### 5.1 PromQL
	PromQL（Prometheus Query Language）是 Prometheus 自己开发的表达式语言，语言表现力很丰富，内置函数也很多。使用它可以对时序数据进行筛选和聚合。
	PromQL 表达式计算出来的值有以下几种类型：

	瞬时向量 (Instant vector): 一组时序，每个时序只有一个采样值
	区间向量 (Range vector): 一组时序，每个时序包含一段时间内的多个采样值
	标量数据 (Scalar): 一个浮点数
	字符串 (String): 一个字符串，暂时未用
	
#### 5.1.1 时序选择器

**a.瞬时向量选择器**

	瞬时向量选择器用来选择一组时序在某个采样点的采样值,比如下面的表达式：
	node_memory_MemAvailable_bytes{job=~'$job',service='optimusExecutor',cluster_id=~'$cluster'}
	
	度量指标名可以使用内部标签 __name__ 来匹配:
	表达式 node_memory_MemAvailable_bytes 也可以写成 {__name__="node_memory_MemAvailable_bytes"}。
	表达式 {__name__=~"job:.*"} 表示匹配所有度量指标名称以 job: 打头的时序。
	
	注意： = (完全相等)，!= (不相等)，=~ (正则表达式匹配)，!~ (正则表达式不匹配)
	
**b.区间向量选择器**

	区间向量选择器是选择过去一段时间的采样值。可以通过在瞬时向量选择器后面添加包含在 [] 里的时长来得到区间向量选择器。eg: 
		node_memory_MemAvailable_bytes{job="Node_Exporter_Job",service='optimusExecutor'}[5m]   //job为Node_Exporter_Job, service为optimusExecutor的时序在过去 5分钟的采样值
		
**c.偏移修饰器**

	前面介绍的选择器默认都是以当前时间为基准时间，偏移修饰器用来调整基准时间，使其往前偏移一段时间。偏移修饰器紧跟在选择器后面，使用 offset 来指定要偏移的量。
		node_memory_MemAvailable_bytes offset 5m       //node_memory_MemAvailable_bytes 的所有时序在 5 分钟前的采样值。
		node_memory_MemAvailable_bytes[5m] offset 1w   //下面的表达式选择 node_memory_MemAvailable_bytes 度量指标在 1 周前的这个时间点过去 5 分钟的采样值。

#### 5.1.2 操作符

**a.算术类二元操作符**

	算术类二元操作符有以下几种：
		+, -, *, /, %, ^
	算术类二元操作符可以使用在标量与标量、向量与标量，以及向量与向量之间,二元操作符上下文里的向量特指瞬时向量，不包括区间向量。

	标量与标量之间，结果很明显，跟通常的算术运算一致。
	向量与标量之间，相当于把标量跟向量里的每一个标量进行运算，这些计算结果组成了一个新的向量。
	向量与向量之间，会稍微麻烦一些。运算的时候首先会为左边向量里的每一个元素在右边向量里去寻找一个匹配元素（匹配规则后面会讲），然后对这两个匹配元素执行计算，这样每对匹配元素的计算结果组成了一个新的向量。如果没有找到匹配元素，则该元素丢弃。

**b.逻辑类二元操作符**

	逻辑操作符仅用于向量与向量之间,具体有：and(交集)，or(合集)，unless(补集)
	规则如下：
	vector1 and vector2 的结果由在 vector2 里有匹配（标签键值对组合相同）元素的 vector1 里的元素组成。
	vector1 or vector2 的结果由所有 vector1 里的元素加上在 vector1 里没有匹配（标签键值对组合相同）元素的 vector2 里的元素组成。
	vector1 unless vector2 的结果由在 vector2 里没有匹配（标签键值对组合相同）元素的 vector1 里的元素组成。
	
### 5.1.5 聚合操作符及辅助函数
	PromQL 提供了比较丰富的聚合操作符：sum（求和），min（最小值），max（最大值），avg（平均值），stddev（标准差），stdvar（方差），count（元素个数），count_values（等于某值的元素个数），bottomk（最小的 k 个元素），topk（最大的 k 个元素），quantile（分位数）等等。聚合操作符语法如下：
		<aggr-op>([parameter,] <vector expression>) [without|by (<label list>)] 
	eg：
		sum(node_memory_MemAvailable_bytes) without (instance)
		sum(node_memory_MemAvailable_bytes) by (application, group)
	Prometheus 内置了一些函数来辅助计算，如：abs(),sqrt(),exp(),ln(),ceil(),floor(),round(),delta(),sort()等等。


https://prometheus.io/docs/prometheus/latest/querying/basics/


### 5.2 Grafana可视化

	    现有的数据可视化工具很多，比如Prometheus 本身自带的可视化工具，我们可以访问Prometheus server的9090端口，在UI上对数据进行可视化编辑，
	但通常很难使用图形和仪表板编辑功能，Prometheus利用控制台模板进行可视化和仪表板编辑，但这些控制台模板的学习曲线起初可能很难。
	    现在比较主流的数据可视化工具是Grafana(http://docs.grafana.org/)，其在可视化和仪表板创建和定制方面是最好的选择，而且功能丰富，易于使用，
	而且非常灵活，目前主要用于大规模指标数据的可视化展现，是网络架构和应用分析中最流行的时序数据展示工具，目前已经支持绝大部分常用的时序数据库。
	Grafana支持许多不同的数据源。每个数据源都有一个特定的查询编辑器，官方支持以下数据源:Graphite，Elasticsearch，InfluxDB，Prometheus，
	Cloudwatch，MySQL和OpenTSDB等。每个数据源的查询语言和能力都是不同的。你可以把来自多个数据源的数据组合到一个仪表板，但每一个面板被绑定到一个特定的数据源,
	它就属于一个特定的组织，这里我们主要是使用Prometheus数据源。
	    在定义Query类型变量时，除了使用PromQL查询时间序列以过滤标签的方式以外，Grafana还提供了几个有用的函数：
	
	label_values(label) 	    返回Promthues所有监控指标中，标签名为label的所有可选值
	label_values(metric,label)  返回Promthues所有监控指标metric中，标签名为label的所有可选值
	metrics(metric)	    	    返回所有指标名称满足metric定义正则表达式的指标名称
	query_result(query)         返回prometheus查询语句的查询结果
	当需要监控Prometheus所有采集任务的状态时，可以使用如下方式获取当前所有采集任务的名称：
			label_values(up, job)			
<div padding="100px"><img src="./granfan- variables.png" width="90%" height="60%" padding="1000"></div>
<div padding="100px"><img src="./dashboard-demo3.png" width="90%" height="60%" padding="1000"></div>


## 6 demo
	下面就是整个monitor最终的效果图了，具体的dashboard可以参考下面的link

<div padding="100px"><img src="./dashboard-demo1.png" width="90%" height="60%" padding="1000"></div>
<div padding="100px"><img src="./dashboard-demo2.png" width="90%" height="60%" padding="1000"></div>
	
https://github.com/pxchen00/Prometheus-on-EMR/tree/Prometheus-on-EMR-pre/dashboards




