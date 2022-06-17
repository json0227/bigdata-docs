# 一、Groovy集成Java的四种方式
## 1.GroovyShell
GroovyShell可以从String、Reader、File、InputStream等中读取脚本执行，而且脚本和程序之间可以进行参数传递，但GroovyShell性能非常差，每次会编译class带来运行效率、GC、perm区爆满各种问题，一般用于简单的做一些demo，实际生产不建议使用。

```
public void groovyShellTest() throws IOException {
        GroovyShell shell = new GroovyShell();
        // 从String中读取
        Object result = shell.evaluate("3 * 5");
        // 从Reader中读取
        Object result1 = shell.evaluate(new StringReader("3 * 5"));
        // 从文件中读取
        String groovyUrl12 = "C:\\Users\\json\\Desktop\\临时文档\\tmp\\groovytest.groovy";
        // 脚本和程序之间参数传递
        Binding binding = new Binding();
        binding.setProperty("input","9|shangguanwaner|23|湛江|shang guanwaner");
        GroovyShell groovyShell = new GroovyShell(binding);
        Object result3 = groovyShell.evaluate(new File(groovyUrl12));
        // 从脚本中获取定义的参数
        String output = (String)binding.getProperty("output");
        System.out.println(result3);
        System.out.println(output);
    }
```
**groovytest.groovy文件**

```
def sayResult(){
    println("当前英雄是shangguanwaner！")
}
sayResult()

def dataConversion(args){
    String result = "";
    String[] fields = input.split("\\|");
    String name = fields[1];
    Integer age = Integer.valueOf(fields[2]);
    if(name.equals("shangguanwaner")){
        if(age<30){
            result = "当前英雄是：" + name + "  结论是：年轻法师，特别适合切后排！"
        }else if(age>=30 && age<50){
            result = "当前英雄是：" + name +  "  结论是：巅峰法师，可以七进七出！"
        }else {
            result = "当前英雄是：" + name +  "  结论是：年龄不合适，可以删除了！"
        }
    }
    result;
}

output = "下面执行数据转换方法！"

dataConversion(input)
```

## 2.GroovyClassLoader
 GroovyClassLoader 动态地加载一个脚本并执行它的行为。GroovyClassLoader是一个定制的类装载器，负责解释加载Java类中用到的Groovy类。GroovyClassLoader在新版groovy中有做优化，对于以文件作为输入时，会对class做缓存，避免对同一个脚本创建多个类，其他输入需要根据groovy脚本变更做reload。各方面性能较好推荐使用。

```
public void classLoaderTest() throws Exception {
        GroovyClassLoader groovyClassLoader = new GroovyClassLoader();
        String groovyUrl = "C:\\Users\\json\\Desktop\\临时文档\\tmp\\classLoadertest.groovy";
        Class<?> groovyClass = groovyClassLoader.parseClass(new File(groovyUrl));
        GroovyObject obj = (GroovyObject) groovyClass.newInstance();
        String result = (String) obj.invokeMethod("dataConversion","9|shangguanwaner|23|湛江|shang guanwaner");
        System.out.println(result);
    }
```
**classLoadertest.groovy**

```
class DC {
    def dataConversion(input){
        String result = ""
        String[] fields = input.split("\\|")
        String name = fields[1];
        Integer age = Integer.valueOf(fields[2])
        if(name.equals("shangguanwaner")){
            if(age<30){
                result = "当前英雄是：" + name + "  结论是：年轻法师，特别适合切后排！"
            }else if(age>=30 && age<50){
                result = "当前英雄是：" + name +  "  结论是：巅峰法师，可以七进七出！"
            }else {
                result = "当前英雄是：" + name +  "  结论是：年龄不合适，可以删除了！"
            }
        }
        result
    }
}
```

## 3.GroovyScriptEngine

GroovySciptEngine本质上是对GroovyClassLoader的封装，各方面性能也较好，可以指定的位置（文件系统，URL，数据库，等等）加载Groovy脚本，并且随着脚本变化而重新加载它们。如同GroovyShell一样，GroovyScriptEngine也允许传入参数值，并能返回脚本的值。内部封装逻辑较多，没法做进一步优化，而且因为每次都要去检查脚本有没有变更并new instance(本质是是轮训而非异步变更)，导致性能不如GroovyClassLoader。

