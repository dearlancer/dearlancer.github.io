---
layout: post
title: SharedPreferences的一切
excerpt: "SharedPreferences是Android平台上一个轻量级的存储类，用来保存应用的一些常用配置，比如Activity状态，Activity暂停时，将此activity的状态保存到SharedPereferences中；当Activity重载，系统回调方法onSaveInstanceState时，再从SharedPreferences中将值取出"
categories: [android]
comments: true
---

## 简介
SharedPreferences是Android平台上一个轻量级的存储类，用来保存应用的一些常用配置，比如Activity状态，Activity暂停时，将此activity的状态保存到SharedPereferences中；当Activity重载，系统回调方法onSaveInstanceState时，再从SharedPreferences中将值取出。
## 原理
SharedPreferences存储数据的本质是在本地新建一个xml文件，以键对值的方式存储数据,文件存放在/data/data/<package name>/shared_prefs目录下
## 使用
一个简单的栗子
```ruby
SharedPreferences sharedPreferences = getSharedPreferences("lizi", Context.MODE_PRIVATE);	//新建名为lizi的xml文件
Editor editor = sharedPreferences.edit();	//获取编辑器
editor.putString("name","sharedPreferences");	//写入string数据
editor.putInt("value",10086);   //写入int数据
editor.commit();	//提交
```

## 方法详解
### 实例化（单例）
```ruby
SharedPreferences sharedPreferences = PreferenceManager.getDefaultSharedPreferences(context);
SharedPreferences sharedPreferences = context.getSharedPreferences("lizi", Context.MODE_PRIVATE);
```
前者内部也是调用后者来实现的，只不过前者默认新建一个包名+"_preferences"的xml文件
这里我们知道，有关文件读写操作的函数其实都是由contextImpl来实现的，所以真正调用的是contextImpl的函数
参数:
context.getSharedPreferences(String name,int mode);
name就是需要新建的xml文件的名字，第二个参数为操作模式，共有以下四种
1. MODE_APPEND: 追加方式存储
2. MODE_PRIVATE: 私有方式存储,其他应用无法访问
3. MODE_WORLD_READABLE: 表示当前文件可以被其他应用读取
4. MODE_WORLD_WRITEABLE: 表示当前文件可以被其他应用写入
5. MODE_MULTI_PROCESS: 适用于多进程访问(3.0以后已经不在支持，google官方推荐使用ContentProvider来实现进程间共享访问)
### SharedPreferences内部的操作函数
* Map<String, ?> getAll();
* String getString(String key, @Nullable String defValue);
* Set<String> getStringSet(String key, @Nullable Set<String> defValues);
* int getInt(String key, int defValue);
* long getLong(String key, long defValue)
* float getFloat(String key, float defValue);
* boolean getBoolean(String key, boolean defValue);
* boolean contains(String key);	//判断key相对的键对值是否存在
* Editor edit();	//获取Editor对象,所有的写操作都是通过Editor对象进行操作的
* void registerOnSharedPreferenceChangeListener(OnSharedPreferenceChangeListener listener); //注册监听函数
* void unregisterOnSharedPreferenceChangeListener(OnSharedPreferenceChangeListener listener);
//注册后必须移除防止内存泄漏
### Editor内部的操作函数
* Editor putString(String key, @Nullable String value);
* Editor putStringSet(String key, @Nullable Set<String> values);
* Editor putInt(String key, int value);
* Editor putLong(String key, long value);
* Editor putFloat(String key, float value);
* Editor putBoolean(String key, boolean value);
* Editor remove(String key);	//删除key对应的值
* Editor clear();	//移除commit或者apply之前所有操作
* boolean commit();		//提交数据并同步提交到硬盘
* void apply();		//原子性提交内容到内存，等所有提交完成后再提交到硬盘中
* mark！ 注意commit和apply的区别：
	1. commit提交是否成功有返回值而apply没有
	2. apply是将修改数据原子提交到内存, 而后异步真正提交到硬件磁盘, 而commit是同步的提交到硬件磁盘，因此，在多个并发的提交commit的时候，他们会等待正在处理的commit保存到磁盘后在操作，从而降低了效率。而apply只是原子的提交到内容，后面有调用apply的函数的将会直接覆盖前面的内存数据，这样从一定程度上提高了很多效率。 
	3. apply方法不会提示任何失败的提示。

## 特别需要注意的点
以上的操作如果都是在同一个进程进行的，那么不会有什么问题，但是SharedPreferences对文件的读写没有实现AIDL方式，所以在不同的进程中进行操作，取得的对象就会不同，相应的，取得的值也会不同。
举个栗子
```ruby
<!--声明两个在不同线程中的activity-->
<activity android:name=".MainActivity">
            <intent-filter>
                <action android:name="android.intent.action.MAIN" />

                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>
        </activity>
<activity android:name=".OtherActivity"
            android:process=":other"/>
```
MainActivity中的代码
```ruby
public class MainActivity extends AppCompatActivity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        SharedPreferences sharedPreferences= PreferenceManager.getDefaultSharedPreferences(this);
        sharedPreferences.edit().putInt("key",10086).commit();
    }
    public void change(View view){
        SharedPreferences sharedPreferences= PreferenceManager.getDefaultSharedPreferences(this);
        sharedPreferences.edit().putInt("key",66666).commit();
        Log.i("mainActivity",sharedPreferences.getInt("key",-1)+"");
    }
    public void startOther(View view){
        Intent intent=new Intent(this,OtherActivity.class);
        startActivity(intent);
    }
}
```
OtherActivity中的代码
```ruby
protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_other);
    }
public void getText(View view){
        SharedPreferences sharedPreferences= PreferenceManager.getDefaultSharedPreferences(this);
        Log.i("otherActivity",sharedPreferences.getInt("key",-1)+"");
    }
```
原理很简单，在mainactivity中首先通过SharedPreferences写入一个值10086
启动otherActivity来读取这个值并打印
在回到mainActivity中修改这个值，在otherActivity中获取这个值并打印
other中第一次获取的值
```ruby
05-09 16:10:58.986 16994-16994/com.spring.sptest:other I/otherActivity: 10086
```
回到mainActivity中改变里面的值后otherActivity再次获取值
```ruby
05-09 16:12:28.629 16098-16098/com.spring.sptest I/mainActivity: 66666
05-09 16:12:31.548 16994-16994/com.spring.sptest:other I/otherActivity: 10086
```
可以看到，otherActivity中的值并没有做相应的改变
由此我们可以得出结论：     
<strong>SharedPreferences对值的操作并非进程同步的</strong>      
原因就是因为不同进程会分别对SharedPreferences做缓存操作以避免复杂的io操作，整个SharedPreferences的获取流程大概如下
```ruby
if(缓存中存在相应的SharedPreferences&&mode==MODE_MULTI_PROCESS&&version<Android3.0){
	直接返回SharedPreferences对象
}else{
	从硬盘中读取xml文件并返回SharedPreferences对象
}
```
由此可以分析上述代码，由于在内存中已经做了缓存造作，所以otherActivity所在的进程直接返回了缓存中的SharedPreferences对象，导致了读取的值没有改变。解决办法，重启进程重新从硬盘读取xml文件



