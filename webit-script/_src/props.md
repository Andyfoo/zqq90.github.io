## 哪些组件可以配置？
Febit Wit 采用IoC的机制管理组件，也就是说只要是提供了setter的组件，随处都可以配置。当前现在支持属性值自动转换为以下类型：String, class, int/Integer, boolean/Boolean，以及他们的数组形式。
例如：

~~~~~
# 格式为 key = value
# 设置输出编码 Engine.setEncoding(String)
engine.encoding=UTF-8
# 设置默认面声明变量 Engine.setVars(String[])
engine.vars=request, response
~~~~~

> 这是一个开放的注入管理器，只要需要，就可以把配置写在这里面，然后通过Engine.get(type)得到配置的实例

> 组件如果标记了 `@org.febit.wit.Init 同时还会在配置注入完成之后执行


## 获取引擎实例

> `Engine.create("配置文件列表", 额外的参数Map)`

### 1.配置文件列表

+ 配置文件之间用`,`分割
+ 配置文件需要放在ClassPath下
+ 配置文件按先后顺序被附加或者覆盖
+ 配置事先加载了ClassPath下的`/default.wim`，不需要另行设置
+ 如果配置文件不存在或者加载失败会报错，加载成功的配置文件将会通过Logger.info() 按加载顺序打印

> 缺省配置文件:[default.wim][default_config]

### 2.额外的参数Map

+ 可以为null
+ 额外的参数如果是String,会附加/覆盖到props配置，使用的插值`${}`会生效
+ 额外参数默认是采取覆盖策略，如果使用附加请在key后面附加`+`,例如`engine.resolvers+`
+ 如果值不是String类型的，会被原样保留并注入到指定的组件中。

## 一段配置文件节选（仅演示，非实际配置）
~~~~~
# 以#开头的是行注释


# 设置了一个数值
DEFAULT_ENCODING=UTF-8

# @开头的一般都有特殊意义
# 依赖的模块列表
@modules=

# 设置一个参数: 类型.字段名=值
# 和 properties 一样
engine.looseVar=false

# 例如 @global 被用来声明哪些是全局组件
# IoC 会自动注入到需要这些组件的字段，例如 NativeFactory.nativeSecurityManager
@global='''
    engine
    logger
    loader
    resolverManager
    textStatement
    global
'''

# 这里先设置区段 [engine]， 这样就可以简短书写了
# 区段中 : 用来定义类型继承自那个类型
#   也可以是定义过类型的区段（实例化的时候总得一层层找到他的类型， 对吧。）
[engine :org.febit.wit.Engine]
encoding=${DEFAULT_ENCODING}
# looseVar=false
# trimCodeBlockBlankLine=true
# vars=
# inits=
# shareRootData=true

# 这里就是个例子，loader 继承自routeLoader包括类型和其他参数
# 另外还记得前面 loader 被添加到全局组件了吧
# 不要担心会有多个 routeLoader, 如果没有`Engine.get("routeLoader")`, 它只是个配置，并不会被实例化
# 不过， Engine.get("loader") 和 Engine.get("routeLoader") 不是同一个实例
# 这个继承关系可以在以后被覆盖，无状态，不会有之前的继承参数
# 例如 改成 [loader :servletLoader]
[loader :routeLoader]
# 日志组件的实现
[logger :simpleLogger]
# 更改日志组件为 slf4j
[logger :slf4jLogger]

[textStatement :simpleTextStatement]

# 下面你可以当做是定义了别名， 以后就可以使用别名了，不用写包名，是不是很省劲
[simpleLogger :org.febit.wit.loggers.impl.SimpleLogger]
[slf4jLogger :org.febit.wit.loggers.impl.Slf4jLogger]
[commonLogger :org.febit.wit.loggers.impl.CommonLogger]

[pathLoader]
encoding=${DEFAULT_ENCODING}
assistantSuffixs+=.wit
suffix=.wit
# appendLostSuffix=false
# root=your/template/path

[classpathLoader :org.febit.wit.loaders.impl.ClasspathLoader]
# @extends 仅仅会把参数继承过来，不会继承类型，覆盖无状态，可以是个列表
@extends=pathLoader

[stringLoader :org.febit.wit.loaders.impl.StringLoader]

# 为了更直观可以使用 \ 转义看不见的换行符
# 当然不换行也是可以的
[resolverManager]
resolvers=\
  org.febit.wit.resolvers.impl.LongOutResolver,\
  org.febit.wit.resolvers.impl.IntegerOutResolver

# 这里 "DEFAULT_ENCODING" 又被使用了一次
# 先设置区段[]为空
[]
# 又可以使用完整的“类型.字段名”了
resolverManager.ignoreNullPointer=true
~~~~~

## 关于配置文件
> 采用内置的[Jodd-props][jodd_props_doc], 文法类似于ini

### 区段头
+ 区段头不一定非要是类型名，段头只作为区段头下的所有配置的前缀（附加一个点`.`）

### 使用插值（宏）
+ 例如例子中的`DEFAULT_ENCODING`

### 多行模式
+ 使用三个单引号`'''`开头 并在尾行以此结尾
+ 类型列表 的分隔符可以是`,` 或者换行，并忽略首位空格
~~~~~
list += '''
  org.febit.wit.resolvers.impl.LongOutResolver
  org.febit.wit.resolvers.impl.IntegerOutResolver
  org.febit.wit.resolvers.impl.InternalVoidResolver