```
public void scriptEngineTest() throws Exception {
        String groovyUrl11 = "C:\\Users\\json\\Desktop\\临时文档\\rules_groovy";
        File file = new File(groovyUrl11);
        // 可以指定一个文件夹作为数据源
        GroovyScriptEngine groovyScriptEngine = new GroovyScriptEngine(file.getAbsolutePath());
        //通过加载的方式获取
        Class<?> groovyClass = groovyScriptEngine.loadScriptByName("DaYe.groovy");
        GroovyObject obj =(GroovyObject)groovyClass.newInstance();
        String result =(String)obj.invokeMethod("dataConversion","14|zhaoyun|44|新疆|zhao yun");
        System.out.println(result);
        //通过创建脚本的方式获取
        Binding binding = new Binding();
        Script script = groovyScriptEngine.createScript("DaYe.groovy", binding);
        Object result1 = script.invokeMethod("dataConversion", "14|zhaoyun|44|新疆|zhao yun");
        System.out.println(result1);

        //执行目录下的所有脚本
        File[] files = file.listFiles();
        assert files != null;
        for (File f : files) {
            String fileName = f.getName();
            System.out.println(fileName);
            Class<?> groovyClass11 = groovyScriptEngine.loadScriptByName(fileName);
            GroovyObject obj11 = (GroovyObject) groovyClass11.newInstance();
            String result11 = (String) obj11.invokeMethod("dataConversion", "9|shangguanwaner|23|湛江|shang guanwaner");
            System.out.println(result11);
        }
    }
```

**DaYe.groovy**

```
def dataConversion(String input){
    String result = "不是打野！";
    String[] fields = input.split("\\|");
    String name = fields[1];
    Integer age = Integer.valueOf(fields[2]);
    Long date = new Date().getTime();
    if(name.equals("zhaoyun")){
        if(age<30){
            result = "处理时间是:" + date + " 当前英雄是：" + name + "  结论是：年轻战士，特别适合切后排！"
        }else if(age>=30 && age<50){
            result = "处理时间是:" + date + " 当前英雄是：" + name +  "  结论是：巅峰战士，可以七进七出！"
        }else {
            result = "处理时间是:" + date + " 当前英雄是：" + name +  "  结论是：年龄不合适，可以删除了！"
        }
    }
    return result
}

```



## 4.JSR-223

JSR-223是java调用其他脚本语言框架的标准API。只能从String和Reader中获取解析脚本
```
public void scriptEngineManagerTest() throws Exception {
        ScriptEngine engine = new ScriptEngineManager().getEngineByName("groovy");
        String fileUrl = "C:\\Users\\json\\Desktop\\临时文档\\rules_groovy\\FaShi.groovy";
        String script = new String(Files.readAllBytes(Paths.get(fileUrl)), StandardCharsets.UTF_8);
        //Bindings bindings = engine.createBindings();
        engine.eval(script);
        Object result = ((Invocable) engine).invokeFunction("dataConversion", "9|shangguanwaner|23|湛江|shang guanwaner");
        System.out.println(result);
        
        engine.eval(new StringReader(script));
        Object result1 = ((Invocable) engine).invokeFunction("dataConversion", "9|zhaoyun|23|湛江|shang guanwaner");
        System.out.println(result1);
    }
```
### FaShi.groovy
```
def dataConversion(input){
    String result = "不是法师！"
    String[] fields = input.split("\\|")
    String name = fields[1]
    Integer age = Integer.valueOf(fields[2])
    if(name.equals("shangguanwaner")){
        if(age<30){
            result = "当前英雄是：" + name + "  结论是：年轻法师，特别适合切后排！"
        }else if(age>=30 && age<50){
            result = "当前英雄是：" + name +  "  结论是：巅峰法师，可以七进七出！"
        }else {
            result = "当前英雄是：" + name +  "  结论是：年龄不合适，可以删除了！"
        }
    }
    result
}
```
# 二、Groovy的ClassLoader体系
Groovy中最主要的3个ClassLoader：

* RootLoader：管理了Groovy的classpath，负责加载Groovy及其依赖的第三方库中的类，它不是使用双亲委派模型。
* GroovyClassLoader：负责在运行时编译groovy源代码为Class的工作，从而使Groovy实现了将groovy源代码动态加载为Class的功能。
* GroovyClassLoader.InnerLoader：Groovy脚本类的直接ClassLoader，它将加载工作委派给GroovyClassLoader，它的存在是为了支持不同源码里使用相同的类名，以及加载的类能顺利被GC。

# 三、Groovy中的script如何转换成类

Groovy同时支持scripts和classes，classes和java基本一样，scripts会被groovy的编译器编译成类。比如当前有一个脚本Test.groovy

```
print "hello groovy"
int add(int a, int b){a + b}
add(10, 20)
```

这个脚本会编译成下面的类

```
import org.codehaus.groovy.runtime.InvokerHelper
class Test extends Script {
    int add(int a, int b){a + b}         
    def run() {
        print "hello groovy"                         
        add(10, 20)             
    }
    static void main(String[] args) {
        InvokerHelper.runScript(Main, args)
    }
}
```

编译过程为：

* 类名： 将文件的名称作为类的名称
* 方法：脚本中的方法直接作为类的成员方法
* 脚本体：在类中会自动生成一个run方法，脚本语句放在run方法中
* main方法： 在类中会自动生成一个main方法，用来执行run方法





