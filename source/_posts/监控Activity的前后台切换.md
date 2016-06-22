---
title: (译)监控Activity的前后台切换
date: 2016-06-20 17:55:29
tags: Android
---
## APP进入后台时获得通知
Api14(Android 4.0/ICS)开始，我们获取到一个方法[Application#onTrimMemory(int level)](https://developer.android.com/reference/android/app/Application.html#onTrimMemory(int)。这个方法包含一个有趣的等级叫[TRIM_MEMORY_UI_HIDDEN](https://developer.android.com/reference/android/content/ComponentCallbacks2.html#TRIM_MEMORY_UI_HIDDEN)，当程序进入后台时来通知我们。

举个栗子：
```bash
public class MyApplication extends Application {
    // ...
    @Override
    public void onTrimMemory(int level) {
        super.onTrimMemory(level);
        if (level == TRIM_MEMORY_UI_HIDDEN) {
           isBackground = true;
           notifyBackground();
        }
    }
    // ...
}
```

Yah!!!现在你可以肯定你的App进入到后台了！

## 屏幕关闭时获得通知
当屏幕关闭时，**onTrimMemory**没有被调用。为了处理这个，你需要注册一个广播[Intent.ACTION_SCREEN_OFF](https://developer.android.com/reference/android/content/Intent.html#ACTION_SCREEN_OFF)。

```bash
public class MyApplication extends Application {
  // ...
  @Override
    public void onCreate() {
        super.onCreate();
        // ...
        IntentFilter screenOffFilter = new IntentFilter(Intent.ACTION_SCREEN_OFF);
        registerReceiver(new BroadcastReceiver() {
            @Override
            public void onReceive(Context context, Intent intent) {
              if (isBackground) {
                  isBackground = false;
                  notifyForeground();
              }
            }
        }, screenOffFilter);
      }
}
```

*注意没有必要监听Intent.ACTION_SCREEN_ON， 它在下面被处理.*

## 返回App时获取通知
当我们的应用重新进入前台时没有一个标记或者等级来通知我们，我们能做的最好的方法是使用**Activity#onResume()**。我们可以把所有的代码都加入到base Activity里面，但是没有必要。我们有一种更加简洁的处理方式，像处理进入后台那样，所以我们来使用[Application#registerActivityLifecycleCallbacks()](https://developer.android.com/reference/android/app/Application.html#registerActivityLifecycleCallbacks(android.app.Application.ActivityLifecycleCallbacks)。这个方法使你能够添加一个[ActivityLifecycleCallbacks](https://developer.android.com/reference/android/app/Application.ActivityLifecycleCallbacks.html)。就像它的名字提示一样，在每一个以及每一次生命周期时间发生时通知你，这意味着我们不用修改任何activity,也能获取每一个Activity#onResume()的效果。

栗子：
```bash
public class MyApplication extends Application {
    // ...
    @Override
       public void onCreate() {
           super.onCreate();

           registerActivityLifecycleCallbacks(new Application.ActivityLifecycleCallbacks() {
               // ...
               @Override
               public void onActivityResumed(Activity activity) {
                 if (isBackground) {
                     isBackground = false;
                     notifyForeground();
                 }
               }
               // ...
           });
       }
       // ...
}
```

## 合在一起看一下咯
下面是一个更加复杂的栗子，当app进入到前后或者退回后台都能被通知到。
```bash
public class MyApplication extends Application {

    // Starts as true in order to be notified on first launch
    private boolean isBackground = true;

    @Override
    public void onCreate() {
        super.onCreate();

        listenForForeground();
        listenForScreenTurningOff();
    }

    private void listenForForeground() {
        registerActivityLifecycleCallbacks(new ActivityLifecycleCallbacks() {
            //...
            @Override
            public void onActivityResumed(Activity activity) {
                if (isBackground) {
                    isBackground = false;
                    notifyForeground();
                }
            }
            //...
        });
    }

    private void listenForScreenTurningOff() {
        IntentFilter screenStateFilter = new IntentFilter(Intent.ACTION_SCREEN_OFF);
        registerReceiver(new BroadcastReceiver() {
            @Override
            public void onReceive(Context context, Intent intent) {
                isBackground = true;
                notifyBackground();
            }
        }, screenStateFilter);
    }

    @Override
    public void onTrimMemory(int level) {
        super.onTrimMemory(level);
        if (level == TRIM_MEMORY_UI_HIDDEN) {
            isBackground = true;
            notifyBackground();
        }

    }

    private void notifyForeground() {
        // This is where you can notify listeners, handle session tracking, etc
    }

    private void notifyBackground() {
        // This is where you can notify listeners, handle session tracking, etc
    }

    public boolean isBackground() {
      return isBackground;
    }
}
```

## 简而言之
* Target API 14
* 程序进入后台时使用[Application#onTrimMemory(int level)](https://developer.android.com/reference/android/app/Application.html#onTrimMemory(int)以及[TRIM_MEMORY_UI_HIDDEN](https://developer.android.com/reference/android/content/ComponentCallbacks2.html#TRIM_MEMORY_UI_HIDDEN)
* 注册一个广播接收[Intent.ACTION_SCREEN_OFF](https://developer.android.com/reference/android/content/Intent.html#ACTION_SCREEN_OFF)来处理关闭屏幕事件
* 注册一个[Application#registerActivityLifecycleCallbacks()](https://developer.android.com/reference/android/app/Application.html#registerActivityLifecycleCallbacks(android.app.Application.ActivityLifecycleCallbacks))在程序重新进入前台时来获取通知

[http://www.developerphil.com/no-you-can-not-override-the-home-button-but-you-dont-have-to/?utm_source=Android+Weekly&utm_campaign=f1e65e3bb5-Android_Weekly_210&utm_medium=email&utm_term=0_4eb677ad19-f1e65e3bb5-337302153](http://www.developerphil.com/no-you-can-not-override-the-home-button-but-you-dont-have-to/?utm_source=Android+Weekly&utm_campaign=f1e65e3bb5-Android_Weekly_210&utm_medium=email&utm_term=0_4eb677ad19-f1e65e3bb5-337302153)
