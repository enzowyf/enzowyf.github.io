---
layout: post
title:  "Android网络开发-请求队列"
date:   2012-05-03 17:00:17 +0800
categories: Android开发
tags: android network
---

因为之前参与的网络开发项目都遇到一些相同的问题：

1. 大量的并发请求造成堵塞，特别是遇上让人无语的```3G```网络，无限loading。。。
2. 一般来说一个网络请求都会用使用到一个异步线程，大量的线程创建、运行、销毁又造成了系统资源的浪费
3. 请求结束得到结果后，如果需要更新```UI```，一个不小心忘了返回```UI```线程，各种崩溃。。。
 
前些日子跟同事商量能不能做个请求队列去进行控制，于是趁着热度没消退说干就干，建了个模型，以备日后使用。
 
在这个模型中，有高中低三个优先级信道如下：高优先级--1，中优先级--3，低优先级--2

规则：
> 
> 1.正常情况下各个优先级使用各自信道（线程）
> 
> 2.高级信道满载、中、低级信道空置，则高级请求可使用低级信道
 
构思：

> ```UI```线程将期望的网络请求url和参数通过一个封装好的```Runnable```提交给```Service```处理（当然也可以交给一个```Thread```处理，本例使用```Service```），```Service```接收到请求，判断优先级，加入到相应线程池中排队。线程池启动线程发起网络请求，最后通过监听器将结果返回给```Service```，```Service```发送广播通知```UI```线程，```UI```线程更新相关界面，结束。
 
废话说完，上例子，首先是封装好的```Runnable```：
 
```java
public class HttpConnRunnable implements Runnable, Parcelable { 
 
    public static final int HIGH_LEVEL = 0; 
    public static final int NORMAL_LEVEL = 1; 
    public static final int LOW_LEVEL = 2; 
     
    private int mPriority = NORMAL_LEVEL;//优先级，默认为普通 
    private String mUrl = ""; 
     
    private HttpConnListener mListener;//监听器 
     
     
    public HttpConnRunnable() { 
        super(); 
    } 
     
    public HttpConnRunnable(int priority) { 
        super(); 
        mPriority = priority; 
    }    
 
    @Override 
    public void run() { 
        Log.i(Thread.currentThread().getName(), "----Start to connect:" + mUrl + ", priority:" + mPriority + "-----"); 
        try { 
            Thread.sleep(10000); 
            //TODO:进行网络请求相关操作，并通过listener返回结果 
            mListener.onSucceed("Connected to " + mUrl + " succeed!"); 
        } 
        catch (InterruptedException e) { 
            e.printStackTrace(); 
        } 
        Log.i(Thread.currentThread().getName(), "----Finish to connect:" + mUrl + ", priority:" + mPriority + "-----"); 
    } 
 
    public int getPriority() { 
        return mPriority; 
    } 
     
    public void setPriority(int priority) { 
        mPriority = priority; 
    } 
     
    public String getURL() { 
        return mUrl; 
    } 
     
    public void setURL(String url) { 
        mUrl = url; 
    } 
     
    public void setHttpConnListener(HttpConnListener listener) { 
        mListener = listener; 
    } 
     
    //序列化，为了传递给Service，如果是使用Thread处理本例，则无需序列化 
    public static final Parcelable.Creator<HttpConnRunnable> CREATOR = new Creator<HttpConnRunnable>() { 
        @Override 
        public HttpConnRunnable createFromParcel(Parcel source) { 
            HttpConnRunnable data = null; 
            Bundle bundle = source.readBundle(); 
            if(bundle != null) { 
                data = new HttpConnRunnable(bundle.getInt("PRIORITY")); 
                data.mUrl = bundle.getString("URL"); 
            } 
            return data; 
        } 
 
        @Override 
        public HttpConnRunnable[] newArray(int size) { 
            return new HttpConnRunnable[size]; 
        } 
    }; 
     
    @Override 
    public int describeContents() { 
        return 0; 
    } 
 
    @Override 
    public void writeToParcel(Parcel dest, int flags) { 
        Bundle bundle = new Bundle(); 
        bundle.putInt("PRIORITY", mPriority); 
        bundle.putString("URL", mUrl); 
        dest.writeBundle(bundle); 
    } 
 
} 
```

