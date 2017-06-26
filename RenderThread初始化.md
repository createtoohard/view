# RenderThread初始化

```puml
Title : RenderThread初始化
ViewRootImpl -> ViewRootImpl : setView()
ViewRootImpl -> ViewRootImpl : enableHardwareAcceleration()
ViewRootImpl -> ThreadedRenderer : isAvailable()
ViewRootImpl -> ThreadedRenderer : create()
ThreadedRenderer -> ThreadedRenderer : ThreadedRenderer()
```
## 打开硬件加速渲染开关
### `ViewRootImpl.setView()`
```java
public void setView(View view, WindowManager.LayoutParams attrs, View panelParentView) {
    synchronized (this) {
        if (mView == null) {
            mView = view;
            //......
            setAccessibilityFocus(null, null);
            // 如果一个view实现来RootViewSurfaceTaker接口
            if (view instanceof RootViewSurfaceTaker) {
                mSurfaceHolderCallback =
                        ((RootViewSurfaceTaker)view).willYouTakeTheSurface();
                if (mSurfaceHolderCallback != null) {
                    // 实例化一个TakenSurfaceHolder对象，赋给mSurfaceHolder
                    mSurfaceHolder = new TakenSurfaceHolder();
                    mSurfaceHolder.setFormat(PixelFormat.UNKNOWN);
                }
            }

            // Compute surface insets required to draw at specified Z value.
            // TODO: Use real shadow insets for a constant max Z.
            if (!attrs.hasManualSurfaceInsets) {
                attrs.setSurfaceInsets(view, false /*manual*/, true /*preservePrevious*/);
            }

            CompatibilityInfo compatibilityInfo =
                    mDisplay.getDisplayAdjustments().getCompatibilityInfo();
            mTranslator = compatibilityInfo.getTranslator();

            // 如果应用程序自己接管view的渲染，则mSurfaceHolder不为null，即不打开硬件渲染开关
            if (mSurfaceHolder == null) {
                enableHardwareAcceleration(attrs);
            }
            //......
        }
    }
}
```


### `ViewRootImpl.enableHardwareAcceleration()` 方法
```java
private void enableHardwareAcceleration(WindowManager.LayoutParams attrs) {
    mAttachInfo.mHardwareAccelerated = false;
    mAttachInfo.mHardwareAccelerationRequested = false;

    // Don't enable hardware acceleration when the application is in compatibility mode
    if (mTranslator != null) return;

    // 如果WindManager.LayoutParams中有FLAG_HARDWARE_ACCELERATED，表示需要开启硬件加速渲染
    final boolean hardwareAccelerated =
            (attrs.flags & WindowManager.LayoutParams.FLAG_HARDWARE_ACCELERATED) != 0;

    // 请求需要开启硬件加速
    if (hardwareAccelerated) {
        // 如果ThreadedRenderer不存在，返回
        if (!ThreadedRenderer.isAvailable()) {
            return;
        }

        // Persistent processes (including the system) should not do
        // accelerated rendering on low-end devices.  In that case,
        // sRendererDisabled will be set.  In addition, the system process
        // itself should never do accelerated rendering.  In that case, both
        // sRendererDisabled and sSystemRendererDisabled are set.  When
        // sSystemRendererDisabled is set, PRIVATE_FLAG_FORCE_HARDWARE_ACCELERATE
        // can be used by code on the system process to escape that and enable
        // HW accelerated drawing.  (This is basically for the lock screen.)

        // 主要针对Persistent和system两个进程。一般是不使用硬件加速渲染的。
        final boolean fakeHwAccelerated = (attrs.privateFlags &
                WindowManager.LayoutParams.PRIVATE_FLAG_FAKE_HARDWARE_ACCELERATED) != 0;
        final boolean forceHwAccelerated = (attrs.privateFlags &
                WindowManager.LayoutParams.PRIVATE_FLAG_FORCE_HARDWARE_ACCELERATED) != 0;

        if (fakeHwAccelerated) {
            // This is exclusively for the preview windows the window manager
            // shows for launching applications, so they will look more like
            // the app being launched.
            mAttachInfo.mHardwareAccelerationRequested = true;
        } else if (!ThreadedRenderer.sRendererDisabled
                || (ThreadedRenderer.sSystemRendererDisabled && forceHwAccelerated)) {
            if (mAttachInfo.mHardwareRenderer != null) {
                mAttachInfo.mHardwareRenderer.destroy();
            }

            final Rect insets = attrs.surfaceInsets;
            final boolean hasSurfaceInsets = insets.left != 0 || insets.right != 0
                    || insets.top != 0 || insets.bottom != 0;
            final boolean translucent = attrs.format != PixelFormat.OPAQUE || hasSurfaceInsets;
            //调用ThreadedRenderer.create()方法，初始化AttachInfo.mHardwareRenderer变量，即创建一个ThreadedRenderer对象
            mAttachInfo.mHardwareRenderer = ThreadedRenderer.create(mContext, translucent);
            if (mAttachInfo.mHardwareRenderer != null) {
                mAttachInfo.mHardwareRenderer.setName(attrs.getTitle().toString());
                mAttachInfo.mHardwareAccelerated =
                        mAttachInfo.mHardwareAccelerationRequested = true;
            }
        }
    }
}
```

### `ThreadedRenderer.isAvailable()` 方法
判断ThreadedRenderer是否存在
```java
public static boolean isAvailable() {
    return DisplayListCanvas.isAvailable();
}
```


## 创建ThreadedRenderer对象
### `ThreadedRenderer.create()` 方法
```java
public static ThreadedRenderer create(Context context, boolean translucent) {
    ThreadedRenderer renderer = null;
    //如果DisplayListCanvas存在，则创建一个ThreadedRenderer对象并返回
    if (DisplayListCanvas.isAvailable()) {
        renderer = new ThreadedRenderer(context, translucent);
    }
    return renderer;
}
```

## ThreadedRenderer构造方法
### `ThreadedRenderer.ThreadedRenderer()` 构造函数
```java
ThreadedRenderer(Context context, boolean translucent) {
    final TypedArray a = context.obtainStyledAttributes(null, R.styleable.Lighting, 0, 0);
    mLightY = a.getDimension(R.styleable.Lighting_lightY, 0);
    mLightZ = a.getDimension(R.styleable.Lighting_lightZ, 0);
    mLightRadius = a.getDimension(R.styleable.Lighting_lightRadius, 0);
    mAmbientShadowAlpha =
            (int) (255 * a.getFloat(R.styleable.Lighting_ambientShadowAlpha, 0) + 0.5f);
    mSpotShadowAlpha = (int) (255 * a.getFloat(R.styleable.Lighting_spotShadowAlpha, 0) + 0.5f);
    a.recycle();

    //调用native层创建一个RootRenderNode对象
    long rootNodePtr = nCreateRootRenderNode();
    //调用RenderNode.adopt()方法，封装rootNodePtr
    mRootNode = RenderNode.adopt(rootNodePtr);
    mRootNode.setClipToBounds(false);

    //调用native层的nCreateProxy()方法创建一个RenderProxy对象
    mNativeProxy = nCreateProxy(translucent, rootNodePtr);

    //调用ProcessInitializer.init()方法
    ProcessInitializer.sInstance.init(context, mNativeProxy);
    //调用loadSystemProperties()方法
    loadSystemProperties();
}
```
