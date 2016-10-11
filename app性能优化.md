##App性能优化
在Java中的每一个类(包括匿名内部类)都会使用大概500 bytes。每一个类的实例花销是12-16 bytes。往HashMap添加一个entry需要额一个额外占用的32 bytes的entry对象。
####1 . 使用SparseArray, SparseBooleanArray, 与 LongSparseArray代替HaspMap
####2 . 枚举类型通常是静态常量的2倍，尽量避免使用枚举
####3 . 尽少使用抽象
    通常他们需要同等量的代码用于可执行。那些代码会被map到内存中。因此如果你的抽象没有显著的提升效率，应该尽量避免他们。
####4 . 避免使用依赖注入框架
    依赖注入框架会通过扫描你的代码执行许多初始化的操作，这会导致你的代码需要大量的RAM来mapping代码，而且mapped pages会长时间的被保留在RAM中。
####5 . [优化整体性能](https://developer.android.com/training/best-performance.html)
####6 . 使用ProGuard进行代码混淆
    ProGuard能够通过移除不需要的代码，重命名类，域与方法等方对代码进行压缩，优化与混淆。使用ProGuard可以使得你的代码更加紧凑，这样能够使用更少mapped代码所需要的RAM。
####7 . 打包后的应用要使用zipalign工具进行优化
    在编写完所有代码，并通过编译系统生成APK之后，你需要使用zipalign对APK进行重新校准。如果你不做这个步骤，会导致你的APK需要更多的RAM，因为一些类似图片资源的东西不能被mapped。
#####zipalign使用方法
######1) . 检查程序包是否进行了对齐：zipalign -c -v 4 application.apk
#####2) . apk签名后，使用以下命令：zipalign -v 4 source.apk destination.apk
####8 . 使用多进程
    如果合适的话，有一个更高级的技术可以帮助你的app管理内存使用：通过	把你的app组件切分成多个组件，运行在不同的进程中。<font color="DD4444">这个技术必须谨慎使用，大多数app都不应该运行在多个进程中。因为如果使用不当，它会显著增加内存的使用，而不是减少。</font><font color="868CFE">当你的app需要在后台运行与前台一样的大量的任务的时候，可以考虑使用这个技术。</font><br>
    app可以切分成2个进程：一个用来操作UI，另外一个用来后台的Service.
    你可以通过在manifest文件中声明'android:process'属性来实现某个组件运行在另外一个进程的操作。<br>

    <service android:name=".PlaybackService"
           android:process=":background" />
           
           
####9 . 避免创建不必要的对象
     	通常来说，需要避免创建更多的临时对象。更少的对象意味者更少的gc动作，gc会对用户体验有比较直接的影响。   
####10 . 除ArrayList外，其它集合遍历时，推荐使用for-each 

比较下面三种循环的方法：

    static class Foo {
      int mSplat;
    }

    Foo[] mArray = ...

     public void zero() {
       int sum = 0;
         for (int i = 0; i < mArray.length; ++i) {
              sum += mArray[i].mSplat;
         }
    }

     public void one() {
        int sum = 0;
        Foo[] localArray = mArray;
        int len = localArray.length;

        for (int i = 0; i < len; ++i) {
            sum += localArray[i].mSplat;
        }
    }

    public void two() {
        int sum = 0;
        for (Foo a : mArray) {
            sum += a.mSplat;
        }
    }
zero()是最慢的，因为JIT没有办法对它进行优化.<br>
one()稍微快些。<br>
two() 在没有做JIT时是最快的，可是如果经过JIT之后，与方法one()是差不多一样快的。它使用了增强的循环方法for-each.<br>
<font color="868CFE">所以请尽量使用for-each的方法，但是对于ArrayList，请使用方法one().</font>
####11 . 其它 
使用包级访问而不是内部类的私有访问<br>
避免使用float类型<br>
使用库函数<br>
谨慎使用native函数<br>


         
##APP内存泄漏         
         
<br>
<br>
Eg:1

	StorageManager storageManager=(StorageManager)getSystemService(Context.STORAGE_SERVICE) ;
改成

    StorageManager storageManager= (StorageManager)getApplication().getSystemService(Context.STORAGE_SERVICE);

Eg:2

    NetworkStatsManager networkStatsManager=(NetworkStatsManager)getApplication().getSystemService(Context.NETWORK_STATS_SERVICE);
改成

    NetworkStatsManager networkStatsManager=(NetworkStatsManager)getSystemService(Context.NETWORK_STATS_SERVICE);
    
Handler是内部类，生命周期跟activiy周期不一致
Thread对象，activity生命周期结束了，thread还在，任务执行后回调ui，泄漏




###### 1 . 非静态成员变量随对象的释放而释放
###### 2 . 局部变量随方法结束释放
###### 3 . 静态成员变量随进程结束而释放。

####Bitmap在Android中的内存管理

######android2.2(level 8): 当系统回收无效数据时，app的线程是停止状态，导致卡顿，性能低
######android2.3.3(levle 10): 增加了并发回收机制，意味着当bitmap不再被使用的时候很快就被回收，性能有所提升;
######<font color=#DD2222>以上，Bitmap与它的pixel data（像素数据）是分离的，Bitmap存在native memory（本机内存）里，pixel data存在Dalvik heap（虚拟机的堆）里.<br>由于pixel data无法预测是否能够释放，所以可能会引起应用超过自身的内存限制，进而造城崩溃</font>
#######在此之前，推荐使用recycle() ，能尽快释放内存，当确定bitmap不再使用的时候调用recycle()释放；<font color=#DD222>如果释放之后，再去使用draw，会出现错误，Canvas: trying to use a recycled bitmap</font>
######Android 3.0 (API level 11) 通过BitmapFactory.Options.inBitmap方法实现了bitmap的重复利用，提升了性能；移除了内存的分配和释放方法<br>
######<font color=#DD222>在android4.4(level 19)以前，使用inBitmap还是有一定限制，只是支持同等大小的bitmap</font>

######android4.4(level 19):一个bitmap从LruCache中移除，bitmap的软引用替代bitmap放入HashSet中，解决了之前出现的不能重draw的问题
    Set<SoftReference<Bitmap>> mReusableBitmaps;
    private LruCache<String, BitmapDrawable> mMemoryCache;

    // If you're running on Honeycomb or newer, create a
    // synchronized HashSet of references to reusable bitmaps.
    if (Utils.hasHoneycomb()) {
  <font color=#DD222>
        mReusableBitmaps =
         Collections.synchronizedSet(newHashSet<SoftReference<Bitmap>>());
         </font>
    }

        mMemoryCache = new LruCache<String,BitmapDrawable>(mCacheParams.memCacheSize) {

    // Notify the removed entry that is no longer being cached.
    @Override
    protected void entryRemoved(boolean evicted, String key,
            BitmapDrawable oldValue, BitmapDrawable newValue) {
        if (RecyclingBitmapDrawable.class.isInstance(oldValue)) {
            // The removed entry is a recycling drawable, so notify it
            // that it has been removed from the memory cache.
            ((RecyclingBitmapDrawable) oldValue).setIsCached(false);
        } else {
            // The removed entry is a standard BitmapDrawable.
            if (Utils.hasHoneycomb()) {
                // We're running on Honeycomb or later, so add the bitmap
                // to a SoftReference set for possible use with inBitmap later.
                mReusableBitmaps.add
                        (new SoftReference<Bitmap>(oldValue.getBitmap()));
            }
        }
    }
    ....
    }

