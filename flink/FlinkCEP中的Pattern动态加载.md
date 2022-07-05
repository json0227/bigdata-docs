# FlinkCEP中的Pattern动态加载

## 1. 整体方案设计

通过groovy书写flinkcep的pattern，并存储在外部系统（文件、数据库等），在flink代码中从外部加载pattern字符串，并通过groovy的GroovyClassLoader加载解析groovy的pattern脚本，从而得到Pattern对象。从而实现一个jar包可以根据不同的规则进行部署任务，而不用另外开发。

## 2. 具体代码实现

**第一步：从配置文件中获取参数**

```scala
//获取参数配置
        val propertiesFilePath = "E:\\workspace\\yjson-flink-demo\\dynamic-rules\\src\\main\\resources\\myArgs.properties"
        val parameter = ParameterTool.fromPropertiesFile(propertiesFilePath)
        val ruleId = parameter.get("rule_id").toInt
        val ruleName = parameter.get("rule_name")
        println(s"ruleId:${ruleId},ruleName:${ruleName}")
```

**第二步：从数据源获取数据**

```scala
val env = StreamExecutionEnvironment.getExecutionEnvironment
        env.enableCheckpointing(1000L)
        env.setParallelism(1)
        //定义WatermarkStrategy
        val ws: WatermarkStrategy[VisitLog] = WatermarkStrategy.forBoundedOutOfOrderness[VisitLog](Duration.ofSeconds(1))
            .withTimestampAssigner(new SerializableTimestampAssigner[VisitLog] {
                override def extractTimestamp(element: VisitLog, recordTimestamp: Long): Long = element.getEventTime * 1000L
            })

        //获取数据流
        val resource = getClass.getResource("/testdata/OrderLog.csv")
        val eventStream = env.readTextFile(resource.getPath)
            .map(data=>{
                val fields = data.split(",")
                new VisitLog(fields(0).toLong, fields(1), fields(3).toLong)
            }).assignTimestampsAndWatermarks(ws)
            .keyBy(_.getOrderId)
```

**第三步：从外部系统获取groovy的pattern脚本，并通过GroovyClassLoader加载解析groovy的pattern脚本，从而得到Pattern对象**

```scala
//从数据库mysql获取groovy规则脚本
        Class.forName("com.mysql.cj.jdbc.Driver")
        val url = "jdbc:mysql://192.168.42.71:3306/test?useUnicode=true&characterEncoding=utf-8"
        val connection = DriverManager.getConnection(url, "root", "qvjjgGRuc5,i123")
        val sql = "select pattern_script from test.cep_patterns where id= ?"
        val statement = connection.prepareStatement(sql)
        statement.setInt(1,ruleId)
        val scripts: ResultSet = statement.executeQuery()
        var script = ""
        while(scripts.next()){
            script = scripts.getString(1)
        }
        //从文件中获取 groovy规则脚本
        /*val groovyUrl = "E:\\workspace\\yjson-flink-demo\\dynamic-rules\\src\\main\\java\\json\\rules\\sRule.groovy"
        val script = new String(Files.readAllBytes(Paths.get(groovyUrl)), StandardCharsets.UTF_8)*/
        println(script)

        // 通过GroovyClassLoader加载groovy规则脚本，并执行run方法获取Pattern对象
        val groovyClassLoader = new GroovyClassLoader
        val groovyClass = groovyClassLoader.parseClass(script)
        val obj = groovyClass.newInstance.asInstanceOf[GroovyObject]
        val pattern = obj.invokeMethod("run",null).asInstanceOf[org.apache.flink.cep.pattern.Pattern[VisitLog,VisitLog]]
```

**第四步：将Pattern应用到流中**

```scala
// 将Pattern对象应用到流中，注意点：通过groovy脚本得到的是java包下的pattern，需要包装成scala包下的pattern
        val patternedStream: PatternStream[VisitLog] = CEP.pattern[VisitLog](eventStream, new Pattern[VisitLog,VisitLog](pattern))
```

**第五步：对匹配的数据进行业务处理**

```scala
// 获取pattern匹配上的数据
        //val result: DataStream[VisitLog] = patternedStream.select(new CustomPatternSelectFunction)
        val result: DataStream[MyOrderResult] = patternedStream.process(new CustomPatternProcessFunction(ruleName))
```

**通过自定义的PatternProcessFunction类将规则名称传入，在处理完后添加到输出数据中**

```scala
class CustomPatternProcessFunction(labelName:String) extends PatternProcessFunction[VisitLog,MyOrderResult]{
        override def processMatch(matched: util.Map[String, util.List[VisitLog]], ctx: PatternProcessFunction.Context,
                                  out: Collector[MyOrderResult]): Unit = {
            val payEvent: VisitLog = matched.get("pay").iterator().next()
            //println(payEvent.name)
            //println(matched.size())
            out.collect(MyOrderResult(payEvent.getOrderId,labelName))
        }
    }
```

## 3.注意点

* GroovyClassLoader加载解析groovy的pattern脚本，从而得到Pattern对象是java包下的Pattern，需要包装成scala包下的Pattern。
* 数据库中存储规则的表的id和名称需要唯一，方便查询，名称可以是标签名称，用来区分是哪个规则的输出。
* 如果确实flink-shaded包下的类，需要另外导入flink-shaded-asm-7-7.1-13.0.jar，可以在本地仓库中找到。