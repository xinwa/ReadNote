#### 添加 Window 流程

Session.windowAddedLocked -> new WindowSesstion -> naticveCreate ->

new SurfaceComposerClient -> 内部又创建了一个 Client 对象 

#### Surface 创建流程

ViewRootImpl -> new Surface ->  ViewRootImpl.performTraversals ->

ViewRootImpl.performTraversals -> WMS.relayoutWindow ->  WindowContainer.createSurfaceControl ->  new SurfaceControl ->

Android_view_SurfaceControl.nativeCreate -> client.createSurfaceChecked ->

mClient.createSurface -> mFlinger.createLayer ->  这里流程不明，可能是创建了

BufferQueueLayer.onFristRef -> createBufferQueue (BufferQueueProducer、BufferQueueConsumer ) -> createMonitorProducer、createBuffrLayrConsumer 

(MotionProducer、BufferLayerConsumer )

在 Java 层中 ViewRootImpl 实例中持有一个 Surface 对象，该 Surface 对象中的 mNativeObject 属性指向 native 层中创建的 Surface 对象，native 层的 Surface 对应 SurfaceFlinger 中的 Layer 对象，它持有 Layer 中的 BufferQueueProducer 生产者指针，在后面的绘制过程中 Surface 会通过这个生产者来请求图形缓存区，在 Surface 上绘制的内容就是存入到这个缓存区里的，最终再交由 SurfaceFlinger 通过 BufferQueueConsumer 消费者取出这些缓存数据，并合成渲染送到显示器显示。



