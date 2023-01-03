# 基于Addax的DolphinDB数据导入工具
## 1. 使用场景
addaxdolphindbwriter插件是解决用户将不同数据来源的数据同步到DolphinDB的场景而开发的，这些数据的特征是改动很少, 并且数据分散在不同的数据库系统中。

## 2. Addax离线数据同步
Addax 是一个异构数据源离线同步工具，最初来源于阿里的 DataX ，致力于实现包括关系型数据库(MySQL、Oracle等)、HDFS、Hive、HBase、FTP等各种异构数据源之间稳定高效的数据同步功能。 [Addax简介](https://wgzhao.github.io/Addax/develop/)。

Addax是可扩展的数据同步框架，将不同数据源的同步抽象为从源头数据源读取数据的Reader插件，以及向目标端写入数据的Writer插件。理论上Addax框架可以支持任意数据源类型的数据同步工作。每接入一套新数据源该新加入的数据源即可实现和现有的数据源互通。


#### Addax插件 ：dolphindbwriter

基于Addax的扩展功能，dolphindbwriter插件实现了向DolphinDB写入数据，使用Addax的现有reader插件结合DolphinDBWriter插件，即可满足从不同数据源向DolphinDB同步数据。

DolphinDBWriter底层依赖于 DolphinDB Java API，采用批量写入的方式，将数据写入分布式数据库。

本插件通常用于一下两个场景：

1-定期从数据源向DolphinDB追加新增数据。

2-定期获取更新的数据，定位DolphinDB中的相同数据并更新。此种模式下，由于需要将历史数据读取出来并在内存中进行匹配，会需要大量的内存，因此这种场景适用于在DolphinDB中容量较小的表，通常建议数据量在200万以下的表。
当前使用的更新数据的模式是通过全表数据提取、更新后删除分区重写的方式来实现，现在的版本还无法保障上述整体操作的原子性，后续版本会针对此种方式的事务处理方面做优化和改进。


## 3. 使用方法
详细信息请参阅 [Addax指南](https://wgzhao.github.io/Addax/develop/quickstart/), 以下仅列出必要步骤。

### 3.1 下载部署Addax

Download [Addax下载地址](https://github.com/wgzhao/Addax/releases/download/4.0.11/addax-4.0.11.tar.gz)

### 3.2 部署Addax-DolphinDBWriter插件

将源码的 ./dolphindbwriter 目录下所有内容拷贝到addax/plugin/writer目录下，即可以使用。

### 3.3 执行Addax任务

进入addax/bin目录下，用 shell 执行 addax.sh 脚本，并指定配置文件地址，示例如下：
```
cd ~/addax/bin/
bash addax.sh ~/addax/job/influx2ddb.json
```

### 3.4 导入实例
使用Addax绝大部分工作都是通过配置来完成，包括双边的数据库连接信息和需要同步的数据表结构信息等。

### 3.4.1 从InfluxDB导出数据到DolphinDB

下面展示一个从InfluxDB导出数据到DolphinDB的任务配置示例。

```
{
  "job": {
    "content": {
      "reader": {
        "name": "influxdb2reader",
        "parameter": {
          "column": ["*"],
          "connection": [
            {
              "endpoint": "http://183.136.170.168:8086",
              "bucket": "demo-bucket",
              "table": [
                  "machinery"
              ],
              "org": "zhiyu"
            }
          ],
          "token": "GLiPjQFQIxzVO0-atASJHH4b075sTlyEZGrqW20XURkelUT5pOlfhi_Yuo2fjcSKVZvyuO00kdXunWPrpJd_kg==",
          "range": [
            "2007-08-09"
          ]
        }
      },
      "writer": {
        "name": "dolphindbwriter",
        "parameter": {
          "userId": "admin",
          "pwd": "123456",
          "host": "115.239.209.122",
          "port": 3134,
          "dbPath": "dfs://demo",
          "tableName": "pt",
          "batchSize": 1000000,
          "saveFunctionName": "transData",
          "saveFunctionDef": "def parseRFC3339(timeStr) {if(strlen(timeStr) == 20) {return localtime(temporalParse(timeStr,'yyyy-MM-ddTHH:mm:ssZ'));} else if (strlen(timeStr) == 24) {return localtime(temporalParse(timeStr,'yyyy-MM-ddTHH:mm:ss.SSSZ'));} else {return timeStr;}};def transData(dbName, tbName, mutable data) {timeCol = exec time from data; timeCol=each(parseRFC3339, timeCol);  writeLog(timeCol);replaceColumn!(data, 'time', timeCol); loadTable(dbName,tbName).append!(data); }",
          "table": [
              {
                "type": "DT_STRING",
                "name": "time"
              },
              {
                "type": "DT_SYMBOL",
                "name": "stationID"
              },
              {
                "type": "DT_DOUBLE",
                "name": "grinding_time"
              },
              {
                "type": "DT_DOUBLE",
                "name": "oil_temp"
              },
              {
                "type": "DT_DOUBLE",
                "name": "pressure"
              },
              {
                "type": "DT_DOUBLE",
                "name": "pressure_target"
              },
              {
                "type": "DT_DOUBLE",
                "name": "rework_time"
              },
              {
                "type": "DT_SYMBOL",
                "name": "state"
              }
          ]
        }
      }
    },
    "setting": {
      "speed": {
        "bytes": -1,
        "channel": 1
      }
    }
  }
}

```

其中，关于InfluxDB 2.0的读插件，配置参数说明，可以参考 [InfluxDB2.0读插件说明](https://wgzhao.github.io/Addax/develop/reader/influxdb2reader/#_3)。

关于DolphinDB的写入插件，参数说明如下:



* 配置文件参数说明

* **host**

	* 描述：Server Host 

 	* 必选：是 <br />

	* 默认值：无 <br />

* **port**

	* 描述：Server Port  <br />

	* 必选：是 <br />

	* 默认值：无 <br />

* **userId**

	* 描述：DolphinDB 用户名 <br />
            导入分布式库时，必须要有权限的用户才能操作，否则会返回
	* 必选：是 <br />

	* 默认值：无 <br />

* **pwd**

	* 描述：DolphinDB 用户密码

	* 必选：是 <br />

	* 默认值：无 <br />

* **dbPath**

	* 描述：需要写入的目标分布式库名称，比如"dfs://MYDB"。

	* 必选：是 <br />

	* 默认值：无 <br />

* **tableName**

	* 描述: 目标数据表名称

	* 必须: 是

	* 默认值: 无
	
* **batchSize**

	* 描述: addax每次写入dolphindb的批次记录数

	* 必须: 否

	* 默认值: 10,000,000

* **table**

	* 描述：写入表的字段集合。内部结构为
	```
	{"name": "columnName", "type": "DT_STRING", "isKeyField":true}
	```
	请注意此处列定义的顺序，需要与原表提取的列顺序完全一致。
	* name ：字段名称。
	* isKeyField：是否唯一键值，可以允许组合唯一键。本属性用于数据更新场景，用于确认更新数据的主键，若无更新数据的场景，无需设置。
	* type 枚举值以及对应Addax数据类型如下。DolphinDB的数据类型及精度，请参考 https://www.dolphindb.cn/cn/help/DataType.html
	
	DolphinDB类型 | 配置值 
	---|---
	 DOUBLE | DT_DOUBLE 
     FLOAT|DT_FLOAT
     BOOL|DT_BOOL
     DATE|DT_DATE
     MONTH|DT_MONTH
     DATETIME|DT_DATETIME
     TIME|DT_TIME
     SECOND|DT_SECOND
     TIMESTAMP|DT_TIMESTAMP
     NANOTIME|DT_NANOTIME
     NANOTIMETAMP|DT_NANOTIMETAMP
     INT|DT_INT
     LONG|DT_LONG
     UUID|DT_UUID
     SHORT|DT_SHORT
     STRING|DT_STRING
     SYMBOL|DT_SYMBOL
	
	* 必选：是 <br />
	* 默认值：无 <br />

* **saveFunctionName**
    * 描述：自定义数据处理函数。若未指定此配置，插件在接收到reader的数据后，会将数据提交到DolphinDB并通过tableInsert函数写入指定库表；如果定义此参数，则会用指定函数替换tableInsert函数。
    * 必选：否 <br />
	* 默认值： 无。 也可以指定自定义函数 <br />

	插件内置了 savePartitionedData(更新分布式表)/saveDimensionData(更新维度表) 两个函数，当saveFunctionDef未定义或为空时, saveFunctionName可以取枚举值之一，对应用于更新分布式表和维度表的数据处理。

* **saveFunctionDef**
    * 描述：数据入库自定义函数。此函数指 用dolphindb 脚本来实现的数据入库过程。
            此函数必须接受三个参数：dfsPath(分布式库路径), tbName(数据表名), data(从addax导入的数据,table格式)
	* 必选：当saveFunctionName参数不为空且非两个枚举值之一时，此参数必填 <br />
	* 默认值：无 <br />
