---
title: Android计步代码
categories: Android
toc: true
abbrlink: f7aeb3e0
date: 2017-05-11 13:58:04
tags:
---


##前言
> Android端计步代码，经测试较为精准。使用了计步传感器（耗能低，准确度高）和加速度传感器（适用度广，基本上各个手机上都有这个传感器），**代码中牵涉到数据的操作，数据库我使用的是GreenDao，相关代码就不贴了，因为配置一下就好了，毕竟重在计步嘛**

主要使用到的技术：
- 传感器的使用
- 广播监听
- 计时类
- 常驻通知
- 使用Messenger进行跨进程通信
- 计步算法
<!-- more -->
效果图

![计步变化](https://pic3.superbed.cn/item/5cfbacb0451253d178d93e9b.gif)

实物不好拍摄，就直接录了个GIF，实际用着还是不错的。

##废话不多说，直接上代码
```
public class StepService extends Service implements SensorEventListener {

    //TAG
    private static String TAG = "StepService";
    //存储间隔
    private static int duration = 3000;
    //当前日期
    private static String CURRENT_DATE = "";
    //传感器
    private SensorManager sensorManager;

    //广播--监听手机状态变化
    private BroadcastReceiver mReceiver;
    //===============================================倒计时
    private TimeCount time;
    //===============================================数据库操作
    private StepsDao stepsDao = DBUtil.getStepsDao();
    //当前步数
    private int CURRENT_STEPS;
    //期望步数
    private float EXPECT_STEPS;
    //计步传感器类型 0-counter 1-detector
    private static  int stepSensor = -1;
    //是否记录
    private boolean hasRecord = false;
    //已经走过的步数
    private int hasStepCount = 0;
    //以前走过的步数
    private int previousStepCount = 0;
    //===============================================采用加速度传感器所要用到的变量

    public static float SENSITIVITY = 10; // SENSITIVITY灵敏度
    private float mLastValues[] = new float[3 * 2];
    private float mScale[] = new float[2];
    private float mYOffset;
    private static long end = 0;
    private static long start = 0;
    /**
     * 最后加速度方向
     */
    private float mLastDirections[] = new float[3 * 2];
    private float mLastExtremes[][] = { new float[3 * 2], new float[3 * 2] };
    private float mLastDiff[] = new float[3 * 2];
    private int mLastMatch = -1;

    //===============================================messenger
    //跨进程通信--使用Messenger方式
    private Messenger messengerFromService = new Messenger(new MessengerHandler());
    private  Messenger messengerFromClient;


    private class MessengerHandler extends Handler{
        @Override
        public void handleMessage(Message msg) {
            super.handleMessage(msg);
            switch (msg.what){
                case Constants.MSG_FROM_CLIENT:
                    messengerFromClient = msg.replyTo;
                    StepService.this.sendMessage();
                    break;
                default:
                    break;
            }
        }
    }
    //===============================================通知相关
    private static final int NOTIFI_ID = 100;
    //格式管理
    private DecimalFormat df = new DecimalFormat("#0.0");


    @Override
    public void onCreate() {
        super.onCreate();
        EXPECT_STEPS = 8000f;
        //初始化广播
        initBroadcastReceiver();
        new Thread(new Runnable() {
            @Override
            public void run() {
                //获取传感器类型
                startStepDetector();
            }
        }).start();
        startTimeCount();
        initTodayData();
    }


    /**
     * 取得今日日期
     * @return
     */
    private String getTodayDate(){
        Date date = new Date(System.currentTimeMillis());
        SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd");
        return sdf.format(date);
    }

    /**
     * 初始化当天的步数
     */
    private void initTodayData() {
        CURRENT_DATE = getTodayDate();
        //===============================================在这里进行数据新的一列的存储
        //通过日期匹配，当数据中有今日步数的行，那么将步数值进行读取，如果没有那么久新增一行，并将CURRENT_STEP存储进去
        QueryBuilder qb = stepsDao.queryBuilder();
        qb.where(StepsDao.Properties.Date.eq(getTodayDate()));
        Steps steps = (Steps) qb.unique();
        if(steps!=null){
            CURRENT_STEPS = steps.getUStep();
        }else{
            //增加一行
            Steps stepsAdd = new Steps();
            stepsAdd.setDate(CURRENT_DATE);
            stepsAdd.setUStep(0);
            stepsAdd.setHasUpLoad(false);
            stepsDao.insert(stepsAdd);
        }
    }
    /**
     * 获取当前步数占有率
     */
    private String getCurrentOccupancy(){
        //默认8000，完善时在Service启动的时候进行复制
        return df.format((float)CURRENT_STEPS/EXPECT_STEPS*100);
    }

    /**
     * 获取传感器实例
     */
    private void startStepDetector() {
        if(sensorManager!=null){
            sensorManager = null;
        }
        sensorManager = (SensorManager) this.getSystemService(SENSOR_SERVICE);
        addCountStepListener();

    }
    Sensor countSensor ;
    Sensor detectorSensor ;
    Sensor accelerateSensor;
    private void addCountStepListener() {
         countSensor = sensorManager.getDefaultSensor(Sensor.TYPE_STEP_COUNTER);
         detectorSensor = sensorManager.getDefaultSensor(Sensor.TYPE_STEP_DETECTOR);
        //利用加速度传感器
        accelerateSensor = sensorManager.getDefaultSensor(Sensor.TYPE_ACCELEROMETER);

        if(countSensor!=null){
            //选择计步传感器
            stepSensor = 0;
            Log.i(TAG,"计步传感器");
            sensorManager.registerListener(StepService.this,countSensor,SENSOR_DELAY_NORMAL);
        }else if(detectorSensor!=null){
            //步数检测器
            stepSensor = 1;
            Log.i(TAG,"步数监测器");
            sensorManager.registerListener(StepService.this,detectorSensor,SENSOR_DELAY_FASTEST);
        }else if(accelerateSensor != null){
            stepSensor = 2;
            int h = 480;
            mYOffset = h * 0.5f;
            mScale[0] = -(h * 0.5f * (1.0f / (SensorManager.STANDARD_GRAVITY * 2)));
            mScale[1] = -(h * 0.5f * (1.0f / (SensorManager.MAGNETIC_FIELD_EARTH_MAX)));
            Log.i(TAG,"加速度传感器");
            sensorManager.registerListener(StepService.this,accelerateSensor,SENSOR_DELAY_FASTEST);
        }

    }

    /**
     * 传感器回调
     * @param event
     */
    @Override
    public void onSensorChanged(SensorEvent event) {
        if(stepSensor == 0){
            int tempStep = (int) event.values[0];
            if(!hasRecord){
                hasRecord = true;
                hasStepCount = tempStep;
            }else{
                int thisStepCount = tempStep -hasStepCount;
                CURRENT_STEPS+=(thisStepCount-previousStepCount);
                previousStepCount = thisStepCount;
            }
            sendMessage();
            setNotification();
        }else if(stepSensor == 1){
            if(event.values[0] == 1.0){
                hasRecord = true;
                CURRENT_STEPS++;
                sendMessage();
                setNotification();
            }
        }else if(stepSensor == 2){
            hasRecord = true;
            synchronized (this) {
                float vSum = 0;
                for (int i = 0; i < 3; i++) {
                    final float v = mYOffset + event.values[i] * mScale[1];
                    vSum += v;
                }
                int k = 0;
                float v = vSum / 3;

                float direction = (v > mLastValues[k] ? 1
                        : (v < mLastValues[k] ? -1 : 0));
                if (direction == -mLastDirections[k]) {
                    // Direction changed
                    int extType = (direction > 0 ? 0 : 1); // minumum or
                    // maximum?
                    mLastExtremes[extType][k] = mLastValues[k];
                    float diff = Math.abs(mLastExtremes[extType][k]
                            - mLastExtremes[1 - extType][k]);

                    if (diff > SENSITIVITY) {
                        boolean isAlmostAsLargeAsPrevious = diff > (mLastDiff[k] * 2 / 3);
                        boolean isPreviousLargeEnough = mLastDiff[k] > (diff / 3);
                        boolean isNotContra = (mLastMatch != 1 - extType);

                        if (isAlmostAsLargeAsPrevious && isPreviousLargeEnough
                                && isNotContra) {
                            end = System.currentTimeMillis();
                            if (end - start > 500) {// 此时判断为走了一步

                                CURRENT_STEPS++;
                               sendMessage();
                               setNotification();
                                mLastMatch = extType;
                                start = end;
                            }
                        } else {
                            mLastMatch = -1;
                        }
                    }
                    mLastDiff[k] = diff;
                }
                mLastDirections[k] = direction;
                mLastValues[k] = v;
            }
            }
        }


    @Override
    public void onAccuracyChanged(Sensor sensor, int accuracy) {

    }

    private void sendMessage(){
        Message msg = Message.obtain(null,Constants.MSG_FROM_SERVICE);
        Bundle bundle = new Bundle();
        bundle.putInt("currentSteps",CURRENT_STEPS);
        msg.setData(bundle);
        try {
            if(hasRecord&&messengerFromClient!=null) {
                messengerFromClient.send(msg);
            }
        } catch (RemoteException e) {
            e.printStackTrace();
        }
    }


    /**
     * 注册广播
     */
    private void initBroadcastReceiver() {
        final IntentFilter filter = new IntentFilter();
        // 屏幕灭屏广播
        filter.addAction(Intent.ACTION_SCREEN_OFF);
        //关机广播
        filter.addAction(Intent.ACTION_SHUTDOWN);
        // 屏幕亮屏广播
        filter.addAction(Intent.ACTION_SCREEN_ON);
        // 屏幕解锁广播
        filter.addAction(Intent.ACTION_USER_PRESENT);
        // 当长按电源键弹出“关机”对话或者锁屏时系统会发出这个广播
        // 所以监听这个广播，当收到时就隐藏自己的对话，如点击pad右下角部分弹出的对话框
        filter.addAction(Intent.ACTION_CLOSE_SYSTEM_DIALOGS);
        //监听日期变化
        filter.addAction(Intent.ACTION_DATE_CHANGED);
        filter.addAction(Intent.ACTION_TIME_CHANGED);
        filter.addAction(Intent.ACTION_TIME_TICK);

        mReceiver = new BroadcastReceiver() {
            @Override
            public void onReceive(Context context, Intent intent) {
                String action = intent.getAction();
                if(Intent.ACTION_SCREEN_ON.equals(action)){
                    Log.i(TAG,"ScreenOn");
                } else if (Intent.ACTION_SCREEN_OFF.equals(action)) {
                    Log.i(TAG,"ScreenOff");
                    //60s
                    duration = 6000;
                }else if (Intent.ACTION_USER_PRESENT.equals(action)){
                    Log.i(TAG,"screen unlock");
                    save();
                    //30s
                    duration = 3000;
                }else if(Intent.ACTION_SHUTDOWN.equals(action)){
                    Log.i(TAG,"shutdown");
                    save();
                }else if(Intent.ACTION_DATE_CHANGED.equals(action)){
                    save();
                    isNewDay();
                }else if(Intent.ACTION_TIME_CHANGED.equals(action)){
                    save();
                    isNewDay();
                }else if(Intent.ACTION_TIME_TICK.equals(action)){
                    save();
//                    isNewDay();
                }else if("ANewacount".equals(action)){

                    //当时一个新用户登录的时候
                    //清除数据
                    Log.i(TAG,"收到新账户广播");
                    initTodayData();
                }
            }
        };
        registerReceiver(mReceiver,filter);
    }

    /**
     * 0点时初始化数据
     */
    private void isNewDay() {
        String time = "00:00";
        if(time.equals(new SimpleDateFormat("HH:mm").format(new Date()))||(!CURRENT_DATE.equals(getTodayDate()))){
            initTodayData();
        }
    }

    /**
     * 存储到数据中去
     */
    private void save() {
        int tempStep = CURRENT_STEPS;
        QueryBuilder qb = stepsDao.queryBuilder();
        qb.where(StepsDao.Properties.Date.eq(getTodayDate()));
        Steps steps = (Steps) qb.unique();
        //不为空时，说明还未到12点，我们进行更新就行，为空说明为最后一次存储
        if(steps!=null){
            steps.setUStep(tempStep);
            steps.setDate(CURRENT_DATE);
            stepsDao.update(steps);
        }else{
            steps = new Steps();
            steps.setUStep(tempStep);
            steps.setDate(CURRENT_DATE);
            stepsDao.update(steps);
        }

    }

    @Nullable
    @Override
    public IBinder onBind(Intent intent) {
        return messengerFromService.getBinder();
    }

    @Override
    public int onStartCommand(Intent intent, int flags, int startId) {
        setNotification();
        return START_STICKY;
        //如果被系统kill掉，系统会自动将kill时的状态保留为开始状态，之后进行重连
    }

    private void setNotification() {
        PendingIntent pd = PendingIntent.getActivity(this,0,new Intent(this, MainViewActivity.class),0);
        //在这里进行前台服务
        Notification.Builder builder = new Notification.Builder(this)
                .setOngoing(true)
                .setSmallIcon(R.drawable.poplog)
                .setLargeIcon(BitmapFactory.decodeResource(this.getResources(),R.drawable.poplog))
                .setContentTitle("当前步数"+CURRENT_STEPS)
                .setContentText("今日完成百分比"+getCurrentOccupancy()+"%")
                .setWhen(System.currentTimeMillis())
                .setContentIntent(pd);
        startForeground(NOTIFI_ID,builder.build());
    }

    @Override
    public void onDestroy() {
        super.onDestroy();
        //取消前台进程
        stopForeground(true);
        unregisterReceiver(mReceiver);
        //注销各传感器
        if(countSensor!= null || detectorSensor != null || accelerateSensor != null){
            sensorManager.unregisterListener(StepService.this);
        }


    }


    /**
     * 开始计时
     */
    private void startTimeCount() {
        time = new TimeCount(duration,1000);
        time.start();
    }

    private class TimeCount extends CountDownTimer{


        /**
         * @param millisInFuture    The number of millis in the future from the call
         *                          to {@link #start()} until the countdown is done and {@link #onFinish()}
         *                          is called.
         * @param countDownInterval The interval along the way to receive
         *                          {@link #onTick(long)} callbacks.
         */
        public TimeCount(long millisInFuture, long countDownInterval) {
            super(millisInFuture, countDownInterval);
        }

        @Override
        public void onTick(long millisUntilFinished) {

        }

        @Override
        public void onFinish() {
            //如果计时器正常结束，则开始计步
            time.cancel();
            save();
            startTimeCount();
        }
    }
}
```
##代码分析
#####onCreate
```

    @Override
    public void onCreate() {
        super.onCreate();
        EXPECT_STEPS = 8000f;
        //初始化广播
        initBroadcastReceiver();
        new Thread(new Runnable() {
            @Override
            public void run() {
                //获取传感器类型
                startStepDetector();
            }
        }).start();
        startTimeCount();
        initTodayData();
    }
```
代码上写得很简单明了，即：**初始广播，获取传感器类型，开始计时，以及初始化日期**

####初始化广播
```
 /**
     * 注册广播
     */
    private void initBroadcastReceiver() {
        final IntentFilter filter = new IntentFilter();
        // 屏幕灭屏广播
        filter.addAction(Intent.ACTION_SCREEN_OFF);
        //关机广播
        filter.addAction(Intent.ACTION_SHUTDOWN);
        // 屏幕亮屏广播
        filter.addAction(Intent.ACTION_SCREEN_ON);
        // 屏幕解锁广播
        filter.addAction(Intent.ACTION_USER_PRESENT);
        // 当长按电源键弹出“关机”对话或者锁屏时系统会发出这个广播
        // 所以监听这个广播，当收到时就隐藏自己的对话，如点击pad右下角部分弹出的对话框
        filter.addAction(Intent.ACTION_CLOSE_SYSTEM_DIALOGS);
        //监听日期变化
        filter.addAction(Intent.ACTION_DATE_CHANGED);
        filter.addAction(Intent.ACTION_TIME_CHANGED);
        filter.addAction(Intent.ACTION_TIME_TICK);

        mReceiver = new BroadcastReceiver() {
            @Override
            public void onReceive(Context context, Intent intent) {
                String action = intent.getAction();
                if(Intent.ACTION_SCREEN_ON.equals(action)){
                    Log.i(TAG,"ScreenOn");
                } else if (Intent.ACTION_SCREEN_OFF.equals(action)) {
                    Log.i(TAG,"ScreenOff");
                    //60s
                    duration = 6000;
                }else if (Intent.ACTION_USER_PRESENT.equals(action)){
                    Log.i(TAG,"screen unlock");
                    save();
                    //30s
                    duration = 3000;
                }else if(Intent.ACTION_SHUTDOWN.equals(action)){
                    Log.i(TAG,"shutdown");
                    save();
                }else if(Intent.ACTION_DATE_CHANGED.equals(action)){
                    save();
                    isNewDay();
                }else if(Intent.ACTION_TIME_CHANGED.equals(action)){
                    save();
                    isNewDay();
                }else if(Intent.ACTION_TIME_TICK.equals(action)){
                    save();
//                    isNewDay();
                }else if("ANewacount".equals(action)){

                    //当时一个新用户登录的时候
                    //清除数据
                    Log.i(TAG,"收到新账户广播");
                    initTodayData();
                }
            }
        };
        registerReceiver(mReceiver,filter);
    }
```
在这段代码里，主要是**监听用户手机状态的改变**，根据状态的改变进行不同的操作，如“用户关机时，将当前步数进行保存”、“当到达凌晨0点时需要进行新的一天的数据库更新”等等。**别忘了注销Receiver**哦

####获取传感器
```
 /**
     * 获取传感器实例
     */
    private void startStepDetector() {
        if(sensorManager!=null){
            sensorManager = null;
        }
        sensorManager = (SensorManager) this.getSystemService(SENSOR_SERVICE);
        addCountStepListener();

    }
    Sensor countSensor ;
    Sensor detectorSensor ;
    Sensor accelerateSensor;
    private void addCountStepListener() {
         countSensor = sensorManager.getDefaultSensor(Sensor.TYPE_STEP_COUNTER);
         detectorSensor = sensorManager.getDefaultSensor(Sensor.TYPE_STEP_DETECTOR);
        //利用加速度传感器
        accelerateSensor = sensorManager.getDefaultSensor(Sensor.TYPE_ACCELEROMETER);

        if(countSensor!=null){
            //选择计步传感器
            stepSensor = 0;
            Log.i(TAG,"计步传感器");
            sensorManager.registerListener(StepService.this,countSensor,SENSOR_DELAY_NORMAL);
        }else if(detectorSensor!=null){
            //步数检测器
            stepSensor = 1;
            Log.i(TAG,"步数监测器");
            sensorManager.registerListener(StepService.this,detectorSensor,SENSOR_DELAY_FASTEST);
        }else if(accelerateSensor != null){
            stepSensor = 2;
            int h = 480;
            mYOffset = h * 0.5f;
            mScale[0] = -(h * 0.5f * (1.0f / (SensorManager.STANDARD_GRAVITY * 2)));
            mScale[1] = -(h * 0.5f * (1.0f / (SensorManager.MAGNETIC_FIELD_EARTH_MAX)));
            Log.i(TAG,"加速度传感器");
            sensorManager.registerListener(StepService.this,accelerateSensor,SENSOR_DELAY_FASTEST);
        }

    }

```
这个没什么好说的，就是获取各个传感器对象，首先选择的是计步传感器，如果没有计步传感器那么久检查是否有加速度传感器，同样的别忘了注销传感器对象。

####计步的核心代码
```
  /**
     * 传感器回调
     * @param event
     */
    @Override
    public void onSensorChanged(SensorEvent event) {
        if(stepSensor == 0){
            int tempStep = (int) event.values[0];
            if(!hasRecord){
                hasRecord = true;
                hasStepCount = tempStep;
            }else{
                int thisStepCount = tempStep -hasStepCount;
                CURRENT_STEPS+=(thisStepCount-previousStepCount);
                previousStepCount = thisStepCount;
            }
            sendMessage();
            setNotification();
        }else if(stepSensor == 1){
            if(event.values[0] == 1.0){
                hasRecord = true;
                CURRENT_STEPS++;
                sendMessage();
                setNotification();
            }
        }else if(stepSensor == 2){
            hasRecord = true;
            synchronized (this) {
                float vSum = 0;
                for (int i = 0; i < 3; i++) {
                    final float v = mYOffset + event.values[i] * mScale[1];
                    vSum += v;
                }
                int k = 0;
                float v = vSum / 3;

                float direction = (v > mLastValues[k] ? 1
                        : (v < mLastValues[k] ? -1 : 0));
                if (direction == -mLastDirections[k]) {
                    // Direction changed
                    int extType = (direction > 0 ? 0 : 1); // minumum or
                    // maximum?
                    mLastExtremes[extType][k] = mLastValues[k];
                    float diff = Math.abs(mLastExtremes[extType][k]
                            - mLastExtremes[1 - extType][k]);

                    if (diff > SENSITIVITY) {
                        boolean isAlmostAsLargeAsPrevious = diff > (mLastDiff[k] * 2 / 3);
                        boolean isPreviousLargeEnough = mLastDiff[k] > (diff / 3);
                        boolean isNotContra = (mLastMatch != 1 - extType);

                        if (isAlmostAsLargeAsPrevious && isPreviousLargeEnough
                                && isNotContra) {
                            end = System.currentTimeMillis();
                            if (end - start > 500) {// 此时判断为走了一步

                                CURRENT_STEPS++;
                               sendMessage();
                              setNotification();
                                mLastMatch = extType;
                                start = end;
                            }
                        } else {
                            mLastMatch = -1;
                        }
                    }
                    mLastDiff[k] = diff;
                }
                mLastDirections[k] = direction;
                mLastValues[k] = v;
            }
            }
        }


```

- 计步传感器
首先从传感器中拿到tempStep，它代表着从计步以来的总步数，所以需要记录上次走的步数，用这次步数减去上次走的步数得到的差值，再叠加，就可以得到当前走的总步数了。
- 加速度传感器
这部分的算法似乎是某个google的大神写的，我也没看懂
- 另外
```sendMessage```和```setNotification```分别为更新UI界面和更新通知栏。

##写在最后
> 因为写这个功能的时候参看了不少代码，而且时间较为久了，参看了不少博文，忘了具体的博主了，这里就感谢所有博主了。ok，上课去了。


