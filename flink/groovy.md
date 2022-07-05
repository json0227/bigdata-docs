# Groovy集成Java的三种方式
## 1.GroovyShell
GroovyShell性能非常差，每次会编译class带来运行效率、GC、perm区爆满各种问题，一般用于简单的做一些demo，实际生产不建议使用。
```
public static void shellTest() throws IOException {
        GroovyShell groovyShell = new GroovyShell();
        String groovyUrl = "E:\\workspace\\FlinkApiTheory\\Flink1.14\\groovy\\Person.groovy";
        String result =(String) groovyShell.evaluate(new File(groovyUrl));
        System.out.println(result);
    }
```
## 2.GroovyClassLoader
GroovyClassLoader保留它创建的所有类的引用，因此很容易创建内存泄漏。特别是，如果您两次执行同一个脚本，如果它是一个字符串，那么您将获得两个不同的类，原因是GroovyClassLoader不跟踪源文本。GroovyClassLoader使用文件作为输入，能够缓存生成的类文件，从而避免在运行时为同一个源创建多个类。
```
public static void classLoaderTest() throws IOException, IllegalAccessException, InstantiationException {
        GroovyClassLoader groovyClassLoader = new GroovyClassLoader();
        String groovyUrl = "E:\\workspace\\FlinkApiTheory\\Flink1.14\\groovy\\Groovy_1.groovy";
        File file = new File(groovyUrl);
        Class<?> groovyClass = groovyClassLoader.parseClass(file);
        GroovyObject obj = (GroovyObject)groovyClass.newInstance();
        String result = (String) obj.invokeMethod("sayHello", "jack|male");
        System.out.println(result);
    }
```
## 3.ScriptEngineManager
GroovySciptEngine本质上是对GroovyClassLoader的封装，各方面性能也较好，可以指定的位置（文件系统，URL，数据库，等等）加载Groovy脚本，并且随着脚本变化而重新加载它们。如同GroovyShell一样，GroovyScriptEngine也允许传入参数值，并能返回脚本的值。内部封装逻辑较多，没法做进一步优化，而且因为每次都要去检查脚本有没有变更并new instance(本质是是轮训而非异步变更)，导致性能不如GroovyClassLoader。
```
public static void scriptEngineManagerTest() throws IOException, ResourceException, ScriptException, javax.script.ScriptException, NoSuchMethodException {
        ScriptEngine engine = new ScriptEngineManager().getEngineByName("groovy");
        Path path = Paths.get("E:\\workspace\\FlinkApiTheory\\Flink1.14\\src\\main\\java\\com\\json\\groovy\\Person.groovy");
        byte[] data = Files.readAllBytes(path);
        String script = new String(data, "utf-8");
        Bindings bindings = engine.createBindings();
        bindings.put("date",new Date());
        engine.eval(script, bindings);
        Object result = ((Invocable) engine).invokeFunction("sayHello");
        System.out.println(result);
    }
```
### Person.groovy
```
def sayHello(){
    println("welcome to this world!")
    return date.getTime();
}
```
# Flink集成Groovy
## 1.整体思路  
Flink采用ScriptEngineManager的方式调用Groovy脚本，Groovy脚本存储再Mysql数据库中，Flink数据流中的每条数据都调用脚本执行，达到动态数据转换的效果。
## 2.Flink主代码
```
object DynamicRule {
    def main(args: Array[String]): Unit = {
        val env = FlinkUtils.getEnv
        env.setParallelism(1)
        val source = KafkaSource.builder[String]
            .setBootstrapServers("rhel071:9092,rhel072:9092,rhel075:9092,rhel076:9092,rhel079:9092")
            .setTopics("user_info")
            .setGroupId("drools_test")
            .setStartingOffsets(OffsetsInitializer.earliest)
            .setValueOnlyDeserializer(new SimpleStringSchema())
            .build
        val ws: WatermarkStrategy[String] = WatermarkStrategy.forBoundedOutOfOrderness(Duration.ofSeconds(10))
        val inputStream: DataStream[String] = env.fromSource(source, ws, "Kafka Source")

        //inputStream.print("input===>")
        //val result = inputStream.map(new MyMap())
        val result = inputStream.process(new MyDynamicRuleProcess())
        result.filter(_.nonEmpty).print("result===>")
        env.execute(DynamicRule.getClass.getSimpleName)
    }

    class MyDynamicRuleProcess() extends ProcessFunction[String,String] {
        var connection: Connection = _
        var engine: ScriptEngine = _
        var statement: Statement = _
        var script: String = ""

        override def open(parameters: Configuration): Unit = {
            Class.forName("com.mysql.jdbc.Driver")
            val url = "jdbc:mysql://192.168.42.71:3306/test"
            connection = DriverManager.getConnection(url, "root", "qvjjgGRuc5,i123")
            statement = connection.createStatement()
            engine = new ScriptEngineManager().getEngineByName("groovy")
        }

        override def processElement(input: String, ctx: ProcessFunction[String, String]#Context,
                                    out: Collector[String]): Unit = {
            println(input)
            val result: ResultSet = statement.executeQuery("select class from test.groovy_class where name='rule'")
            var script = ""
            while (result.next()) {
                script = result.getString("class")
                engine.eval(script)
                val outputResult = engine.asInstanceOf[Invocable].invokeFunction("dataConversion", input).asInstanceOf[String]
                out.collect(outputResult)
            }
        }

        override def close(): Unit = {
            if(engine != null){
                engine = null
            }
            if(statement != null){
                statement.close()
            }
            if(connection != null){
                connection.close()
            }
        }
    }
}
```
## 3.Groovy转换规则
```
package com.json.groovy;
public class MyRule {
    static String dataConversion(String input){
        String result = "";
        String[] fields = input.split("\\|");
        String name = fields[1];
        if(name.equals("sunbin")){
            result = "sunbin 是一个辅助，特别适合打团！";
        }else if(name.equals("shangguanwaner")){
            result = "shangguanwaner 是一个法师，特别适合击杀射手！";
        }else {
            result = "不是什么好玩的英雄，建议删除";
        }
        return result;
    }
}
```





