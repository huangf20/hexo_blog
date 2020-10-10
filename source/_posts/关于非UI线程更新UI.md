---
title: 关于非UI线程更新UI
date: 2020-10-09 11:45:50
summary: 通常的Android开发中不会在非UI线程中更新UI，突然碰到一种情况在子线程更新UI也不会报错，现在就来稍微探索一下其中缘由。
categories: Android
tags:
  - Android
  - 多线程
---
代码如下，可以正常运行不会报错，注释中的代码加入后，报错。
{% codeblock lang:java %}
protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        mTextView=findViewById(R.id.tv_hello);
        mTextView.setOnClickListener(this);
        new Thread()
        {
            @Override
            public void run()
            {
                /*try {
                    Thread.sleep(2000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }*/
                mTextView.setText("10");
            }
        }.start();
    }
{% endcodeblock %}

报错信息：
{% asset_img 0001.png 报错信息 %}

一步步看，TextView中，setText()方法会更新UI，重点在于setText()方法中的：
{% asset_img 0002.png setText() %}

checkForRelayout()方法，即检查是否需要更新layout，断点调试可知，不执行Thread.sleep(2000)时，mLayout为null，执行了则mLayout有值，执行checkForRelayout()方法，以至于执行后面一系列View.requestLayout、ViewRootIml.checkThread方法。
ViewRootIml.checkThread方法：  

{% asset_img 0003.png checkThread() %}

  checkThread方法会检查当前线程是否为UI线程，如果不是，便抛出异常。  
  
  到这里，便可得出结论，这跟Activity的生命周期有关了，在onCreate()中直接开启子线程更新UI，此时View还没有画出来，用户不可见，即mLayout 为空，也就不会执行后面的方法，检查当前线程是否为UI线程了；  
    
  
  而子线程延时一段时间，此时OnResume和onStart都已经执行完成，View已经出来了，再更新UI就会执行TextView中checkForRelayout()和后面一系列方法，最后ViewRootIml的checkThread()，程序崩溃。Android要求只能在主UI线程更新UI，这是因为Android的单线程模型，View的修改若是支持多线程，那么带来的线程同步和线程安全问题难以解决。再有一点自己的理解，在一般的APP中，UI的更新都是在View绘制完成之后用户交互的时候的，那么在子线程更新UI必定会崩溃。