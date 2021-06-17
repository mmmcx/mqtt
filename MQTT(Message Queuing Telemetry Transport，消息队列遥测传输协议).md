## MQTT(Message Queuing Telemetry Transport，消息队列遥测传输协议)

### 1.简述

MQTT 是一种基于发布/订阅模式的轻量级通讯协议，基于TCP/IP协议，由IBM于1999年发布，最大优点在于，可以以极少的代码和有限的带宽，为连接远程设备提供实时可靠的消息服务。

作为一种低开销、低带宽占用的即时通讯协议，使其在物联网、小型设备、移动应用等方面有较广泛的应用。

MQTT是一个基于客户端-服务器的消息发布/订阅传输协议。MQTT协议是轻量、简单、开放和易于实现的，这些特点使它适用范围非常广泛。在很多情况下，包括受限的环境中，如：机器与机器（M2M）通信和物联网（IoT）。其在，通过卫星链路通信传感器、偶尔拨号的医疗设备、智能家居、及一些小型化设备中已广泛使用

### 2.设计原则

由于物联网的环境是非常特别的，所以MQTT遵循以下设计原则：

- （1）精简，不添加可有可无的功能；
- （2）发布/订阅（Pub/Sub）模式，方便消息在传感器之间传递；
- （3）允许用户动态创建主题，零运维成本；
- （4）把传输量降到最低以提高传输效率；
- （5）把低带宽、高延迟、不稳定的网络等因素考虑在内；
- （6）支持连续的会话控制；
- （7）理解客户端计算能力可能很低；
- （8）提供服务质量管理；
- （9）假设数据不可知，不强求传输数据的类型与格式，保持灵活性。



### 3.MQTT协议实现方式

实现MQTT协议需要客户端和服务器端通讯完成，在通讯过程中，MQTT协议中有三种身份：

**发布者(publish)、代理(Broker)（服务器）、订阅者(Subscribe)**。

其中，消息的发布者和订阅者都是客户端，消息代理是服务器，消息发布者可以同时是订阅者。

MQTT传输的消息分为：**主题（Topic）**和 **负载（payload）**两部分：

- （1）Topic，可以理解为消息的类型，订阅者订阅（Subscribe）后，就会收到该主题的消息内容（payload）；
- （2）payload，可以理解为消息的内容，是指订阅者具体要使用的内容。



### 4.基于docker-compose 搭建mosquitto 服务

#### 4.1.创建目录

```shell
mkdir -p /usr/local/work/mqtt/config
mkdir -p /usr/local/work/mqtt/data
mkdir -p /usr/local/work/mqtt/log
```

#### 4.2.初始化配置文件

```shell
vim /usr/local/work/mqtt/config/mosquitto.conf
```

**mosquitto.conf文件如下：**

```shell

persistence true
persistence_location /mosquitto/data
log_dest file /mosquitto/log/mosquitto.log

listener 1883
listener 9001
protocol websockets

#关闭匿名模式
allow_anonymous false

#指定密码文件
password_file /mosquitto/config/pwfile.conf

```

#### 4.3. 为目录添加权限

```powershell
chmod -R 777 /usr/local/work/mqtt
```

#### 4.4. 在/usr/local/work/mqtt下创建docker-compose 文件

**docker-compose.yml内容如下：**

```shell
version: '3'
services:
  mqtt:
    image: eclipse-mosquitto:latest  #镜像
    container_name: mos_mqtt  #定义容器名称
    restart: always  #开机启动，失败也会一直重启
    #environment:
     # - "cluster.name=elasticsearch" #设置集群名称为elasticsearch
      #- "discovery.type=single-node" #以单一节点模式启动
     # - "ES_JAVA_OPTS=-Xms1024m -Xmx1024m" #设置使用jvm内存大小
    volumes:
      - /usr/local/work/mqtt/config/mosquitto.conf:/mosquitto/config/mosquitto.conf #配置文件挂载
      - /usr/local/work/mqtt/data:/mosquitto/data #数据文件挂载
      - /usr/local/work/mqtt/log:/mosquitto/log/ #日志文件挂载  
    ports:
      - 1883:1883
      - 9001:9001

```

在该目录下执行docker-compose up -d ，启动容器。

#### 4.5. 以交互式方式进入容器，配置权限

```shell
docker exec -it 7a07aaf9f790 sh
```

