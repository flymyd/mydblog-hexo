---
layout: android
title: Android BLE（低功耗蓝牙）在Android不同版本的适配问题.md
date: 2020-04-16 15:08:49
tags: [代码片段, 工作记录]
categories: [Android开发]
---
### 一、前言

截止到本文完成的日期为止（2020年04月16日），笔者对Android 5.0~Android 10的部分手机进行了适配测试。该文中所遇到的问题基本都出现在国产定制系统（EMUI、MIUI、ColorOS）上。开发环境为macOS+idea。

### 二、相关代码

+ 1、（基本）在AndroidManifest.xml中静态申请如下权限：

+ ~~~xml
  <uses-permission android:name="android.permission.BLUETOOTH" />
  <uses-permission android:name="android.permission.BLUETOOTH_ADMIN" />
  <uses-permission android:name="android.permission.ACCESS_FINE_LOCATION" />
  <uses-permission android:name="android.permission.ACCESS_COARSE_LOCATION" />
  <uses-permission android:name="android.permission.ACCESS_BACKGROUND_LOCATION" />
  <uses-feature
          android:name="android.hardware.bluetooth_le"
          android:required="true" />
  ~~~

  *BLE的扫描需要开启位置信息相关的权限，否则将会无法扫描到BLE设备。

  *在MIUI11（Android 10）中，需要另行打开位置服务（GPS），否则也不能扫描到BLE设备。部分其它国产ROM在Android 10也可能存在此问题，解决办法同上。判断标准是：BluetoothLeScanner被正确地实例化（不为null），但始终无法发现附近的BLE设备。

+ 2、（Android 6.0及以上）在onCreate中动态申请位置信息权限：

  ~~~java
  if (Build.VERSION.SDK_INT <= Build.VERSION_CODES.M) {
      Log.i(TAG, "sdk < Android M");
      if (ActivityCompat.checkSelfPermission(this,
          Manifest.permission.ACCESS_FINE_LOCATION) != PackageManager.PERMISSION_GRANTED
          || ActivityCompat.checkSelfPermission(this,
              Manifest.permission.ACCESS_COARSE_LOCATION) != PackageManager.PERMISSION_GRANTED) {
          String[] strings =
              {Manifest.permission.ACCESS_FINE_LOCATION, Manifest.permission.ACCESS_COARSE_LOCATION};
          ActivityCompat.requestPermissions(this, strings, 1);
      }
  } else {
      if (ActivityCompat.checkSelfPermission(this,
          Manifest.permission.ACCESS_FINE_LOCATION) != PackageManager.PERMISSION_GRANTED
          || ActivityCompat.checkSelfPermission(this,
              Manifest.permission.ACCESS_COARSE_LOCATION) != PackageManager.PERMISSION_GRANTED
          || ActivityCompat.checkSelfPermission(this,
              "android.permission.ACCESS_BACKGROUND_LOCATION") != PackageManager.PERMISSION_GRANTED) {
          String[] strings = {android.Manifest.permission.ACCESS_FINE_LOCATION,
              android.Manifest.permission.ACCESS_COARSE_LOCATION,
              "android.permission.ACCESS_BACKGROUND_LOCATION"};
          ActivityCompat.requestPermissions(this, strings, 2);
      }
  }
  ~~~

  检测是否打开位置服务：

  ~~~java
  public static final boolean isLocationEnable(Context context) {
      LocationManager locationManager = (LocationManager) context.getSystemService(Context.LOCATION_SERVICE);
      boolean networkProvider = locationManager.isProviderEnabled(LocationManager.NETWORK_PROVIDER);
      boolean gpsProvider = locationManager.isProviderEnabled(LocationManager.GPS_PROVIDER);
      if (networkProvider || gpsProvider) return true;
      return false;
  }
  ~~~

  如果未打开位置服务，让用户去打开：

  ~~~java
  private static final int REQUEST_CODE_LOCATION_SETTINGS = 2;
  private void setLocationService() {
      Intent locationIntent = new Intent(Settings.ACTION_LOCATION_SOURCE_SETTINGS);
      this.startActivityForResult(locationIntent, REQUEST_CODE_LOCATION_SETTINGS);
  }
  ~~~

  进入设置，让用户自己选择是否打开位置服务，然后获取选择结果：

  ~~~java
  @Override
  protected void onActivityResult(int requestCode, int resultCode, Intent data) {
      if (requestCode == REQUEST_CODE_LOCATION_SETTINGS) {
          if (isLocationEnable(this)) {
              //定位已打开的处理
          } else {
              //定位依然没有打开的处理
          }
      } else super.onActivityResult(requestCode, resultCode, data);
  } 
  ~~~

