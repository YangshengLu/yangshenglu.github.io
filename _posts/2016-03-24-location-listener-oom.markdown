---
layout: "post"
title:  "NmeaListener、LocationListener与Looper错误使用引起的内存泄漏"
date: "2016-03-24"
header-img: "img/post-bg-03.jpg"
author:  "MrYang"
categories: "Android"
---
### 内存泄漏根源分析
这个内存泄漏引起的根源，是在没有移除NmeaListener、LocationListener的情况下，使调用addNmeaListener的线程退出。首现来看看Android SDK中addNmeaListener是怎么实现的

``` Java
public boolean addNmeaListener(GpsStatus.NmeaListener listener) {
    boolean result;

    if (mNmeaListeners.get(listener) != null) {
        // listener is already registered
        return true;
    }
    try {
        GpsStatusListenerTransport transport = new GpsStatusListenerTransport(listener);
        result = mService.addGpsStatusListener(transport, mContext.getPackageName());
        if (result) {
            mNmeaListeners.put(listener, transport);
        }
    } catch (RemoteException e) {
        Log.e(TAG, "RemoteException in registerGpsStatusListener: ", e);
        result = false;
    }

    return result;
}
```

其中`GpsStatusListenerTransport`继承`IGpsStatusListener.Stub`，是IGpsStatusListener AIDL的实现，意味着GpsStatusListenerTransport的各个回调函数是在Binder Thread中被回调的，而GpsStatusListenerTransport也创建了一个handler，使用的Looper是myLooper，用来确保NmeaListener的回调与addNmeaListener在同一个线程中执行，代码如下

``` Java
@Override
public void onNmeaReceived(long timestamp, String nmea) {
    if (mNmeaListener != null) {
        synchronized (mNmeaBuffer) {
            mNmeaBuffer.add(new Nmea(timestamp, nmea));
        }
        Message msg = Message.obtain();
        msg.what = NMEA_RECEIVED;
        // remove any NMEA_RECEIVED messages already in the queue
        mGpsHandler.removeMessages(NMEA_RECEIVED);
        mGpsHandler.sendMessage(msg);
    }
}

private final Handler mGpsHandler = new Handler() {
    @Override
    public void handleMessage(Message msg) {
        if (msg.what == NMEA_RECEIVED) {
            synchronized (mNmeaBuffer) {
                int length = mNmeaBuffer.size();
                for (int i = 0; i < length; i++) {
                    Nmea nmea = mNmeaBuffer.get(i);
                    mNmeaListener.onNmeaReceived(nmea.mTimestamp, nmea.mNmea);
                }
                mNmeaBuffer.clear();
            }
        } else {
            // synchronize on mGpsStatus to ensure the data is copied atomically.
            synchronized(mGpsStatus) {
                mListener.onGpsStatusChanged(msg.what);
            }
        }
    }
};
```

通过上面的知识，我们可以确定，如果自己创建了线程，并且调用过`Looper.prepare()`然后再`addNmeaListener`，却最终没有调用`Looper.loop()`，或者是在`Looper.loop()`退出之后，没有`removeNmeaListener`，都会引起mNmeaBuffer的堆积，最终造成OOM

### 内存泄漏代码

``` Java
new Thread() {
    @Override
    public void run() {
        try {
            Looper.prepare();
            locationManager.addNmeaListener(new GpsStatus.NmeaListener() {
                @Override
                public void onNmeaReceived(long timestamp, String nmea) {
                    // do onNmeaReceived
                }
            });
            locationManager.requestLocationUpdates("gps", 10, 0, new LocationListener() {
                @Override
                public void onLocationChanged(Location location) {

                }

                @Override
                public void onStatusChanged(String provider, int status, Bundle extras) {

                }

                @Override
                public void onProviderEnabled(String provider) {

                }

                @Override
                public void onProviderDisabled(String provider) {

                }
            });
            // doSomethingMayThrow();
            // Looper.loop();
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}.start();
```

上面的代码中，如果`doSomethingMayThrow()`抛出一个异常，会造成`Looper.loop()`不会被执行，并且后续也没有`removeNmeaListener`。感兴趣的朋友可以随便写个小程序运行一下上面那段代码，可以发现java堆的大小不断的上涨，GC也无法回收，一段时间之后OOM，如果期间dump下app的内存，可以看到mNmeaBuffer占据了绝大多数的内存，我就插一张图吧，<strong><font color="red">下面的图是线上OOM的dump文件分析出来的，不是hello world！！！</font></strong>

-----
![MAT_OOM_1](http://mryangyang.github.io/assets/nmea_listener_oom_1.png)

-----
![MAT_OOM_2](http://mryangyang.github.io/assets/nmea_listener_oom_2.png)

### 修复内存泄漏代码示例

``` Java
new Thread() {
    @Override
    public void run() {
        final GpsStatus.NmeaListener nmeaListener = new GpsStatus.NmeaListener() {
            @Override
            public void onNmeaReceived(long timestamp, String nmea) {
                // do onNmeaReceived
            }
        };

        final LocationListener locationListener = new LocationListener() {
            @Override
            public void onLocationChanged(Location location) {
                // do onLocationChanged
            }

            @Override
            public void onStatusChanged(String provider, int status, Bundle extras) {
                // do onStatusChanged
            }

            @Override
            public void onProviderEnabled(String provider) {
                // do onProviderEnabled
            }

            @Override
            public void onProviderDisabled(String provider) {
                // do onProviderDisabled
            }
        };
        try {
            Looper.prepare();
            locationManager.addNmeaListener(nmeaListener);
            locationManager.requestLocationUpdates("gps", 10, 0 , locationListener);
            doSomethingMayThrow();
            Looper.loop();
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            locationManager.removeNmeaListener(nmeaListener);
            locationManager.removeUpdates(locationListener);
            Looper myLooper = Looper.myLooper();
            if (myLooper != null) {
                myLooper.quit();
            }
        }
    }
}.start();
```

修复的方法总结起来就是，如果`addNmeaListener`等函数是在非主线程当中调用的，那么只要该线程跳出`Looper.loop()`，一定要确保`removeNmeaListener`、`removeUpdates`和`myLooper.quit()`被执行
