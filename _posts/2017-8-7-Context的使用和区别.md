---
layout: post
title: Context的使用和区别
excerpt: “ Context，中文直译为“上下文”，SDK中对其说明如下：
Interface to global information about an application environment. This is an abstract class whose implementation is provided by the Android system. It allows access to application-specific resources and classes, as well as up-calls 
for application-level operations such as launching activities, broadcasting and receiving intents, etc”
categories: [android]
comments: true
---


## Context作用
Context，中文直译为“上下文”，SDK中对其说明如下：
Interface to global information about an application environment. This is an abstract class whose implementation is provided by the Android system. It allows access to application-specific resources and classes, as well as up-calls for application-level operations such as launching activities, broadcasting and receiving intents, etc
从中可得，它的作用如下
1.描述应用程序的环境信息,即上下文
2.该类是一个抽象(abstract class)类，Android提供了该抽象类的具体实现类(后面我们会讲到是ContextIml类)
3.可以进行一些应用级别级别操作，例如启动acticity，service，接收intent信息等
4.获取应用资源和类

## Context主要方法
1.getAssets() //获取AssetManager对象，可以加载asset文件夹下面的资源
2.getResources() //获取Resources资源，可以加载value文件下的资源
3.getPackageManager() //获取PackageManager对象，获取包相关信息
4.getMainLooper(). //获取主线程,可以对线程进行操作，但是不建议使用
5.getApplicationContext() //获取Application级别的context
6.getXXX(Id).  //获取一些资源，本质是调用Resources的一些方法
7.startXxx() //启动某个组件
8.checkPemission()  //检测权限

## Application,Activity,Service三种Context的区别
一个应用中，Context的数量=Activity数量+Service数量+1(Application的Context)
Application的Context生命周期同应用相同，相当于一个全局变量，所以在一些需要长期存在或者生命周期不固定的对象中，可以使用Applicatiion的Context来调用系统资源
Activity的Context生命伴随Activity的onCreate()函数而生成，onDestory()后销毁,如果对象的生命周期同Activity，建议首先使用Acitivty的Context;
Service的Context在startService()或者bindService()被调用时创建,生命周期同Service,在程序退出时，Service依然可能存在

## 在使用Context进行应用级别的操作时需要注意的点
1.调用startActivity方法，首先建议使用Activity的Context因为这会给新建的Activity新建栈信息，如果使用Application或者Service的来启动，需要手动添加栈信息,intent.addFlag(ACTIVITY_NEW_STACK)；
2.Dialog传入的Context不能为Application或者Service的Context，Dialog是依赖于Window存在的，而window是和Activity想对应的，如果传入其他Context，生命周期就会难以同步


