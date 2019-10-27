# Android 基础知识

## Activity
---

1、Activity生命周期

2、Activity启动模式：standard， singleTop, singleTask ,singleInstance, (TaskAffinity)

3、Fragment生命周期：

onAttach() -> onCreate() -> onCreateView() ->onViewCreate() -> onActivityCreated()

4、onSaveInstanceState()

5、Service生命周期：

start方式：onCreate()——>onStartCmmmand()->onDestory()

bind方式：onCreate()——>onBind()->onUnbind()->onDestory()

先start后bind：onCreate()——>onStartCmmmand()——>onBind()——>onUnbind()[如果重写了此方法并返回了true]——>onRebind()


6、BroadcastReceiver注册方式：静态注册，动态注册，onReceiver()

## 布局
---

1、LinearLayout, RelativeLayout, FrameLayout

2、TableLayout, AbsoluteLayout, GridLayout

3、Constraintlayout

## Drawable
---

1、BitmapDrawable —— NinePathceDrawable

2、ShapeDrawable —— shape/corners/solid/stroke/gradient/padding/size

3、StateListDrawable ——  pressed/focused/selected/checked/enabled

4、LevelListDrawable/LayerDrawable/InsetDrawable/ScaleDrawable/ClipDrawable


## 动画
---

1、View动画：translate， sclale， rotate， alpha

  LayoutAnimation，activity(overridePendingTransition)，fragment(setCustomAnimations)
  
2、帧动画：animation-list

3、动画集合：AnimationSet

4、属性动画：ValueAnimator  ObjectAnimator(ofFloat/ofInt/ofObject) 

  内部封装PropertyValuesHolder保存动画过程中属性与值

  插值器/估值器

5、基于物理的动画：spring/fling animation  Lottie库
  


## 内存泄露
---

1、单例模式，静态变量

2、handler，线程，非静态内部类

3、资源未关闭，webview

## ListView
---

1、优化：ViewHolder，getView

2、recycleView：ViewHolder，layoutManager，recycler


## Handler
---

**1、Looper：**

Looper.prepareLooper()：threadLocal.set ，Looper()初始化消息队列/线程，ActivityThread.main()中调用

Loop.lopp()：queue.next() -> msg.target.dispatchMessage(msg) -> msg.recycle()

Handler.dispatchMessage()：mCallback不为空则post()，为空则sendMessage()


**2、MessageQueue**

enqueueMessage：同步入队(比较when)，neewake（native方法唤醒next）

next：nativePollOnce阻塞线程，消息屏障判断，message时间判断，处理不紧急任务

消息屏障：优先处理异步消息（例如UI绘制消息）


**3、使用**

创建Handler：绑定当前线程对应消息队列

sendmessage:enqueueMessage同步入队

Message创建：通过对象池