然后是```Service```的处理:

```java
public class HttpConnService extends Service implements HttpConnListener { 
    public static final String HTTP_POOL_PARAM_KEYWORD = "HttpPoolParam";           //网络参数传递的关键字 
     
    private final int HIGH_POOL_SIZE = 1; 
    private final int NORMAL_POOL_SIZE = 3; 
    private final int LOW_POOL_SIZE = 2; 
     
    // 可重用固定线程数的线程池 
    private ThreadPoolExecutor mHighPool; 
    private ThreadPoolExecutor mNormalPool; 
    private ThreadPoolExecutor mLowPool; 
     
    @Override 
    public void onCreate() { 
        //初始化所有 
        mHighPool = (ThreadPoolExecutor) Executors.newFixedThreadPool(HIGH_POOL_SIZE); 
        mNormalPool = (ThreadPoolExecutor) Executors.newFixedThreadPool(NORMAL_POOL_SIZE); 
        mLowPool = (ThreadPoolExecutor) Executors.newFixedThreadPool(LOW_POOL_SIZE); 
 
        super.onCreate(); 
    } 
 
    @Override 
    public int onStartCommand(Intent intent, int flags, int startId) { 
        //接受到来自UI线程的请求 
        //取出Runnable，并加入到相应队列 
        Bundle bundle = intent.getExtras(); 
        HttpConnRunnable httpConnRunnable = bundle.getParcelable(HTTP_POOL_PARAM_KEYWORD); 
        if (httpConnRunnable != null) { 
            httpConnRunnable.setHttpConnListener(HttpConnService.this); 
            int level = httpConnRunnable.getPriority(); 
            switch (level) { 
                case HttpConnRunnable.HIGH_LEVEL: 
                    //如果高级池满而低级池未满交由低级池处理 
                    //如果高级池满而普通池未满交由普通池处理 
                    //如果高级池未满则交给高级池处理，否则，交由高级池排队等候 
                    if (mHighPool.getActiveCount() == HIGH_POOL_SIZE && mLowPool.getActiveCount() < LOW_POOL_SIZE) { 
                        mLowPool.execute(httpConnRunnable); 
                    } 
                    else if (mHighPool.getActiveCount() == HIGH_POOL_SIZE && mNormalPool.getActiveCount() < NORMAL_POOL_SIZE) { 
                        mNormalPool.execute(httpConnRunnable); 
                    } 
                    else { 
                        mHighPool.execute(httpConnRunnable); 
                    } 
                    break; 
         
                case HttpConnRunnable.NORMAL_LEVEL: 
                    //如果普通池满而低级池未满交由低级池处理 
                    //如果普通池未满则交给普通池处理，否则，交由普通池排队等候 
                    if (mNormalPool.getActiveCount() == NORMAL_POOL_SIZE && mLowPool.getActiveCount() < LOW_POOL_SIZE) { 
                        mLowPool.execute(httpConnRunnable); 
                    } 
                    else { 
                        mNormalPool.execute(httpConnRunnable); 
                    } 
                    break; 
         
                case HttpConnRunnable.LOW_LEVEL: 
                    mLowPool.execute(httpConnRunnable); 
                    break; 
            } 
        } 
        return super.onStartCommand(intent, flags, startId); 
    } 
 
    @Override 
    public void onDestroy() { 
        mHighPool.shutdownNow(); 
        mNormalPool.shutdownNow(); 
        mLowPool.shutdownNow(); 
 
         
        mNormalPool = null; 
        mLowPool = null; 
        super.onDestroy(); 
    } 
 
    @Override 
    public IBinder onBind(Intent intent) { 
        return null; 
    } 
 
    @Override 
    public void onSucceed(String result) { 
        Intent intent = new Intent(); 
        intent.setAction("com.ezstudio.connpool.HttpConnReceiver"); 
 
        // 要发送的内容 
        intent.putExtra("RESULT", result); 
 
        // 发送 一个无序广播 
        sendBroadcast(intent); 
    } 
 
    @Override 
    public void onFailed() { 
        // TODO Auto-generated method stub 
    } 
} 
``` 
 
