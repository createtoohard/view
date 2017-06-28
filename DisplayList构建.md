# DisplayList构建
Android应用程序窗口UI的绘制过程是从`ViewRootImpl.performDraw()`方法开始的

## DisplayList构建时序图
```puml
Title : DisplayList构建

ViewRootImpl -> ViewRootImpl : performDraw()
ViewRootImpl -> ViewRootImpl : draw()
ViewRootImpl -> ThreadedRenderer : draw()
ThreadedRenderer -> ThreadedRenderer : updateRootDisplayList()
ThreadedRenderer -> ThreadedRenderer : updateViewTreeDisplayList()

ThreadedRenderer -> View : updateDisplayListIfDirty()
View -> RenderNode : start()
View -> View : draw()

View -> RenderNode : end()
View -> View : setDisplayListProperties()


ThreadedRenderer -> RenderNode : start()
ThreadedRenderer -> DisplayListCanvas : drawRenderNode()
ThreadedRenderer -> RenderNode : end()
```

### `ViewRootImpl.performDraw()` 方法
```java
private void performDraw() {
    if (mAttachInfo.mDisplayState == Display.STATE_OFF && !mReportNextDraw) {
        return;
    }

    final boolean fullRedrawNeeded = mFullRedrawNeeded;
    mFullRedrawNeeded = false;

    mIsDrawing = true;
    Trace.traceBegin(Trace.TRACE_TAG_VIEW, "draw");
    try {
        //调用draw()方法
        draw(fullRedrawNeeded);
    } finally {
        mIsDrawing = false;
        Trace.traceEnd(Trace.TRACE_TAG_VIEW);
    }
    //......
}
```

### `ViewRootImpl.draw()` 方法
```java
private void draw(boolean fullRedrawNeeded) {
    Surface surface = mSurface;
    //mSurface不存在返回
    if (!surface.isValid()) {
        return;
    }
    //......
    final Rect dirty = mDirty;
    //......
    // 当dirty区域不为空，或者窗口当前有动画要执行，
    if (!dirty.isEmpty() || mIsAnimating || accessibilityFocusDirty) {
        //ThreadedRenderer不为空且开启状态
        if (mAttachInfo.mHardwareRenderer != null && mAttachInfo.mHardwareRenderer.isEnabled()) {
            // If accessibility focus moved, always invalidate the root.
            boolean invalidateRoot = accessibilityFocusDirty || mInvalidateRootRequested;
            mInvalidateRootRequested = false;

            // Draw with hardware renderer.
            mIsAnimating = false;

            if (mHardwareYOffset != yOffset || mHardwareXOffset != xOffset) {
                mHardwareYOffset = yOffset;
                mHardwareXOffset = xOffset;
                invalidateRoot = true;
            }

            if (invalidateRoot) {
                mAttachInfo.mHardwareRenderer.invalidateRoot();
            }

            dirty.setEmpty();

            // Stage the content drawn size now. It will be transferred to the renderer
            // shortly before the draw commands get send to the renderer.
            final boolean updated = updateContentDrawBounds();

            if (mReportNextDraw) {
                // report next draw overrides setStopped()
                // This value is re-sync'd to the value of mStopped
                // in the handling of mReportNextDraw post-draw.
                mAttachInfo.mHardwareRenderer.setStopped(false);
            }

            if (updated) {
                requestDrawWindow();
            }
            //调用ThreadedRenderer.draw()方法
            mAttachInfo.mHardwareRenderer.draw(mView, mAttachInfo, this);
        } else {
            //......
            //软件绘制
            if (!drawSoftware(surface, mAttachInfo, xOffset, yOffset, scalingRequired, dirty)) {
                return;
            }
        }
    }

    if (animating) {
        mFullRedrawNeeded = true;
        scheduleTraversals();
    }
}
```


