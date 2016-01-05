---
layout: post
title:  "响应式编程 ReactiveX"
date:   2016-01-05 15:07:22 +0800
categories: RxJava
---

响应式编程 ReactiveX
=====

[响应式编程官方介绍](http://reactivex.io/intro.html)

[给 Android 开发者的 RxJava 详解](http://gank.io/post/560e15be2dca930e00da1083#toc_16)

### 1. RxJava 的观察者模式大致如下图： ###
![RxJava 观察者模式](http://7xn2zf.com1.z0.glb.clouddn.com/rxjavarxjava_3.png)

### 2. RxJava 线程控制 —— Scheduler ###
RxJava 已经内置了几个 Scheduler:
Schedulers.io(); Schedulers.computation(); AndroidSchedulers.mainThread();

subscribeOn(): 指定 subscribe() 所发生的线程，即 Observable.OnSubscribe 被激活时所处的线程。或者叫做事件产生的线程。

observeOn(): 指定 Subscriber 所运行在的线程。或者叫做事件消费的线程。


    Observable.just(1, 2, 3, 4)
    .subscribeOn(Schedulers.io()) // 指定 subscribe() 发生在 IO 线程
    .observeOn(AndroidSchedulers.mainThread()) // 指定 Subscriber 的回调发生在主线程
    .subscribe(new Action1<Integer>() {
        @Override
        public void call(Integer number) {
            Log.d(tag, "number:" + number);
        }
    });
        

### 3. 两个例子，RxJava 分别替代 AsyncTask和Handler ###

#### AsyncTask的实现 ####
checkRange调用成功后，将数据本地化后再开始上传

    ArrayList<Date> supplements = supplement((ArrayList<Date>) obj, startTime, endTime);
    BackupUntilNowTask task = new BackupUntilNowTask().init(supplements, "" + uid, pCode, new SucceedAndFailedHandler() {

    @Override
    public void onSuccess(Object obj) {
    upload(true, Constants.REMAINPERCENT);
    }

    @Override
    public void onFailure(int errorCode) {
    mCurrentBackupState.backupFailed(BatchDataManager.this);
    }
    });
    task.execute();

BackupUntilNowTask的代码

    private class BackupUntilNowTask extends AsyncTask<Void, Void, Void> {
        String uid = null;
        String pCode = null;
        SucceedAndFailedHandler handler = null;
        ArrayList<Date> dates = null;

        public BackupUntilNowTask init(ArrayList<Date> dates, String id, String code, SucceedAndFailedHandler handler) {
            uid = id;
            pCode = code;
            this.handler = handler;
            this.dates = dates;

            return this;
        }

        @Override
        protected Void doInBackground(Void... params) {
            for (Date date : dates)
                addBatchedDataToUploading(date.startOfCurrentDayInGMT8(), date.startOfCurrentDayInGMT8().twentyFourHoursNext(), uid, pCode);
            Date now = Date.now().startOfCurrentDayInGMT8();
            setNeedBackupTime(now.getTime());
            addBatchedDataToUploading(now, now.twentyFourHoursNext(), uid, pCode);
            return null;
        }

        @Override
        protected void onPostExecute(Void result) {
            handler.onSuccess(null);
            // BatchDataUploaderSetting.getInstance().setAlarm();
        }

    }

#### RxJava的方式进行替代后的代码效果 ####

    Observable.just(mNeedBackupTime).observeOn(Schedulers.io()).subscribe(new Subscriber<Long>() {
            @Override
            public void onCompleted() {
                // 跨天的处理，不考虑状态
                startUploading();
            }

            @Override
            public void onError(Throwable e) {

            }

            @Override
            public void onNext(Long startTime) {
                // 已经备份到的时间
                Date timeForBackupDate = Date.dateWithMilliSeconds(startTime);
                // 当前的时间
                Date endTime = Date.now().startOfCurrentDayInGMT8();

                while (timeForBackupDate.before(endTime)) {
                    // 将这天加入备份中
                    addBatchedDataToUploading(timeForBackupDate, timeForBackupDate.twentyFourHoursNext(), UserUtil.userId() + "", UserUtil.deviceId());

                    // 备份记录往后移一天
                    setNeedBackupTime(timeForBackupDate.twentyFourHoursNext().getTime());
                    timeForBackupDate = timeForBackupDate.twentyFourHoursNext();
                }

            }
            });

-------------------------------------------------------------------------------
#### Handler的实现 ####

ViewPagerFragment中对页面上圆环更新时的实现

    public MyHandler mHandler;

    /**
     * 接受消息,处理消息 ,此Handler会与当前主线程一块运行
     */
    public static class MyHandler extends Handler {
        private WeakReference<ViewPagerFragment> mFragmentRef;

        public MyHandler(ViewPagerFragment fragment) {
            mFragmentRef = new WeakReference(fragment);
        }

        // 子类必须重写此方法,接受数据
        @Override
        public void handleMessage(Message msg) {
            if (mFragmentRef.get() == null)
                return;

            mFragmentRef.get().processMessage(msg);
        }
    }

通过发送handler的消息进行通信

    mHandler.sendEmptyMessage(MESSAGE_DAILYSTATS_UPDATE);

#### RxJava 替代方案 ####
直接运行该updateCircle函数，即可以实现圆环的更新

    public void updateCircle(final boolean isAnimator) {
        // 先获取DailyStats数据，然后进行UI的更新
        HomePageDataManager.getDailyStatsObservable(Date.now())
                .observeOn(AndroidSchedulers.mainThread())
                .subscribe(new Action1<DailyStats>() {
                    @Override
                    public void call(DailyStats o) {
                        mMinCircleView.updateData(o, isAnimator);
                    }
                });
    }

    public static Observable getDailyStatsObservable(Date date) {

        return Observable.just(date)
                .map(new Func1<Date, DailyStats>() {
                    @Override
                    public DailyStats call(Date date) {
                        return HomePageDataManager.getDailyStatsByDate(date);
                    }
                })
                .subscribeOn(Schedulers.io());
    }