最后的```Receiver```处理比较简单：

```java
public class HttpConnReceiver extends BroadcastReceiver { 
    private HttpConnListener mListener; 
     
    public void setHttpConnListener (HttpConnListener listener) { 
        mListener = listener; 
    } 
     
    @Override 
    public void onReceive(Context context, Intent intent) { 
        String action = intent.getAction(); 
        if (action.equals("com.ezstudio.connpool.HttpConnReceiver")) { 
            String result = intent.getStringExtra("RESULT"); 
            mListener.onSucceed(result); 
        } 
    } 
 
} 
```
 
ok,流程走完了，写个测试界面：

 ![](./../assets/img/2012-05-03-android_network_queue/1.png)
 
看结果非常满意：

```bash
05-03 16:42:20.225: I/HighPollingThread(2318): ----HighQuqeue is empty-----
05-03 16:42:20.225: I/NormalPollingThread(2318): ----NormalQuqeue is empty-----
05-03 16:42:20.233: I/LowPollingThread(2318): ----LowQuqeue is empty-----
05-03 16:42:20.233: I/HighPollingThread(2318): ----HighQuqeue is empty-----
05-03 16:42:20.233: I/pool-1-thread-1(2318): ----Start to connect:www.0.com, priority:0-----
05-03 16:42:22.374: I/NormalPollingThread(2318): ----NormalQuqeue is empty-----
05-03 16:42:22.374: I/pool-2-thread-1(2318): ----Start to connect:www.1.com, priority:1-----
05-03 16:42:22.780: I/LowPollingThread(2318): ----LowQuqeue is empty-----
05-03 16:42:22.780: I/pool-3-thread-1(2318): ----Start to connect:www.2.com, priority:2-----
05-03 16:42:23.428: I/NormalPollingThread(2318): ----NormalQuqeue is empty-----
05-03 16:42:23.428: I/pool-2-thread-2(2318): ----Start to connect:www.3.com, priority:1-----
05-03 16:42:23.835: I/HighPollingThread(2318): ----HighQuqeue is empty-----
05-03 16:42:23.835: I/pool-3-thread-2(2318): ----Start to connect:www.4.com, priority:0-----
05-03 16:42:24.171: I/NormalPollingThread(2318): ----NormalQuqeue is empty-----
05-03 16:42:24.171: I/pool-2-thread-3(2318): ----Start to connect:www.5.com, priority:1-----
05-03 16:42:24.507: I/LowPollingThread(2318): ----LowQuqeue is empty-----
05-03 16:42:24.764: I/NormalPollingThread(2318): ----NormalQuqeue is empty-----
05-03 16:42:25.030: I/HighPollingThread(2318): ----HighQuqeue is empty-----
05-03 16:42:25.335: I/NormalPollingThread(2318): ----NormalQuqeue is empty-----
05-03 16:42:25.647: I/LowPollingThread(2318): ----LowQuqeue is empty-----
05-03 16:42:25.936: I/NormalPollingThread(2318): ----NormalQuqeue is empty-----
05-03 16:42:26.163: I/NormalPollingThread(2318): ----NormalQuqeue is empty-----
05-03 16:42:26.389: I/NormalPollingThread(2318): ----NormalQuqeue is empty-----
05-03 16:42:26.694: I/LowPollingThread(2318): ----LowQuqeue is empty----- 
```
 
### 总结：

如果不算```Service```一共最多使用了3个线程池，6个线程，或许可以考虑将三个池合并为一个。但却也大量减少了单独发起请求时的线程创建和销毁的消耗。