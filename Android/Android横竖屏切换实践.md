# Android横竖屏切换实践

## 1、前言

前段时间，PM提了一个需求，在APP内直播观众和回放页面，需要增加一个横竖屏切换的按钮，和当前众多视频APP的横竖屏切换行为保持一致。

当拿到这个需求的时候，觉得是easy模式，无非就是`setRequestedOrientation(横屏/竖屏)`就ok了。但交互评审和查看B站、虎牙、钉钉等APP之后，并不是自己想得那么easy。因为忽略了一个很重要的设置——屏幕自动旋转。

## 2、需求分析

先将需求分析清楚，其实从系统锁定（下拉菜单里面是否打开了自动旋转）的角度分为两种情况：

- 系统关闭自动旋转。
- 系统打开自动旋转。

> 这里先不要考虑Activity的模式是`ActivityInfo.SCREEN_ORIENTATION_SENSOR`, 即使是，也是一样处理，这点后面会讲。

对于这两种情况，我们来看看延生出来的情况

![](https://raw.githubusercontent.com/Numen-fan/BlogPicRepo/main/img/20230213193249.png)

【说明】

1. 对于屏幕自动旋转关闭的情况下，情况比较简单，用户手动切换了横竖屏后，保持不动即可。
2. 如果屏幕自动旋转打开的情况，用户手动切换了横竖屏，此时需要增加一个策略，如果用户旋转屏幕到对应的方向，需要恢复自动旋转。
   1. 举个栗子：在竖屏观看状态下，用户点击切换，此时屏幕切换到横屏，手机仍然是竖屏状态。用户旋转90度到手机也是横屏时，随后用户再竖直回手机，屏幕能够自动旋转回竖屏。（读起来有点饶）

## 3、功能实现

经过上面的分析，我们可以得出两个需要去解决的关键点

【关键点1】监听系统自动旋转设置。

【关键点2】监听用户旋转到用户设置的方向。

### 3.1 监听系统自动旋转设置

首先，我们要知道如何读取系统设置的有关屏幕自动旋转的值

```java
// 0 off / 1 on
Settings.System.getInt(getContentResolver(), Settings.System.ACCELEROMETER_ROTATION）
```

我们可以封一个方法，表示系统自动旋转是否打开，可以考虑放到某个Util工具类中。

```java
private boolean canActivityAutoRotate() {
        try {
            int value = Settings.System.getInt(getContentResolver(), Settings.System.ACCELEROMETER_ROTATION);
            return value == 1;
        } catch (Settings.SettingNotFoundException e) {
            LogUtils.INSTANCE.error(TAG, "canActivityAutoRotate error", e);
        }
        return false;
    }
```

我的知道了如何获取设置值，那么什么时候去读取呢，当然是系统监听到设置的值发生了变化。

可以使用内容观察者`ContentObserver`来监听设置值的变化。我们新建一个类，继承自`ContentObserver`, 并在内部定义一个接口，将观测结果回调。这样的封装，可以提供多处使用。

```java
/**
 * Created by Numen_fan on 2023/2/11
 * Desc: 监听系统设置中是否打开自动选择，部分手机厂商叫方向锁定
 */
public class ScreenRotationSettingObserver extends ContentObserver {

    private static final String TAG = "RotationObserver";
    final ContentResolver mResolver;

    private ScreenRotationSettingListener listener;

    public ScreenRotationSettingObserver(Handler handler, ContentResolver resolver) {
        super(handler);
        mResolver = resolver;
    }

    public void setSystemOrientationSettingListener(ScreenRotationSettingListener l) {
        this.listener = l;
    }

    /**
     *  屏幕旋转设置改变时调用
     */
    @Override
    public void onChange(boolean selfChange) {
        super.onChange(selfChange);
        if (listener != null) {
            listener.onRotationSettingChanged();
        }
    }

    public void startObserver() {
        if (mResolver == null) {
            LogUtils.INSTANCE.warn(TAG, "mResolver is null");
            return;
        }
        mResolver.registerContentObserver(Settings.System.getUriFor(Settings.System.ACCELEROMETER_ROTATION),
                false, this);
    }

    public void stopObserver() {
        if (mResolver == null) {
            LogUtils.INSTANCE.warn(TAG, "mResolver is null");
            return;
        }
        mResolver.unregisterContentObserver(this);
    }

    /**
     *  监听回调
     */
    public interface ScreenRotationSettingListener {
        void onRotationSettingChanged();
    }


}
```

### 3.2 监听屏幕旋转

这里的监听屏幕旋转，并不是简单的`onConfigChanged()`，准确来说是监听屏幕的旋转角度。

上面提到，我们需要知道用户手动设置了横竖屏后，手机何时旋转到对应的位置，此时需要恢复Activity到默认的旋转行为。

我们可以借助`OrientationEventListener`，实时的获取当前手机的旋转角度，从而计算出当前手机的横竖状态。这里可以补一下旋转角度的知识。

- 正常竖直状态（信号电量状态栏朝上），orientation = 0。
- 横屏状态1（信号电量状态栏朝左），orientation = 270。
- 反向竖直状态（信号电量状态栏朝下），orientation = 180。
- 横屏状态2 （信号电量状态栏朝右），orientation = 90。

同样我们可以封装一下

```java
/**
 * Created by Numen_fan on 2023/2/11
 * Desc: 时刻检测屏幕的旋转角度，并计算当前的横竖屏状态
 */
public class ScreenOrientationDetector extends OrientationEventListener {
    private int mCurrentOrientation;

    public static final int ORIENTATION_UNDEFINED = 0;

    public static final int ORIENTATION_PORTRAIT = 1;

    public static final int ORIENTATION_LANDSCAPE = 2;

    private int currentOrientation = ORIENTATION_UNDEFINED;

    private OrientationChangeListener listener;

    public ScreenOrientationDetector(Context context, int rate) {
        super(context, rate);
    }

    public void setOrientationChangeListener(OrientationChangeListener l) {
        this.listener = l;
    }

    private int getOrientation() {
        if (this.mCurrentOrientation != 0 && this.mCurrentOrientation != 180) {
            return this.mCurrentOrientation != 90 && this.mCurrentOrientation != 270
                    ? ORIENTATION_UNDEFINED : ORIENTATION_LANDSCAPE;
        } else {
            return ORIENTATION_PORTRAIT;
        }
    }

    @Override
    public void onOrientationChanged(int orientation) {
        if (orientation != ORIENTATION_UNKNOWN) {
            if (orientation >= 45 && orientation <= 315) {
                if (orientation > 45 && orientation < 135) {
                    this.mCurrentOrientation = 90;
                } else if (orientation > 135 && orientation < 225) {
                    this.mCurrentOrientation = 180;
                } else if (orientation > 225 && orientation < 315) {
                    this.mCurrentOrientation = 270;
                }
            } else {
                this.mCurrentOrientation = 0;
            }

            int newOrientation = getOrientation();
            if (ORIENTATION_UNDEFINED != newOrientation && newOrientation != currentOrientation) {
                currentOrientation = newOrientation;
                if (listener != null) {
                    listener.onOrientationChanged(currentOrientation);
                }
            }
        }
    }

    public void initOrientation() {
        currentOrientation = ORIENTATION_UNDEFINED;
    }

    /**
     * 计算出屏幕发生旋转，就会触发
     */
    public interface OrientationChangeListener {
        void onOrientationChanged(int orientation);
    }
}
```

注意这里有一个`initOrientation` 方法，这个后面会讲，它在什么场景下会被调用。

### 3.3 使用

这里我们用一个Activity来实现一下 ，功能很简单，在页面上放了一个按钮，点击后，切换横竖屏就好了。

```
public class OrientationActivity extends BaseActivity implements ScreenRotationSettingObserver.ScreenRotationSettingListener,
        ScreenOrientationDetector.OrientationChangeListener {

    private static final String TAG = "OrientationActivity";

    private static final int CHANGE_ORIENTATION = 10086;

    private ScreenRotationSettingObserver mScreenRotationSettingObserver;
    private ScreenOrientationDetector mScreenOrientationDetector;
    private Handler mHandler;

    @Override
    public int getContentResId() {
        return R.layout.activity_orientation;
    }

    @Override
    public void initUI() {

    }

    @Override
    public void initParam() {
        mScreenRotationSettingObserver = new ScreenRotationSettingObserver(mHandler, getContentResolver());
        mScreenOrientationDetector = new ScreenOrientationDetector(this, SensorManager.SENSOR_DELAY_NORMAL);
        mScreenRotationSettingObserver.setSystemOrientationSettingListener(this);
        mScreenOrientationDetector.setOrientationChangeListener(this);
        // 初始化Handler
        mHandler = new Handler(getMainLooper()) {
            @Override
            public void handleMessage(@NonNull Message msg) {
                super.handleMessage(msg);
                if (msg.what == CHANGE_ORIENTATION) {
                    setRequestedOrientation(ActivityInfo.SCREEN_ORIENTATION_UNSPECIFIED);
                    mScreenOrientationDetector.disable(); // 恢复设置后，结束检测
                }
            }
        };
    }

    @Override
    public void initListener() {
        findViewById(R.id.btn_change_orientation).setOnClickListener(v -> changeOrientation());
        // 开启屏幕自动旋转开关的监听
        mScreenRotationSettingObserver.startObserver();
    }

    @Override
    protected void onDestroy() {
        super.onDestroy();
        mScreenRotationSettingObserver.setSystemOrientationSettingListener(null);
        mScreenOrientationDetector.setOrientationChangeListener(null);
        mScreenOrientationDetector.disable();
        mScreenRotationSettingObserver.stopObserver();
        mHandler.removeMessages(CHANGE_ORIENTATION);
        mHandler = null;
    }

    /**
     * 屏幕自动旋转开关发生变化
     */
    @Override
    public void onRotationSettingChanged() {
        LogUtils.INSTANCE.warn(TAG, "系统自动选择设置发生变化");
        startScreenOrientationListen();
    }

    /**
     * 实时计算的横竖屏发生了变化
     *
     * @param orientation 当前横屏还是竖屏
     */
    @Override
    public void onOrientationChanged(int orientation) {
        LogUtils.INSTANCE.warn(TAG, "横竖屏计算发生变化，当前状态 = " + orientation);
        if (canActivityAutoRotate() && getResources().getConfiguration().orientation == orientation) {
         // 当手机旋转到和手动设置的同一个方向，恢复默认的设置。
            mHandler.sendEmptyMessageDelayed(CHANGE_ORIENTATION, 500);
        }
    }

    /**
     * 手动改变横竖屏
     */
    @SuppressLint("SourceLockedOrientationActivity")
    private void changeOrientation() {
        mHandler.removeMessages(CHANGE_ORIENTATION); // 手动切换，移除之前的延迟任务，避免快速点击带来的问题。
        if (isLandscape()) {
            setRequestedOrientation(ActivityInfo.SCREEN_ORIENTATION_PORTRAIT);
        } else {
            setRequestedOrientation(ActivityInfo.SCREEN_ORIENTATION_LANDSCAPE);
        }
        startScreenOrientationListen();
    }

    /**
     * 打开屏幕旋转监听
     */
    private void startScreenOrientationListen() {
        // 如果系统自动旋转打开，则开启横竖屏切换检测
        if (canActivityAutoRotate()) {
            LogUtils.INSTANCE.warn(TAG, "开启屏幕旋转检测");
            mHandler.postDelayed(() -> {
                mScreenOrientationDetector.initOrientation();
                mScreenOrientationDetector.enable();
            }, 500);
        } else {
            LogUtils.INSTANCE.warn(TAG, "关闭了自动旋转");
            mScreenOrientationDetector.disable(); // 如果关闭了自动旋转，取消一次横竖屏监听
        }
    }

    private boolean isLandscape() {
        return getResources().getConfiguration().orientation == Configuration.ORIENTATION_LANDSCAPE;
    }


    private boolean canActivityAutoRotate() {
        try {
            int value = Settings.System.getInt(getContentResolver(), Settings.System.ACCELEROMETER_ROTATION);
            return value == 1;
        } catch (Settings.SettingNotFoundException e) {
            LogUtils.INSTANCE.error(TAG, "canActivityAutoRotate error", e);
        }
        return false;
    }
}
```

【说明】

1. 各个初始化的中，不需要关注太多，在initParam中`startScreenOrientationListen`开启监听，监听系统中自动旋转的变化。
2. Handler用处1：用于延迟生效屏幕旋转监听。因为`OrientationEventListener` 的回调是很频繁的，频率大概是200ms。如果我们手动切换后立刻开启，当用户在旋转的过程中，可能Sensor回调偏差，导致orientation计算出错，就会导致恢复默认，画面来回切换，这也是为何在开启检测和恢复默认的时候都有延迟执行。
3. Handler用户2：延迟执行恢复默认行为，注意由于有延迟机制，每次手动切换都要记得移除前一次的事件，否则在快速点击切换的时候会有问题，这点不难理解。（其实这里可以优化一下，orientation的判断应该拿在延迟事件中一起判断）
4. `mScreenOrientationDetector.initOrientation();` 是为了确保每次开启监听一定会收到回调，解决在竖屏状态下，手动点击切换横屏，保持不动的情况下，继续切回竖屏之后能够收到回调，恢复默认行为。
5. 因为`OrientationEventListener` 的执行频繁，所以要做好`disable` 的处理。

### 3.4 流程总结

依据3.3节中的实现，总结如下流程

![](https://raw.githubusercontent.com/Numen-fan/BlogPicRepo/main/img/20230214001214.png)

## 资料

- [Android ContentObserver](https://juejin.cn/post/7031511196125626398)
- [重力传感器——屏幕旋转OrientationEventListener监听](https://blog.csdn.net/dongxianfei/article/details/120093409)
- [OrientationEventListener](https://developer.android.com/reference/android/view/OrientationEventListener?hl=en)









