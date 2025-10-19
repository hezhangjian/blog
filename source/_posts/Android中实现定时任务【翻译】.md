---
title: Android中实现定时任务【翻译】
link:
date: 2016-09-08T09:34:23
tags:
---
原文：[http://android-developers.blogspot.sg/2007/11/stitch-in-time.html?m=0](https://link.jianshu.com/?t=http://android-developers.blogspot.sg/2007/11/stitch-in-time.html?m=0)

1.利用TimerTask实现任务的定时执行 

```java
        TextView hezhangjian;
        int count = 0;//用于计数
        @Override
        protected void onCreate(Bundle savedInstanceState) {
            super.onCreate(savedInstanceState);
            setContentView(R.layout.activity_main);
            hezhangjian = (TextView) findViewById(R.id.hezhangjian);
            Timer timer = new Timer();//新建一个Timer
            timer.schedule(new UpdateTimeTask(),100,200);
            //通过schedule方法执行一个TimerTask，Timertask是一个抽象类，必须重写它的run方法。
            //task,long a,long b代表的是先等待a毫秒的延迟执行任务，然后每次等待大约b时间执行任务。
        }
        class UpdateTimeTask extends TimerTask{

            @Override
            public void run() {
                count++;
                runOnUiThread(new Runnable() {
                    @Override
                    public void run() {
                        hezhangjian.setText("这是"+"第"+count+"次");
                    }
                });
            }
        }
```
2.利用Handler实现定时任务的操作

  ```java
    TextView hezhangjian;
    int count = 0;
    private Handler mHandler;
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        mHandler = new Handler();//初始化handler
        hezhangjian = (TextView) findViewById(R.id.hezhangjian);
        mHandler.postDelayed(new UpdateTimeTask(),200);//延迟200，执行这个任务
    }
    class UpdateTimeTask extends TimerTask{
        @Override
        public void run() {
            count++;
            runOnUiThread(new Runnable() {
                @Override
                public void run() {
                    hezhangjian.setText("这是"+"第"+count+"次");//执行完毕
                    mHandler.postDelayed(new UpdateTimeTask(),100);//延迟100，再执行这个任务
                }
            });
        }
    }
  ```

如果你想要取消这个post事件，你可以使用handler的removeCallbacks(TimerTask task)方法。
