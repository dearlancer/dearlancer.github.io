---
layout: post
title: 自定义Dialog
excerpt: “”
categories: [android]
comments: true
---


## Dialog代码,使用Builder模式构建
```ruby
public class XxDialog extends Dialog {
    private Context mContext;
    private String mTitle;
    private String mContent;

    protected XxDialog(Context context) {
        super(context, R.style.marketdialog);
        this.mContext = context;
    }

    public void setmTitle(String mTitle) {
        this.mTitle = mTitle;
    }


    public void setmContent(String mContent) {
        this.mContent = mContent;
    }

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.dialog_layout);
        getWindow().setGravity(Gravity.CENTER);   //设置dialog位置
        WindowManager.LayoutParams layoutParams = getWindow().getAttributes();
        layoutParams.width = WindowManager.LayoutParams.WRAP_CONTENT;
        layoutParams.height = WindowManager.LayoutParams.WRAP_CONTENT;
        getWindow().setAttributes(layoutParams);
        setupView();
    }
    public static class Build {
            private Context mContext;
            private String mTitle;
            private String mContent;
            public Build(final Context mContext) {
            this.mContext = mContext;
        }

        public Build setmContent(String mContent) {
            this.mContent = mContent;
            return this;
        }

        public Build setmTitle(String mTitle) {
            this.mTitle = mTitle;
            return this;
        }

        public void setImgId(@DrawableRes int imgId) {
            this.imgId = imgId;
        }

        private XxDialog create() {
            XxDialog dialog = new XxDialog(mContext);
            dialog.setmTitle(mTitle != null ? mTitle : mContext.getResources().getString(R.string.dialog_title));
            dialog.setImgId(imgId != 0 ? imgId : R.drawable.dialog_icon);
            dialog.setmContent(mContent != null ? mContent : mContext.getResources().getString(R.string.dialog_content));
            return dialog;
        }

        public XxDialog show() {
            final XxDialog dialog = create();
            dialog.show();
            return dialog;
        }
    }
}
```


## Dialog主题属性
1.设置Dialog标题,true:显示,false:不显示 

```ruby
<item name="android:windowNoTitle">true</item>
```
2.设置对话框的背景颜色   

```ruby
 <item name="android:background">@android:color/white</item>
 ```
3.设置点击dialog以外的区域，dialog是否消失 true:消失,false:不消失
```ruby
<item name="android:windowCloseOnTouchOutside">false</item>
```
4.是否允许对话框的背景变暗 true:变暗 false:不改变   
```ruby
<item name="android:backgroundDimEnabled">false</item>
```
5.对话框的背景变暗的程度，值越大越暗    
```ruby
<item name="android:backgroundDimAmount">0.3</item>
```
6.dialog的动画效果      
```ruby
<item name="android:windowAnimationStyle">@style/animation</item>
进入/退出动画 
<style name="animation">
    <item name="windowEnterAnimation">@anim/dialog_enter</item>
    <item name="windowExitAnimation">@anim/dialog_exit</item>
</style>
```
7.设置dialog的背景颜色    
```ruby
 <item name="android:windowBackground">@android:color/holo_red_dark</item>
 ```
8.Dialog的windowFrame框  
值为null时，Dialog边框显示为透明   
```ruby
<item name="android:windowFrame">@android:color/holo_red_dark</item>
```
9.是否悬浮在activity上   
```ruby
<item name="android:windowIsFloating">true</item>
```