+ 3、（基本）检测是否支持BLE蓝牙及蓝牙开关状态

  ```java
  //是否支持
  public static boolean isSupportBle(Context context) {
        if (context == null || !context.getPackageManager().hasSystemFeature(PackageManager.FEATURE_BLUETOOTH_LE)) {
            return false;
        }
        BluetoothManager manager = (BluetoothManager) context.getSystemService(Context.BLUETOOTH_SERVICE);
        return manager.getAdapter() != null;
  }
  //是否开启
  public static boolean isBleEnable(Context context) {
        if (!isSupportBle(context)) {
            return false;
        }
        BluetoothManager manager = (BluetoothManager) context.getSystemService(Context.BLUETOOTH_SERVICE);
        return manager.getAdapter().isEnabled();
  }
  //开启蓝牙
  public static void enableBluetooth(Activity activity, int requestCode) {
        Intent intent = new Intent(BluetoothAdapter.ACTION_REQUEST_ENABLE);
        activity.startActivityForResult(intent, requestCode);
  }
  ```

+ 4、（基本&&Android 7.0+适配特别注意！）进行BLE设备扫描

  在Android 7.0及以后的系统中，BluetoothAdapter.startLeScan()方法被废弃，所以需要根据系统版本的不同来执行不同的扫描方法，否则在MIUI11及EMUI10的部分机型上会扫描不到BLE设备。

  ```java
      private void startScan() throws Exception {
          isScanning = true;
          handler.postDelayed(scanRunnable, TIME_OUT);
          BluetoothManager bluetoothManager = (BluetoothManager) getSystemService(Context.BLUETOOTH_SERVICE);
          if (Build.VERSION.SDK_INT <= Build.VERSION_CODES.M) {
              LogUtil.e("系统版本：","<7.0");
              bluetoothManager.getAdapter()
                      .startLeScan(mLeScanCallback);
          } else {//安卓7.0及以上的方案
              LogUtil.e("系统版本：",">=7.0");
              bleScanner = bluetoothManager.getAdapter().getBluetoothLeScanner();
              bleScanner.startScan(scanCallback);
          }
      }
  ```

  ~~~java
  //停止扫描
      private void stopScan() {
          isScanning = false;
          BluetoothManager bluetoothManager = (BluetoothManager) getSystemService(Context.BLUETOOTH_SERVICE);
          if (Build.VERSION.SDK_INT <= Build.VERSION_CODES.M) {
              //安卓6.0及以下版本BLE操作的代码
              bluetoothManager.getAdapter().stopLeScan(mLeScanCallback);
          } else
              //安卓7.0及以上版本BLE操作的代码
              bleScanner.stopScan(scanCallback);
      }
  ~~~

  ~~~java
  //扫描结果回调-Android 7.0-
      private BluetoothAdapter.LeScanCallback mLeScanCallback = new BluetoothAdapter.LeScanCallback() {
          @SuppressLint("HandlerLeak")
          @Override
          public void onLeScan(BluetoothDevice device, int rssi, byte[] scanRecord) {
              LogUtil.e(TAG, "开始搜索设备：" + mac);
              if (device.getAddress().equals(mac)) {
                  LogUtil.e(TAG, "搜索到设备 " + mac);
                  new Handler() {
                      @Override
                      public void handleMessage(Message msg) {
                          if (mBluetoothLeService != null) {
                              mBluetoothLeService.connect(mac);
                          }
                      }
                  }.sendEmptyMessageDelayed(3000, 200);
                  stopScan();
                  isScanning = false;
                  handler.removeCallbacks(scanRunnable);
              }
          }
      };
  
  //扫描结果回调-7.0+
      private ScanCallback scanCallback = new ScanCallback() {
          @SuppressLint("HandlerLeak")
          @RequiresApi(api = Build.VERSION_CODES.LOLLIPOP)
          @Override
          public void onScanResult(int callbackType, ScanResult result) {
              super.onScanResult(callbackType, result);
              BluetoothDevice device = result.getDevice();
              String macAddr = device.getAddress();
              LogUtil.e("发现设备：", macAddr);
              if (macAddr.equals(mac)) {
                  LogUtil.e(TAG, "搜索到匹配的蓝牙设备(6.0+): " + mac);
                  new Handler() {
                      @Override
                      public void handleMessage(Message msg) {
                          if (mBluetoothLeService != null) {
                              mBluetoothLeService.connect(mac);
                          }
                      }
                  }.sendEmptyMessageDelayed(3000, 200);
                  stopScan();
                  isScanning = false;
                  handler.removeCallbacks(scanRunnable);
              }
              Log.i(TAG, macAddr);
          }
      };
  ~~~

  ~~~java
  //重连计时器
  private Runnable scanRunnable = new Runnable() {
          @Override
          public void run() {
              connTimeOutTimes++;
              LogUtil.e(TAG, "重连第" + connTimeOutTimes + "次");
              if (connTimeOutTimes < RECONNECT_TIMES) {
                  handler.postDelayed(scanRunnable, TIME_OUT);
              } else {
                  stopScan();
                  connTimeOutTimes = 0;
                  if (connectFunction != null) {
                      LogUtil.e(TAG, "连接超时：" + mBluetoothLeService.getConnectState());
                      connectFunction.onCallBack("{\"code\":-1,\"msg\":\"连接超时\",\"data\":\"false\"}");
                      connectFunction = null;
                  }
                  if (cardConnectFunction != null) {
                      cardConnectFunction.onCallBack("{\"code\":-1,\"msg\":\"连接超时\",\"data\":\"false\"}");
                      cardConnectFunction = null;
                  }
                  mBluetoothLeService.setConnectionState(0);
                  mBluetoothLeService.release();
                  bindStatus = 0;
              }
          }
      };
  ~~~

