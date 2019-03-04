1. 创建Oozie工作流目录
2. 在工作流目录下创建lib目录用于添加依赖库文件
3. 编辑workflow.xml(定义Oozie工作流)
```
<?xml version="1.0" encoding="UTF-8"?>
<!--
  Licensed to the Apache Software Foundation (ASF) under one
  or more contributor license agreements.  See the NOTICE file
  distributed with this work for additional information
  regarding copyright ownership.  The ASF licenses this file
  to you under the Apache License, Version 2.0 (the
  "License"); you may not use this file except in compliance
  with the License.  You may obtain a copy of the License at

       http://www.apache.org/licenses/LICENSE-2.0

  Unless required by applicable law or agreed to in writing, software
  distributed under the License is distributed on an "AS IS" BASIS,
  WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
  See the License for the specific language governing permissions and
  limitations under the License.
-->
<workflow-app xmlns="uri:oozie:workflow:0.2" name="${workflowName}">
    <start to="create-hive-table"/>

    <fork name="create-hive-table">
        <path start="import-data-to-hdfs" />
        <path start="import-data-into-hive" />
    </fork>

    <action name="import-data-into-hive">
        <sqoop xmlns="uri:oozie:sqoop-action:0.2">
            <job-tracker>${jobTracker}</job-tracker>
            <name-node>${nameNode}</name-node>
            <prepare>
                <delete path="${nameNode}/apps/hive/warehouse/inner_${mysqlTable}"/>
            </prepare>
            <job-xml>${nameNode}/user/${wf:user()}/${workflowName}/hive-site.xml</job-xml>
            <configuration>
                <property>
                    <name>mapred.job.queue.name</name>
                    <value>${queueName}</value>
                </property>
            </configuration>
            <arg>import</arg>
            <arg>--connect</arg>
            <arg>jdbc:mysql://${mysqlHost}:${mysqlPort}/${mysqlDatabase}</arg>
            <arg>--username</arg>
            <arg>${mysqlUser}</arg>
            <arg>--password</arg>
            <arg>${mysqlPassword}</arg>
            <arg>--table</arg>
            <arg>${mysqlTable}</arg>            
            <arg>--target-dir</arg>
            <arg>/apps/hive/warehouse/inner_${mysqlTable}</arg>
            <arg>--hive-import</arg>
            <arg>--hive-database</arg>
            <arg>default</arg>
            <arg>--hive-table</arg>
            <arg>inner_${mysqlTable}</arg>
            <arg>-m</arg>
            <arg>1</arg>
        </sqoop>
        <ok to="finish-create-hive-table"/>
        <error to="fail"/>
    </action>

    <action name="import-data-to-hdfs">
        <sqoop xmlns="uri:oozie:sqoop-action:0.2">
            <job-tracker>${jobTracker}</job-tracker>
            <name-node>${nameNode}</name-node>
            <prepare>
                <delete path="${nameNode}/user/${wf:user()}/${workflowName}/output/import-data"/>
                <mkdir path="${nameNode}/user/${wf:user()}/${workflowName}/output"/>
            </prepare>
            <configuration>
                <property>
                    <name>mapred.job.queue.name</name>
                    <value>${queueName}</value>
                </property>
            </configuration>
            <arg>import</arg>
            <arg>--connect</arg>
            <arg>jdbc:mysql://${mysqlHost}:${mysqlPort}/${mysqlDatabase}</arg>
            <arg>--username</arg>
            <arg>${mysqlUser}</arg>
            <arg>--password</arg>
            <arg>${mysqlPassword}</arg>
            <arg>--table</arg>
            <arg>${mysqlTable}</arg>            
            <arg>--target-dir</arg>
            <arg>/user/${wf:user()}/${workflowName}/output/import-data</arg>
            <arg>-m</arg>
            <arg>1</arg>
        </sqoop>
        <ok to="decide-import-data"/>
        <error to="fail"/>
    </action>

    <decision name="decide-import-data">
        <switch>
            <case to="create-hive-external-table">${fs:exists(concat(concat(concat(concat("/user/",wf:user()),"/"), workflowName),"/output/import-data"))}</case> 
            <default to="fail"/>
        </switch>
    </decision>

    <action name="create-hive-external-table">
        <hive xmlns="uri:oozie:hive-action:0.2">
            <job-tracker>${jobTracker}</job-tracker>
            <name-node>${nameNode}</name-node>
            <configuration>
                <property>
                    <name>mapred.job.queue.name</name>
                    <value>${queueName}</value>
                </property>
            </configuration>
            <script>create-hive-external-table.q</script>
            <param>INPUT=/user/${wf:user()}/${workflowName}/output/import-data</param>
            <param>TABLE_NAME=external_${mysqlTable}</param>
        </hive>
        <ok to="finish-create-hive-table"/>
        <error to="fail"/>
    </action>

    <join name="finish-create-hive-table" to="end"/>

    <kill name="fail">
        <message>example failed, error message[${wf:errorMessage(wf:lastErrorNode())}]</message>
    </kill>
    <end name="end"/>
</workflow-app>
```
4. 编辑job.properties（配置工作流）
```
workflowName=example
nameNode=hdfs://xxxx:8020 # hdfs-site.xml中dfs.namenode.rpc-address指定的地址
jobTracker=xxxx:8050 # yarn-site.xml中yarn.resourcemanager.address指定的地址
queueName=default
oozie.wf.application.path=${nameNode}/user/${user.name}/${workflowName}
oozie.use.system.libpath=true
oozie.action.sharelib.for.sqoop=hive,sqoop #指定具体action使用的共享库
mysqlHost=xxxx
mysqlPort=xxxx
mysqlDatabase=xxxx
mysqlUser=xxxx
mysqlPassword=xxxx
mysqlTable=xxxx
```
5. 上传Oozie工作流目录至HDFS
6. 上传Oozie共享库至HDFS
```
hadoop fs -put ${oozie-path}/share /user/oozie
```
> oozie.action.sharelib.for.sqoop=hive,sqoop
修改Oozie工作流配置文件job.properties指定具体action使用的共享库，用于解决异常：Encountered IOException running import job: java.io.IOException: Cannot run program "hive": error=2, No such file or directory。
7. 启动工作流
```
oozie job -oozie http://xxxx:11000/oozie -config job.properties -run
```


