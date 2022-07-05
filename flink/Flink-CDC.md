# 通过Flink-CDC实时读取Mysql表更新数据

## 1. 依赖

```maven
		<dependency>
			<groupId>com.ververica</groupId>
			<artifactId>flink-connector-mysql-cdc</artifactId>
			<version>2.1.0</version>
		</dependency>
```

## 2.读取Mysql中表的数据

```scala
val env = StreamExecutionEnvironment.getExecutionEnvironment
        env.enableCheckpointing(5000L)
        val config = env.getCheckpointConfig
        config.setCheckpointingMode(CheckpointingMode.EXACTLY_ONCE)
        config.setExternalizedCheckpointCleanup(ExternalizedCheckpointCleanup.RETAIN_ON_CANCELLATION)

        val mysqlSource: MySqlSource[String] = MySqlSource.builder()
            .hostname("localhost")
            .port(3306)
            .username("root")
            .password("password")
            .databaseList("test")
            .tableList("test.dynamicrules")
            .deserializer(new CustomDeserializationSchema())
            .startupOptions(StartupOptions.initial())
            .build()

        val mysqlDS = env.fromSource(mysqlSource,WatermarkStrategy.noWatermarks(),"mysql source")
```

## 3. 自定义序列化

```java
public class CustomDeserializationSchema implements DebeziumDeserializationSchema<String> {

    @Override
    public void deserialize(SourceRecord sourceRecord, Collector<String> collector) throws Exception {

        JSONObject result = new JSONObject();
        String[] split = sourceRecord.topic().split("\\.");
        result.put("db",split[1]);
        result.put("tb",split[2]);
        Envelope.Operation operation = Envelope.operationFor(sourceRecord);
        result.put("op",operation.toString().toLowerCase());

        Struct value =(Struct)sourceRecord.value();
        JSONObject after = getValueBeforeAfter(value, "after");
        JSONObject before = getValueBeforeAfter(value, "before");

        if (after!=null){result.put("after",after);}
        if (before!=null){result.put("before",before);}

        collector.collect(result.toJSONString());

    }

    public JSONObject getValueBeforeAfter(Struct value, String type){
        Struct midStr = (Struct)value.get(type);
        JSONObject result = new JSONObject();
        if(midStr!=null){
            List<Field> fields = midStr.schema().fields();
            for (Field field : fields) {
                result.put(field.name(),midStr.get(field));
            }
            return result;
        }return null;
    }

    @Override
    public TypeInformation<String> getProducedType() {
        return BasicTypeInfo.STRING_TYPE_INFO;
    }
}
```



## 3.注意事项

* 读取的表需要有主键
* Flink和Flink-CDC的版本需要对应

