# Prometheus在EMR上的实践



## 1. 什么是Prometheus
	Prometheus(普罗米修斯)这款开源监控工具，名字和功能一样酷，本文是一个干货入门，动手来部署一个实验环境。用Prometheus+Grafana来监控EMR。
	首先Prometheus，它支持多维度的指标数据模型，服务端通过HTTP协议定时拉取数据后，通过灵活的查询语言，实现监控的目的。客户端记录相关数据，
	并提供对外查询接口，服务端则通过服务器发现客户端，并定时抓取形成时序数据存储起来，最后通过可视化工具加以展现，其整体架构如下图：
<div padding="100px"><img src="./architecture.png" width="80%" height="60%" padding="1000"></div>

## 2. 什么是EMR集群
	为了充分利用集群资源，我们需要对集群状态有非常深刻的了解，由于team的Spark业务是run在EMR集群上面的，我们需要对整个EMR集群的运行状态有管理，
	比如磁盘的状态，内存的状态，集群是否稳定等等，都会对我们的pipeline 造成很大的影响；另一方面，我们也是为了能够充分的利用集群资源，做好cost saving.


## 3. 什么是Exporter
### 3.1 Graphite Exporter
	纯文本协议中导出的指标的导出器。 它通过TCP和UDP接收数据，并进行转换并将其公开以供Prometheus使用。该导出器对于从现有Graphite设置导出度量标准以及核心Prometheus导出器（例如Node Exporter）未涵盖的度量标准非常有用。https://github.com/prometheus/graphite_exporter, 下面是安装 Graphite Exporter 的流程:
#### step 1: Download node_exporter release from original repo
	curl -L -O  https://github.com/prometheus/graphite_exporter/releases/download/v0.9.0/graphite_exporter-0.9.0.linux-amd64.tar.gz
	tar -xzvf graphite_exporter-0.9.0.linux-amd64.tar.gz
	sudo cp graphite_exporter-0.9.0.linux-amd64/graphite_exporter /usr/local/bin/
	rm -rf graphite_exporter-0.9.0.linux-amd64.tar.gz
	rm -rf graphite_exporter-0.9.0.linux-amd64
#### step 2: Add graphite_exporter's mapping file
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

#### step 3: Add graphite_exporter as systemd service
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
	
### 5.1 PromQL查询
	PromQL（Prometheus Query Language）是 Prometheus 自己开发的表达式语言，语言表现力很丰富，内置函数也很多。使用它可以对时序数据进行筛选和聚合。

### 5.1.1 数据类型
	PromQL 表达式计算出来的值有以下几种类型：

	瞬时向量 (Instant vector): 一组时序，每个时序只有一个采样值
	区间向量 (Range vector): 一组时序，每个时序包含一段时间内的多个采样值
	标量数据 (Scalar): 一个浮点数
	字符串 (String): 一个字符串，暂时未用
	
### 5.1.2 时序选择器
#### 5.1.2.1 瞬时向量选择器
	瞬时向量选择器用来选择一组时序在某个采样点的采样值。
	最简单的情况就是指定一个度量指标，选择出所有属于该度量指标的时序的当前采样值。比如下面的表达式：
	http_requests_total
	可以通过在后面添加用大括号包围起来的一组标签键值对来对时序进行过滤。比如下面的表达式筛选出了 job 为 prometheus，并且 group 为 canary 的时序：

	http_requests_total{job="prometheus", group="canary"}
	匹配标签值时可以是等于，也可以使用正则表达式。总共有下面几种匹配操作符：

	=：完全相等
	!=： 不相等
	=~： 正则表达式匹配
	!~： 正则表达式不匹配
	下面的表达式筛选出了 environment 为 staging 或 testing 或 development，并且 method 不是 GET 的时序：

	http_requests_total{environment=~"staging|testing|development",method!="GET"}
	度量指标名可以使用内部标签 __name__ 来匹配，表达式 http_requests_total 也可以写成 {__name__="http_requests_total"}。表达式 {__name__=~"job:.*"} 匹配所有度量指标名称以 job: 打头的时序。
	
