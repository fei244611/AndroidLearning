# Android View机制

## View绘制：
---

1、requestRootImpl.requestLayout

线程检查

scheduleTraversals()：mTraversalScheduled（去除重复绘制）, 
   Choreographer提交绘制任务doTraversal()，Handler post异步消息屏障

Handler回调performTraversals：performMeasure,performLayout,performDrow

#### 2、Measure：

**1）MeasureSpec获取：**

SpecMode+SpecSizeSpecMode+SpecSize（EXACTLY，AT_MOST，UNSPECIFIED）

UNSPECIFIED：用于NestedScrollView和ScrollView

ViewRootImpl.performTraversals()
ViewGroup.measureChildWithMargins()

```
childwidthMeasureSpec = getRootMeasureSpec(desiredWindowWidth, lp.width);
childHeightMeasureSpec = getRootMeasureSpec(desiredWindowHeight, lp.width);
performMeasure(childwidthMeasureSpec,childheightMeasureSpec);
```


**2）调用流程：**

View.measure() -> View.onMeasure()(setMeasuredDimension/getDefaultSize) -> ViewGroup.measureChidren

ViewGroup类提供了measureChildren, measureChild, measureChildWithMargins方法，简化了父子View的尺寸计算。
measureChildren内部实质只是循环调用measureChild，measureChild和measureChildWithMargins的区别是margin和padding也作为子视图的大小

```
    // onMeasure默认实现，通过getDefaultSize对成员变量mMeasuredWidth和mMeasuredHeight进行赋值
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        setMeasuredDimension(getDefaultSize(getSuggestedMinimumWidth(), widthMeasureSpec),
                getDefaultSize(getSuggestedMinimumHeight(), heightMeasureSpec));
    }
    
    public static int getDefaultSize(int size, int measureSpec) {
        int result = size;
        int specMode = MeasureSpec.getMode(measureSpec);
        int specSize = MeasureSpec.getSize(measureSpec);

        switch (specMode) {
        case MeasureSpec.UNSPECIFIED:
            result = size;
            break;
        case MeasureSpec.AT_MOST:
        case MeasureSpec.EXACTLY:
            result = specSize;
            break;
        }
        return result;
    }
    
```


#### 3、Layout过程：

layout（setFrame确定四个顶点位置） onlayout(具体view重写，setChildFrame调用子layout方法)

```
// ViewRootImpl.java
private void performLayout(WindowManager.LayoutParams lp, int desiredWindowWidth, int desiredWindowHeight) {
    ...
    host.layout(0, 0, host.getMeasuredWidth(), host.getMeasuredHeight());
    ...
}

// View.java
public void layout(int l, int t, int r, int b) {
    ...
    // 通过setFrame方法来设定View的四个顶点的位置，即View在父容器中的位置
    boolean changed = isLayoutModeOptical(mParent) ? 
    set OpticalFrame(l, t, r, b) : setFrame(l, t, r, b);

    ...
    onLayout(changed, l, t, r, b);
    ...
}

// 空方法，子类如果是ViewGroup类型，则重写这个方法，通过setChildFrame调用子layout方法
protected void onLayout(boolean changed, int left, int top, int right, int bottom) {

}

```


#### 4、Draw过程：

```
public void draw(Canvas canvas) {
    ...
    // 步骤一：绘制View的背景
    drawBackground(canvas);

    ...
    // 步骤二：如果需要的话，保持canvas的图层，为fading做准备
    saveCount = canvas.getSaveCount();
    ...
    canvas.saveLayer(left, top, right, top + length, null, flags);

    ...
    // 步骤三：绘制View的内容
    onDraw(canvas);

    ...
    // 步骤四：绘制View的子View
    dispatchDraw(canvas);

    ...
    // 步骤五：如果需要的话，绘制View的fading边缘并恢复图层
    canvas.drawRect(left, top, right, top + length, p);
    ...
    canvas.restoreToCount(saveCount);

    ...
    // 步骤六：绘制View的装饰(例如滚动条等等)
    onDrawForeground(canvas)
}
```

<img src="../image/c2f3fbe8.png" width="600"/>


## View事件分发

#### 1、ViewGroup.dispatchTouchEvent()流程

是否判断拦截：ACTION_DOWN， mFirstTouchTarget != null， FLAG_DISALLOW_INTERCEPT(不拦截标记位，子View requestDisallowInterceptTouchEvent)

拦截事件onInterceptTouchEvent：默认false，子类可重写

