# Springboot-eureka-example
多模块的基于springboot的eureka项目示例

## Eureka是什么?
1. Eureka是Netflix开发的服务发现框架，本身是一个基于REST的服务，主要用于定位运行在AWS域中的中间层服务，以达到负载均衡和中间层服务故障转移的目的。SpringCloud将它集成在其子项目spring-cloud-netflix中，以实现SpringCloud的服务发现功能。</br>
2. Eureka包含两个组件：Eureka Server和Eureka Client。</br>
3. Eureka Server提供服务注册服务，各个节点启动后，会在Eureka Server中进行注册，这样Eureka Server中的服务注册表中将会存储所有可用服务节点的信息，服务节点的信息可以在界面中直观的看到。</br>
4. Eureka Client是一个java客户端，用于简化与Eureka Server的交互，客户端同时也就是一个内置的、使用轮询(round-robin)负载算法的负载均衡器。</br>
在应用启动后，将会向Eureka Server发送心跳,默认周期为30秒，如果Eureka Server在多个心跳周期内没有接收到某个节点的心跳，Eureka Server将会从服务注册表中把这个服务节点移除(默认90秒)。</br>
6. Eureka Server之间通过复制的方式完成数据的同步，Eureka还提供了客户端缓存机制，即使所有的Eureka Server都挂掉，客户端依然可以利用缓存中的信息消费其他服务的API。</br>
综上，Eureka通过心跳检查、客户端缓存等机制，确保了系统的高可用性、灵活性和可伸缩性。</br>

## 关键依赖
```
<dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
</dependency>
```

## 本项目的模块说明
```
cloud.miles4j.eureka.server 注册服务
cloud.miles4j.eureka.provider 服务提供者
cloud.miles4j.eureka.consumer 服务消费者
```

## Eureka 配置
注意: defaultZone为Eureak服务的地址,不要填写错误,Provider和Consumer应该都是填写Eureak服务的地址,而不是它本身或者Provider的地址.

### Server:

```
server:
  port: 8000
spring:
  application:
    name: cloud.miles4j.eureka.server
eureka:
  client:
    registerWithEureka: false
    fetchRegistry: false
    serviceUrl:
      defaultZone: http://127.0.0.1:${server.port}/eureka/
  server:
    enable‐self‐preservation: false
    eviction‐interval‐timer‐in‐ms: 60000
```

### Provider
```
server:
  port: 8080
spring:
  application:
    name: cloud.miles4j.eureka.provider
eureka:
  client:
    serviceUrl:
      defaultZone: http://127.0.0.1:8000/eureka/
```

### Consumer
```
server:
  port: 8090
spring:
  application:
    name: cloud.miles4j.eureka.consumer
eureka:
  client:
    serviceUrl:
      defaultZone: http://127.0.0.1:8000/eureka/
```

### 流程
1. 先启动 cloud.miles4j.eureka.server
2. 再启动 cloud.miles4j.eureka.provider
3. 最后启动 cloud.miles4j.eureka.consumer

### 关键
@EnableEurekaServer 注解开启Eureak的服务

```
package cloud.miles4j.eureka.server;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.netflix.eureka.server.EnableEurekaServer;

/**
 * @author richard
 */
@EnableEurekaServer
@SpringBootApplication
public class ServerApplication {
    public static void main(String[] args) {
        SpringApplication.run(ServerApplication.class, args);
    }
}
```

@EnableEurekaClient 注解开启客户端,并注册服务

```
/**
 * Eureka提供方(其实也是客户端)
 */
@RestController
@EnableEurekaClient
@SpringBootApplication
public class ProviderApplication {

    public static void main(String[] args) {
        SpringApplication.run(ProviderApplication.class, args);
    }


    /**
     * 假如这个客户端要提供一个getUser的方法
     * @return
     */
    @GetMapping(value = "/getUser")
    @ResponseBody
    public Map<String,Object> getUser(@RequestParam Integer id){
        Map<String,Object> data = new HashMap<>();
        data.put("id",id);
        data.put("userName","admin");
        data.put("from","provider-A");
        return data;
    }

}
```

@EnableEurekaClient 注解开启客户端,并消费服务(其实无论是提供方还是消费方,在Eureka来说都是客户端),这里需要注意的是访问服务不是输入提供方的地址,也不是输入Eureka服务的地址,而是使用之前在Provider的application.yml中配置的注册服务名,切记不要搞错.
1. 之前在Provider的application.yml中配置的注册服务名
```
spring:
  application:
    name: cloud.miles4j.eureka.provider
```

2. 调用的时候要指定的是注册服务的名字,而不是其他地址
```
@GetMapping(value = "/getUser")
@ResponseBody
public Map<String, Object> getUser(@RequestParam Integer id) {
    Map<String, Object> data = new HashMap<>();

    data = restTemplate.getForObject("http://cloud.miles4j.eureka.provider/getUser?id=" + id, Map.class);

    return data;

}
```

3. 完整的消费示例代码,其实就是普通的rest api调用,只是地址是输入Provider注册时指定的服务名称:
```
@RestController
@EnableEurekaClient
@SpringBootApplication
public class ConsumerApplication {
    @Autowired
    RestTemplate restTemplate;

    public static void main(String[] args) {
        SpringApplication.run(ConsumerApplication.class, args);
    }


    /**
     * 实例化RestTemplate
     *
     * @return
     */
    @LoadBalanced
    @Bean
    public RestTemplate rest() {
        return new RestTemplate();
    }

    @GetMapping(value = "/getUser")
    @ResponseBody
    public Map<String, Object> getUser(@RequestParam Integer id) {
        Map<String, Object> data = new HashMap<>();

        data = restTemplate.getForObject("http://cloud.miles4j.eureka.provider/getUser?id=" + id, Map.class);

        return data;

    }
}
```