### 三、踩过坑的机型/ROM

* 1、小米/红米

  MIUI11：必须打开位置服务才可搜索到BLE设备；

  MIUI11及更早：

  初始化BlueToothGatt时务必将connectGatt的autoConnect设为false，否则会被系统杀掉；

  ~~~java
  package android.bluetooth;
  public final class BluetoothDevice implements Parcelable {
  ...
  public BluetoothGatt connectGatt(Context context, boolean autoConnect,
              BluetoothGattCallback callback) {
          return (connectGatt(context, autoConnect, callback, TRANSPORT_AUTO));
      }
  }
  ~~~

  必须针对Android M及以上的MIUI配置调用BluetoothLeScanner.startScan()方法来搜索BLE设备，而非被废弃的BluetoothAdapter.startLeScan()，否则将无法搜索到设备，日志如下：

  ~~~
  D/BluetoothAdapter: startLeScan(): null					**注意，此处为null
  D/BluetoothAdapter: isLeEnabled(): ON
  D/BluetoothLeScanner: onScannerRegistered() - status=0 scannerId=6 mScannerId=0
  ~~~

* 2、华为/荣耀

  EMUI10（仅Mate 30/P30系列）：必须打开位置服务才可搜索到BLE设备

* 3、部分国产ROM（包括上述两类）：

  部分机型需打开位置服务才可搜索到BLE设备；

  部分机型需动态申请位置权限才可搜索到BLE设备；

  部分机型需将connectGatt的autoConnect设为false（MIUI、ColorOS）；

  部分机型需针对Android M及以上的系统版本配置调用BluetoothLeScanner.startScan()方法来搜索BLE设备；

### 四、附录：连接日志（MIUI）

~~~
E/系统版本：: >=6.0
D/BluetoothAdapter: isLeEnabled(): ON
D/BluetoothLeScanner: onScannerRegistered() - status=0 scannerId=8 mScannerId=0
E/发现设备：: xx:xx:xx:xx:xx:xx
E/WebViewActivity: 搜索到匹配的蓝牙设备(6.0+): xx:xx:xx:xx:xx:xx
D/BluetoothAdapter: isLeEnabled(): ON
I/WebViewActivity: xx:xx:xx:xx:xx:xx
E/BlueToothService: xx:xx:xx:xx:xx:xx
D/BluetoothGatt: onClientConnectionState() - status=0 clientIf=7 device=xx:xx:xx:xx:xx:xx
D/BluetoothGatt: discoverServices() - device: xx:xx:xx:xx:xx:xx
E/WebViewActivity: 连接成功：2
D/BluetoothGatt: onConnectionUpdated() - Device=xx:xx:xx:xx:xx:xx interval=6 latency=0 timeout=500 status=0
D/BluetoothGatt: onSearchComplete() = Device=xx:xx:xx:xx:xx:xx Status=0
D/BluetoothGatt: setCharacteristicNotification() - uuid: 0000fff4-xxxx-xxxx-xxxx-00xxxx9bxxxx enable: true
D/BluetoothGatt: onConnectionUpdated() - Device=xx:xx:xx:xx:xx:xx interval=36 latency=0 timeout=500 status=0
~~~

### 五、联系方式
flymyd@foxmail.com