dispatchTransformedTouchEvent：调用view或子view.dispatchTouchEvent

```
    public boolean dispatchTouchEvent(MotionEvent ev) {
        ...
        
        boolean handled = false;
        if (onFilterTouchEventForSecurity(ev)) {
            final int action = ev.getAction();
            final int actionMasked = action & MotionEvent.ACTION_MASK;
            // down事件则重新开始事件流
            if (actionMasked == MotionEvent.ACTION_DOWN) {
                cancelAndClearTouchTargets(ev);
                resetTouchState();
            }

            // 判断是否拦截
            final boolean intercepted;
            if (actionMasked == MotionEvent.ACTION_DOWN
                    || mFirstTouchTarget != null) {
                final boolean disallowIntercept = (mGroupFlags & FLAG_DISALLOW_INTERCEPT) != 0;
                if (!disallowIntercept) {
                    intercepted = onInterceptTouchEvent(ev);
                    ev.setAction(action);
                } else {
                    intercepted = false;
                }
            } else {
                intercepted = true;
            }
            ...

            final boolean split = (mGroupFlags & FLAG_SPLIT_MOTION_EVENTS) != 0;
            TouchTarget newTouchTarget = null;
            boolean alreadyDispatchedToNewTouchTarget = false;
            if (!canceled && !intercepted) {
                ...
                  // 循环遍历子View
                  for (int i = childrenCount - 1; i >= 0; i--) 
                    ...
                    // 调用子View.dispatchTouchEvent,设置mFirstTouchTarget
                    dispatchTransformedTouchEvent(ev, false, child, idBitsToAssign)) 
                     ...
            }

            
            if (mFirstTouchTarget == null) {
                // 调用当前view.dispatchTouchEvent
                handled = dispatchTransformedTouchEvent(ev, canceled, null,
                        TouchTarget.ALL_POINTER_IDS);
            } else{
                // 除Down事件外的事件交由指定子View处理
                ...
            }
              ...
        return handled;
    }

  // 是否拦截事件，子类重写，默认数false 
  public boolean onInterceptTouchEvent(MotionEvent ev)
  
```

#### 2、View.dispatchTouchEvent

onTouchEvent()：Clickable或LongClickable为true则返回true，ACTION_UP事件会触发OnClickListener

```
public boolean dispatchTouchEvent(MotionEvent event) {
    ...
    // 判断OnTouchListener
    if (onFilterTouchEventForSecurity(event)) {
        ListenerInfo li = mListenerInfo;
        if (li != null && li.mOnTouchListener != null
                && (mViewFlags & ENABLED_MASK) == ENABLED
                && li.mOnTouchListener.onTouch(this, event)) {
            result = true;
        }

        if (!result && onTouchEvent(event)) {
            result = true;
        }
    }
    ...
    return result;
}


public boolean onTouchEvent(MotionEvent event) {
    ...
    if ((viewFlags & ENABLED_MASK) == DISABLED) {
        if (action == MotionEvent.ACTION_UP && (mPrivateFlags & PFLAG_PRESSED) != 0) {
            setPressed(false);
        }
        // 是否可点击
        return (((viewFlags & CLICKABLE) == CLICKABLE
                || (viewFlags & LONG_CLICKABLE) == LONG_CLICKABLE)
                || (viewFlags & CONTEXT_CLICKABLE) == CONTEXT_CLICKABLE);
    }
    ...
    
    if (((viewFlags & CLICKABLE) == CLICKABLE ||
            (viewFlags & LONG_CLICKABLE) == LONG_CLICKABLE) ||
            (viewFlags & CONTEXT_CLICKABLE) == CONTEXT_CLICKABLE) {
        switch (action) {
            case MotionEvent.ACTION_UP:
                // 判断是否调用onClickListenser()
                performClick();
            
            // 检查是否在scrollView内，设置点击态    
            case MotionEvent.ACTION_DOWN:
                setPressed(true, x, y);
                ... 
            // 重置状态    
            case MotionEvent.ACTION_CANCEL:
                ...
            // 判断点击区域是否在view内
            case MotionEvent.ACTION_MOVE:
                ...
        }
        return true;
    }
    return false;
}

```

3、滑动冲突：

外部拦截法：重写父类onInterceptTouchEvent方法

内部拦截法：重写父类onInterceptTouchEvent方法， 重写子类dispatchTouchEvent方法， 调用requestDisallowInterceptTouchEvent


## 自定义View

1、继承View：

重写onDraw(padding)

重写onMeasure(wrap_content)