#### 5.1.2.2 区间向量选择器
	区间向量选择器类似于瞬时向量选择器，不同的是它选择的是过去一段时间的采样值。可以通过在瞬时向量选择器后面添加包含在 [] 里的时长来得到区间向量选择器。比如下面的表达式选出了所有度量指标为 		http_requests_total 且 job 为 prometheus 的时序在过去 5 分钟的采样值。eg:
		http_requests_total{job="prometheus"}[5m]
		
### 5.1.3 偏移修饰器
	前面介绍的选择器默认都是以当前时间为基准时间，偏移修饰器用来调整基准时间，使其往前偏移一段时间。偏移修饰器紧跟在选择器后面，使用 offset 来指定要偏移的量。
		http_requests_total offset 5m   //http_requests_total 的所有时序在 5 分钟前的采样值。
		http_requests_total[5m] offset 1w   //下面的表达式选择 http_requests_total 度量指标在 1 周前的这个时间点过去 5 分钟的采样值。
		
## 5.2. PromQL 操作符
### 5.2.1 二元操作符
	PromQL 的二元操作符支持算术类、比较类、逻辑类三大类。

#### 5.2.1.1 算术类
	算术类二元操作符有以下几种：
		+, -, *, /, %, ^
	算术类二元操作符可以使用在标量与标量、向量与标量，以及向量与向量之间,二元操作符上下文里的向量特指瞬时向量，不包括区间向量。

	标量与标量之间，结果很明显，跟通常的算术运算一致。
	向量与标量之间，相当于把标量跟向量里的每一个标量进行运算，这些计算结果组成了一个新的向量。
	向量与向量之间，会稍微麻烦一些。运算的时候首先会为左边向量里的每一个元素在右边向量里去寻找一个匹配元素（匹配规则后面会讲），然后对这两个匹配元素执行计算，这样每对匹配元素的计算结果组成了一个新的向量。如果没有找到匹配元素，则该元素丢弃。
#### 5.2.1.2 比较类
	比较类二元操作符有以下几种：

		== (equal)
		!= (not-equal)
		> (greater-than)
		< (less-than)
		>= (greater-or-equal)
		<= (less-or-equal)
	比较类二元操作符同样可以使用在标量与标量、向量与标量，以及向量与向量之间。默认执行的是过滤，也就是保留值。

	也可以通过在运算符后面跟 bool 修饰符来使得返回值 0 和 1，而不是过滤。

	标量与标量之间，必须跟 bool 修饰符，因此结果只可能是 0（false） 或 1（true）。
	向量与标量之间，相当于把向量里的每一个标量跟标量进行比较，结果为真则保留，否则丢弃。如果后面跟了 bool 修饰符，则结果分别为 1 和 0。
	向量与向量之间，运算过程类似于算术类操作符，只不过如果比较结果为真则保留左边的值（包括度量指标和标签这些属性），否则丢弃，没找到匹配也是丢弃。如果后面跟了 bool 修饰符，则保留和丢弃时结果	相应为 1 和 0。
	
#### 5.2.1.3 逻辑类
	逻辑操作符仅用于向量与向量之间,具体有：and(交集)，or(合集)，unless(补集)
	规则如下：
	vector1 and vector2 的结果由在 vector2 里有匹配（标签键值对组合相同）元素的 vector1 里的元素组成。
	vector1 or vector2 的结果由所有 vector1 里的元素加上在 vector1 里没有匹配（标签键值对组合相同）元素的 vector2 里的元素组成。
	vector1 unless vector2 的结果由在 vector2 里没有匹配（标签键值对组合相同）元素的 vector1 里的元素组成。
	
### 5.2.2 向量匹配
	前面算术类和比较类操作符都需要在向量之间进行匹配。共有两种匹配类型，one-to-one 和 many-to-one / one-to-many。