### `ThreadedRenderer.draw()` 方法
```java
void draw(View view, AttachInfo attachInfo, HardwareDrawCallbacks callbacks) {
    attachInfo.mIgnoreDirtyState = true;

    final Choreographer choreographer = attachInfo.mViewRootImpl.mChoreographer;
    choreographer.mFrameInfo.markDrawStart();
    //调用updateRootDisplayList()方法，构建参数view描述的视图的DisplayList（DecorView）
    updateRootDisplayList(view, callbacks);

    attachInfo.mIgnoreDirtyState = false;

    // register animating rendernodes which started animating prior to renderer
    // creation, which is typical for animators started prior to first draw
    if (attachInfo.mPendingAnimatingRenderNodes != null) {
        final int count = attachInfo.mPendingAnimatingRenderNodes.size();
        for (int i = 0; i < count; i++) {
            registerAnimatingRenderNode(
                    attachInfo.mPendingAnimatingRenderNodes.get(i));
        }
        attachInfo.mPendingAnimatingRenderNodes.clear();
        // We don't need this anymore as subsequent calls to
        // ViewRootImpl#attachRenderNodeAnimator will go directly to us.
        attachInfo.mPendingAnimatingRenderNodes = null;
    }

    final long[] frameInfo = choreographer.mFrameInfo.mFrameInfo;
    //调用nSyncAndDrawFrame()方法通知RenderThread绘制下一帧
    int syncResult = nSyncAndDrawFrame(mNativeProxy, frameInfo, frameInfo.length);
    if ((syncResult & SYNC_LOST_SURFACE_REWARD_IF_FOUND) != 0) {
        setEnabled(false);
        attachInfo.mViewRootImpl.mSurface.release();
        // Invalidate since we failed to draw. This should fetch a Surface
        // if it is still needed or do nothing if we are no longer drawing
        attachInfo.mViewRootImpl.invalidate();
    }
    if ((syncResult & SYNC_INVALIDATE_REQUIRED) != 0) {
        attachInfo.mViewRootImpl.invalidate();
    }
}
```


### `ThreadedRenderer.updateRootDisplayList()` 方法
构建DecorView的DisplayList
设置Barrier是为了将一个view的所有的Draw Op及其子view对应的Draw op记录在一个chunk中，
```java
private void updateRootDisplayList(View view, HardwareDrawCallbacks callbacks) {
    //调用updateViewTreeDisplayList()方法，构建参数view描述的视图的DisplayList，即DecorView的DisplayList
    updateViewTreeDisplayList(view);
    //mRootNodeNeedsUpdate为true或者mRootNode还未构建，表示要更新mRootNode的DisplayList
    if (mRootNodeNeedsUpdate || !mRootNode.isValid()) {
        //调用RenderNode.start()方法获得一个HardwareCanvas
        DisplayListCanvas canvas = mRootNode.start(mSurfaceWidth, mSurfaceHeight);
        try {
            final int saveCount = canvas.save();
            canvas.translate(mInsetLeft, mInsetTop);
            callbacks.onHardwarePreDraw(canvas);
            //调用HardwareCanvas.insertReorderBarrier()方法，设置一个ReorderBarrier
            canvas.insertReorderBarrier();
            //调用HardwareCanvas.drawRenderNode()方法将view的DisplayList绘制在里面
            canvas.drawRenderNode(view.updateDisplayListIfDirty());
            //调用HardwareCanvas.insertInorderBarrier()方法，设置一个InorderBarrier
            canvas.insertInorderBarrier();

            callbacks.onHardwarePostDraw(canvas);
            canvas.restoreToCount(saveCount);
            mRootNodeNeedsUpdate = false;
        } finally {
            //调用RootRender.end()方法，取出已经绘制好的HardwareCanvas的数据，并且作为RenderNode的新的DisplayList
            mRootNode.end(canvas);
        }
    }
}
```


#### `ThreadedRenderer.updateViewTreeDisplayList()` 方法
构建DecorView的DisplayList
```java
private void updateViewTreeDisplayList(View view) {
    view.mPrivateFlags |= View.PFLAG_DRAWN;
    view.mRecreateDisplayList = (view.mPrivateFlags & View.PFLAG_INVALIDATED)
            == View.PFLAG_INVALIDATED;
    view.mPrivateFlags &= ~View.PFLAG_INVALIDATED;
    //调用View.updateDisplayListIfDirty()方法
    view.updateDisplayListIfDirty();
    view.mRecreateDisplayList = false;
}
```

