---
title: Android 读取其他应用通知栏消息
date: 2018-2-2 18:03:00
categories: 
tags: Android
---

> `NotificationListenerService`(API 21)，获取通知相关信息（通知的新增和删除，获取当前通知数量，通知内容信息）。
>
> 这些信息可通过`NotificationListenerService`类及`StatusBarNotification`类对象获取。

#### NotificationListenerService主要方法(成员变量)： 

cancelAllNotifications() ：删除系统中所有可被清除的通知； 
cancelNotification(String pkg, String tag, int id) ：删除具体某一个通知； 
getActiveNotifications() ：返回当前系统所有通知到StatusBarNotification[]数组； 
onNotificationPosted(StatusBarNotification sbn) ：当系统收到新的通知后出发回调； 
onNotificationRemoved(StatusBarNotification sbn) ：当系统通知被删掉后出发回调；

StatusBarNotification主要方法(成员变量)： 
getId()：返回通知对应的id； 
getNotification()：返回通知对象； 
getPackageName()：返回通知对应的包名； 
getPostTime()：返回通知发起的时间； 
getTag()：返回通知的Tag，如果没有设置返回null； 
getUserId()：返回UserId，用于多用户场景； 
isClearable()：返回该通知是否可被清楚，FLAG_ONGOING_EVENT、FLAG_NO_CLEAR； 
isOngoing()：检查该通知的flag是否为FLAG_ONGOING_EVENT；

#### Step 1. 新建通知消息监听类

新建一个类继承自`NotificationListenerService`，复写两个重要方法。

```java
public class AppNotificationListenerService extends NotificationListenerService {

    @Override
    public void onNotificationPosted(StatusBarNotification sbn) {
        super.onNotificationPosted(sbn);
        Log.d("NotificationListener","Notification Posted");
        Log.d("NotificationListener",sbn.getPackageName());
    }

    @Override
    public void onNotificationRemoved(StatusBarNotification sbn) {
        super.onNotificationRemoved(sbn);
        Log.d("NotificationListener","onNotification Removed");
    }
    
}
```

#### Step 2. 注册Service

在AndroidManifest.xml中注册Service并声明权限。

```xml
<service
	android:name=".MyNotificationListenerService"
	android:label="@string/service_name"
	android:permission="android.permission.BIND_NOTIFICATION_LISTENER_SERVICE">
    <intent-filter>
		<action android:name="android.service.notification.NotificationListenerService" />
	</intent-filter>
</service>
```

#### Step 3. 获取权限

##### 判断权限是否开启

> 使用NotificationListenerService的应用如果开启了Notification access，系统会将包名等相关信息写入SettingsProver数据库中，因此可以从数据库中获取相关信息并过滤，从而判断应用是否开启了Notification access。

```java
private static final String ENABLED_NOTIFICATION_LISTENERS = "enabled_notification_listeners";

private boolean isNotificationListenersEnabled() {
	String pkgName = getPackageName();
	String flat = Settings.Secure.getString(getContentResolver(),ENABLED_NOTIFICATION_LISTENERS);
	if (!TextUtils.isEmpty(flat)) {
		String[] names = flat.split(":");
		for (String name : names) {
			ComponentName cn = ComponentName.unflattenFromString(name);
            if (cn != null) {
            	return TextUtils.equals(pkgName, cn.getPackageName());
            }
        }
    }
    return false;
}
```

##### 跳转通知访问权限开启界面

```java
public static boolean goToNotificationAccessSetting(Context context) {
	try {
		Intent intent = new Intent("android.settings.ACTION_NOTIFICATION_LISTENER_SETTINGS");
        intent.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
        context.startActivity(intent);
        return true;
        } catch (ActivityNotFoundException e) {//普通情况下找不到的时候需要再特殊处理找一次
        	try {
        		Intent intent = new Intent();
            	intent.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
            	ComponentName cn = new ComponentName("com.android.settings", "com.android.settings.Settings$NotificationAccessSettingsActivity");
            	intent.setComponent(cn);
            	intent.putExtra(":settings:show_fragment", "NotificationAccessSettings");
            	context.startActivity(intent);
            	return true;
            } catch (Exception e1) {
                e1.printStackTrace();
            }
            Toast.makeText(context, "对不起，您的手机暂不支持", Toast.LENGTH_SHORT).show();
            e.printStackTrace();
            return false;
        }
    }
```

##### 针对应用被清理重启后`NotificaitonListenerService`失效

应用被杀死，重启应用后，需要让系统重新执行授权(也就是改变数据库的状态)，可以在应用启动的时候使用该方法。

```java
public static void toggleNotificationListenerService(Context context) {
    Log.e(tag, "toggleNotificationListenerService");
    PackageManager pm = context.getPackageManager();
    pm.setComponentEnabledSetting(new ComponentName(context, AppNotificationListenerService.class), PackageManager.COMPONENT_ENABLED_STATE_DISABLED, PackageManager.DONT_KILL_APP);
    pm.setComponentEnabledSetting(new ComponentName(context, AppNotificationListenerService.class), PackageManager.COMPONENT_ENABLED_STATE_ENABLED, PackageManager.DONT_KILL_APP);
}
```

让系统重新绑定服务。

#### Step 4. 统计通知栏消息数

在AppNotificationListenerService服务中的onCreate方法中调用getActiveNotifications获取StatusBarNotification消息数组时，需要延时获取，因为这个service还没有创建成功，StatusBarNotification数组的lenth即当前通知栏的消息数。

```java
@Override
public void onCreate() { 
    super.onCreate()；
    new Handler().postDelayed(new Runnable() {
		@Override
        public void run() {
        	Log.e(TAG, "onCreate, notification count:" + getActiveNotifications().length);
        }
    },1000);
}
```

在接收到新消息或者移除消息时，即在AppNotificationListenerService服务中的onNotificationPosted，onNotificationRemoved方法中统计消息数，或者处理StatusBarNotification消息

```java
public class AppNotificationListenerService extends NotificationListenerService {  
        @Override  
        public void onNotificationPosted(StatusBarNotification sbn) {  
            Log.e(TAG,,"Notification posted");  
            Log.e(TAG, "notification count ：" + getActiveNotifications().length);
            //处理sbn通知消息
        }  

        @Override  
        public void onNotificationRemoved(StatusBarNotification sbn) {  
            Log.e(TAG,,"Notification removed");   
            Log.e(TAG, "notification count ：" + getActiveNotifications().length);
            //处理sbn通知消息
        }  
} 
```

