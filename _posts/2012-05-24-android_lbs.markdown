---
layout: post
title:  "Android那些事儿之LBS定位"
date:   2012-05-24 11:37:42 +0800
categories: Android开发
tags: android
---

最近为了做```LBS```功能模块，到网上搜了一些资料，大多数介绍都是使用繁琐的基站定位，要自己去读取什么```CellId```，```LocationAreaCode```， ```MobileCountryCode```，```MobileNetworkCode```等参数，而且多数是针对```GSM/UMTS```。而自己使用的```CDMA```，跟上面的参数叫法不一样，还得自己一个一个去对应。虽然最后算是解决了，但是难道就没有更好的办法吗。

翻了翻```Android Developer```找到一个不错的东西```LocationManager```。
```LocationManager```是通过```listener```的方式来告知调用者，而原来写好的模块是直接```return```的，于是得稍微改造一下：

首先定义一个```Model```：

```java
public class LocationData {  
    String lat;  
    String lon;  
    String address;  
}  
```

然后```LBS```的所有功能都封装到一个工具类里面：

首先在构造函数里面获取系统服务中的```LocationManager```：

```java
public class LBSTool {  
    private Context mContext;  
    private LocationManager mLocationManager;   
    private LocationData mLocation;  
    private LBSThread mLBSThread;  
    private MyLocationListner mNetworkListner;  
    private MyLocationListner mGPSListener;  
    private Looper mLooper;  
      
    public LBSTool(Context context) {  
        mContext = context;  
        //获取Location manager  
        mLocationManager = (LocationManager)mContext.getSystemService(Context.LOCATION_SERVICE);  
    }  
  
......  
}  
```

然后是入口方法，这里会启动一个子线程去获取地理位置信息，并让主线程进入等待，时长通过```timeout```设置

```java
/**  
 * 开始定位   
 * @param timeout 超时设置  
 * @return LocationData位置数据，如果超时则为null  
 */  
public LocationData getLocation(long timeout) {  
    mLocation = null;  
    mLBSThread = new LBSThread();  
    mLBSThread.start();//启动LBSThread  
    timeout = timeout > 0 ? timeout : 0;  
      
    synchronized (mLBSThread) {  
        try {  
            Log.i(Thread.currentThread().getName(), "Waiting for LocationThread to complete...");  
            mLBSThread.wait(timeout);//主线程进入等待，等待时长timeout ms  
            Log.i(Thread.currentThread().getName(), "Completed.Now back to main thread");  
        }  
        catch (InterruptedException e) {  
            e.printStackTrace();  
        }  
    }  
    mLBSThread = null;  
    return mLocation;  
} 
```

子线程通过调用```registerLocationListener```开启位置服务的监听，并且讲监听器分配给指定```looper```：

```java
private class LBSThread extends Thread {  
    @Override  
    public void run() {  
        setName("location thread");   
        Log.i(Thread.currentThread().getName(), "--start--");   
        Looper.prepare();//给LBSThread加上Looper  
        mLooper = Looper.myLooper();  
        registerLocationListener();  
        Looper.loop();  
        Log.e(Thread.currentThread().getName(),  "--end--");  
          
    }  
}  
  
private void registerLocationListener () {  
    Log.i(Thread.currentThread().getName(), "registerLocationListener");          
    if (isGPSEnabled()) {  
        mGPSListener=new MyLocationListner();    
  
        //五个参数分别为位置服务的提供者，最短通知时间间隔，最小位置变化，listener，listener所在消息队列的looper  
        mLocationManager.requestLocationUpdates(LocationManager.GPS_PROVIDER, 5000, 0, mGPSListener, mLooper);    
    }  
    if (isNetworkEnabled()) {  
        mNetworkListner=new MyLocationListner();    
 
        mLocationManager.requestLocationUpdates(LocationManager.NETWORK_PROVIDER, 3000, 0, mNetworkListner, mLooper);    
    }  
} 
```
 
 ```isGPSEnabled和isNetworkEnabled```分别为判断当前手机是否开启了```GPS```以及网络的状况（包含了是否开启```wifi```和移动网络），以决定使用哪一种服务提供者：```GPS_PROVIDER```或者```NETWORK_PROVIDER```。

```java
/**  
 * 判断GPS是否开启  
 * @return  
 */  
public boolean isGPSEnabled() {  
    if(mLocationManager.isProviderEnabled(LocationManager.GPS_PROVIDER)) {  
        Log.i(Thread.currentThread().getName(), "isGPSEnabled");  
        return true;  
    }   
    else {  
        return false;  
    }  
}  
  
/**  
 * 判断Network是否开启(包括移动网络和wifi)  
 * @return  
 */  
public boolean isNetworkEnabled() {  
    return (isWIFIEnabled() || isTelephonyEnabled());   
}  
  
/**  
 * 判断移动网络是否开启  
 * @return  
 */  
public boolean isTelephonyEnabled() {  
    boolean enable = false;  
    TelephonyManager telephonyManager = (TelephonyManager) mContext.getSystemService(Context.TELEPHONY_SERVICE);  
    if (telephonyManager != null) {  
        if (telephonyManager.getNetworkType() != TelephonyManager.NETWORK_TYPE_UNKNOWN) {  
            enable = true;  
            Log.i(Thread.currentThread().getName(), "isTelephonyEnabled");  
        }  
    }  
      
    return enable;  
}  
  
/**  
 * 判断wifi是否开启  
 */  
public boolean isWIFIEnabled() {  
    boolean enable = false;  
    WifiManager wifiManager = (WifiManager)mContext.getSystemService(Context.WIFI_SERVICE);  
    if(wifiManager.isWifiEnabled()) {  
        enable = true;  
        Log.i(Thread.currentThread().getName(), "isWIFIEnabled");   
    }   
    return enable;  
} 
```

