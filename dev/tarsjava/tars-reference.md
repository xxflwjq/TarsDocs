# Tars参考

Tars协议是一种基于 IDL 实现的协议，与 Protocol Buffer 类似，它与语言无关，是一种类 C++ 标识符的语言，用于生成具体的服务接口文件。同时，作为一种二进制协议，相较于常见的 JSON 等文本协议，它的编解码效率更高、网络包占用空间更小。

Tars文件使用.tars作为扩展名，对于.tars文件中的每个服务，生成代码时都会对应产生一个Java接口，若为服务端接口代码生成时会加上Servant后缀，若为客户端接口则会加上Prx后缀。Tars语言的语法规则请参考[Tars协议](../../base/tars-protocol.md)



## Tars插件

Tars的maven插件需要在pom.xml文件中添加相关依赖：

```xml
<!--tars2java插件-->
<plugin>
	<groupId>com.tencent.tars</groupId>
	<artifactId>tars-maven-plugin</artifactId>
	<version>1.7.0</version>
	<configuration>
		<tars2JavaConfig>
			<!-- tars文件位置 -->
			<tarsFiles>
				<tarsFile>${basedir}/src/main/resources/hello.tars</tarsFile>
			</tarsFiles>
			<!-- 源文件编码 -->
			<tarsFileCharset>UTF-8</tarsFileCharset>
			<!-- 生成服务端代码 -->
			<servant>true</servant>
			<!-- 生成源代码编码 -->
			<charset>UTF-8</charset>
			<!-- 生成的源代码目录 -->
			<srcPath>${basedir}/src/main/java</srcPath>
			<!-- 生成源代码包前缀 -->
			<packagePrefixName>com.qq.tars.quickstart.server.</packagePrefixName>
		</tars2JavaConfig>
	</configuration>
</plugin>
```

其中的一些配置项如下：

- tarsFiles：用于指示.tars文件的位置
- tarsFileCharset：源文件的编码格式
- servant：true表示生成服务端代码，即生成Servant为后缀的接口，false表示生成客户端代码，即生成Prx为后缀的接口
- charset：生成的源代码的编码格式
- srcPath：生成的源代码目录
- packagePrefixName：生成的源代码的包前缀



## 服务端Tars文件

以tars-quick-start中服务端的hello.tars为例：

```text
module TestApp
{
	interface Hello
	{
	    string hello(int no, string name);
	};
};
```

其中TestApp为名称空间，所有的struct，interface必须在名字空间中。

服务端代码生成时需要设置pom.xml文件中的tars-maven-plugin依赖的servant项为true，即可获得Servant为后缀的接口文件，HelloServant.java：

```java
@Servant
public interface HelloServant {

	public String hello(int no, String name);
}
```

之后，根据业务逻辑来实现接口即可。



## 客户端Tars文件

在客户端进行服务调用时，首先需要获取服务端的tars文件，之后在客户端代码生成时需要设置pom.xml文件中的tars-maven-plugin依赖的servant项为false，即可获得Prx为后缀的接口文件，HelloPrx.java：

```java
@Servant
public interface HelloPrx {

	 String hello(int no, String name);

	CompletableFuture<String>  promise_hello(int no, String name);

	 String hello(int no, String name, @TarsContext java.util.Map<String, String> ctx);

	 void async_hello(@TarsCallback HelloPrxCallback callback, int no, String name);

	 void async_hello(@TarsCallback HelloPrxCallback callback, int no, String name, @TarsContext java.util.Map<String, String> ctx);
}
```

接口提供了三种调用方式：

- 同步调用
- 异步调用
- promise调用

此外，在客户端代码生成时还生成了HelloPrxCallback.java，这是一个普通异步回调处理的抽象类：

```java
public abstract class HelloPrxCallback extends TarsAbstractCallback {

	public abstract void callback_hello(String ret);

}
```

该类继承了TarsAbstractCallback这个抽象类：

```java
@TarsCallback(comment = "Callback")
public abstract class TarsAbstractCallback implements com.qq.tars.net.client.Callback<TarsServantResponse> {

    @Override
    public final void onCompleted(TarsServantResponse response) {
        TarsServantRequest request = response.getRequest();
        try {
            Method callback = getCallbackMethod("callback_".concat(request.getFunctionName()));
            callback.setAccessible(true);
            callback.invoke(this, ((TarsCodec) response.getSession().getProtocolFactory().getDecoder()).decodeCallbackArgs(response));
        } catch (Throwable ex) {
            throw new ClientException(ex);
        }
    }

    private Method getCallbackMethod(String methodName) throws NoSuchMethodException {
        Method[] methods = getClass().getDeclaredMethods();
        for (Method method : methods) {
            if (methodName.equals(method.getName())) {
                return method;
            }
        }
        throw new NoSuchMethodException("no such method " + methodName);
    }

    @Override
    public final void onException(Throwable ex) {
        callback_exception(ex);
    }

    @Override
    public final void onExpired() {
        callback_expired();
    }

    public abstract void callback_exception(Throwable ex);

    public abstract void callback_expired();
}

```

通过实现callback_exception，callback_expired和callback_hello方法，可以执行相应的回调逻辑。

- 当接收到服务端返回时，HelloPrxCallback的callback_hello会被响应。
- 如果调用返回异常或超时，则callback_exception会被调用。



## 代码生成

在工程的根目录下执行`mvn tars:tars2java`命令，即可生成对应的代码。