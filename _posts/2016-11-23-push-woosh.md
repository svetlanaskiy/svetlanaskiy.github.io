---
layout: post
title: "PushWoosh-уведомления в Android"
date: 2016-11-23
backgrounds:
    - https://dl.dropboxusercontent.com/u/18322837/cdn/Streetwill/code-screen.jpg
    - https://dl.dropboxusercontent.com/u/18322837/cdn/Streetwill/the-desk.jpg
thumb: https://dl.dropboxusercontent.com/u/18322837/cdn/Streetwill/thumbs/coding.jpg
categories: development work
tags: home work office coding design
---
В данном посте я опишу процесс подключения Push-уведомлений в Android-приложение с использованием PushWoosh

Push-уведомления являются важнейшей частью мобильного приложения. Новостные рассылки, маркетинговые акции, уведомления о лайках в инстаграмме - все это позволяют сделать
Push-уведомления. В связи со скорым закрытием Parse все больше и больше разработчиков в поисках альтернатив. Одной из них является сервис PushWoosh. Итак, давайте начнем.
PushWoosh работает через Firebase Cloud Messaging (FCM). Как настроить PushWoosh -  проект для работы через FCM отлично показано [здесь](http://docs.pushwoosh.com/docs/fcm-configuration)

Для использования PushWoosh в Android-приложении вам нужно будет добавить в AndroidManifest.xml следующее:

{% highlight scss %}
<uses-permission android:name="android.permission.WAKE_LOCK" />
    <uses-permission android:name="your.package.name.pushwooshtest.permission.C2D_MESSAGE" />
    <uses-permission android:name="com.google.android.c2dm.permission.RECEIVE" />

    <permission
        android:name="your.package.name.permission.C2D_MESSAGE"
        android:protectionLevel="signature" />

    <uses-permission android:name="your.package.name.permission.C2D_MESSAGE" />

    <receiver
        android:name="com.google.android.gms.gcm.GcmReceiver"
        android:exported="true"
        android:permission="com.google.android.c2dm.permission.SEND">
        <intent-filter>
            <action android:name="com.google.android.c2dm.intent.RECEIVE" />

            <category android:name="your.package.name.pushwooshtest" />
        </intent-filter>
    </receiver>

    <service
        android:name="com.pushwoosh.GCMListenerService"
        android:exported="false">
        <intent-filter>
            <action android:name="com.google.android.c2dm.intent.RECEIVE" />
        </intent-filter>
    </service>
    <service
        android:name="com.pushwoosh.GCMInstanceIDListenerService"
        android:exported="false">
        <intent-filter>
            <action android:name="com.google.android.gms.iid.InstanceID" />
        </intent-filter>
    </service>
    <service
        android:name="com.pushwoosh.GCMRegistrationService"
        android:exported="false"></service>
{% endhighlight %}

Далее нам необходимо добавить мета-информацию внутри  application тэга

{% highlight scss %}
        <meta-data
            android:name="PW_APPID"
            android:value="XXXXX-XXXXX" />
        <meta-data
            android:name="PW_PROJECT_ID"
            android:value="A12345678991" />
 {% endhighlight %}

 ВАЖНО! PW_PROJECT_ID это ID проекта в Google Console c префиксом A. То есть, если google project id = 12345, то value=”A12345”
 Вот и все! Если вы правильно прописали разрешения, id проекта, то уже можно отправить push прямо на телефон из консоли управления.

 ![Отправка push](http://www.joshmorony.com/wp-content/uploads/2014/10/pw-control.png)
 Вы можете отправить тестовый пуш к себе на девайс зарегистрировав его в разделе Test Devices. Для этого, запустите только что собранный проект в Android Studio и посмотрите лог в дебаг консоли. Токен можно найти - по строке token. Этот токен нужен будет для ввода в поле Device token.

## Передаем  json и показываем кастомные нотификации

 Если же вам, необходимо отравлять какие-либо данные на устройство в виде json, то для этого нужно имплементировать собтсвенный класс-фабрику.

{% highlight java %}
 public class PushNotificationFactory extends AbsNotificationFactory {

   @Override
   public Notification onGenerateNotification(PushData pushData) {

       final String notificationTitle = "YoutTitleHere";
       JSONObject aps = null;
       String notificationAlert = null;
       try {
           aps = new JSONObject(pushData.getExtras().getString(“SomeStringKey”));
           notificationAlert = aps.getString(“SomeStringKey2”);
       } catch (Exception e) {
           e.printStackTrace();
       }

       if (null != aps && TextUtils.isEmpty(notificationAlert)) {
           notificationAlert = aps.optString(PushApi.ALERT);
       }

       // Create our notification here
       NotificationCompat.BigTextStyle bigTextStyle = new NotificationCompat.BigTextStyle();
       bigTextStyle.setBigContentTitle(notificationTitle);
       bigTextStyle.bigText(notificationAlert);

       final NotificationCompat.Builder builder = new NotificationCompat.Builder(getContext())
               .setSmallIcon(R.drawable.app_icon)
               .setContentTitle(notificationTitle)
               .setDefaults(Notification.DEFAULT_SOUND)
               .setContentText(notificationAlert)
               .setStyle(bigTextStyle);

       final Notification notification = builder.build();
       notification.flags |= Notification.FLAG_AUTO_CANCEL;

       return notification;
   }

   @Override
   public void onPushReceived(PushData pushData) {
   }

   @Override
   public void onPushHandle(Activity activity) {
   }
}

       {% endhighlight %}

Создадим класс PushWooshHelper, у которого будем вызывать  метод init(). Его можно использовать в коде в котором инициализируем PushManager (может быть Activity или Application).

		{% highlight java %}
public class PushWooshHelper {
   public static void init(Context context) {

       // Create and start push manager
       PushManager pushManager = PushManager.getInstance(context);
       // Start push manager, this will count app open for Pushwoosh stats as well
       try {
           pushManager.onStartup(context);
       } catch (Exception e) {
           // Push notifications are not available or AndroidManifest.xml is not configured properly
           e.printStackTrace();
       }

       // Register for push
       pushManager.registerForPushNotifications();
       // Create custom factory in order to create custom notification UI
       pushManager.setNotificationFactory(new PushNotificationFactory());
   }
}
		{% endhighlight %}

   Для того чтобы по тапу на нотификейшн можно было переходить на конкретную активтити, нужно создать собственный BroadcastReceiver и указать это в манифесте в application секции:

    {% highlight java %}
    <receiver android:name=".PushNotificationReceiver" />
	<meta-data
   		android:name="PW_NOTIFICATION_RECEIVER"
 		android:value="your.package.name.pushwooshtest.PushNotificationReceiver" />
    {% endhighlight %}

    {% highlight java %}
    public class PushNotificationReceiver extends BroadcastReceiver {

   @Override
   public void onReceive(Context context, Intent incomingIntent) {
       if (incomingIntent == null)
           return;

       final Bundle pushBundle = PushManagerImpl.preHandlePush(context, incomingIntent);
       if (pushBundle == null) {
           return;
       }
       String id = pushBundle.getString(“id”);
       final Intent pushIntent = new Intent(context.getApplicationContext(), PushReceiverActivity.class);
       pushIntent.setFlags(Intent.FLAG_ACTIVITY_CLEAR_TOP | Intent.FLAG_ACTIVITY_SINGLE_TOP | Intent.FLAG_ACTIVITY_NEW_TASK);
       pushIntent.putExtra(PushNotificationFactory.PushApi.SCHEME, schemeValue);
       context.startActivity(pushIntent);
       PushManagerImpl.postHandlePush(context, incomingIntent);
   }
}
   {% endhighlight %}

   Вот и все, теперь, чтобы отправить json из консоли нужно вставить json в текстовое поле Android root params 
   ![Отправка json](https://files.readme.io/irTvWOecTwiL25Rz1E7i_andr_sendpush.png)

   Полезные ссылки:
   [Manual Integration](http://docs.pushwoosh.com/docs/native-android-sdk)

   [FCM (GCM) Configuration](http://docs.pushwoosh.com/docs/fcm-configuration)

   [Customizing Android app](http://docs.pushwoosh.com/docs/androidmanifestxml-modifications#using-local-notifications-with-pushwoosh)
   
   [Example](http://www.programcreek.com/java-api-examples/index.php?source_dir=pushwoosh-native-samples-master/Android/src/com/pushwoosh/test/tags/sample/app/NotificationFactorySample.java)





