# Flink连接加密Kafka

## 1. 代码

代码中需要对kafka进行参数配置

```scala
def getKafkaProperties(jobConfig: Properties): Properties ={
        val properties = new Properties
        properties.setProperty("bootstrap.servers", jobConfig.getProperty("bootstrap.servers"))
        properties.setProperty("group.id", jobConfig.getProperty("group.id"))
        properties.setProperty("key.deserializer", "org.apache.kafka.common.serialization.StringDeserializer")
        properties.setProperty("value.deserializer", "org.apache.kafka.common.serialization.StringDeserializer")
        properties.setProperty("auto.offset.reset", jobConfig.getProperty("auto.offset.reset"))
        properties.put(ConsumerConfig.REQUEST_TIMEOUT_MS_CONFIG, "240000")
        properties.put(ConsumerConfig.SESSION_TIMEOUT_MS_CONFIG, "240000")
        // 添加kafka的sasl认证
        if (jobConfig.containsKey("security.protocol")) {
            properties.setProperty("security.protocol", jobConfig.getProperty("security.protocol"))
        }
        if (jobConfig.containsKey("sasl.mechanism")) {
            properties.setProperty("sasl.mechanism", jobConfig.getProperty("sasl.mechanism"))
        }
        properties.setProperty("sasl.kerberos.service.name", "kafka")
        properties
    }
```

## 2.任务提交

任务提交时需要指定配置文件的目录

```
## 1、先配置临时环境变量：
export JAVA_HOME=/usr/local/jdk1.8
export HADOOP_HOME=/cmss/bch/3.0.1/hadoop
export PATH="$HADOOP_HOME/bin:$JAVA_HOME/bin:$PATH"
export HADOOP_CONF_DIR=/cmss/bch/3.0.1/hadoop/etc/hadoop
export HADOOP_CLASSPATH=/cmss/bch/3.0.1/hadoop/share/hadoop/common/lib/*

##  Kerboros 身份验证
kinit -kt /data/test_user01.keytab  test_user01@HOSTNAME

## 2、执行 flink 作业提交
/opt/flink-1.12.1/bin/flink run -d \
-m yarn-cluster \
-yqu root.queue.testqueue01 \
-ynm automarket_test \
-ytm 4096 \
-yjm 2048 \
-ys 2 \
-p 2 \
-yD taskmanager.memory.framework.off-heap.size=1024m \
-yD taskmanager.memory.managed.fraction=0.1 \
-yD env.java.opts=-Djavax.security.auth.useSubjectCredsOnly=false \
-yD env.java.opts=-Djava.security.auth.login.config=kafka_jaas.conf \
-yt /data/cy_workspace/streamdata_import/automarket/kafka_jaas.conf \
-c automarketing.UpLoadPicture2Hive /data/automarket/label2hive/mcloud_rc_prj_1.0.jar /data/automarket/label2hive/project_args.conf
```

## 3.kafka配置文件

* kafka_jaas.conf

```
KafkaClient {
org.apache.kafka.common.security.plain.PlainLoginModule required
username="hcy"
password="hcy-sec";
};
```

* project_args.conf
```
checkpoint.interval = 1800000
fixed-delay.attempts = 10
fixed-delay.delay = 30000
bootstrap.servers=10.153.98.201:9092
topic_uop = hecaiyun-uop
topic_uop_risk_tracking = uop_risk_tracking
topic_bi=hecaiyun-bi
auto.offset.reset=latest
group.id=label2hvie_0526
security.protocol=SASL_PLAINTEXT
sasl.mechanism=PLAIN
hiveconfdir=/cmss/bch/3.0.1/hive/conf
```

## 4.配置写在代码中

```
prop.setProperty("security.protocol","SASL_PLAINTEXT");
prop.setProperty("sasl.mechanism","PLAIN")
prop.setProperty("sasl.jaas.config","org.apache.kafka.common.security.plain.PlainLoginModule required 
username=\"name\" 
password=\"pwd\");
prop.setProperty("sasl.kerberos.service.name","kafka");
```

注意点：sasl.jaas.config项配置的时候需要如同配置文件里一样，username和password都需要进行换行
