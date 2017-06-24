# ViewRootImpl
* ViewRootImpl实现了ViewParent接口，作为整个控件树的根部，它是控件树正常运作的动力
* 控件的测量、布局、绘制以及输入事件的派发处理都由ViewRootImpl触发
* ViewRootImpl是WindowManagerGlobal工作的实际实现者，因此它需要与WMS交互通信以调整窗口位置的大小，以及对来自WMS的时间进行处理

## ViewRootImpl的创建
ViewRootImpl的创建主要在以下两个方法中完成的
* `ViewRootImpl.ViewRootImpl()`构造方法
    * ViewRootImpl的构造方法主要就是初始化一些重要成员的变量
    * ViewRootImpl在`WindowManagerGlobal.addView()`方法中构造
    * `final ViewRootHandler mHandler = new ViewRootHandler();`
        * 一个依附于创建ViewRootImpl的线程，即主线程上，用于将某些必须在主线程上进行的操作调度到主线程执行
        * mChoreographer处理消息时具有VSYNC特性，因此它主要用于处理与重绘相关的操作，由于mChoreographer需要等待VSYNC的垂直同步事件来触发对下一条消息的处理，即时性要比mHandler差
        * mHandler的作用是为了将发生在其他线程中（指来自WMS的事件）的事件安排在主线程上执行
    * `final Surface mSurface = new Surface();`
        * Surface实例，此时是一个空壳子，在WMS通过relayoutWindow()为其分配一块Surface之前尚不能使用
    * `mPendingContentInsets、mPendingContentInsets、mWinFrame、mWidth、mHeight`
        * 这几个成员变量都是存储了窗口布局相关的信息
        * mPendingContentInsets、mPendingContentInsets、mWinFrame与窗口在WMS中的Frame、ContentInsets、VisiableInsets是保持同步的
            * 在relayoutWindow()是会作为参数传出去
            * 在WMS回调IWindow.Stub.resize()时，会更新
        * `mWinFrame、mWidth、mHeight`
* `ViewRootImpl.setView()`方法
    * 创建窗口、建立输入事件接收机制、触发第一次遍历

### `ViewRootImpl.ViewRootImpl()`构造函数
```java
public ViewRootImpl(Context context, Display display) {
    mContext = context;
    //WindowSession是与WMS通信的代理，从WindowManagerGlobal中获得
    mWindowSession = WindowManagerGlobal.getWindowSession();
    //保存参数Display对象，后面setView()会把窗口添加到这个Display上
    mDisplay = display;
    //保存应用名称
    mBasePackageName = context.getBasePackageName();
    //保存当前线程到mThread，这个赋值体现了创建ViewRootImpl的线程如何成为UI主线程
    //在ViewRootImpl处理来自控件树的请求时，会检查发起请求的thread与这个mThread是否相同，不同会拒绝这个请求
    mThread = Thread.currentThread();

    mLocation = new WindowLeaked(null);
    mLocation.fillInStackTrace();
    //宽、高
    mWidth = -1;
    mHeight = -1;
    //mDirty用于收集窗口中的无效区域
    //无效区域就是由于数据或状态发生变化而需要进行重绘的区域
    //通过view.invalidate()方法将需要重绘的区域添加到mDirty中
    mDirty = new Rect();
    mTempRect = new Rect();
    mVisRect = new Rect();
    //mWinFrame描述了当前窗口的位置和尺寸。与WMS中的WindowState.mFrame保持一致
    mWinFrame = new Rect();
    //创建一个W类型的实例，W是IWindow.Stub的子类，即它将在WMS中作为新窗口的ID，并接收来自WMS的回调
    mWindow = new W(this);
    mTargetSdkVersion = context.getApplicationInfo().targetSdkVersion;
    mViewVisibility = View.GONE;
    mTransparentRegion = new Region();
    mPreviousTransparentRegion = new Region();
    mFirst = true; // true for the first time the view is added
    mAdded = false;
    //创建View.AttachInfo实例
    //AttachInfo存储了当前空间数所贴附的窗口的各种信息，并且会派发给控件树中的每一个控件。
    //这些控件会将这个对象保存在自己的mAttachInfo变量中
    //mAttachInfo中所保存的信息有WindowSession、窗口的实例（即mWindow）、ViewRootImpl实例、窗口所属的Display、窗口的Surface以及在屏幕上的位置等
    //当需要在一个View中查询与当前窗口相关的信息时，非常值得在mAttachInfo中搜索一下
    mAttachInfo = new View.AttachInfo(mWindowSession, mWindow, display, this, mHandler, this);
    mAccessibilityManager = AccessibilityManager.getInstance(context);
    mAccessibilityInteractionConnectionManager =
        new AccessibilityInteractionConnectionManager();
    mAccessibilityManager.addAccessibilityStateChangeListener(
            mAccessibilityInteractionConnectionManager);
    mHighContrastTextManager = new HighContrastTextManager();
    mAccessibilityManager.addHighTextContrastStateChangeListener(
            mHighContrastTextManager);
    mViewConfiguration = ViewConfiguration.get(context);
    //dpi
    mDensity = context.getResources().getDisplayMetrics().densityDpi;
    mNoncompatDensity = context.getResources().getDisplayMetrics().noncompatDensityDpi;
    //创建FallbackEventHandler实例，这个类同PhoneWindowManager一样，定义在policy包中
    //它的实现为PhoneFallbackEventHandler。主要处理一个未经任何人消费的输入事件的场所
    mFallbackEventHandler = new PhoneFallbackEventHandler(context);
    //创建一个依附于当前线程，即主线程的Choreographer，用于通过VSYNC特性安排重绘行为
    mChoreographer = Choreographer.getInstance();
    mDisplayManager = (DisplayManager)context.getSystemService(Context.DISPLAY_SERVICE);
    loadSystemProperties();
}
```
