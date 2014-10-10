title: "从Java的反射机制说开去<2>"
date: 2014-10-10 11:18:09
tags: [Java, 反射机制]
categories: Java
---
## Android Hotspot的使用
接上文：[从Java的反射机制说开去<1>](http://vensent.github.io/2014/10/10/%E4%BB%8EJava%E7%9A%84%E5%8F%8D%E5%B0%84%E6%9C%BA%E5%88%B6%E8%AF%B4%E5%BC%80%E5%8E%BB-1/)。
我们知道Android有很多的应用使用了Ap（Access Point）热点。而很多的应用也开始越来越多地使用Ap，比如最近很火的“闪传”，“QQ近传”等等。笔者在写这样的一个简单的开启热点的时候却遇到了问题。具体来说就是必须要使用到反射机制来解决，现在先看代码。

## 调用的代码
```java
public static void turnOnOffHotspot(Context context, boolean isTurnToOn) {
    WifiManager wifiManager = (WifiManager) context
    		.getSystemService(Context.WIFI_SERVICE);
    WifiApControl apControl = WifiApControl.getApControl(wifiManager);
    if (apControl != null ) {
    	// 使用WifiManager来关闭wifi
    	wifiManager.setWifiEnabled(false);
        apControl.setWifiApEnabled(apControl.getWifiApConfiguration(),
                isTurnToOn);
    }
}
```
其中WifiApControl是自己写好的一个类，代码比较长，不过如果理解了反射机制的话还是很好理解的，贴下来如下：
```java
/**
 * This class is use to handle all Hotspot related information.
 * 
 */
public class WifiApControl {
    private static Method getWifiApState;
    private static Method isWifiApEnabled;
    private static Method setWifiApEnabled;
    private static Method getWifiApConfiguration;
 
    public static final String WIFI_AP_STATE_CHANGED_ACTION = "android.net.wifi.WIFI_AP_STATE_CHANGED";
 
    public static final int WIFI_AP_STATE_DISABLED = WifiManager.WIFI_STATE_DISABLED;
    public static final int WIFI_AP_STATE_DISABLING = WifiManager.WIFI_STATE_DISABLING;
    public static final int WIFI_AP_STATE_ENABLED = WifiManager.WIFI_STATE_ENABLED;
    public static final int WIFI_AP_STATE_ENABLING = WifiManager.WIFI_STATE_ENABLING;
    public static final int WIFI_AP_STATE_FAILED = WifiManager.WIFI_STATE_UNKNOWN;
 
    public static final String EXTRA_PREVIOUS_WIFI_AP_STATE = WifiManager.EXTRA_PREVIOUS_WIFI_STATE;
    public static final String EXTRA_WIFI_AP_STATE = WifiManager.EXTRA_WIFI_STATE;
 
    static {
        // lookup methods and fields not defined publicly in the SDK.
        Class<?> cls = WifiManager.class;
        // nameList用于打印出WifiManager中的所有方法
        ArrayList<String> nameList = new ArrayList<String>();
        for (Method method : cls.getDeclaredMethods()) {
            String methodName = method.getName();
            nameList.add(methodName);
            if (methodName.equals("getWifiApState")) {
                getWifiApState = method;
            } else if (methodName.equals("isWifiApEnabled")) {
                isWifiApEnabled = method;
            } else if (methodName.equals("setWifiApEnabled")) {
                setWifiApEnabled = method;
            } else if (methodName.equals("getWifiApConfiguration")) {
                getWifiApConfiguration = method;
               
            }
        }
        // 用在调试
        nameList.add("0");
    }
 
    public static boolean isApSupported() {
        return (getWifiApState != null && isWifiApEnabled != null
                && setWifiApEnabled != null && getWifiApConfiguration != null);
    }
 
    private WifiManager mgr;
 
    private WifiApControl(WifiManager mgr) {
        this.mgr = mgr;
    }
 
    public static WifiApControl getApControl(WifiManager mgr) {
        if (!isApSupported())
            return null;
        return new WifiApControl(mgr);
    }
 
    public boolean isWifiApEnabled() {
        try {
            return (Boolean) isWifiApEnabled.invoke(mgr);
        } catch (Exception e) {
            Log.v("BatPhone", e.toString(), e); // shouldn't happen
            return false;
        }
    }
 
    public int getWifiApState() {
        try {
            return (Integer) getWifiApState.invoke(mgr);
        } catch (Exception e) {
            Log.v("BatPhone", e.toString(), e); // shouldn't happen
            return -1;
        }
    }
 
    public WifiConfiguration getWifiApConfiguration() {
        try {
            return (WifiConfiguration) getWifiApConfiguration.invoke(mgr);
        } catch (Exception e) {
            Log.v("BatPhone", e.toString(), e); // shouldn't happen
            return null;
        }
    }
 
    public boolean setWifiApEnabled(WifiConfiguration config, boolean enabled) {
        try {
            return (Boolean) setWifiApEnabled.invoke(mgr, config, enabled);
        } catch (Exception e) {
            Log.v("BatPhone", e.toString(), e); // shouldn't happen
            return false;
        }
    }
}
```

## 代码分析
通过反射机制，可以看出来在WifiApControl中是为了获得WifiManager类中的`getWifiApState`、`isWifiApEnabled`、`setWifiApEnabled`、`getWifiApConfiguration`这四个方法。但是其中WifiManager中没有这些方法么？我们具体点开可以看看，以下是代码中显示的WifiManager的所有方法：
<div style="text-align:center"><img src ="https://vensent.github.io/img/WifiManager_Methods.jpg" /></div>
而以上四种方法是都没有的。
那没有的话为什么可以获取到呢？我们通过调试进入断点看一看：
<div style="text-align:center"><img src ="https://vensent.github.io/img/WifiManager_nameList_1.jpg" /><img src ="https://vensent.github.io/img/WifiManager_nameList_2.jpg" /></div>
可以看出在用反射机制去获得WifiManager中的所有方法的时候，是可以得到的这四个方法的。
那么究竟是怎么一回事呢？

## 原因分析
进入Android的platform_framworks_base的源码去看一下。代码在这里也可以看到：[WifiManager.java](https://github.com/android/platform_frameworks_base/blob/master/wifi/java/android/net/wifi/WifiManager.java)。
以`getWifiApState`为例，其中定义代码在如下：
```java
    /**
     * Gets the Wi-Fi enabled state.
     * @return One of {@link #WIFI_AP_STATE_DISABLED},
     *         {@link #WIFI_AP_STATE_DISABLING}, {@link #WIFI_AP_STATE_ENABLED},
     *         {@link #WIFI_AP_STATE_ENABLING}, {@link #WIFI_AP_STATE_FAILED}
     * @see #isWifiApEnabled()
     *
     * @hide Dont open yet
     */
    public int getWifiApState() {
        try {
            return mService.getWifiApEnabledState();
        } catch (RemoteException e) {
            return WIFI_AP_STATE_FAILED;
        }
    }
```
其中在注释中的`@hide Dont open yet`是就是我们找到的原因。原来API自己把这些方法隐藏了。而在Eclipse中通过.class的反编译的结果查看的时候是看不到这些方法的。
笔者猜测Google这样做的原因可能是因为这些API属于内部逻辑，不想对外暴露，也有可能是API接口还未最终确定下来。所以在低版本Android上的隐藏API不一定能在高版本的Android上使用。这点是一定要注意的。也就说隐藏API的兼容性比较差。因此利用反射调用隐藏API时，一定要注意根据Android的版本采用不同的方式去反射。

