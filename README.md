# Prometheus在EMR上是实践



## 1. 什么是Prometheus
	Prometheus(普罗米修斯)这款开源监控工具，名字和功能一样酷，本文是一个干货入门，
	动手来部署一个实验环境。用Prometheus+Grafana来监控EMR。
	首先Prometheus，它支持多维度的指标数据模型，服务端通过HTTP协议定时拉取数据后，通过灵活的查询语言，实现监控的目的。
	客户端记录相关数据，并提供对外查询接口，服务端则通过服务器发现客户端，并定时抓取形成时序数据存储起来，最后通过可视化工具加以展现，其整体架构如下图：

<div padding="100px"><img src="./architecture.png" width="50%" height="50%" padding="1000"></div>

## 2. 为什么需要监控EMR集群
	为了充分利用集群资源，我们需要对集群状态有非常深刻的了解，
	由于team的Spark业务是run在EMR集群上面的，我们需要对整个EMR集群的运行状态有管理，
	比如磁盘的状态，内存的状态，集群是否稳定等等，都会对我们的pipeline 造成很大的影响；
	另一方面，我们也是为了能够充分的利用集群资源，做好cost saving.


## 3. Exporter和EMR集群的集成

	Graphite Exporter
	纯文本协议中导出的指标的导出器。 它通过TCP和UDP接收数据，并进行转换并将其公开以供Prometheus使用。
	该导出器对于从现有Graphite设置导出度量标准以及核心Prometheus导出器（例如Node Exporter）未涵盖的度量标准非常有用。
	https://github.com/prometheus/graphite_exporter

	JMX Exporter
	JMX到Prometheus导出器：一个收集器，该收集器可以可配置地抓取和公开JMX目标的mBean。该导出程序旨在作为Java代理运行，公开HTTP服务器并提供本地JVM的度量。 
	它也可以作为独立的HTTP服务器运行，并刮擦远程JMX目标，但这有许多缺点，例如难以配置和无法公开进程指标（例如，内存和CPU使用率）。 
	因此，强烈建议将导出程序作为Java代理运行。
	https://github.com/prometheus/jmx_exporter

	Node Exporter
	Prometheus导出程序，用于* NIX内核公开的硬件和操作系统指标，使用可插入的指标收集器用Go编写。
	建议Windows用户使用Windows导出程序。 要公开NVIDIA GPU指标，可以使用prometheus-dcgm。
	https://github.com/prometheus/node_exporter

	目前我们只集成这3种Exporter, 具体的集成方式：
	step 1. 在build 




## 4. 指标的收集
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
		
		

5. 监控数据的可视化

	