#### `View.updateDisplayListIfDirty()` 方法
```java
public RenderNode updateDisplayListIfDirty() {
    final RenderNode renderNode = mRenderNode;
    if (!canHaveDisplayList()) {
        // can't populate RenderNode, don't try
        return renderNode;
    }

    if ((mPrivateFlags & PFLAG_DRAWING_CACHE_VALID) == 0
            || !renderNode.isValid()
            || (mRecreateDisplayList)) {
        // Don't need to recreate the display list, just need to tell our
        // children to restore/recreate theirs
        if (renderNode.isValid()
                && !mRecreateDisplayList) {
            mPrivateFlags |= PFLAG_DRAWN | PFLAG_DRAWING_CACHE_VALID;
            mPrivateFlags &= ~PFLAG_DIRTY_MASK;
            dispatchGetDisplayList();

            return renderNode; // no work needed
        }

        // If we got here, we're recreating it. Mark it as such to ensure that
        // we copy in child display lists into ours in drawChild()
        mRecreateDisplayList = true;

        int width = mRight - mLeft;
        int height = mBottom - mTop;
        int layerType = getLayerType();

        final DisplayListCanvas canvas = renderNode.start(width, height);
        canvas.setHighContrastText(mAttachInfo.mHighContrastText);

        try {
            if (layerType == LAYER_TYPE_SOFTWARE) {
                buildDrawingCache(true);
                Bitmap cache = getDrawingCache(true);
                if (cache != null) {
                    canvas.drawBitmap(cache, 0, 0, mLayerPaint);
                }
            } else {
                computeScroll();

                canvas.translate(-mScrollX, -mScrollY);
                mPrivateFlags |= PFLAG_DRAWN | PFLAG_DRAWING_CACHE_VALID;
                mPrivateFlags &= ~PFLAG_DIRTY_MASK;

                // Fast path for layouts with no backgrounds
                if ((mPrivateFlags & PFLAG_SKIP_DRAW) == PFLAG_SKIP_DRAW) {
                    dispatchDraw(canvas);
                    if (mOverlay != null && !mOverlay.isEmpty()) {
                        mOverlay.getOverlayView().draw(canvas);
                    }
                } else {
                    draw(canvas);
                }
            }
        } finally {
            renderNode.end(canvas);
            setDisplayListProperties(renderNode);
        }
    } else {
        mPrivateFlags |= PFLAG_DRAWN | PFLAG_DRAWING_CACHE_VALID;
        mPrivateFlags &= ~PFLAG_DIRTY_MASK;
    }
    return renderNode;
}
```


## 获取一个DisplayListCanvas对象
### `RenderNode.start()` 方法
```java
public DisplayListCanvas start(int width, int height) {
    //调用DisplayListCanvas.obtain()方法，返回一个DisplayListCanvas对象
    return DisplayListCanvas.obtain(this, width, height);
}
```

#### `DisplayListCanvas.obtain()` 方法
```java
static DisplayListCanvas obtain(@NonNull RenderNode node, int width, int height) {
    if (node == null) throw new IllegalArgumentException("node cannot be null");
    //从sPool种申请一个DisplayListCanvas，如果没有就新建一个
    DisplayListCanvas canvas = sPool.acquire();
    if (canvas == null) {
        canvas = new DisplayListCanvas(width, height);
    } else {
        nResetDisplayListCanvas(canvas.mNativeCanvasWrapper, width, height);
    }
    //将RenderNode参数保存到.mNode中。这里的宽高是Surface的宽高
    canvas.mNode = node;
    canvas.mWidth = width;
    canvas.mHeight = height;
    return canvas;
}
```

#### `DisplayListCanvas.DisplayListCanvas()` 构造函数
```java
private DisplayListCanvas(int width, int height) {
    //调用natvie层的nCreateDisplayListCanvas()函数，并将返回值作为参数调用Canvas的对应的构造函数
    super(nCreateDisplayListCanvas(width, height));
    mDensity = 0; // disable bitmap density scaling
}
```

#### `createDisplayListCanvas()` 函数
```cpp
static jlong android_view_DisplayListCanvas_createDisplayListCanvas(JNIEnv* env, jobject clazz,
        jint width, jint height) {
    return reinterpret_cast<jlong>(Canvas::create_recording_canvas(width, height));
}
```

#### `Canvas::create_recording_canvas()` 函数
创建一个对象并将地址返回
```cpp
Canvas* Canvas::create_recording_canvas(int width, int height) {
#if HWUI_NEW_OPS
    return new uirenderer::RecordingCanvas(width, height);
#else
    return new uirenderer::DisplayListCanvas(width, height);
#endif
}
```

#### `Canvas.Canvas()` 构造函数
```java
public Canvas(long nativeCanvas) {
    if (nativeCanvas == 0) {
        throw new IllegalStateException();
    }
    mNativeCanvasWrapper = nativeCanvas;
    mFinalizer = NoImagePreloadHolder.sRegistry.registerNativeAllocation(
            this, mNativeCanvasWrapper);
    mDensity = Bitmap.getDefaultDensity();
}
```


## 将DisplayList绘制到RenderNode
### `DisplayListCanvas.drawRenderNode()` 方法
```java
public void drawRenderNode(RenderNode renderNode) {
    //调用jni层的nDrawRenderNode()函数，mNativeCanvasWrapper即native层Canvas的地址
    nDrawRenderNode(mNativeCanvasWrapper, renderNode.getNativeDisplayList());
}
```