**4.5.1 进入容器内的/mosquitto/config 目录，生成密码**

```shell
touch /mosquitto/config/pwfile.conf
chmod -R 777 /mosquitto/config/pwfile.conf
```

**4.5.2 使用mosquitto_passwd 命令创建用户，第一个是用户名admin，第二个是密码(123456)**

```shell
mosquitto_passwd -b /mosquitto/config/pwfile.conf admin 123456
```

#### 4.6.重启mqtt服务

```shell
docker restart 7a07aaf9f790
```

### 5.使用MQTT.fx进行连接测试

MQTT.fx下载和使用参考：

[参考]: https://blog.csdn.net/tiantang_1986/article/details/85101366

[参考](https://blog.csdn.net/tiantang_1986/article/details/85101366)

### 6.MQTT在Java中的使用及常用api

#### MqttClient 构造方法

```java
//创建客户端
MqttClient mqttClient = new MqttClient(broker,clientId,new MemoryPersistence());
```

- broker：代理的地址，一般是启动的代理机的Ip+端口（1883）
- clientId：MqttClient的唯一标识，不同的MqttClient需要不同的MqttClient
- new MemoryPersistence()：固定参数，直接new即可。

#### 连接信息的构造方法

```java
//创建连接参数
MqttConnectOptions connectOptions = new MqttConnectOptions();
```

常用设置属性：

```java
// 在重新启动和重新连接时记住状态
connOpts.setCleanSession(false);
// 设置连接的用户名
connOpts.setUserName(userName);  
// 设置连接的密码
connOpts.setPassword(password.toCharArray());
// 超时时间 单位为秒
connOpts.setConnectionTimeout(10);  
// 设置会话心跳时间 单位为秒 服务器会每隔1.5*20秒的时间向客户端发送个消息判断客户端是否在线，但这个方法并没有重连的机制
connOpts.setKeepAliveInterval(20);

```

```java
mqttClient.connect(connectOptions);
//设置回调信息
mqttClient.setCallback(new MqttCallback() { 
    // 客户端掉线走这条
 	public void connectionLost(Throwable cause) {
    		  System.out.println("connectionLost");
    			 cause.printStackTrace();
                }

    // 收到消息推送走这条
  public void messageArrived(String topic, MqttMessage message) throws SQLException, MqttException {
           String content = new String(message.getPayload());
                    System.out.println("topic:"+topic);
                    System.out.println("Qos:"+message.getQos());
                    System.out.println("收到的消息:"+content);
                }

    public void deliveryComplete(IMqttDeliveryToken token) {
                    System.out.println("deliveryComplete---------"+ token.isComplete());
                }
            }

```

通过MqttClient的对象可以直接取消曾经的订阅

```java
mqttClient.unsubscribe(topic);
```

mqtt在取消订阅后还是会处于阻塞状态。

**发布消息**

```java
MqttMessage message = new MqttMessage(next.getBytes());  // 创建消息
message.setQos(qos);   // 设置消息的服务质量
sampleClient.publish(topic, message);  // 发布消息

```

断开连接

```java
mqttClient.disconnect();
//关闭客户端
mqttClient.close();
```

代码如下：

```java
package com.study.mqtt.config;
import java.util.ArrayList;
import java.util.Arrays;
import java.util.List;
import java.util.Objects;

import lombok.extern.slf4j.Slf4j;
import org.apache.coyote.Constants;
import org.eclipse.paho.client.mqttv3.*;


import org.eclipse.paho.client.mqttv3.persist.MemoryPersistence;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.integration.mqtt.core.DefaultMqttPahoClientFactory;
import org.springframework.stereotype.Service;
import org.springframework.util.StringUtils;

/**
 * MQTT 工具类
 */
@Slf4j
@Service
public class MqttUtils {

    @Value("${mqtt.server.producer.url}")
    private String serviceUrl;

    @Value("${mqtt.server.producer.clientId}")
    private String clientId;

    @Value("${mqtt.server.producer.defaultTopic}")
    private String defaultTopic;

    @Value("${mqtt.server.producer.username}")
    private String username;

    @Value("${mqtt.server.producer.password}")
    private String password;

    /**
     * 订阅主题
     * @param theme
     * @return
     */
    public boolean subscribe(String theme){
        if(StringUtils.isEmpty(theme)){
            theme = "test";
        }

        //mqtt 连接设置
        try {
            MqttClient mqttClient = new MqttClient(serviceUrl,clientId,new MemoryPersistence());
            MqttConnectOptions connectOptions = new MqttConnectOptions();
            connectOptions.setUserName(username);
            connectOptions.setPassword(password.toCharArray());
            connectOptions.setKeepAliveInterval(Constants.STAGE_KEEPALIVE);
            connectOptions.setConnectionTimeout(1000);
            List<String> serviceUrlList = new ArrayList<>();
            serviceUrlList.add(serviceUrl);
            connectOptions.setServerURIs(serviceUrlList.toArray(new String[serviceUrlList.size()]));
            connectOptions.setCleanSession(true);
            //如项目中需要知道客户端是否掉线可以调用该方法。设置最终端口的通知消息
//            connectOptions.setWill(new MqttTopic(),new byte[](),0,false);
//            connectOptions.setMaxInflight(0);
//            connectOptions.setSocketFactory(new SocketFactory());
//            connectOptions.setSSLProperties(new Properties());
//            connectOptions.setSSLHostnameVerifier(new HostnameVerifier());
//            connectOptions.setMqttVersion(0);
//            connectOptions.setAutomaticReconnect(false);
//            mqttClient.setTimeToWait(0L);
            mqttClient.connect(connectOptions);
            //回调方法
            mqttClient.setCallback(new PushCallback());
            return true;
        } catch (MqttException e) {
            e.printStackTrace();
            return false;
        }

    }

    /**
     * 推送消息
     * @return
     */
    public boolean pushMsg(String clientId, String theme, String msg){
        if(StringUtils.isEmpty(theme)){
            theme = "test";
        }

        MqttTopic connect = this.getTopic(clientId, theme);
        MqttMessage message = new MqttMessage();
        //保证消息能到达一次
        message.setQos(1);
        message.setRetained(true);
        //String str = "{\"msg\":\"" + msg + "\"}";
        message.setPayload(msg.getBytes());
        try {
            this.publish(connect, message);
            return true;
        } catch (Exception e) {
            log.error("推送消息--异常\nclientId:{}\nexception:{}", clientId, e.toString());
            return false;
        }

    }

    /**
     * 获取订阅
     */
    private MqttTopic getTopic(String clientId, String theme) {
        MqttClient mqttClient = this.mqttClient(clientId);
        MqttConnectOptions options = new MqttConnectOptions();
        options.setCleanSession(false);
        options.setUserName(username);
        options.setPassword(password.toCharArray());
        // 设置超时时间
        options.setConnectionTimeout(10);
        // 设置会话心跳时间
        options.setKeepAliveInterval(20);
        try {
            mqttClient.setCallback(new PushCallback());
            mqttClient.connect(options);
            return mqttClient.getTopic(theme);
        } catch (Exception e) {
            System.out.println("获取订阅--异常\nclientId:{}\ntheme:{}\nexception:{}"+clientId+theme+e.toString());
        }
        return null;
    }
    private void getTopic2(String clientId, String theme,String msg) {
        MqttClient mqttClient = this.mqttClient(clientId);
        MqttConnectOptions options = new MqttConnectOptions();
        options.setCleanSession(false);
        options.setUserName(username);
        options.setPassword(password.toCharArray());
        // 设置超时时间
        options.setConnectionTimeout(10);
        // 设置会话心跳时间
        options.setKeepAliveInterval(20);
        try {
            mqttClient.setCallback(new PushCallback());
            mqttClient.connect(options);
            MqttMessage message = new MqttMessage();
            message.setQos(1);
            message.setPayload(msg.getBytes());
            message.setRetained(true);
        } catch (Exception e) {
            System.out.println("获取订阅--异常\nclientId:{}\ntheme:{}\nexception:{}"+clientId+theme+e.toString());
        }
    }
    /**
     * 连接mqtt服务
     */
    private MqttClient mqttClient(String clientId) {
        MqttClient mqttClient = null;
        try {
            mqttClient = new MqttClient(serviceUrl, clientId, new MemoryPersistence());
        } catch (MqttException e) {
            System.out.println("连接MQTT服务器--异常\nclientId:{}\nexception:{}"+clientId+ e.toString());
        }
        return mqttClient;
    }

    /**
     * 推送消息
     */
    private void publish(MqttTopic mqttTopic, MqttMessage message) throws Exception {
        if (Objects.nonNull(mqttTopic)) {
            MqttDeliveryToken token = mqttTopic.publish(message);
            token.waitForCompletion();
            if (!token.isComplete()) {
                System.out.println("推送消息--失败--msg：{}"+message);
            }
        } else {
            System.out.println("推送消息--失败--msg：{}"+message);
        }
    }


}

```

回调类：

```java
package com.study.mqtt.config;

import com.alibaba.fastjson.JSON;
import lombok.extern.slf4j.Slf4j;
import org.eclipse.paho.client.mqttv3.IMqttDeliveryToken;
import org.eclipse.paho.client.mqttv3.MqttCallback;
import org.eclipse.paho.client.mqttv3.MqttMessage;

/**
 * 发布消息的回调类
 * <p>
 * 必须实现MqttCallback的接口并实现对应的相关接口方法CallBack 类将实现 MqttCallBack。
 * 每个客户机标识都需要一个回调实例。在此示例中，构造函数传递客户机标识以另存为实例数据。
 * 在回调中，将它用来标识已经启动了该回调的哪个实例。
 * 必须在回调类中实现三个方法：
 * public void messageArrived(MqttTopic topic, MqttMessage message)接收已经预订的发布。
 * public void connectionLost(Throwable cause)在断开连接时调用。
 * public void deliveryComplete(MqttDeliveryToken token))
 * 接收到已经发布的 QoS 1 或 QoS 2 消息的传递令牌时调用。
 * 由 MqttClient.connect 激活此回调。
 * </p>
 */
@Slf4j
public class PushCallback implements MqttCallback {
    @Override
    public void connectionLost(Throwable cause) {
        //连接丢失后，一般在这里面进行重连
//        log.info("PushCallback--连接断开" + JSON.toJSONString(cause));
        log.info("PushCallback--连接断开" + JSON.toJSONString(cause));
    }

    @Override
    public void deliveryComplete(IMqttDeliveryToken token) {
        if (!token.isComplete()) {
            log.info("PushCallback--deliveryComplete--异常：" + JSON.toJSONString(token));
        }
    }

    @Override
    public void messageArrived(String theme, MqttMessage message) {
        //TODO 订阅成功返回的数据在这里获取，根据自己的业务进行处理
        try {
            log.info("【订阅回调】\ttheme：{}\tqos：{}\npayload：{}", theme, message.getQos(), new String(message.getPayload()));
        } catch (Exception e) {
            log.error("【订阅回调】--异常\ttheme：{}\nmessage：{}", theme, JSON.toJSONString(message));
        }
    }
}

```

controller:

```java
package com.study.mqtt.controller;

import com.study.mqtt.config.MqttUtils;
import com.study.mqtt.service.TestGateway;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.bind.annotation.RestController;

@RestController
@RequestMapping("/mqtt")
public class MqttFunController {


    @Value("${mqtt.server.producer.clientId}")
    private String clientId;

    @Autowired
    private TestGateway gateway;

    @Autowired
    private MqttUtils mqttUtils;

    @RequestMapping("/testMqtt")
    public String sendMsgMqtt(@RequestParam("topic") String topic ,@RequestParam("message") String message){

        gateway.sendToMqtt(message,topic);
        return "success";
    }

    /**
     * 发送消息
     * @param theme
     * @param msg
     * @return
     */
    @GetMapping("pushMsg")
    public String pushMsg(@RequestParam(value = "theme", required = false) String theme, @RequestParam(value = "msg") String msg){
        boolean isSuccess = mqttUtils.pushMsg(clientId, theme, msg);
        return isSuccess?"成功":"失败";
    }

    /**
     * 订阅
     * @param theme
     * @return
     */
    @GetMapping("subscribe")
    public String subscibe(@RequestParam(value = "theme", required = false) String theme){
        boolean subscribe = mqttUtils.subscribe(theme);
        return subscribe?"订阅成功":"订阅失败";
    }



}

```

application.yml

```yaml
server:
  port: 8080


spring:
  application:
    name: mqtt



mqtt:
  server:
    producer:
      url: tcp://192.168.5.100:1883
      clientId: 04b24288543849b69a0c3aa49426233c
      defaultTopic: mqtt
      username: admin
      password: 123456


```











## 附录：

参考网址：

[基于docker构建mosquitto服务](https://blog.csdn.net/lanpiao_87/article/details/100025722)