'''
~~~~~

### 列表值的附加

+ 使用`属性名 += 值`的形式即可
+ `=` 将会根据先来后到的规则覆盖掉最早设置的值，使用`+=`将会跨配置文件保留先前的值，并附加上当前的值
+ 附加的值是使用的逗号`,`分割的

### 属性继承

~~~~~
[classpathLoader :org.febit.wit.loaders.impl.ClasspathLoader]
@extends=pathLoader
~~~~~
这样将会使得 classpathLoader 继承所有以"pathLoader."开头的所有配置

> 注意：这种继承之间的附加操作"+=" 是无效的，原生的将直接覆盖掉继承来的值

### 依赖模块的

+ 使用 `@modules=值` 可以加载依赖的配置文件(或者叫模块)
+ 依赖的模块按照设定的顺序依次加载
+ 加载完毕依赖模块才会加载本配置文件
+ 依赖的依赖也会被加载，同一个配置文件不会被重复加载

~~~~~
@modules +='''
lib-assert.wim
lib-type.wim
lib-cache.wim
'''
~~~~~


## 常用配置一览
<table class="table table-striped table-bordered "><tbody>
<tr>
	<td>DEFAULT_ENCODING</td>
	<td>字符串</td>
	<td>默认编码</td>
</tr>
<tr>
	<td>engine.encoding</td>
	<td>字符串</td>
	<td>输出编码</td>
</tr>
<tr>
	<td>engine.looseVar</td>
	<td>Boolean</td>
	<td>是否启用宽松的变量声明</td>
</tr>
<tr>
	<td>engine.trimCodeBlockBlankLine</td>
	<td>Boolean</td>
	<td>是否删除指令所占行</td>
</tr>
<tr>
	<td>engine.textStatementFactory</td>
	<td>类型</td>
	<td>TextStatment生成器</td>
</tr>
<tr>
	<td>engine.vars</td>
	<td>列表</td>
	<td>免声明变量名</td>
</tr>
<tr>
	<td>engine.inits</td>
	<td>列表</td>
	<td>初始化模板</td>
</tr>
<tr>
	<td>engine.shareRootData</td>
	<td>Boolean</td>
	<td>对子模版共享传入的参数</td>
</tr>
<tr>
	<td>routeLoader.defaultLoader</td>
	<td>类型</td>
	<td>路由加载器的缺省加载器</td>
</tr>
<tr>
	<td>routeLoader.loaders</td>
	<td>路由规则</td>
	<td>路由加载器的路由规则</td>
</tr>
<tr>
	<td>resolverManager.resolvers</td>
	<td>类型列表</td>
	<td>bean属性解释器</td>
</tr>
<tr>
	<td>resolverManager.ignoreNullPointer</td>
	<td>Boolean</td>
	<td>是否忽略属性读取时bean为null的异常</td>
</tr>
<tr>
	<td>global.registers</td>
	<td>类型列表</td>
	<td>全局变量/常量 注册器</td>
</tr>
<tr>
	<td>nativeSecurity.list</td>
	<td>列表</td>
	<td>Native黑白名单</td>
</tr>
</tbody></table>

## 资源加载器加载器

### 基本的资源加载器

+ Classpath `classpathLoader`
+ Web 根目录 `servletLoader`
+ 普通文件系统 `fileLoader`
+ String类型的模板  `stringLoader`

~~~~~~
## Classpath 资源加载
[classpathLoader]
# 模板根路径
root=your/template/path

## Web 根目录 资源加载
[servletLoader]
root=/your/template/path

## 普通文件系统 资源加载
[fileLoader]
root=/your/template/path
~~~~~~

### Loader路由：routeLoader

+ 根据进行前缀的贪婪匹配
+ 匹配的Loader之后 会删除设置的前缀
+ 理论上可以使用任意前缀
+ 可以设置失败后缺省的Loader
+ 同一个Loader实例可以设置多个匹配的前缀

~~~~~
# 新增两个 loader， 继承自 classpathLoader
[classpathLoader-root :classpathLoader]
[classpathLoader-root2 :classpathLoader]

# 同上，还可以修改参数
[codeLoader :stringLoader]
codeFirst=true

# routeLoader 已经配置为全局 loader
# [loader :routeLoader]
# 改成其他的可以使用:
# [loader :classpathLoader]
# ^ 这样 所有资源直接使用 classpathLoader 来加载，不会走 routeLoader

[routeLoader]
# 缺省的Loader
default=classpathLoader
# 路由规则
loaders +='''
  classpath:   classpathLoader-root
  classpath2:  classpathLoader-root2
  /sub/path    classpathLoader-root
  str:         stringLoader
  code:        codeLoader
