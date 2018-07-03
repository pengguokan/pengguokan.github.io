---
layout: post
title: Android实现布局自动宽高比
categories: [Android,UI]
description: 
keywords: UI
---

UI开发过程中，通常会遇到类似这样的场景：图片宽度占满屏幕，高度是宽度的xxx倍。一般情况下，我是这么解决的：
```java
View view = LayoutInflator.from(getContext()).inflate(R.layout.xxx, parent, false);
//为了获取LayoutParam，必须知道父容器的layout类型，否则Crash
FrameLayout.LayoutParam lp = (FrameLayout.LayoutParam)view.getLayoutParam();
//如果width已经是match_parent，那就按要求算出高度
int height = lp.getWidth() / 1.5;
//更新view的高度
lp.height = height;
```
 这种实现，一方面不直观，另外也要我们找到合适的初始化位置修改。

进一步思考：既然xml布局文件里边，可以用match_parent、wrap_content这种方式指定宽高，那能否增加宽高比的参数，来自动计算呢？答案是肯定的。

首先了解下View的正常宽高是如何计算的。

View的绘制包括Measure、Layout和Draw三部分，宽高的计算由Measure给出。
```java
private void performTraversals{
...
int childWidthMeasureSpec = getRootMeasureSpec(mWidth, lp.width);
int childHeightMeasureSpec = getRootMeasureSpec(mHeight, lp.height);
...
//执行测量流程
performanceMeasure(childWidthMeasureSpec, childHeightMeasureSpec);
...
//执行布局流程
performLayout(lp, desiredWindowWidthm desiredWindowHeight);
...
//执行绘制流程
performDraw();
}
```
 MeasureSpec是一个32位整形，高2位表示测量模式：EXACTLY（精确测量）、AT_MOST（最大值测量）还有一个不常见的UNSPECIFIED。

EXACTLY：当该View的layout_width或者layout_height指定为具体数值或者match_parent时候，表示父容器已经决定了子View的精确大小，这种模式下View的测量值就是SpecSize的值。

AT_MOST：当该View的layout_width和layout_height指定为wrap_content时生效，此时子View的尺寸可以是不超过父容器允许的最大尺寸。

Measure操作用来计算View的实际大，从performMeasure开始：

```java
private void performanceMeasure(int childWidthMeasureSpec, int childHeightMeasureSpec){
...
mView.measure(childWidthMeasureSpec, childHeightMeasureSpec)
...
}
```
 具体的测量时分发给ViewGroup的，由ViewGroup在它的measureChild方法中传递给子View：ViewGroup通过遍历自身所有的子View，并逐个调用子View的measure方法；并且最终会通过onMeasure，回调开发者自定义实现：
```java
public final void meassure(int widthMeassureSpec, int heightMeasureSpec){
...
onMeassure(widthMeassureSpec, heightMeasureSpec);
...
}
```
 所以如果要更改View的宽高，可以自己实现Layout的onMeasure方法。

为了能够用xml指定宽高比，需要定义属性。在res/value目录下新建一个属性文件：
```xml
<resources>
    <declare-styleable name="ExtensionLayout">
        <attr name="width_to_height" format="float"/>
        <attr name="calc_by_width" format="boolean"/>
    </declare-styleable>
</resources>
```
其中width_to_height用来代表宽高比，calc_by_width指定了是按宽固定或是高固定计算另外一边的值。

以FrameLayout为例，我们可以这么实现：
```java
public class ExtensionFrameLayout extends FrameLayout {
    private float mWidthToHeight = 1.0f;
    private boolean mCalcByWidth = true;

    public ExtensionFrameLayout(Context context) {
        super(context);
    }

    public ExtensionFrameLayout(Context context, AttributeSet attrs) {
        super(context, attrs);
        TypedArray array = context.obtainStyledAttributes(attrs, R.styleable.ExtensionFrameLayout);
        mWidthToHeight = array.getFloat(R.styleable.ExtensionLayout_width_to_height, 1.0f);
        mCalcByWidth = array.getBoolean(R.styleable.ExtensionLayout_calc_by_width, true);
        mCornerRadius = 
        array.recycle();
    }

    public ExtensionFrameLayout(Context context, AttributeSet attrs, int defStyleAttr) {
        super(context, attrs, defStyleAttr);
    }

    @Override
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        if(mWidthToHeight > 0.0f){
            if(mCalcByWidth){
                int screenHeight = DeviceManager.getScreenHeight(getContext());
                int height = (int) (View.MeasureSpec.getSize(widthMeasureSpec) / mWidthToHeight);
                if(height > screenHeight){
                    height = screenHeight;
                }
                heightMeasureSpec = View.MeasureSpec.makeMeasureSpec(height, View.MeasureSpec.EXACTLY);
            }else{
                int screenWidth = DeviceManager.getScreenWidth(getContext());
                int width = (int) (View.MeasureSpec.getSize(heightMeasureSpec) / mWidthToHeight);
                if(width > screenWidth){
                    width = screenWidth;
                }
                widthMeasureSpec = View.MeasureSpec.makeMeasureSpec(width, View.MeasureSpec.EXACTLY);
            }
        }
        super.onMeasure(widthMeasureSpec, heightMeasureSpec);
    }
}
```
 在使用的时候，只要添加这两个参数，就可以轻松的实现宽高比：
![](/images/posts/auto_calc.png)


## 总结

核心是通过重写Layout的onMeasure方法，通过xml传入参数，计算好宽高。这样可以帮助开发者，专注其他逻辑，不用关心布局的调整。
