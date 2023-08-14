# BusinessTrace

 一个完整的业务流程往往是跨时间、跨服务甚至跨团队的，对于业务的流程追踪往往是在代码中编写相应的记录逻辑，侵入性过高，通过 skywalking 获得灵感，使用java agen + asm 实现无侵入式的、可配置的业务追踪项目。

// TODO git死活传不上去。。。代码敬请期待。。。



## 实现原理

通过字节码技术在 class 文件加载阶段修改方法的字节码 —— 插入一个方法调用将目标方法的所有参数按照模板发送到指定位置。

1. **听起来有点像是一个日志记录功能。**完全没毛病，就是相当于在代码中用 aop 记录下方法的入参，只不过完全无侵入
2. **只是记录方法参数，和业务追踪好像没什么关系。**确实，日志记录可以做到通用，但是业务不太可能，这里只是采用完全无侵入式的方式记录日志，至于怎么做到业务链路可视化追踪等等，基本的数据日志有了，怎么做就是查询问题。

## 现在可以做什么

1. 使用 java agent 技术无侵入式的对字节码逻辑进行增强
2. 通过配置文件指定方法和模板，BusinessTrace 会将方法的入参按照模板进行构建并发送到指定地址

## 怎么做

先来看一下样例配置文件

```json
{
  "springVersion": "",
  "http": {},
  "headers": {},
  "bizConfig": {
    "请假流程": {
      "http": {
        "method": "post",
        "url": "http://127.0.0.1:8080/post"
      },
      "headers": {
        "app-name": "${spring.application.name}",
        "active": "${spring.profiles.active}"
      },
      "target": {
        "com.xxx.model1.controller.*": {
          "method*()": {
            "template": {
              "field1": "${arg1.id}",
              "field2": "${arg2.getName()}",
              "field3": "${arg3.get('str')}",
              "field4": "${arg3.get2('${arg1.id})}",
              "field5": "${statics['com.example.demo.test.HeaderStore'].getName()}",
              "field6": "${statics['com.example.demo.test.HeaderStore'].FOO}",
              "field7": "${statics['com.example.demo.test.HeaderStore'].get3('${arg2.getName()}')}"
            }
          }
        },
        "com.xxx.model1.*": {
          "*": {
            "template": {}
          }
        }
      }
    },
    "报销流程": {
      "http": {},
      "headers": {},
      "target": {
        "com.xxx.model2.XxxController": {
          "createTask": {
            "template": {}
          },
          "upload()": {
            "template": {}
          },
          "examine(int,long,java.lang.Integer,java.lang.String)": {
            "template": {}
          },
          "approve*": {
            "template": {}
          }
        }
      }
    }
  }
}
```

### 配置文件字段详解

- `springVersion`: spring的版本，由于需要记录 spring 应用程序上下文，因此需要提供版本（目前这字段没用）
- `http`: 消息发送方式 BusinessTrace 会将方法参数按照此地址发送，后续会支持 kafka 等方式
- `headers`: 发送时候的消息头数据，按照指定的 key 从 spring 应用程序上下文中获取
- `bizConfig`: 核心配置，用于将**指定方法**的参数按照**指定模板**发送到**指定位置**
- `bizConfig.bizName`:  业务名称，会自动带入到消息模板的 bizName 字段，除非你在模板中设置了这个字段
- `bizConfig.bizName.http`: 同 `http`，可将其内容覆盖，若不指定则用 `http`的配置
- `bizConfig.bizName.headers`: 同 `headers`，可将其内容覆盖，若不指定则用 `headers`的配置
- `bizConfig.bizName.target`: 核心的核心，用于**指定方法和模板**
- `bizConfig.bizName.target.classPath`: 指定方法所在的类，支持通配符
- `bizConfig.bizName.target.classPath.method`: 指定方法签名，仅方法名字支持通配符。
- `bizConfig.bizName.target.classPath.method.template`: 指定模板，将当前方法的参数按照此模板填充

### 配置文件字段规则详解

- `bizConfig.bizName`:  即样例配置文件中的`请假流程、报销流程`所在的位置直属于`bizConfig`，意思是该属性的配置属于`请假流程`或`报销流程`业务流的配置
- `bizConfig.bizName.target.classPath`: 即样例配置文件中的`com.xxx.model1.controller.*`所在位置，直属于`bizConfig.bizName.target`的字段，key 为类的通配符，value 为该类下的方法等配置
- `bizConfig.bizName.target.classPath.method`: 即样例配置文件中的`method*()`所在位置，直属于`bizConfig.bizName.target.classPath`的字段，key 为方法签名，方法名支持通配符，参数不支持。当只有方法名而没有 `()`时，会按照方法名对该类下所有方法进行匹配，当存在 `()`时，若`()`内部没有参数，则在按照方法名匹配后取无参数的方法，若内部有参数，则严格按照参数顺序以及类型进行匹配。
- `bizConfig.bizName.target.classPath.method.template`: 参数填充模板，采用的 freeMarker 模板引擎，语法同 freeMarker。`arg1`为第一个参数`arg2`为第二个参数，以此类推。样例中 field5、field6、field7 的写法分别为调用 HeaderStore 类的静态无参方法、静态字段（必须 public 修饰）、静态有参方法。

## 怎么用。

启动命令改为

```shell
java -jar .../app.jar -javaagent:.../BusinessTrace-0.1-SNAPSHOT.jar=nacos://127.0.0.1.:8848?namespace=public&username=root&password=root&group=DEFAULT_GROUP&dataId=bizconfig.json
```

后面的参数为 BusinessTrace 的配置的地址，目前只支持 nacos（没错，本地文件都不支持~~~）

## 将来可以做什么

- 目前只支持参数的发送，后续会考虑将返回值也纳入
- 发送异步化，目前还是同步发送，等于在方法最开始调用了一个方法将参数发送一下
- 发送方式将支持更多，kafka 一定会支持的，其他 mq 也会考虑
- 配置中心也会支持更多，项目的 resources 目录下配置一定会支持，Apollo 等其他配置方式，...，不太想考虑
- 