'''
~~~~~

> 以上面的配置为例："classpath:/a.wit","/sub/path/a.wit" 都将会使用"classpathLoader"加载"/a.wit"； "code:var a =1;"将会作为字符串模版被加载为"var a =1;"；"classpath2:/b.wit" 将会使用"classpathLoader-root2"进行加载，其余的如 "/a.wit" "/b/sub/path/a.wit" 会使用缺省的"classpathLoader" 进行加载

### 延迟模版过期检查：lazyLoader

> 可以包裹一个普通loader，并设置一定时间内不检查更新（使用一些检查成本较高的模板时非常有用）

### 简单的安全过滤：simpleSecurityLoader

> 可以包裹一个普通loader，并设置安全名单 

## 全局常量/变量/函数注册

> 首先，继承接口GlobalRegister

> 然后在方法`void regist(GlobalManager manager)`中进行注册

~~~~~
//如：
public void regist(final GlobalManager manager) {

	//全局变量
	SimpleBag globalBag = manager.getGlobalBag();
	globalBag.set("MY_GLOBAL", "MY_GLOBAL");
	globalBag.set("MY_GLOBAL_2", "MY_GLOBAL_2");

	//全局常量
	SimpleBag constBag = manager.getConstBag();
	constBag.set("MY_CONST", "MY_CONST");
	constBag.set("MY_CONST_2", "MY_CONST_2");

	//全局Native 函数
	constBag.set("new_list", this.nativeFactory.createNativeConstructorDeclare(ArrayList.class, null));
	constBag.set("list_size", this.nativeFactory.createNativeMethodDeclare(List.class, "size", null));
	constBag.set("list_add", this.nativeFactory.createNativeMethodDeclare(List.class, "add", new Class[]{Object.class}));
	constBag.set("substring", this.nativeFactory.createNativeMethodDeclare(String.class, "substring", new Class[]{int.class, int.class}));
	
	//全局自定函数
  constBag.set("is_bool", new MethodDeclare() {

      public Object invoke(Context context, Object[] args) {
          return ArrayUtil.get(args, 0, null) instanceof Boolean;
      }
  });
}
~~~~~

> 配置文件里注册该注册器

~~~~~
[global]
registers+= my.pkg.MyGlobalRegister
~~~~~



## TextStatment工厂

> TextStatment的作用是缓存模版文件中的文本片段，根据最终输出的类型不同可分别保存成不同的形式，如：char[]、byte[]。对于模版引擎的渲染，io以及编码消耗了很大一部分性能，选择一个更接近目标输出的形式有利于优化性能。

+ byte[]流：ByteArrayTextStatementFactory
+ char[]流：CharArrayTextStatementFactory
+ byte[]流 & char[]流：SimpleTextStatementFactory

> 另外可以继承`TextStatementFactory`提供更适合自己的TextStatement



## Native安全管理--黑白名单
+ 列表采用 `,``\r``\n` 分割，并自动删除前后空白字符
+ 标识可具体到包、类、方法名
+ 白名单规则：标识前加`+` 或者无修饰 为白名单
+ 黑名单规则：标识前加`-` 为黑名单
+ 同一标识，黑名单优先于白名单，如无特殊设置，将继承上一级

> 默认配置已经添加了基本类型为白名单
> 示例：

~~~~~
# 示例白名单：
#    com.dog
#    com.cat
#    com.mouse.jerry
# 黑名单：
#    com.mouse
#    com.cat.tom

[defaultNativeSecurity]
list='''
  com.dog
  com.cat
  com.mouse.jerry
- com.mouse
- com.cat.tom
'''
~~~~~


[default_config]: https://github.com/febit/wit/blob/master/wit-core/src/main/resources/default.wim
[jodd_props_doc]: http://jodd.org/doc/props.html