当```LocationManager```在大于最短时间且检测到最小位置变化时，就会通知给监听器，然后我们就可以通过返回的经纬度信息去```google```服务器查找对应的地址，然后停止```LocationManger```的工作，解除```LBSThread```中的```Looper```，让```LBSThread```结束，最后通知主线程可以继续，整个流程结束。

```java
private class MyLocationListner implements LocationListener{    
  
    @Override  
    public void onLocationChanged(Location location) {    
        // 当LocationManager检测到最小位置变化时，就会回调到这里  
        Log.i(Thread.currentThread().getName(), "Got New Location of provider:"+location.getProvider());  
        unRegisterLocationListener();//停止LocationManager的工作  
        try {  
            synchronized (mLBSThread) {   
                parseLatLon(location.getLatitude()+"", location.getLongitude()+"");//解析地理位置  
                mLooper.quit();//解除LBSThread的Looper，LBSThread结束  
                mLBSThread.notify();//通知主线程继续  
            }  
        }  
        catch (Exception e) {  
            e.printStackTrace();  
        }  
    }    
 
    //后3个方法此处不做处理 
    @Override  
    public void onStatusChanged(String provider, int status, Bundle extras) {}    
 
    @Override  
    public void onProviderEnabled(String provider) {}    
 
    @Override  
    public void onProviderDisabled(String provider) {}  
  
};  
  
/**  
 * 使用经纬度从goole服务器获取对应地址   
 * @param 经纬度  
 */  
private void parseLatLon(String lat, String lon) throws Exception {  
    Log.e(Thread.currentThread().getName(),  "---parseLatLon---");  
    Log.e(Thread.currentThread().getName(),  "---"+lat+"---");  
    try {  
        HttpClient httpClient = new DefaultHttpClient();  
        HttpGet get = new HttpGet("http://ditu.google.cn/maps/geo?output=json&q="+lat+","+lon);  
        HttpResponse response = httpClient.execute(get);  
        String resultString = EntityUtils.toString(response.getEntity());  
          
        JSONObject jsonresult = new JSONObject(resultString);  
        if(jsonresult.optJSONArray("Placemark") != null) {  
            mLocation = new LocationData();  
            mLocation.lat = lat;  
            mLocation.lon = lon;  
            mLocation.address = jsonresult.optJSONArray("Placemark").optJSONObject(0).optString("address");  
        }  
    }  
    catch (Exception e) {  
        e.printStackTrace();  
    }  
}  
  
/**  
 * 注销监听器   
 */  
private void unRegisterLocationListener () {  
    if(mGPSListener!=null){    
        mLocationManager.removeUpdates(mGPSListener);    
        mGPSListener=null;    
    }   
    if(mNetworkListner!=null){    
        mLocationManager.removeUpdates(mNetworkListner);    
        mNetworkListner=null;    
    }   
} 
```

接下来可以在界面上安放个```button```:

```java
locationBtn.setOnClickListener(new OnClickListener() {  
              
    @Override  
    public void onClick(View v) {  
        //return mode  
        LBSTool lbs = new LBSTool(LBStestActivity.this);  
        LocationData location = lbs.getLocation(120000);  
        if (location != null) {  
            Log.i("---lat---",location.lat);  
            Log.i("---lon---",location.lon);  
            Log.i("---address---",location.address);  
            Toast.makeText(LBStestActivity.this, location.lat + " " + location.lon + " " + location.address, Toast.LENGTH_LONG).show();  
 
        }  
          
    }  
}); 
```

最后别忘了加入权限：

```xml
<uses-permission android:name="android.permission.INTERNET" />  
<uses-permission android:name="android.permission.ACCESS_NETWORK_STATE" />  
<uses-permission android:name="android.permission.ACCESS_WIFI_STATE"/>  
<uses-permission android:name="android.permission.ACCESS_COARSE_LOCATION" />  
<uses-permission android:name="android.permission.ACCESS_FINE_LOCATION" />  
```
 
 
此外，```LocationManager```还有些高级的用法，比如设置一些关键参数，以及获取最后一次定位信息等等：

```java
Criteria criteria = new Criteria(); 
criteria.setAccuracy(Criteria.ACCURACY_FINE); // 高精度 
criteria.setAltitudeRequired(false); 
criteria.setBearingRequired(false); 
criteria.setCostAllowed(true); 
criteria.setPowerRequirement(Criteria.POWER_LOW); // 低功耗 
 
String bestprovider = locationManager.getBestProvider(criteria, true); // 获取GPS信息 
Location location = locationManager.getLastKnownLocation(bestprovider); // 通过GPS获取位置 
Log.e("--bestprovider--", bestprovider); 
Log.e("--bestprovider--", location.getLatitude()+""); 
```