#### `RenderNode.getNativeDisplayList()` 方法
```java
long getNativeDisplayList() {
    if (!mValid) {
        throw new IllegalStateException("The display list is not valid.");
    }
    //mNativeRenderNode保存的就是native层的RootRenderNode对象，在RenderNode.adopt()方法种初始化
    return mNativeRenderNode;
}
```

#### `drawRenderNode()` 函数
```cpp
static void android_view_DisplayListCanvas_drawRenderNode(JNIEnv* env,
        jobject clazz, jlong canvasPtr, jlong renderNodePtr) {
    Canvas* canvas = reinterpret_cast<Canvas*>(canvasPtr);
    RenderNode* renderNode = reinterpret_cast<RenderNode*>(renderNodePtr);
    //调用native层的canvas.drawRenderNode()函数，renderNode即native层的RootRenderNode对象
    canvas->drawRenderNode(renderNode);
}
```

### `DisplayListCanvas::drawRenderNode()` 函数
```cpp
void DisplayListCanvas::drawRenderNode(RenderNode* renderNode) {
    //将RenderNode封装成一个DrawRenderNodeOp对象
    DrawRenderNodeOp* op = new (alloc()) DrawRenderNodeOp(
            renderNode,
            *mState.currentTransform(),
            mState.clipIsSimple());
    //调用addRenderNodeOp()函数
    addRenderNodeOp(op);
}
```

### `DisplayListCanvas::addRenderNodeOp()` 函数
```cpp
size_t DisplayListCanvas::addRenderNodeOp(DrawRenderNodeOp* op) {
    int opIndex = addDrawOp(op);
#if !HWUI_NEW_OPS
    int childIndex = mDisplayList->addChild(op);

    // update the chunk's child indices
    DisplayList::Chunk& chunk = mDisplayList->chunks.back();
    chunk.endChildIndex = childIndex + 1;

    if (op->renderNode->stagingProperties().isProjectionReceiver()) {
        // use staging property, since recording on UI thread
        mDisplayList->projectionReceiveIndex = opIndex;
    }
#endif
    return opIndex;
}
```


##
### `RenderNode.end()` 方法
```java
public void end(DisplayListCanvas canvas) {
    //返回一个DisplayList对象
    long displayList = canvas.finishRecording();
    nSetDisplayList(mNativeRenderNode, displayList);
    canvas.recycle();
    mValid = true;
}
```

#### `DisplayListCanvas.finishRecording()` 方法
```java
long finishRecording() {
    return nFinishRecording(mNativeCanvasWrapper);
}
```

#### `nFinishRecording()` 函数
调用natvie层对应的Canvas的finishRecording()函数
```cpp
static jlong android_view_DisplayListCanvas_finishRecording(JNIEnv* env,
        jobject clazz, jlong canvasPtr) {
    Canvas* canvas = reinterpret_cast<Canvas*>(canvasPtr);
    return reinterpret_cast<jlong>(canvas->finishRecording());
}
```

#### `DisplayListCanvas::finishRecording()` 函数
```cpp
DisplayList* DisplayListCanvas::finishRecording() {
    flushRestoreToCount();
    flushTranslate();

    mPaintMap.clear();
    mRegionMap.clear();
    mPathMap.clear();
    DisplayList* displayList = mDisplayList;
    mDisplayList = nullptr;
    mSkiaCanvasProxy.reset(nullptr);
    return displayList;
}
```


### `nSetDisplayList()` 函数
```cpp
static void android_view_RenderNode_setDisplayList(JNIEnv* env,
        jobject clazz, jlong renderNodePtr, jlong displayListPtr) {
    class RemovedObserver : public TreeObserver {
    public:
        virtual void onMaybeRemovedFromTree(RenderNode* node) override {
            maybeRemovedNodes.insert(sp<RenderNode>(node));
        }
        std::set< sp<RenderNode> > maybeRemovedNodes;
    };

    RenderNode* renderNode = reinterpret_cast<RenderNode*>(renderNodePtr);
    DisplayList* newData = reinterpret_cast<DisplayList*>(displayListPtr);
    RemovedObserver observer;
    renderNode->setStagingDisplayList(newData, &observer);
    for (auto& node : observer.maybeRemovedNodes) {
        if (node->hasParents()) continue;
        onRenderNodeRemoved(env, node.get());
    }
}
```
