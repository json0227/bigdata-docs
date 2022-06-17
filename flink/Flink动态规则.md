# Flink实现动态处理数据

## 1. 方案设计

通过将业务数据流和规则流进行双流连接，规则流需要转变成广播流；规则流的规则存储在广播状态中，业务数据流通过广播状态获取处理规则，从而根据规则动态处理业务数据。规则使用groovy语言编写。

## 2. 获取规则数据流和业务数据流
* 规则流
规则存储在数据库mysql中，flink通过读取mysql的binlog获取规则更新数据
```
// 规则流
        val mysqlSource: MySqlSource[String] = MySqlSource.builder()
            .hostname("host")
            .port(3306)
            .username("username")
            .password("password")
            .databaseList("db")
            .tableList("db.table")
            .deserializer(new CustomDeserializationSchema())
            .startupOptions(StartupOptions.latest())
            .build()
        val mysqlStream = env.fromSource(mysqlSource,WatermarkStrategy.noWatermarks(),"mysql source")
```
* 自定义反序列化器类
```
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
* 业务数据流
业务数据从kafka获取，可以保证精确一次消费语义
```
// 业务数据流
        val kafkaSource = KafkaSource.builder[String]
            .setBootstrapServers("rhel071:9092,rhel075:9092,rhel076:9092,rhel079:9092")
            .setTopics("flink_user_info")
            .setGroupId("dynamic_rule")
            .setStartingOffsets(OffsetsInitializer.earliest)
            .setValueOnlyDeserializer(new SimpleStringSchema())
            .build
        val ws: WatermarkStrategy[String] = WatermarkStrategy.forBoundedOutOfOrderness(Duration.ofSeconds(10))
        val dataStream: DataStream[String] = env.fromSource(kafkaSource, ws, "Kafka Source")
```
## 3. 将规则流进行广播
```
        // 定义广播变量
        val rulesBroadcastState: MapStateDescriptor[String, GroovyObject] = new MapStateDescriptor[String, GroovyObject]("RulesBroadcastState", classOf[String], classOf[GroovyObject])
        //将规则流变成广播流
        val ruleStream: BroadcastStream[String] = mysqlStream.broadcast(rulesBroadcastState)
```
## 4. 将双流进行连接，并通过BroadcastProcessFunction进行处理
```
// 将业务数据流和广播规则流进行连接
        val connectedStream: BroadcastConnectedStream[String, String] = dataStream.connect(ruleStream)
        // 对连接流进行业务处理
        val labelStream = connectedStream.process(new MyBroadcastProcessFunction(rulesBroadcastState))
        dataStream.print("kafka:")
        mysqlStream.print("mysql:")
        labelStream.filter(_.nonEmpty).print("result:")
        env.execute("DynamicRuleWithJoin")

```
## 5. 通过实现BroadcastProcessFunction对双流数据具体处理
```
class MyBroadcastProcessFunction(rulesBroadcastState: MapStateDescriptor[String, GroovyObject]) extends BroadcastProcessFunction[String,String,String]{
        // groovy 类加载器
        var classLoader:GroovyClassLoader = _
        // groovy脚本的类
        var groovyClass: Class[_] = _
        // groovy脚本的类创建的对象
        var groovyObject: GroovyObject = _

        override def open(parameters: Configuration): Unit = {
            classLoader = new GroovyClassLoader()
        }
    
        override def processElement(input: String, readOnlyContext: BroadcastProcessFunction[String, String, String]#ReadOnlyContext,
                                    out: Collector[String]): Unit = {
    
            val rulesMap: ReadOnlyBroadcastState[String, GroovyObject] = readOnlyContext.getBroadcastState(rulesBroadcastState)
            val rules= rulesMap.immutableEntries()
            // 遍历广播状态，获取所有的业务规则，对输入数据按照规则处理
            for(rule <- rules){
                val obj: GroovyObject = rule.asInstanceOf[util.Map.Entry[String, GroovyObject]].getValue
                val result = obj.invokeMethod("dataConversion", input).asInstanceOf[String]
                out.collect(result)
            }
        }
    
        override def processBroadcastElement(ruleData: String, context: BroadcastProcessFunction[String, String, String]#Context,
                                             collector: Collector[String]): Unit = {
            val ruleState: BroadcastState[String, GroovyObject] = context.getBroadcastState(rulesBroadcastState)
            // 对规则流中的输入数据进行解析
            val jsonObject = JSON.parseObject(ruleData)
            val op = jsonObject.getString("op")
            val afterData = jsonObject.getString("after")
            val beforeData = jsonObject.getString("before")
            var afterName: String = ""
            var afterRule: String = ""
            var beforeName: String = ""
    
            if(afterData != null){
                val afterObject = JSON.parseObject(afterData)
                afterName = afterObject.getString("rulename")
                afterRule = afterObject.getString("rulescript")
                // 解析groovy脚本，获取类对象
                groovyClass = classLoader.parseClass(afterRule)
                // 根据类创建对象
                groovyObject = groovyClass.newInstance.asInstanceOf[GroovyObject]
            }
            if(beforeData != null){
                val beforeObject = JSON.parseObject(beforeData)
                beforeName = beforeObject.getString("rulename")
            }
            // 根据操作类型对广播状态进行更新
            op match {
                case "read" =>
                    ruleState.put(afterName,groovyObject)
                case "create" =>
                    ruleState.put(afterName,groovyObject)
                case "update" =>
                    ruleState.put(afterName,groovyObject)
                case "delete" =>
                    ruleState.remove(beforeName)
                case "truncate" =>
                    ruleState.clear()
                case _ =>
            }
    
        }
    }
```