```
    // wrap_parent 适配
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
    	//第一步:调用super.onMeasure()
    	super.onMeasure(widthMeasureSpec , heightMeasureSpec);
	    int widthSpecMode = MeasureSpec.getMode(widthMeasureSpec);
	    int widthSpceSize = MeasureSpec.getSize(widthMeasureSpec);
	    int heightSpecMode=MeasureSpec.getMode(heightMeasureSpec);
	    int heightSpceSize=MeasureSpec.getSize(heightMeasureSpec);
	    
	    //第二步:处理子View的大小为wrap_content的情况
	    if(widthSpecMode==MeasureSpec.AT_MOST&&heightSpecMode==MeasureSpec.AT_MOST){
	    	setMeasuredDimension(mWidth, mHeight);
	    }else if(widthSpecMode==MeasureSpec.AT_MOST){
	    	setMeasuredDimension(mWidth, heightSpceSize);
	    }else if(heightSpecMode==MeasureSpec.AT_MOST){
	    	setMeasuredDimension(widthSpceSize, mHeight);
	    } 
	 }
```


2、继承特定View

3、继承ViewGroup

需支持wrap_content， padding， margin

4、继承特定ViewGroup

#### 5、注意点：

自定义属性

wrap_content， padding， margin处理

多线程使用post

避免内存泄露（线程/动画停止）

滑动冲突

## Window / WindowManager

Window 是一个抽象类，它的具体实现是 PhoneWindow。WindowManager 是外界访问 Window 的入口，Window 的具体实现位于 WindowManagerService 中，WindowManager 和 WindowManagerService 的交互是一个 IPC 过程。Android 中所有的视图都是通过 Window 来呈现，因此 Window 实际是 View 的直接管理者。

| Window 类型 | 说明 | 层级
|--|--|--
| Application Window | 对应着一个 Activity | 1~99
| Sub Window | 不能单独存在，只能附属在父 Window 中，如 Dialog 等 | 1000~1999
| System Window | 需要权限声明，如 Toast 和 系统状态栏等 | 2000~2999


```java

// WindowManagerImpl.java
@Override
public void addView(@NonNull View view, @NonNull ViewGroup.LayoutParams params) {
    applyDefaultToken(params);
    mGlobal.addView(view, params, mContext.getDisplay(), mParentWindow);
}

@Override
public void updateViewLayout(@NonNull View view, @NonNull ViewGroup.LayoutParams params) {
    applyDefaultToken(params);
    mGlobal.updateViewLayout(view, params);
}

@Override
public void removeView(View view) {
    mGlobal.removeView(view, false);
}
```

