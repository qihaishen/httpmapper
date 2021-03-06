# httpmapper
httpmapper是对httpasyncclient的简单封装，用于发送http请求并将返回数据转换成对象，以mapper接口的方式使用http-client，应用代码只需要写个interface的声明即可

## Get started
声明mapper接口：
```java
public interface TestServiceFacade {

  /**
   * 异步调用
   */
  @Request("http://localhost:8080/home/index.json?name=#{name}&test=1")
  Future<JsonResult<TestBean>> get(@HttpParam("name") String name);
  
  /**
   * 异步调用
   */
  @Request("http://localhost:8080/home/index.json?name=#{name}&test=1")
  ListenableFuture<JsonResult<TestBean>> getForListening(@HttpParam("name") String name);

  /**
   * 同步调用
   */
  @Request("http://localhost:8080/home/index.json?name=#{name}")
  @POST
  JsonResult<TestBean> post(@HttpParam("name") String name);
}
```
说明：httpmapper默认使用fastjson做json反序列化，JsonResult可换成任意对象（json可转成对应的对象类型）

初始化接口（后文会有与Spring集成的方式）：
```java
public void handleInvocation() throws Exception {

    final Configuration configuration = Configuration.newBuilder()
        .parse(TestServiceFacade.class)
        .build();
    
    final TestServiceFacade testServiceFacade = configuration.newMapper(TestServiceFacade.class);
    JsonResult<TestBean> result = testServiceFacade.get("name").get();
    System.out.println(result);
    
    result = testServiceFacade.post("name").get();
    System.out.println(result);
    
    ListenableFuture<JsonResult<TestBean>> f = testServiceFacade.getForListening("name");
    f.addListener(new Runnable() {xxx}, executor);

}

```
http请求返回的内容在默认情况下会使用FastJson做反序列化，支持自定义反序列化方式（见下文）

异步调用还可以用ResponseCallback接口，例如：
```java
public interface TestServiceFacade {

  /**
   * 异步调用，通过回调的方式
   */
  @Request("http://localhost:8080/home/index.json?name=#{name}&test=1")
  void getString(@HttpParam("name") String name, ResponseCallback<JsonResult<TestBean>> callback);
}

```

## 设置post请求的HttpEntity
@POST注解中的entity属性可指定使用哪种entity，目前支持，FORM, JSON_STRING和SERIALIZER
三种，如果需要自定义，可以使用ReqeuestPostProcessor（下文会有介绍）
## 自定义反序列化HttpResponse
### 设置默认的反序列化方式
可使用Configuration.setDefaultResponseHandler()方法设置默认反序列化，例如：
```java
configuration.setDefaultResponseHandler(new FastJsonResponseHandler());
```
或者：
```java
configuration.setDefaultResponseHandler(new ToStringResponseHandler());
```
### 为请求定制ResponseHandler
可使用@Response注解为请求定制ResponseHandler，@Response注解提供两种使用方式：
* 将@Response标记在接口上，表示此接口默认使用的ResponseHandler，例如：
```java
@Response(FastJsonResponseHandler.class)
public interface TestServiceFacade {
}
```
* 将@Response注解标记在方法上，表示此方法使用的ResponseHandler，例如：
```java
public interface TestServiceFacade {
  
  @Request("http://localhost:8080/home/index.json?name=#{name}&test=1")
  @Response(ToStringResponseHandler.class)
  String getString(@HttpParam("name") String name);
}

```
两种方式可同时使用:
```java
@Response(FastJsonResponseHandler.class)
public interface TestServiceFacade {

  @Request("http://localhost:8080/home/index.json?name=#{name}&test=1")
  @PostProcessors({KeepHeaderPostProcessor.class})
  Future<JsonResult<TestBean>> get(@HttpParam("name") String name);

  @Request("http://localhost:8080/home/index.json?name=#{name}&test=1")
  @Response(ToStringResponseHandler.class)
  String getString(@HttpParam("name") String name);

  @Request("http://localhost:8080/home/index.json?name=#{name}")
  @POST
  JsonResult<TestBean> post(@HttpParam("name") String name);

}

```
如果即没有在接口上标记@Response注解，也没有在方法上标记@Response注解，则使用Configuration.getDefaultResponseHandler()
## 定制HttpClient
可以调用Configuration.setHttpClientFactory()方法提供一个HttpClientFactory
来替代默认的DefaultHttpClientFactory，用于定制HttpClient对象

