
## 前言
对于云端的开发，如果是服务端开发者，其实没必要太深入了解设备端的开发，但是了解设备端的开发能让我们更深入知道IOT的流程。（IOT新增了[虚拟设备](https://help.aliyun.com/document_detail/88471.html?spm=a2c4g.11186623.2.16.fffb7411TJKJA7)的概念，方便云端应用开发者快速开发）


## 1 设备端开发
IOT提供设备端SDK开发，详情可见文末，IOT提供了各种环境下的SDK，每种SDK对应的功能也并不相同。
![设备端SDK](http://image.talkmoney.cn/2019-1-10/2019-1-10_阿里云-IOT平台开发-设备端开发（3）/1547088202708.png)


### 1.1 设备端连接云端
我们这边采用Java SDK，使用基于Tcp的MQTT连接云端，使用MQTT客户端域名直连的设备认证模式。

``` javas
    public static void main(String... strings) throws Exception {
        //客户端设备自己的一个标记，建议是MAC或SN，不能为空，32字符内
        String clientId = InetAddress.getLocalHost().getHostAddress();

        //设备认证
        Map<String, String> params = new HashMap<String, String>();
        params.put("productKey", productKey); //这个是对应用户在控制台注册的 设备productkey
        params.put("deviceName", deviceName); //这个是对应用户在控制台注册的 设备name
        params.put("clientId", clientId);
        String t = System.currentTimeMillis() + "";
        params.put("timestamp", t);

        //MQTT服务器地址，TLS连接使用ssl开头
        String targetServer = "ssl://" + productKey + ".iot-as-mqtt.cn-shanghai.aliyuncs.com:1883";

        //客户端ID格式，两个||之间的内容为设备端自定义的标记，字符范围[0-9][a-z][A-Z]
           String mqttclientId = clientId + "|securemode=2,signmethod=hmacsha1,timestamp=" + t + "|";
        String mqttUsername = deviceName + "&" + productKey; //mqtt用户名格式
        String mqttPassword = SignUtil.sign(params, secret, "hmacsha1"); //签名

        System.err.println("mqttclientId=" + mqttclientId);

        connectMqtt(targetServer, mqttclientId, mqttUsername, mqttPassword, deviceName);
    }
```
我们注意一下，上面的targetServer对象。我们可以知道，MQTT客户端域名直连，使用了TLS加密连接。


具体连接过程在connectMqtt方法中，方法比较长，可直接看文末的github地址进行下载。

``` javas
 public static void connectMqtt(String url, String clientId, String mqttUsername,
                                   String mqttPassword, final String deviceName) throws Exception {
        MemoryPersistence persistence = new MemoryPersistence();
        SSLSocketFactory socketFactory = createSSLSocket();
        final MqttClient sampleClient = new MqttClient(url, clientId, persistence);
        MqttConnectOptions connOpts = new MqttConnectOptions();
        connOpts.setMqttVersion(4); // MQTT 3.1.1
        connOpts.setSocketFactory(socketFactory);

        //设置是否自动重连
        connOpts.setAutomaticReconnect(true);

        //如果是true，那么清理所有离线消息，即QoS1或者2的所有未接收内容
        connOpts.setCleanSession(false);


        connOpts.setUserName(mqttUsername);
        connOpts.setPassword(mqttPassword.toCharArray());
        connOpts.setKeepAliveInterval(65);

        LogUtil.print(clientId + "进行连接, 目的地: " + url);
        sampleClient.connect(connOpts);
		。。。。。。。
		。。。。。。。
}
```


### 1.2 设备端发送消息
我们可以知道当前设置的MQTT协议的版本是3.1.1，那么设备端如何发送消息到云端呢？上文有讲到IOT有基于MQTT协议的，那么发送消息也是基于MQTT的Pub/Sub模式。
下面介绍如何使用PUb/Sub发送一条消息

``` javas
String content = "消息"+i++;
                System.out.println("消息:"+content);
                MqttMessage message = null;
                try {
                    message = new MqttMessage(content.getBytes("utf-8"));
                } catch (UnsupportedEncodingException e) {
                    e.printStackTrace();
                }
                //有0和1
                message.setQos(1);
                //System.out.println(System.currentTimeMillis() + "消息发布:---");
                try {
                    sampleClient.publish(pubTopic, message);
                } catch (MqttException e) {
                    e.printStackTrace();
                }
```
我们可以知道设置的消息为Qos1（至少发送1次），Qos有0,1,2。但是IOT只支持0和1。
![enter description here](http://image.talkmoney.cn/2019-1-10/2019-1-10_阿里云-IOT平台开发-设备端开发（3）/1547089957261.png)
![enter description here](http://image.talkmoney.cn/2019-1-10/2019-1-10_阿里云-IOT平台开发-设备端开发（3）/1547089968387.png)

那么，设备发布的消息能不能被自己接收呢？我们可以设置sampleClient.subscribe方法。但是需要配置对应的权限，我们可以在物联网平台上设置Topic的权限。
![设备具有权限](http://image.talkmoney.cn/2019-1-10/2019-1-10_阿里云-IOT平台开发-设备端开发（3）/1547090423353.png)

**注意：Pub方法会阻塞线程，是同步方法**


### 1.3 设备接收消息
刚才说到了设备接收消息可以使用subscribe接收消息，**需要注意的是Topic并不是能随便订阅，Topic必须是当前设备拥有权限的**，不然订阅Topic失败。

![订阅RRPC消息](http://image.talkmoney.cn/2019-1-10/2019-1-10_阿里云-IOT平台开发-设备端开发（3）/1547111510849.png)
我们知道，上面代码订阅了/sys/ productKey/deviceName /rrpc/request/，我们可以知道这是RRPC通信方式下的订阅Topic方式（参考上篇文章）。
RRPC下的设备的响应消息类型只支持Qos0，因此我们只能设置消息Qos0，同样发送消息是同步方法，也必须交由线程处理。

## 总结
设备端与云端之间的通信使用的协议可以采用MQTT协议（不仅仅只有这种协议，还可以自定义协议），连接到云端的方式也有多种方式（建议使用MQTT-TCP连接）。

对于Topic的发布和订阅，都需要相应的权限，创建设备时会为我们默认创建一些固定的Topic。


##### 参考文献
[虚拟设备](https://help.aliyun.com/document_detail/88471.html?spm=a2c4g.11186623.2.16.fffb7411TJKJA7)
[MQTT官网](http://mqtt.org/?spm=a2c4g.11186623.2.12.5ac778dcoCzsJI)
[设备端开发](https://help.aliyun.com/document_detail/42648.html?spm=a2c4g.11186623.6.633.5ac778dcoCzsJI)


##### 项目代码
[MQTT设备端](https://github.com/229319258/mqttclient)