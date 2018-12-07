# Firebase使用教程

## 1.添加依赖的jar包

```java
<dependency>
  <groupId>com.google.firebase</groupId>
  <artifactId>firebase-admin</artifactId>
  <version>6.5.0</version>
</dependency>
```



## 2.初始化SDK

### 2.1 官网展示初始化demo

```java
//serviceAccountKey.json 是firebase相关账户信息
FileInputStream serviceAccount = new FileInputStream("path/to/serviceAccountKey.json");

FirebaseOptions options = new FirebaseOptions.Builder()
    .setCredentials(GoogleCredentials.fromStream(serviceAccount))
    //<DATABASE_NAME> 是firebase数据库信息
    .setDatabaseUrl("https://<DATABASE_NAME>.firebaseio.com/")
    .build();

FirebaseApp.initializeApp(options);
```

### 2.2 官方SDK初始化示例

```java
public static synchronized FirebaseApp ensureDefaultApp() {
    if (masterApp == null) {
      FirebaseOptions options =
          FirebaseOptions.builder()
              .setDatabaseUrl(getDatabaseUrl())
              .setStorageBucket(getStorageBucket())
              .setCredentials(TestUtils.getCertCredential(getServiceAccountCertificate()))
              .setFirestoreOptions(FirestoreOptions.newBuilder()
                  .setTimestampsInSnapshotsEnabled(true)
                  .build())
              .build();
      masterApp = FirebaseApp.initializeApp(options);
    }
    return masterApp;
  }
```

### 2.3 message_api项目中使用的初始化(需要优化)

```java
public void initFirebase() throws FileNotFoundException, IOException {
		FirebaseApp firebaseApp = null;
		List<FirebaseApp> firebaseApps = FirebaseApp.getApps();
		if (firebaseApps != null && !firebaseApps.isEmpty()) {
			for (FirebaseApp app : firebaseApps) {
				if (app.getName().equals(FirebaseApp.DEFAULT_APP_NAME))
					firebaseApp = app;
			}
		} else {
			String path = System.getProperty("spring.profiles.active") == null ? ""
					: System.getProperty("spring.profiles.active");
			String pathStr = FirebaseServiceImpl.class.getClassLoader().getResource(path + "serviceAccountKey.json")
					.getPath();
			pathStr=pathStr.replace("%23", "#").replace("%23", "#");
			FileInputStream serviceAccount = new FileInputStream(pathStr);
			FirebaseOptions options = new FirebaseOptions.Builder()
					.setCredentials(GoogleCredentials.fromStream(serviceAccount))
                //<DATABASE_NAME> 是firebase数据库信息
					.setDatabaseUrl("https://<DATABASE_NAME>.firebaseio.com").build();
			firebaseApp = FirebaseApp.initializeApp(options);
			log.info("Firebase initialise:" + firebaseApp.getName());
		}
	}
```



### 2.4 [Google 应用默认凭据](https://developers.google.com/identity/protocols/application-default-credentials?refresh=1)（2.1，2.2，2.3在使用无效后看这个）

```java
//Admin SDK 可以使用其他凭据类型进行身份验证。
//例如，如果您在 Google Cloud Platform 中运行自己的代码，则可以利用 Google 应用默认凭据让 Admin SDK 自行代表您获取服务帐号：
FirebaseOptions options = new FirebaseOptions.Builder()
    .setCredentials(GoogleCredentials.getApplicationDefault())
    .setDatabaseUrl("https://<DATABASE_NAME>.firebaseio.com/")
    .build();

FirebaseApp.initializeApp(options);
```

## 3.发送消息(发送消息前请确保初始化完成)

### 3.1 单个设备推送消息

#### 3.1.1 向个别设备发送消息(官方demo)

```java
//每个 Firebase 客户端 SDK 都能够生成适用于以下平台的注册令牌：iOS、Android、Web、C++ 和 Unity。
//您可以在消息有效负载中添加注册令牌并将其传递给 Admin SDK 的 send() 方法：
// This registration token comes from the client FCM SDKs.
//注册令牌来自于用户发送到服务端数据库中保存的令牌数据
String registrationToken = "YOUR_REGISTRATION_TOKEN";

// See documentation on defining a message payload.
Message message = Message.builder()
    .putData("score", "850")
    .putData("time", "2:45")
    .setToken(registrationToken)
    .build();

// Send a message to the device corresponding to the provided
// registration token.
String response = FirebaseMessaging.getInstance().send(message);
// Response is a message ID string.
System.out.println("Successfully sent message: " + response);
```