## 使用RequestPostProcessor
RequestPostProcessor可对请求进行拦截，接口定义如下：
```java
public interface RequestPostProcessor {
  boolean postProcessRequest(HttpUriRequest request, MappedRequest mr, Map<String, Object> params);

  void postProcessResponse(HttpResponse response, MappedRequest mr);
}

```
在RequestPostProcessor中可对HttpRequest和HttpResponse做任何的定制.
例如：
```java
public class KeepHeaderPostProcessor implements RequestPostProcessor {

  @Override
  public boolean postProcessRequest(HttpUriRequest request, MappedRequest mr, Map<String, Object> params) {
    request.addHeader("testHeader", "testHeader");
    return true;
  }

  @Override
  public void postProcessResponse(HttpResponse response, MappedRequest mr) {
  }
}

```
有了RequestPostProcessor后，可以通过@PostProcessors在mapper接口上使用，三种级别的使用，

1.全局的RequestPostProcessor

使用Configuration设置全局的RequestPostProcessor，全局的RequestPostProcessor对所有请求有效

2.接口级别的RequestPostProcessor

将@RequestPostProcessors标记在接口上
```java
@PostProcessors({KeepHeaderPostProcessor.class})
public interface TestServiceFacade {

  /**
   * 异步调用
   */
  @Request("http://localhost:8080/home/index.json?name=#{name}&test=1")
  Future<JsonResult<TestBean>> get(@HttpParam("name") String name);

  /**
   * 同步调用
   */
  @Request("http://localhost:8080/home/index.json?name=#{name}")
  @POST
  JsonResult<TestBean> post(@HttpParam("name") String name);
}
```

2.方法级别的RequestPostProcessor

将@RequestPostProcessors标记在方法上
```java
public interface TestServiceFacade {

  @Request("http://localhost:8080/home/index.json?name=#{name}&test=1")
  @PostProcessors({KeepHeaderPostProcessor.class})
  Future<JsonResult<TestBean>> get(@HttpParam("name") String name);

  @Request("http://localhost:8080/home/index.json?name=#{name}")
  @POST
  Future<JsonResult<TestBean>> post(@HttpParam("name") String name);
}

```
一个mapper接口的方法在发送请求时使用的RequestPostProcessor为：
全局 + 接口级别 + 方法级别

## 与Spring集成
在Spring中添加配置:
```xml
  <bean class="cn.yxffcode.httpmapper.spring.HttpMapperAutoConfigurer">
    <property name="basePackages">
      <array>
        <value>cn.yxffcode.httpmapper.spring</value>
      </array>
    </property>
    <property name="annotation" value="org.springframework.stereotype.Component"/>
  </bean>
```
可配置DefaultResponseHandler, 全局的RequestPostProcessors以及HttpClientFactory
```xml
  <bean class="cn.yxffcode.httpmapper.spring.HttpMapperAutoConfigurer">
    <property name="basePackages">
      <array>
        <value>cn.yxffcode.httpmapper.spring</value>
      </array>
    </property>
    <property name="annotation" value="org.springframework.stereotype.Component"/>

    <!--以下是可选-->
    <property name="commonRequestPostProcessors">
      <list>
        <ref bean="keepHeaderPostProcessor"/>
      </list>
    </property>
    <property name="defaultResponseHandler" ref="fastJsonResponseHandler"/>
    <property name="httpClientFactory" ref="defaultHttpClientFactory"/>
  </bean>
```