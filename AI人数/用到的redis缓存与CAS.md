## 用redis来做缓存队列，CAS保证线程安全  

###代码

在 FetchParamsController 类中（这是给图片端调用的接口），图片端同时调用该接口相当于生产者，将众多的待下载任务加入到redis缓存队列中。

```java
	@RequestMapping(value = "/photo_ready")
    public String fetchPhotosParmas(@RequestParam(value = "cameraCode") String cameraCode,
                                  @RequestParam(value = "photoUrl") String photoUrl,
                                  @RequestParam(value = "callTime") String callTime){
        if (cameraCode == null || photoUrl == null || callTime == null || cameraCode.equals("") || photoUrl.equals("") || callTime.equals("")){
            return "ERROR";
        }else{
            Long id = idGenerator.getId();
            PhotoData data = new PhotoData(id, cameraCode, photoUrl, changeDataFormat(Long.parseLong(callTime)), 0, 0, 0, "0");
            
            //加入到redis缓存队列中
            photoDataCacheService.addToDownLoadList(data);
            //使用线程池下载，executeConcurrent()的实现中用了CAS
            downloader.executeConcurrent();
            
            return "SUCCESS";
        }
    }
```

在 PictureDownloaderImpl 类下

```java
	/**
     * 并行执行下载任务：
     * 首先通过isBusy来判断是否有线程正在操作任务队列，如果有，则直接返回，如果没有则对任务队列中
     * 包含的所有任务依次进行出队、下载操作。此处使用了固定大小的线程池，由于此处是IO密集型的任务
     * 因此可以将线程池的数量设定的大一些。
     * 队列中的所有任务完成之后将isBusy置为false，代表当前没有线程访问任务队列。
     * @param
     */
    @Override
    public void executeConcurrent() {
        if(!isBusy.compareAndSet(false, true)) return;
        PhotoData currentTask;
        while ((currentTask = downloadList.getOneFromDownLoadList()) != null) {
            /**
             * 将插入数据库操作放到此处执行，以免在定时任务中取到的数据在缓存队列中仍旧存在
             * 新的任务加到缓存队列，等待执行
             * 定时任务从数据库中查询未下载任务，其中包含新任务
             * 造成新任务执行两次
             *
             */
            //将下载任务生成记录存入数据库
            photoDataService.add(currentTask);
            //使用线程池下载（消费者），下载成功会改变数据库中记录的标志位。
            fixedThreadPool.execute(new DownloadTask(currentTask));
        }
        isBusy.set(false);
    }
```

该方法相当于是消费者，通过CAS从redis队列中取出一个下载任务，进行下载。  

注意此处设计成先将待下载任务存入数据库中，在下载成功后将数据库相应记录的标志位进行改变。这样做的目的是在数据库中保存住图片在整个处理周期的各个阶段的状态。从而在出错后能直接判断是哪个阶段出错的。更重要的是，由于去远程图片端下载图片难免出错，如果出错了，数据库中的标志位没有改变，表示下载失败，此时由于系统中**存在一个定时任务每十分钟查询一次数据库中未被下载的记录重新下载**，这样就可以达到避免图片一次下载失败就漏掉该图片的情况发生。  

###思路

1、这样设计的好处是什么？

2、CAS的实现原理？

3、为什么使用redis，它的特点是什么？

4、改进方式？

**改进的地方：**当前代码是在每次线程池下载之前存入数据库，这其实是会影响效率的，在多线程下载的时候相当于要频繁写数据库，而且如果下载成功了，还要从数据库中查出该条记录并更新其标志位，这就相当于每次下载都要写数据库然后立马又读数据库再更改数据库，这对数据库的操作过于频繁。可以设计成存数据库之前先存入一个redis缓存队列，先存入缓存队列，如果下载成功则只需更改缓存队列中相应记录的标志位，缓存队列再慢慢存入数据库。这样就只需写一次数据库，避免了下载之后再读数据库，可以缓解在下载高峰期数据库的读写压力。

