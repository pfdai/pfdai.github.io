---
layout: post
title:  "Volley的扩展"
categories: Volley
---

## Volley使用Post方法 ##

volley默认只支持Get方法的请求，需要通过重载Request的getParams()方法，将Post的内容传入Request中，同时将Request的请求方法设置为Method.POST

        StringRequest stringRequest = new StringRequest(Request.Method.POST, url,
                new Response.Listener<String>() {

                    @Override
                    public void onResponse(String s) {
                        vh.onSuccess(s);
                    }
                },
                new Response.ErrorListener() {

                    @Override
                    public void onErrorResponse(VolleyError volleyError) {
                        vh.onFailure(1);
                    }
                }) {

            @Override
            protected Map<String, String> getParams() {
                return para.getPara();
            }
        };

## Request请求的缓存 ##

# 自定义缓存机制 #

[Android Volley + JSONObjectRequest Caching](http://stackoverflow.com/questions/16781244/android-volley-jsonobjectrequest-caching)
默认的页面缓存机制来自于服务器所传输的Header头部内容，根据这些内容来处理缓存的机制 Cache-Control
需要重构Request对应的parseNetworkResponse方法

# 需要重载的对应代码 #

对应的源代码是通过判断 request.shouldCache(), 获取具体内容的键值对为 request.getCacheKey(),默认的key为mUrl，但
post请求是包含body的所有这里需要重载post请求来支持缓存。


        // If the request is uncacheable, skip the cache queue and go straight to the network.
        if (!request.shouldCache()) {
            mNetworkQueue.add(request);
            return request;
        }

        // Insert request into stage if there's already a request with the same cache key in flight.
        synchronized (mWaitingRequests) {
            String cacheKey = request.getCacheKey();
            if (mWaitingRequests.containsKey(cacheKey)) {
                // There is already a request in flight. Queue up.
                Queue<Request<?>> stagedRequests = mWaitingRequests.get(cacheKey);
                if (stagedRequests == null) {
                    stagedRequests = new LinkedList<Request<?>>();
                }
                stagedRequests.add(request);
                mWaitingRequests.put(cacheKey, stagedRequests);
                if (VolleyLog.DEBUG) {
                    VolleyLog.v("Request for cacheKey=%s is in flight, putting on hold.", cacheKey);
                }
            } else {
                // Insert 'null' queue for this cacheKey, indicating there is now a request in
                // flight.
                mWaitingRequests.put(cacheKey, null);
                mCacheQueue.add(request);
            }
            return request;
        }

## 图片的内存和本地缓存 ##

Volley对于图片的处理，有ImageRequest、ImageLoader等，ImageLoader是管理协调大量的ImageRequest的类；
通过引入DiskLruCache和BitmapLruCache的方式可以实现图片在内存和本地两层缓存机制

需要重载的方法为 getBitmap和setBitmap

    @Override
    public Bitmap getBitmap(String urlKey) {
        // 先从内存中获取
        Bitmap bitmap = get(urlKey);

        // 内存中没有，从本地获取
        if (bitmap == null && mDiskLruCache != null && !mDiskLruCache.isClosed()) {
            String key = Md5.md5(urlKey);
            try {
                if (mDiskLruCache.get(key) != null)
                    bitmap = getBitmapFromDiskLruCache(key);
            } catch (IOException e) {
                e.printStackTrace();
            }
        }

        return bitmap;
        // return mCache.get(url);
    }

    @Override
    public void putBitmap(String urlKey, Bitmap bitmap) {
        put(urlKey, bitmap);

        // 放入本地缓存
        String key = Md5.md5(urlKey);
        try {
            if (bitmap != null && mDiskLruCache != null && !mDiskLruCache.isClosed() && null == mDiskLruCache.get(key)) {
                putBitmapToDiskLruCache(key, bitmap);
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
    }




