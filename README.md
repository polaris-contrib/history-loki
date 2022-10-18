# Report historical operation log to Loki plugin

[author: alexwanglei](https://github.com/alexwanglei)

English | [中文](./README-zh.md)

The plug-in implements sending historical operation records to the Loki log system, and retrieves and visualizes the logs in Loki through Grafana.

## 1. Preparation
### Install and deploy loki

Download binary package
````
$ curl -O -L "https://github.com/grafana/loki/releases/download/v2.6.1/loki-linux-amd64.zip"
# extract the binary
$ unzip "loki-linux-amd64.zip"
# make sure it is executable
$ chmod a+x "loki-linux-amd64"
````
Download configuration file
````
$ wget https://raw.githubusercontent.com/grafana/loki/v2.6.0/cmd/loki/loki-local-config.yaml
````
Start Loki
````
$ nohup ./loki-linux-amd64 -config.file=loki-local-config.yaml > ./loki.log 2>&1 &
````
For more installation methods, please refer to [Loki official installation documentation](https://grafana.com/docs/loki/latest/installation/)

### Install and deploy Grafana
RPM package installation
````
$ wget https://dl.grafana.com/enterprise/release/grafana-enterprise-9.1.6-1.x86_64.rpm
$ sudo yum install grafana-enterprise-9.1.6-1.x86_64.rpm
````
Start the Grafana service
````
$ sudo systemctl daemon-reload
$ sudo systemctl start grafana-server
$ sudo systemctl status grafana-server
````
For more installation methods, please refer to [Grafana official installation documentation](https://grafana.com/docs/grafana/latest/setup-grafana/installation/)

## 2. Plug-in configuration and use

### Compile plugin

1. git clone https://github.com/polarismesh/polaris.git
2. Go to the polaris directory
3. Execute **go get github.com/polaris-contrib/polaris-server-plugin-history-loki**
4. Execute **go mod tidy -compat=1.17**
5. Edit polaris/plugin.go, add in the import block
   ````go
   import (
        ...
        _ "github.com/polarismesh/polaris/plugin/history/loki" # Introduce the plugin and trigger the plugin's automatic registration logic
   )
   ````
5. Execute build.sh to build polaris-server

### Enable on polaris-server

In the polaris-server.yaml configuration file, in the plugin configuration section, the history plugin configuration block is set as follows:

````YAML
plugin:
  history:
    name: HistoryLoki # [Required] Plugin name
    option:
      queueSize: 1024 # [Required] The length of the log cache queue
      pushURL: http://127.0.0.1:3100/loki/api/v1/push # [Required] loki push interface address
      tenantID: "test" # [optional] set loki log tenant
      labels: # [optional] custom log labels
        key1: "value1"
        key2: "value2"
      timeout: 5s # [Optional] Send log http request timeout, default 10s
````
After polaris-server starts, the plugin will be loaded according to the configuration file, and historical operation records will be reported to the Loki log system through the plugin.

## 3. Grafana query Loki log

### Grafana configure Loki data source
Click the configuration icon on the Grafana page, click Data Sources, and add the Loki data source:
- Name: Set the data source name
- URL: set the address of the Loki instance

![Configure Loki Data Source|800](img/data_source_loki.png)

After the configuration is complete, click the `Save & Test` button at the bottom of the page. If the Loki system is running and a log is reported to the system, it will prompt that the data source is connected and the tag is found.

![save test|800](img/save_test.png)

### Query Loki logs
Create a Grafana panel to display logs, select Loki as the data source, and filter logs based on tags to generate query conditions. The plugin will set three tags, `source`, `resource_type` and `operation_type` by default, and `source` is the plugin name. `HistoryLoki`, `resource_type` is the resource type in the operation record, `operation_type` is the operation type in the operation record.

![Query log|800](img/panel_loki_log.png)

For more configuration and usage, please refer to [Using Loki in Grafana](https://grafana.com/docs/grafana/latest/datasources/loki/)