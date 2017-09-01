---
layout: post
title: recycleView多接口适配器实现
excerpt:大多数时候,我们在实现recycleView的时候，都会封装适配器实现网络请求接口，子类只需要实现这个接口，填入要请求的地址就可以获取对应的数据，但是这样也有弊端，通常情况下，一个适配器就只能请求一个网络接口，需要请求两个以上接口时，这样的适配器就不太适用了。
categories: 
- android
- 适配器
comments: true
---
# 实现原理
先从recycleView.Adapter的实现说起,新建一个MyAdapter继承自RecyclerView.Adapter,系统会自动为我们成需要完成的函数实现，为了区分绑定的ViewHolder,我们还可以重写getItemViewType
```ruby
public class MyAdapter  extends RecyclerView.Adapter<RecyclerView.ViewHolder>{
    @Override
    public RecyclerView.ViewHolder onCreateViewHolder(ViewGroup parent, int viewType) {
        return null;
    }

    @Override
    public void onBindViewHolder(RecyclerView.ViewHolder holder, int position) {

    }

    @Override
    public int getItemCount() {
        return 0;
    }
    @Override
    public int getItemViewType(int position) {

    }
}
```