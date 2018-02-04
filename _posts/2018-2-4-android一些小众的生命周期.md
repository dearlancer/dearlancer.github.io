---
layout: post
title: android中一些小众的生命周期函数
excerpt: "介绍一些在activity中不常使用的小众生命周期函数"
categories:
- android
- 源码
comments: true
---

## Android中activity的生命周期
在android开发中，对生命周期的把握和理解是最基础也是最重要的知识点，以往使用中，我们使用的最多的一般是一些主流的生命周期函数，如下：
```ruby
onCreate()->onStart()->onResume()->onPause()->onstop()->onDestory()
```
其实，在activity中还有一些比较小众，但是用好了也很有裨益的生命周期函数。
* onAttachFragment(Fragment fragment) 该函数会在activity界面每一次重新显示的时候被调用到,在这个时候，你可以获取到对应的fragment引用，在activity中对fragment进行相应操作。
* onContentChanged 当activity的布局内容变化时被回调，可以在这里执行findView操作
* onPostCreate(@Nullable Bundle savedInstanceState) 会在每一次onCreate函数被执行的时候调用，执行时期为onStart和onResume之间，同onCreate一样，只在activity初始化的时候被执行一次。
* onPostResume() 同onPostCreate类似，在onResume执行完毕之后被回调，每一次activity的onresume执行该函数也会被执行，所以同onResume一样，避免在这里做一些初始化或者耗时操作。
* onAttachedToWindow() activity被添加至Window对象时被回调，只回调一次。
* onWindowFocusChanged(boolean hasFocus) 当前activity获取或者失去焦点的时候会被回调
* onUserInteraction() 任何用户操作比如点击，触摸，返回都会出发这个函数的回调
* onTrimMemory(int level) 我们可以根据传入的等级进行对应的内存释放操作，建议在这个函数中处理那些不需要常驻内存并且占比较大的对象资源，比如图片缓存等。 
* onLowMemory 在低内存的情况下会回调这个函数，在这里释放那些不必要或者占比较大的对象资源

所以整个activity的完整的生命周期如下:
```ruby
//正常进入activity然后点击返回退出
onCreate->onAttachFragment->onContentChanged->onStart->onPostCreate->onResume->onAttachTowindow->onWindowFocusChanged->onUserInteraction(双击返回)->onPause->onWindowFocusChanged->onStop->onDestroy->onDetachedfromWindow
```
![activity_life](/img/activity_life.png)
```ruby
//点击home键进入后台再进入
onWindowFocusChanged->onUserInteraction->onPause->onStop->onTrimMemory->onRestart->onStart->onResume->onPostResume->onWindowFocusChanged
```
![activity_pause](/img/activity_pause.png)