#### 5.2.2.1 One-to-one 向量匹配
	这种匹配模式下，两边向量里的元素如果其标签键值对组合相同则为匹配，并且只会有一个匹配元素。可以使用 ignoring 关键词来忽略不参与匹配的标签，或者使用 on 关键词来指定要参与匹配的标签。语法	如下：

		<vector expr> <bin-op> ignoring(<label list>) <vector expr>
		<vector expr> <bin-op> on(<label list>) <vector expr>
		比如对于下面的输入：

		method_code:http_errors:rate5m{method= "get", code="500"} 24
		method_code:http_errors:rate5m{method= "get", code="404"} 30
		method_code:http_errors:rate5m{method= "put", code="501"} 3
		method_code:http_errors:rate5m{method= "post", code="500"} 6
		method_code:http_errors:rate5m{method= "post", code="404"} 21

		method:http_requests:rate5m{method= "get"} 600
		method:http_requests:rate5m{method= "del"} 34
		method:http_requests:rate5m{method= "post"} 120
		执行下面的查询：

		method_code:http_errors:rate5m{code="500"} / ignoring(code) method:http_requests:rate5m
		得到的结果为：

		{method= "get"} 0.04 // 24 / 600
		{method= "post"} 0.05 // 6 / 120
		也就是每一种 method 里 code 为 500 的请求数占总数的百分比。由于 method 为 put 和 del 的没有匹配元素所以没有出现在结果里。

#### 5.2.2.2 Many-to-one / one-to-many 向量匹配
	这种匹配模式下，某一边会有多个元素跟另一边的元素匹配。这时就需要使用 group_left 或 group_right 组修饰符来指明哪边匹配元素较多，左边多则用 group_left，右边多则用 group_right。其	语法如下：

		<vector expr> <bin-op> ignoring(<label list>) group_left(<label list>) <vector expr>
		<vector expr> <bin-op> ignoring(<label list>) group_right(<label list>) <vector expr>
		<vector expr> <bin-op> on(<label list>) group_left(<label list>) <vector expr>
		<vector expr> <bin-op> on(<label list>) group_right(<label list>) <vector expr>
	组修饰符只适用于算术类和比较类操作符。

	对于前面的输入，执行下面的查询：

		method_code:http_errors:rate5m / ignoring(code) group_left method:http_requests:rate5m
	将得到下面的结果：

		{method= "get", code="500"} 0.04 // 24 / 600
		{method= "get", code="404"} 0.05 // 30 / 600
		{method= "post", code="500"} 0.05 // 6 / 120
		{method= "post", code="404"} 0.175 // 21 / 120
	也就是每种 method 的每种 code 错误次数占每种 method 请求数的比例。这里匹配的时候 ignoring 了 code，才使得两边可以形成 Many-to-one 形式的匹配。由于左边多，所以需要使用 		group_left 来指明。

		Many-to-one / one-to-many 过于高级和复杂，要尽量避免使用。很多时候通过 ignoring 就可以解决问题。

### 5.2.3 聚合操作符
	PromQL 提供了比较丰富的聚合操作符：sum（求和），min（最小值），max（最大值），avg（平均值），stddev（标准差），stdvar（方差），count（元素个数），count_values（等于某值的元素个数），bottomk（最小的 k 个元素），topk（最大的 k 个元素），quantile（分位数）等等。聚合操作符语法如下：

		<aggr-op>([parameter,] <vector expression>) [without|by (<label list>)] 
	eg：
		sum(http_requests_total) without (instance)
		sum(http_requests_total) by (application, group)

## 5.3 函数
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

<div padding="100px"><img src="./dashboard-demo1.png" width="90%" height="60%" padding="1000"></div>
<div padding="100px"><img src="./dashboard-demo2.png" width="90%" height="60%" padding="1000"></div>
	
https://github.com/pxchen00/Prometheus-on-EMR/tree/Prometheus-on-EMR-pre/dashboards




