# 撩一撩Ｂ妹(BitMap)

### 为什么要撩BitMap?

做android必然会撩到BitMap，撩过BitMap的都知道，BitMap可不是你想撩就能撩，她脾气可是很不好的，是一言不合就ＯＯＭ.So 学好撩妹姿势是多么的重要

### 历史演进

言归正传，BitMap会出现OOM其实是有历史原因的：

* 在Android 2.2 (api level 8)以及之前，当垃圾回收和线程是不能并发的，当发生垃圾回收的时候，线程就会被暂停，这就会导致延迟滞后甚至卡顿，系统效率低下。从Android 2.3开始，添加了并发垃圾回收的机制， 这意味着在一个Bitmap不再被引用之后，它所占用的内存会被立即回收。

* 在Android 2.3.3 (API level 10)以及之前, **一个Bitmap的像素级数据（pixel data）是存放在Native内存空间中的。 这些数据与Bitmap本身是隔离的，Bitmap本身被存放在Dalvik堆中**。我们无法预测在Native内存中的像素级数据何时会被释放，这意味着程序容易超过它的内存限制并且崩溃。 自Android 3.0 (API Level 11)开始， **像素级数据则是与Bitmap本身一起存放在Dalvik堆中**。

介于现在已经minSdk至少在14以上了,所以接下来就不讨论手动回收的问题；

### BitMap在到底占多大？

* 在android 中其实是可以通过api方法来调用获取BitMap大小的　
```java
public final int getByteCount() {
    // int result permits bitmaps up to 46,340 x 46,340
    return getRowBytes() * getHeight();
}
```
举个栗子在nexus 5x上一张xxhdpi下512*512大小的PNG图片所生成的BitMap 占用大小是802816　在xxxhdpi下占用内存是451584
至于怎么算的具体请看http://bugly.qq.com/bbs/forum.php?mod=viewthread&tid=498　这个链接有详细解答，涉及到一些native c代码的跟踪，还是值得学习一下；
那么在nexus 5x 上的density 是420（为什么不是４８０ 不是号称是1920X1080?）在xxhdpi上应该是这么算的
```
512/480 * 420 * 512/480 * 420 * 4 = 802816
```
计算的过程会有精度丢失，我觉得可以忽略不计算了,在BitMapFactory.cpp中有这么一个计算规则
```java
  scaledWidth = int(scaledWidth * scale + 0.5f);
  scaledHeight = int(scaledHeight * scale + 0.5f);
```

至于为什么会乘以４，那就看源码吧：
```c
static const uint8_t gSize[] = {
    0,  // Unknown
    1,  // Alpha_8
    2,  // RGB_565
    2,  // ARGB_4444
    4,  // RGBA_8888
    4,  // BGRA_8888
    1,  // kIndex_8
  };
```

### 小结

BitMap 的大小涉及到的因素有：

* 转换色彩的格式是RGB_565 还是ARGB_888等；

* 所存放资源文件的draweble资源目录；

* 机器本身的像素密度，密度越大，所占内存越大；

## 防止她红杏出墙(OOM)

### BitMap内存管理

防止BitMap　OOM 就应该先了解系统对BitMap的管理，请移驾到：
　* https://developer.android.com/training/displaying-bitmaps/manage-memory.html

 * 英文和我一样烂的同学请看http://hukai.me/android-training-course-in-chinese/graphics/displaying-bitmaps/manage-memory.html

###　将oom 降低到最小发生几率

由于android以及第三方手机厂商平台的差异化，oom永远都会是crash上的常客;

图片加载的问题目前开源社区有了很多优秀的depends，例如:fresco ,picaso, glide 。我个人不建议自己从零开始造轮子写加载库，但是可以去查看他们的源码，学习他的实现方式；

如果很不幸，App中会隔三差五来个OOM 那个不要慌张：

* 操作BitMapFactory

BitmapFactory.Options提供一些附加属性来指定decode的选项，解析Bitmap时用到2个重要参数：
1.inJustDecodeBounds
设置为true后，decode方法解析Bitmap时会返回一个null，只讲这个图片的原始大小（单位是像素）存入BitmapFactory.Options对象的options.outHeight和options.outWidth中，这样可以在不分配内存的情况下得到图片的尺寸信息。
2.inSampleSize
这个参数代表缩小比例，如果是1，代表原始尺寸，如果>1,假设为2，则缩小后图片像素值为原图的1/4（长1/2，宽1/2），同等格式下，占用内存也变为原来的1/4。decoder以2的幂作为系数，接近2的幂的数值都会被处理为最接近的2的幂值，3.4～4，2.1～2，这样。

缩小比例值
```java

public static int calculateInSampleSize(  
            BitmapFactory.Options options, int reqWidth, int reqHeight) {  
    final int height = options.outHeight;  
    final int width = options.outWidth;  
    int inSampleSize = 1;  

    if (height > reqHeight || width > reqWidth) {  

        final int halfHeight = height / 2;  
        final int halfWidth = width / 2;  

        while ((halfHeight / inSampleSize) > reqHeight  
                && (halfWidth / inSampleSize) > reqWidth) {  
            inSampleSize *= 2;  
        }  
    }  

    return inSampleSize;  
}   
```
在调用以上方法前，记得设置options.inJustDecodeBounds = true;
调用后算出比例后，则调用`BitmapFactory.decodexxxx(res, resId, options);  `

举个栗子
```java
public static Bitmap decodeSampledBitmapFromResource(Resources res, int resId,  
        int reqWidth, int reqHeight) {  

    final BitmapFactory.Options options = new BitmapFactory.Options();  
    options.inJustDecodeBounds = true;  
    BitmapFactory.decodeResource(res, resId, options);  

    options.inSampleSize = calculateInSampleSize(options, reqWidth, reqHeight);  

    options.inJustDecodeBounds = false;  
    return BitmapFactory.decodeResource(res, resId, options);  
}  
```
就可以`mImageView.setImageBitmap(decodeSampledBitmapFromResource(getResources(), R.id.myimage, 100, 100));`

* 如果非常不幸的是你要操作(缓存)很多BitMap实例（这是有多糟糕）那么请用LruCache;LruCache 采用Lru算法将BitMap用链表的方式存入缓存，可以有效的控制BitMap 大小

```java
int cacheSize = 4 * 1024 * 1024; // 4MiB　一般是可用内存的１/８
LruCache<String, Bitmap> bitmapCache = new LruCache<String, Bitmap>(cacheSize) {
    protected int sizeOf(String key, Bitmap value) {
        return value.getByteCount();
    }
}
```
* 简单粗暴－－捕捉错误
  其实从根源上来说这不是解决ＯＯＭ的办法，这是一个不是办法的办法，可以有效规避程序的崩溃，从而不至于严重影响程序的崩溃；
  ```java
  try {
     ......
  } catch (IOException e) {
      e.printStackTrace();
      ......
  }
```
