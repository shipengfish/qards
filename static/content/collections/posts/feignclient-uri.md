---
title: FeignClient 配置动态 URI
created_at: 2018-10-18T02:30:09.291Z
tags:
  - Feign
authors: 再见孙悟空
categories: programing
meta:
  description: feign
  keywords: feign
isPage: false
isFeatured: false
hero:
  alt: image
  image: /images/uploads/andrew-ridley-76547-unsplash.jpg
excerpt: 使用 FeignClient 配置动态路由
---
{"widget":"qards-code","config":"eyJjb2RlIjoiZXhwb3J0IGRlZmF1bHQgY2xhc3MgUG9zdFRhZ3MgZXh0ZW5kcyBDb21wb25lbnQ8UHJvcHMsIFN0YXRlPiB7XG4gICAgcmVuZGVyKCkge1xuICAgICAgICBjb25zdCB7cG9zdCwgLi4ucHJvcHN9ID0gdGhpcy5wcm9wcztcbiAgICAgICAgY29uc3Qge3RhZ3N9ID0gcG9zdDtcblxuICAgICAgICByZXR1cm4gKFxuICAgICAgICAgICAgPFdyYXBwZXIgZmxleFdyYXA9e1wid3JhcFwifSBhbGlnbkl0ZW1zPXtcInNwYWNlLWJldHdlZW5cIn0gey4uLnByb3BzfT5cbiAgICAgICAgICAgICAgICB7dGFncy5tYXAoKHRhZykgPT4ge1xuXG4gICAgICAgICAgICAgICAgICAgIHJldHVybiA8TGluayB0bz17YC90YWdzLyR7dGFnLnNsdWd9L2B9IGtleT17dGFnLmlkfT5cbiAgICAgICAgICAgICAgICAgICAgICAgIHt0YWcudGl0bGV9XG4gICAgICAgICAgICAgICAgICAgIDwvTGluaz5cbiAgICAgICAgICAgICAgICB9KX1cbiAgICAgICAgICAgIDwvV3JhcHBlcj5cbiAgICAgICAgKTtcbiAgICB9XG59IiwibGFuZ3VhZ2UiOiJqYXZhc2NyaXB0In0="}


#### 一般的调用方式

使用 [SpringBoot](https://spring.io/projects/spring-boot) 开发项目时，如果我们需要调用三方的 `API` 时，可以使用 `SpringBoot` 内置集成的 `RestTemplate`，例如你可以这样发起一个 `POST` 请求:

```java
public void call(RestTemplateBuilder restTemplateBuilder) {
    ResponseEntity<Object> responseEntity = restTemplateBuilder.build().postForEntity("service-url", "request", Object.class);
}
```

当然我们还可以使用 [OpenFeign](https://github.com/OpenFeign/feign) 来调用，例如根据城市 ID 获取城市:

```java
@FeignClient("user-service")
public interface InternalCityClient {
    @RequestMapping(value = "/api/city/{id}", method = RequestMethod.GET)
    CityDTO getCity(@PathVariable("id") Integer cityId);
}
``` 

其中 `user-service` 是我们注册中心的一个服务，配置了 `name = user-service` 之后，`feign` 就会在服务注册中心找到名称为 `user-service` 的服务，并且匹配到 `/api/city/{id}` 的路由。然后将接口响应结果反序列化成 `CityDTO`。

当然，假如我们没有使用注册中心的时候，还可以直接设置 `URL` 进行调用，例如:

```java
public class Example {
  public static void main(String[] args) {
      GitHub github = Feign.builder()
                     .encoder(new JacksonEncoder())
                     .decoder(new JacksonDecoder())
                     .target(GitHub.class, "https://api.github.com");
  }
}
```

有时候，我们需要动态配置请求的地址，比如说生产环境和测试环境地址不一样，并且需要同时访问 `prod` 和 `test` 两个地址，这个时候如果用 `FeignClient` 需要怎么做呢？


#### 配置动态 url

1. 为了使用动态 url，我们需要使用 `@RequestLine` 注解，重点是参数 **`URI`**， 例如:

	```java
	@FeignClient(
	        name = "user-service",
	        url = "#{'${user.service.testUrl}'}",
	        configuration = FeignConfiguration.class,
	        fallback = UserClientFallback.class)
	public interface UserClient {
	
	    /**
	     * 向 user-service 发起请求.
	     */
	    @RequestLine("POST")
	    String post(URI uri, @QueryMap Map<String, Object> queryMap);
	}
	```

	`name` 是必须要指定的，不然 `Feign` 会报错；

	`url` 值会从 `.yaml` 配置文件中读取，`yaml` 文件格式如下:

	```
	user:
		service:
			testUrl: test-api.xxx.com
	```

2. 我们还需要为该接口配置特定的 `FeignConfiguration`，一定要设置 **`feignContract`**，如果不这样做，`FeignClient` 将可能不会生效(大概率启动时候就报错)

	`FeignConfiguration.java` 代码如下:
	
	```java
	public class FeignConfiguration {
	
	    @Bean
	    public Contract feignContract() {
	        return new Contract.Default();
	    }
	
	    /**
	     * 用于将返回的结果 decode.
	     */
	    @Bean(name = "snake_decode")
	    public Decoder feignDecoder() {
	        HttpMessageConverter jacksonConverter = new MappingJackson2HttpMessageConverter();
	        ObjectFactory<HttpMessageConverters> objectFactory = () -> new HttpMessageConverters(jacksonConverter);
	        return new ResponseEntityDecoder(new SpringDecoder(objectFactory));
	    }
	
	    /**
	     * 将请求的驼峰字段转换为下划线.
	     */
	    @Bean(name = "snake_encode")
	    public Encoder feignEncoder(ObjectMapper objectMapper) {
	        ObjectMapper snakeObjectMapper = objectMapper.copy().setPropertyNamingStrategy(SNAKE_CASE).setSerializationInclusion(JsonInclude.Include.NON_NULL);
	        HttpMessageConverter jacksonConverter = new MappingJackson2HttpMessageConverter(snakeObjectMapper);
	        ObjectFactory<HttpMessageConverters> objectFactory = () -> new HttpMessageConverters(jacksonConverter);
	        return new SpringEncoder(objectFactory);
	    }
	
	    @Bean
	    Request.Options feignOptions() {
	        return new Request.Options(10000, 15000);
	    }
	
	}
	```
	
3. 客户端调用时候，传入 `new URI("/api/city/"+id)`，就会找到 `test-api.xxx.com/api/city/{id}` ，还可以传入 `new URI("https://produ-api.xxx.com/api/city/"+id)`，就会访问 `https://produ-api.xxx.com/api/city/{id}`，这样就达到了动态配置 `URI` 的效果。