#### 3.1.2 message_api项目向单个设备发送消息示例

```java
/**
 *deviceNo：设备的令牌，title：消息标题,content：消息文本
 */
public String pushMsg(String deviceNo, String title, String content)
			throws InterruptedException, ExecutionException {
		// This registration token comes from the client FCM SDKs.
		// String registrationToken =
		// "f9VVNR6oXUs:APA91bERf3fnZYC7ssiqrGbzIZq6kUpiXhbG9W9sfp0GS6fucewCryz_RcSBylz95MkpPde9H0TJULMMiNSXoz68x-WLxEg4kZUhdLu5m4TUmGtbMXpQGnO0-PkBESw2fzHwgt29hFLY";
		// See documentation on defining a message payload.
		Message message = Message.builder().setNotification(new Notification(title, content)).setToken(deviceNo)
				.build();
		// Send a message to the device corresponding to the provided registration
		// token.
		String response = FirebaseMessaging.getInstance().sendAsync(message).get();
		// Response is a message ID string.
		log.info("deviceNo {}, Firebase response: {}", deviceNo, response);
		return response;
	}
```

### 3.2 通过主题(Topic)多台设备推送消息

- 主题消息传递不限制每个应用拥有的主题和订阅数。
- 主题消息传递最适合传递新闻、天气或其他可通过公开途径获得的信息。
- 主题消息的优化重心是吞吐量而非延迟。要将消息快速安全地传送到单台设备或小规模设备组，应[将消息定位至注册令牌](https://firebase.google.com/docs/cloud-messaging/admin/send-messages?refresh=1#send_to_individual_devices)，而非主题。
- 如果您需要向一位用户的多台设备发送消息，可考虑在这些使用场景中使用[设备组消息传递](https://firebase.google.com/docs/cloud-messaging/admin/legacy-fcm?refresh=1#send_to_a_device_group)。

#### 3.2.1 订阅主题

将这些注册令牌的列表传递给 [`subscribeToTopic()`](https://firebase.google.com/docs/reference/admin/node/admin.messaging.Messaging?refresh=1#subscribeToTopic) 方法，以便为相应的设备订阅主题：

##### 官网demo

```java
// These registration tokens come from the client FCM SDKs.
List<String> registrationTokens = Arrays.asList(
    "YOUR_REGISTRATION_TOKEN_1",
    // ...
    "YOUR_REGISTRATION_TOKEN_n"
);

// Subscribe the devices corresponding to the registration tokens to the
// topic.
TopicManagementResponse response = FirebaseMessaging.getInstance().subscribeToTopicAsync(
    registrationTokens, topic).get();
// See the TopicManagementResponse reference documentation
// for the contents of response.
System.out.println(response.getSuccessCount() + " tokens were subscribed successfully");
```

在单个请求中，您最多可以为 1000 台设备订阅主题。如果您提供的数组所含的注册令牌超过 1000 个，则系统将无法处理此请求，并且会显示 `messaging/invalid-argument` 错误。

`subscribeToTopic()` 方法会返回一个包含 FCM 响应的[对象](https://firebase.google.com/docs/reference/admin/node/admin.messaging.MessagingTopicManagementResponse?refresh=1)。无论请求中指定的注册令牌的数量为何，返回类型的格式都是相同的。

如果出现错误（身份验证失败、令牌或主题无效等）， `subscribeToTopic()` 会返回错误。如需错误代码的完整列表（包括说明和解决步骤），请参阅 [Admin FCM API 错误](https://firebase.google.com/docs/cloud-messaging/admin/errors?refresh=1)。

##### 官方SDK 测试demo

```java
public void testSubscribe() throws Exception {
    FirebaseMessaging messaging = FirebaseMessaging.getInstance();
    TopicManagementResponse results = messaging.subscribeToTopicAsync(
        ImmutableList.of(TEST_REGISTRATION_TOKEN), "mock-topic").get();
    assertEquals(1, results.getSuccessCount() + results.getFailureCount());
  }
```

#### 3.2.2 取消订阅

利用 Admin FCM API，您还可以通过将注册令牌传递给 [`unsubscribeFromTopic()`](https://firebase.google.com/docs/reference/admin/node/admin.messaging.Messaging?refresh=1#unsubscribeFromTopic) 方法来为设备退订主题：

##### 官网 demo

```java
// These registration tokens come from the client FCM SDKs.
List<String> registrationTokens = Arrays.asList(
    "YOUR_REGISTRATION_TOKEN_1",
    // ...
    "YOUR_REGISTRATION_TOKEN_n"
);

// Unsubscribe the devices corresponding to the registration tokens from
// the topic.
TopicManagementResponse response = FirebaseMessaging.getInstance().unsubscribeFromTopicAsync(
    registrationTokens, topic).get();
// See the TopicManagementResponse reference documentation
// for the contents of response.
System.out.println(response.getSuccessCount() + " tokens were unsubscribed successfully");
```

**注意**：在单个请求中，您最多可以为 1000 台设备退订主题。如果您提供的数组所含的注册令牌超过 1000 个，则系统将无法处理此请求，并且会显示 `messaging/invalid-argument` 错误。

`unsubscribeFromTopic()` 方法会返回一个包含 FCM 响应的[对象](https://firebase.google.com/docs/reference/admin/node/admin.messaging.MessagingTopicManagementResponse?refresh=1)。无论请求中指定的注册令牌的数量为何，返回类型的格式都是相同的。

如果出现错误（身份验证失败、令牌或主题无效等）， `unsubscribeFromTopic()` 会返回错误。如需错误代码的完整列表（包括说明和解决步骤），请参阅 [Admin FCM API 错误](https://firebase.google.com/docs/cloud-messaging/admin/errors?refresh=1)。

##### 官方SDK 测试demo

```java
public void testUnsubscribe() throws Exception {
    FirebaseMessaging messaging = FirebaseMessaging.getInstance();
    TopicManagementResponse results = messaging.unsubscribeFromTopicAsync(
        ImmutableList.of(TEST_REGISTRATION_TOKEN), "mock-topic").get();
    assertEquals(1, results.getSuccessCount() + results.getFailureCount());
  }
```

#### 3.2.3 发送消息前先了解消息的顶层定义参数

下面的Message.builder()中可以自定义的消息字段

```properties
data=键值对映射，其中所有键和值都是字符串。
notification=一个包含 title 和 body 字段的对象。
android=一个由 Android 消息专用字段组成的对象。要了解详情，请参阅 Android 专用字段。
apns=一个由 Apple 推送通知服务 (APNS) 专用字段组成的对象。要了解详情，请参阅 APNS 专用字段。
webpush=一个由 WebPush 协议专用字段组成的对象。要了解详情，请参阅 WebPush 专用字段。
token=一个用于标识消息的收件人设备的注册令牌。
topic=要将消息发送至的主题名称。该主题名称不能包含 /topics/ 前缀。
condition=发送消息时所依据的条件，例如 "foo" in topics && "bar" in topics
```



#### 3.2.4 通过主题向安卓设备发送消息

##### 官网 demo

```java
FirebaseMessaging messaging = FirebaseMessaging.getInstance();
Message message = Message.builder()
    .setAndroidConfig(AndroidConfig.builder()
        .setTtl(3600 * 1000) // 1 hour in milliseconds
        .setPriority(AndroidConfig.Priority.NORMAL)//消息优先级，必须是normal和high中一个
        .setNotification(AndroidNotification.builder()//消息对象
            .setTitle("$GOOG up 1.43% on the day")//消息标题
            .setBody("$GOOG gained 11.80 points to close at 835.67, up 1.43% on the day.")//消息体
            .setIcon("stock_ticker_update")//通知图标
            .setColor("#f45342")//通知图标的颜色
            .build())
        .build())
    .setTopic("industry-tech")//设置主题
    .build();
String id = messaging.sendAsync(message, true).get();
```

AndroidConfig.builder()中可设置的参数

```properties
collapseKey=一组消息的标识符，这组消息可以折叠，以便当恢复传送时只发送最后一条消息。在任意给定时间最多允许 4 个不同的折叠键。

priority=消息优先级。必须是 normal 和 high 中的一个。

ttl=消息的生存时间。这是当目标设备离线时消息在 FCM 存储空间中保留的时长。允许的最长时间是 4 周，这也是默认设置。设为 0 可立即发送消息（发后即忘）。在 Node.js 和 Java 中，此时长是以毫秒为单位设置的。在 Python 中，可以将其指定为“datimetime.timedelta”值。在 Go 中，它被指定为“time.Duration”。

restrictedPackageName=应用的软件包名称，其注册令牌必须匹配才能接收消息。

data=键值对映射，其中所有键和值都是字符串。此参数在指定后会替换顶层消息中设置的 data 字段。

notification=一个由 Android 通知专用字段组成的对象。如需受支持字段的列表，请参阅下表。
```

AndroidNotification.builder()使用以下字段

```properties
title:通知的标题。此参数在设置后会替换顶层消息通知中设置的 title 字段。
body:通知的正文。此参数在设置后会替换顶层消息通知中设置的 body 字段。
icon:通知图标。如果未指定此参数，FCM 将会显示您的应用清单文件中指定的启动器图标。
color:通知的图标颜色，以 #rrggbb 格式来表示。
sound:设备收到通知时要播放的声音。支持 default 或应用中绑定的声音资源的文件名。声音文件必须位于 /res/raw/. 中。

tag:用于替换抽屉式通知栏中现有通知的标识符。如果未指定此参数，则每次请求均会创建一条新的通知。如果已指定此参数，且已显示带有相同标记的通知，则新通知将替换抽屉式通知栏中的现有通知。

clickAction:与用户点击通知相关的操作。如果指定了此参数，则当用户点击通知时，将会启动带有匹配 intent 过滤器的 Activity。

bodyLocKey:应用的字符串资源中正文字符串的键，用于将正文文字本地化为用户当前的本地化设置语言。
bodyLocArgs:将用于替换 bodyLocKey（用来将正文文字本地化为用户当前的本地化设置语言）中的格式说明符的字符串值列表。

titleLocKey:应用的字符串资源中标题字符串的键，用于将标题文字本地化为用户当前的本地化设置语言。
titleLocArgs:将用于替换 titleLocKey（用来将标题文字本地化为用户当前的本地化设置语言）中的格式说明符的字符串值列表。
```

##### 官方SDK 测试demo

```java
FirebaseMessaging messaging = FirebaseMessaging.getInstance();
Message message = Message.builder()
            .setAndroidConfig(AndroidConfig.builder()
                .setPriority(AndroidConfig.Priority.HIGH)
                .setTtl(TimeUnit.SECONDS.toMillis(123))
                .setRestrictedPackageName("test-package")
                .setCollapseKey("test-key")
                .setNotification(AndroidNotification.builder()
                    .setClickAction("test-action")
                    .setTitle("test-title")
                    .setBody("test-body")
                    .setIcon("test-icon")
                    .setColor("#112233")
                    .setTag("test-tag")
                    .setSound("test-sound")
                    .setTitleLocalizationKey("test-title-key")
                    .setBodyLocalizationKey("test-body-key")
                    .addTitleLocalizationArg("t-arg1")
                    .addAllTitleLocalizationArgs(ImmutableList.of("t-arg2", "t-arg3"))
                    .addBodyLocalizationArg("b-arg1")
                    .addAllBodyLocalizationArgs(ImmutableList.of("b-arg2", "b-arg3"))
                    .setChannelId("channel-id")
                    .build())
                .build())
            .setTopic("test-topic")
            .build();
String id = messaging.sendAsync(message, true).get();
```

#### 3.2.5 通过主题向IOS设备发送消息

##### 官网 demo

```java
FirebaseMessaging messaging = FirebaseMessaging.getInstance();
Message message = Message.builder()
    .setApnsConfig(ApnsConfig.builder()
        .putHeader("apns-priority", "10")
        .setAps(Aps.builder()
            .setAlert(ApsAlert.builder()
                .setTitle("$GOOG up 1.43% on the day")
                .setBody("$GOOG gained 11.80 points to close at 835.67, up 1.43% on the day.")
                .build())
            .setBadge(42)
            .build())
        .build())
    .setTopic("industry-tech")
    .build();
String id = messaging.sendAsync(message, true).get();
```

ApnsConfig.builder()中字段

```properties
headers:在 Apple 推送通知服务中定义的 HTTP 请求标头。要了解受支持的标头，请参阅 APNS 请求标头。
payload:包含 aps 字典和其他自定义键值对的 APNS 有效负载。要了解受支持的字段，请参阅 APNS 有效负载键参考。Admin SDK 为常用的 APNS 有效负载字段提供了类型化 setter 方法。
```

##### 官方SDK demo

```java
FirebaseMessaging messaging = FirebaseMessaging.getInstance();
Message message = Message.builder()
            .setApnsConfig(ApnsConfig.builder()
                .putHeader("h1", "v1")
                .putAllHeaders(ImmutableMap.of("h2", "v2", "h3", "v3"))
                .putAllCustomData(ImmutableMap.<String, Object>of("k1", "v1", "k2", true))
                .setAps(Aps.builder()
                    .setBadge(42)
                    .setAlert(ApsAlert.builder()
                        .setTitle("test-title")
                        .setSubtitle("test-subtitle")
                        .setBody("test-body")
                        .build())
                    .build())
                .build())
            .setTopic("test-topic")
            .build();
String id = messaging.sendAsync(message, true).get();
```

3.2.6 通过主题向web推送消息

官网 demo

```java
FirebaseMessaging messaging = FirebaseMessaging.getInstance();
Message message = Message.builder()
    .setWebpushConfig(WebpushConfig.builder()
        .setNotification(new WebpushNotification(
            "$GOOG up 1.43% on the day",
            "$GOOG gained 11.80 points to close at 835.67, up 1.43% on the day.",
            "https://my-server/icon.png"))
        .build())
    .setTopic("industry-tech")
    .build();
String id = messaging.sendAsync(message, true).get();
```

WebpushConfig.builder()字段

```properties
headers:在 WebPush 协议中定义的 HTTP 请求标头。要了解受支持的标头，请参阅 WebPush 规范。
data:键值对映射，其中所有键和值都是字符串。此参数在指定后会替换顶层消息中设置的 data 字段。
notification:一个包含 title、body 和 icon 字段的对象。标题和正文会替换顶层消息通知中的相应字段
```

官方SDK demo

```java
FirebaseMessaging messaging = FirebaseMessaging.getInstance();
Message.builder()
            .setWebpushConfig(WebpushConfig.builder()
                .putHeader("h1", "v1")
                .putAllHeaders(ImmutableMap.of("h2", "v2", "h3", "v3"))
                .putData("k1", "v1")
                .putAllData(ImmutableMap.of("k2", "v2", "k3", "v3"))
                .setNotification(WebpushNotification.builder()
                    .setTitle("test-title")
                    .setBody("test-body")
                    .setIcon("test-icon")
                    .setBadge("test-badge")
                    .setImage("test-image")
                    .setLanguage("test-lang")
                    .setTag("test-tag")
                    .setData(ImmutableList.of("arbitrary", "data"))
                    .setDirection(Direction.AUTO)
                    .setRenotify(true)
                    .setRequireInteraction(false)
                    .setSilent(true)
                    .setTimestampMillis(100L)
                    .setVibrate(new int[]{200, 100, 200})
                    .addAction(new Action("action1", "title1"))
                    .addAllActions(ImmutableList.of(new Action("action2", "title2", "icon2")))
                    .putCustomData("k4", "v4")
                    .putAllCustomData(ImmutableMap.<String, Object>of("k5", "v5", "k6", "v6"))
                    .build())
                .build())
            .setTopic("test-topic")
            .build();
String id = messaging.sendAsync(message, true).get();
```

#### 3.2.6 综合应用

官网 demo

```java
FirebaseMessaging messaging = FirebaseMessaging.getInstance();
Message message = Message.builder()
    .setNotification(new Notification(
        "$GOOG up 1.43% on the day",
        "$GOOG gained 11.80 points to close at 835.67, up 1.43% on the day."))
    .setAndroidConfig(AndroidConfig.builder()
        .setTtl(3600 * 1000)
        .setNotification(AndroidNotification.builder()
            .setIcon("stock_ticker_update")
            .setColor("#f45342")
            .build())
        .build())
    .setApnsConfig(ApnsConfig.builder()
        .setAps(Aps.builder()
            .setBadge(42)
            .build())
        .build())
    .setTopic("industry-tech")
    .build();
String id = messaging.sendAsync(message, true).get();
```



官方SDK demo

```java
FirebaseMessaging messaging = FirebaseMessaging.getInstance();
Message message = Message.builder()
        .setNotification(new Notification("Title", "Body"))
        .setAndroidConfig(AndroidConfig.builder()
            .setRestrictedPackageName("com.google.firebase.testing")
            .build())
        .setApnsConfig(ApnsConfig.builder()
            .setAps(Aps.builder()
                .setAlert(ApsAlert.builder()
                    .setTitle("Title")
                    .setBody("Body")
                    .build())
                .build())
            .build())
        .setWebpushConfig(WebpushConfig.builder()
            .putHeader("X-Custom-Val", "Foo")
            .setNotification(new WebpushNotification("Title", "Body"))
            .build())
        .setTopic("foo-bar")
        .build();
String id = messaging.sendAsync(message, true).get();
```

# Firebase-demo