# Bitmap
![](https://upload-images.jianshu.io/upload_images/2618044-cd996dd172cce293.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1000/format/webp)

## 配置信息与压缩方式
**Bitmap 中有两个内部枚举类：**
- Config 是用来设置颜色配置信息
- CompressFormat 是用来设置压缩方式

> 通常我们优化 Bitmap 时，当需要做性能优化或者防止 OOM，我们通常会使用 Bitmap.Config.RGB_565 这个配置，因为 Bitmap.Config.ALPHA_8 只有透明度，显示一般图片没有意义，Bitmap.Config.ARGB_4444 显示图片不清楚， Bitmap.Config.ARGB_8888 占用内存最多。



## 常用操作
### 裁剪、缩放、旋转、移动
```java
Matrix matrix = new Matrix();  
// 缩放 
matrix.postScale(0.8f, 0.9f);  
// 左旋，参数为正则向右旋
matrix.postRotate(-45);  
// 平移, 在上一次修改的基础上进行再次修改 set 每次操作都是最新的 会覆盖上次的操作
matrix.postTranslate(100, 80);
// 裁剪并执行以上操作
Bitmap bitmap = Bitmap.createBitmap(source, 0, 0, source.getWidth(), source.getHeight(), matrix, true);
````
> 虽然Matrix还可以调用postSkew方法进行倾斜操作，但是却不可以在此时创建Bitmap时使用。

### 保存与释放
```java
Bitmap bitmap = BitmapFactory.decodeResource(getResources(), R.drawable.test);
File file = new File(getFilesDir(),"test.jpg");
if(file.exists()){
    file.delete();
}
try {
    FileOutputStream outputStream=new FileOutputStream(file);
    bitmap.compress(Bitmap.CompressFormat.JPEG,90,outputStream);
    outputStream.flush();
    outputStream.close();
} catch (FileNotFoundException e) {
    e.printStackTrace();
} catch (IOException e) {
    e.printStackTrace();
}
//释放bitmap的资源，这是一个不可逆转的操作
bitmap.recycle();
```

### 图片压缩
```java
public static Bitmap compressImage(Bitmap image) {
    if (image == null) {
        return null;
    }
    ByteArrayOutputStream baos = null;
    try {
        baos = new ByteArrayOutputStream();
        image.compress(Bitmap.CompressFormat.JPEG, 100, baos);
        byte[] bytes = baos.toByteArray();
        ByteArrayInputStream isBm = new ByteArrayInputStream(bytes);
        Bitmap bitmap = BitmapFactory.decodeStream(isBm);
        return bitmap;
    } catch (OutOfMemoryError e) {
        e.printStackTrace();
    } finally {
            if (baos != null) {
                baos.close();
            }
    }
    return null;
}
```

Bitmap 类的构造方法都是私有的，所以开发者不能直接 new 出一个 Bitmap 对象，只能通过 BitmapFactory 类的各种静态方法来实例化一个 Bitmap。

生成 Bitmap 对象最终都是通过 JNI 调用方式实现的，所以需要调用 recycle() 方法来释放 C 部分的内存。

## 屏幕适配

1、dp适配

2、宽高限定符

3、头条适配方案


```java
private static void setCustomDensity(@NonNull Activity activity, @NonNull final Application application) {
    final DisplayMetrics appDisplayMetrics = application.getResources().getDisplayMetrics();
    if (sNoncompatDensity == 0) {
        sNoncompatDensity = appDisplayMetrics.density;
        sNoncompatScaledDensity = appDisplayMetrics.scaledDensity;
        // 监听字体切换
        application.registerComponentCallbacks(new ComponentCallbacks() {
            @Override
            public void onConfigurationChanged(Configuration newConfig) {
                if (newConfig != null && newConfig.fontScale > 0) {
                    sNoncompatScaledDensity = application.getResources().getDisplayMetrics().scaledDensity;
                }
            }

            @Override
            public void onLowMemory() {

            }
        });
    }
    
    // 适配后的dpi将统一为360dpi
    final float targetDensity = appDisplayMetrics.widthPixels / 360;
    final float targetScaledDensity = targetDensity * (sNoncompatScaledDensity / sNoncompatDensity);
    final int targetDensityDpi = (int)(160 * targetDensity);

    appDisplayMetrics.density = targetDensity;
    appDisplayMetrics.scaledDensity = targetScaledDensity;
    appDisplayMetrics.densityDpi = targetDensityDpi;

    final DisplayMetrics activityDisplayMetrics = activity.getResources().getDisplayMetrics();
    activityDisplayMetrics.density = targetDensity;
    activityDisplayMetrics.scaledDensity = targetScaledDensity;
    activityDisplayMetrics.densityDpi = targetDensityDpi
}
```

4、刘海屏适配
- Android P 刘海屏适配方案

Android P 支持最新的全面屏以及为摄像头和扬声器预留空间的凹口屏幕。通过全新的 DisplayCutout 类，可以确定非功能区域的位置和形状，这些区域不应显示内容。要确定这些凹口屏幕区域是否存在及其位置，使用 getDisplayCutout() 函数。

| DisplayCutout 类方法 | 说明
|--|--
| getBoundingRects() | 返回Rects的列表，每个Rects都是显示屏上非功能区域的边界矩形
| getSafeInsetLeft () | 返回安全区域距离屏幕左边的距离，单位是px
| getSafeInsetRight () | 返回安全区域距离屏幕右边的距离，单位是px
| getSafeInsetTop () | 返回安全区域距离屏幕顶部的距离，单位是px
| getSafeInsetBottom() | 返回安全区域距离屏幕底部的距离，单位是px

Android P 中 WindowManager.LayoutParams 新增了一个布局参数属性 layoutInDisplayCutoutMode：

| 模式 | 模式说明
|--|--
| LAYOUT_IN_DISPLAY_CUTOUT_MODE_DEFAULT | 只有当DisplayCutout完全包含在系统栏中时，才允许窗口延伸到DisplayCutout区域。 否则，窗口布局不与DisplayCutout区域重叠。
| LAYOUT_IN_DISPLAY_CUTOUT_MODE_NEVER | 该窗口决不允许与DisplayCutout区域重叠。
| LAYOUT_IN_DISPLAY_CUTOUT_MODE_SHORT_EDGES | 该窗口始终允许延伸到屏幕短边上的DisplayCutout区域。





