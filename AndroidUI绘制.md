# AndroidUI绘制
**AndroidUI从绘制到显示分为两步进行：**
1. Android应用进程中，将UI绘制成一个图形缓冲区，并将图形缓冲区交给SurfaceFlinger
    * Android应用进程UI绘制分为 **软件绘制** 和 **硬件绘制**
2. SurfaceFlinger将图形缓冲区合成并输入屏幕显示（硬件加速的方式）
    * 硬件加速渲染即调用GPU进行渲染
    * GPU作为一个硬件，用户是无法直接调用的，它是GPU厂商根据OpenGL规范实现的，驱动间接调用

# RenderNode
* 只有硬件加速绘制时才会有关联的RenderNode
* 在Android应用窗口中，每一个View都抽象为一个RenderNode
  * 如果一个设有Background，该Background也会被抽象为一个RenderNode
* 在OpengGLRender库种没有view的概念，所有每一个可以绘制的元素都被抽象成一个RenderNode

# DisplayList
* DisplayList是一个绘制命令缓冲区
  * 当`view.onDraw()`方法被调用，进行绘制时，会调用Canvas类的drawxxx方法来绘制图形时，实际上是将绘制命令以参数的形式保存在 **DisplayList** 中。
* 每一个 **RnederNode** 都关联一个 **DisplayListRenderer** ，用于执行 **DisplayList** 命令，这个过程成为DisplayListReplay
* 只有使用硬件加速绘制的view才会有关联的RenderNode，才会使用到DisplayList

### DisplayList的优点
在不需要更新时，跳过`onDraw()`方法，减少cpu的计算执行时间。主要分为两点：
1. 如果下一帧绘制中，view的内容不需要更新，那就不用更新它的DisplayList，也就是不需要调用它的`onDraw()`方法
2. 如果下一帧绘制时，view仅仅是一些简单的属性变化，也不需要重建它的DisplayList，只需要更新一下上一帧DisplayList对应的属性，也不需要调用它的`onDraw()`方法


# 非硬件加速渲染的view
* **Gpu不支持的ui绘制的view**，只能通过软件渲染
  * 创建一个新的Canvas，这个Canvas的底层是一个bitmap，也就是说绘制发生在这个bitmap上
  * 绘制完成后，这个bitmap再被记录在其ParentView的DisplayList中
  * 当其ParentView的DisplayList命令被执行时，记录在里面的Bitmap再通过OpenGL命令来绘制

* **TextureView** 不是通过DisplayList来绘制的
  * 它的底层是一个OpenGL纹理，因此可以直接跳过DisplayList这一步，从而提高效率
  * OpenGL纹理通过LayerRenderer来封装，LayerRenderer和DisplayList是同一个级别的

# RootView
* Android应用窗口的View是通过树形结构来组织的，在他们的`onDraw()`方法被调用期间，他们都是将自己的UI绘制在ParentView的DisplayList中
* 最顶层的ParentView是一个 **RootView** ，它关联的RootNode成为 **RootRenderNode**
* **RootRenderNode** 对应的 **DisplayList** 将会包含一个窗口的所有绘制命令
* 在绘制窗口的下一帧时，RootRenderNode的DisplayList都会通过一个 **OpenGLRenderer** 真正的通过Open GL命令绘制在一个 **GraphicBuffer** 中，最后这个GraphicBuffer被交给SurfaceFliger服务进行合成和显示

# RenderThread
* Android 5.0引进，以减轻MianThread的工作，
* **MainThread** 主要负责调用`view.onDraw()`方法来构造他们的 **DisplayList**
* 在下一帧Vsync信号到来时，通过 **RenderProxy** 向 **RenderThread** 发出一个 **drawFrame** 命令
* **RenderThread** 内部由一个 **TaskQueue**，从MainThread发送过来的drawFrame命令就保存在TaskQueue中，等待RenderThread处理
