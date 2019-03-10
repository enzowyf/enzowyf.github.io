---
layout: post
title:  "Glide自定义Transformations"
date:   2018-09-05 16:30:42 +0800
categories: Android开发
tags: android glide
---

大家在使用`Glide`库的时候，都会用到`Transformations`，来完成一些对图片的变换操作，比如使用常见的`CenterCrop`、`CenterInside`和`fitCenter`，来适配`ImageView`的尺寸。

除此之外，`Glide`还提供了两种常用的裁剪工具：

* 给方形裁圆角 - RoundedCorners
* 裁圆形 - CircleCrop

使用也非常简单：

```kotlin
val requestOptions = RequestOptions.bitmapTransform(CircleCrop())
Glide.with(this)
    .load("https://blog.ezstudio.cn/assets/img/profile.jpeg")
    .apply(requestOptions)
    .into(image_view)
        
```

## 自定义Transformations

以上这些其实都是`BitmapTransformation`的子类，我们可以通过继承它，来完成一些自定义的变换或者做一些图片的加工，[wasabeef/glide-transformations](https://github.com/wasabeef/glide-transformations)就提供了一些现成的例子。自定义的过程也不难，就是通过覆写父类的`transform`方法，在里面对传递进来`toTransform`做一些加工就可以了：

```java
protected abstract Bitmap transform(@NonNull BitmapPool pool, @NonNull Bitmap toTransform, int outWidth, int outHeight);
```

例如假设传递进来的`toTransform`是一个已经通过`CircleCrop`裁切好的圆形，我们要对他描边，只需要：

```kotlin
override fun transform(
    pool: BitmapPool,
    toTransform: Bitmap,
    outWidth: Int,
    outHeight: Int
): Bitmap {
    //创建一个新的bitmap
    val result = pool.get(toTransform.width, toTransform.height, Bitmap.Config.ARGB_8888)
    result.setHasAlpha(true)

    lock.withLock {
        val canvas = Canvas(result)

        val paint = Paint().apply {
            isAntiAlias = true
            strokeWidth = borderWidth
            color = borderColor
            isDither = true
            style = Paint.Style.STROKE
        }

        //绘制传递进来的toTransform
        canvas.drawBitmap(toTransform, 0f, 0f, paint)

        //绘制描边
        val center = result.width / 2.0f
        val radius = center - borderWidth / 2.0f
        canvas.drawCircle(center, center, radius, paint)

        //清理canvas
        canvas.setBitmap(null) 
    }
    return result
}
```

当然这里还需要`Canvas`和`Paint`的相关知识，这里就不展开了。

## 多重变换之一
上一节描边的例子是建立在已经通过`CircleCrop`裁切好的圆形，也就是需要两种变换的叠加，假设用于描边处理的类是`CircleBorderTransformation`，多种变换的叠加就需要用到`MultiTransformation`：

```kotlin
val requestOptions = RequestOptions.bitmapTransform(
    MultiTransformation(
        CircleCrop(),
        CircleBorderTransformation(8f, Color.parseColor("#3498db"))
    )
)	
Glide.with(this)
    .load("https://blog.ezstudio.cn/assets/img/profile.jpeg")
    .apply(requestOptions)
    .into(image_view)
```

## 多重变换之二
上一节中的`MultiTransformation`用起来其实挺麻烦的，当需要变换的层次多了，参数就会变得很长，代码不漂亮。而且每次一层变换，都要去创建一个新的`bitmap`，把上一层绘制到新的一层中，再传递下去，创建`bitmap`占内存不说还浪费时间。

那有没有更好的办法呢，当然是有的。还是已裁切圆形加描边来说，扒开`CircleCrop`，会发现：

```java
@Override
protected Bitmap transform(@NonNull BitmapPool pool, @NonNull Bitmap toTransform, int outWidth, int outHeight) {
    return TransformationUtils.circleCrop(pool, toTransform, outWidth, outHeight);
}
```

`CircleCrop`的`transform`方法其实是间接调用了`TransformationUtils.circleCrop()`，所以改造一下`CircleBorderTransformation`：

```kotlin
override fun transform(
    pool: BitmapPool,
    toTransform: Bitmap,
    outWidth: Int,
    outHeight: Int
): Bitmap {
    val width = toTransform.width
    val height = toTransform.height
    
    //裁圆形
    val result = TransformationUtils.circleCrop(pool, toTransform, width, height)
    
    lock.withLock {
        val canvas = Canvas(result)

        val paint = Paint().apply {
            isAntiAlias = true
            strokeWidth = borderWidth
            color = borderColor
            isDither = true
            style = Paint.Style.STROKE
        }

        //绘制描边
        val center = result.width / 2.0f
        val radius = center - borderWidth / 2.0f
        canvas.drawCircle(center, center, radius, paint)
        
        //清理canvas
        canvas.setBitmap(null)
    }
    return result
}
```

这样一来使用的时候只需要：

```kotlin
val requestOptions = RequestOptions.bitmapTransform(CircleBorderTransformation(8f, Color.parseColor("#3498db")))
Glide.with(this)
    .load("https://blog.ezstudio.cn/assets/img/profile.jpeg")
    .apply(requestOptions)
    .into(image_view)   
```

这种做法，很好的减少了创建`bitmap`的开销。当然也有人会说欠缺灵活，这就看取舍，是灵活重要还是性能重要。

## Transformation的缓存
`Transformation`配合`Glide`的缓存策略（`DiskCacheStrategy.ALL`或者`DiskCacheStrategy.RESOURCE`），可以将上次下载并完成变换加工的图片直接拿来使用，有效减少开销。比如上面使用`CircleCrop`加载头像的例子，第一次下载加裁圆形需要大约`300ms+`，而配合缓存策略，再次加载仅需`3ms`。

Glide提供`Transformations`的子类都有这种效果，而我们前面自定义的`CircleBorderTransformation`却没有，这是为什么呢。

因为我们只是覆写了`transform`方法，这是不够的，还需要覆写另外三个方法：`updateDiskCacheKey`、`equals`和`hashCode`。

来自源码中解释：
>  Transforms the given resource and returns the transformed resource.
> 
>  <p>If the original resource object is not returned, the original resource will be
>  recycled and it's internal resources may be reused. This means it is not safe to rely on the
>  original resource or any internal state of the original resource in any new resource that is
>  created. Usually this shouldn't occur, but if absolutely necessary either the original resource
>  object can be returned with modified internal state, or the data in the original resource can
>  be copied into the transformed resource.
> 
>  <p>If a Transformation is updated, {@link #equals(Object)}, {@link #hashCode()}, and
>  {@link #updateDiskCacheKey(java.security.MessageDigest)} should all change. If you're using a
>  simple String key an easy way to do this is to append a version number to your key. Failing to
>  do so will mean users may see images loaded from cache that had the old version of the
>  Transformation applied. Changing the return values of those methods will ensure that the cache
>  key has changed and therefore that any cached resources will be re-generated using the updated
>  Transformation.
> 
>  <p>During development you may need to either using {@link
>  com.bumptech.glide.load.engine.DiskCacheStrategy#NONE} or make sure {@link
>  #updateDiskCacheKey(java.security.MessageDigest)} changes each time you make a change to the
>  Transformation. Otherwise the resource you request may be loaded from disk cache and your
>  Transformation may not be called.

   
所以完整版的`CircleBorderTransformation`：

```kotlin
class CircleCropTransformation(private val borderWidth: Float, private val borderColor: Int) :
    BitmapTransformation() {
    private val lock = ReentrantLock()

    override fun transform(
        pool: BitmapPool,
        toTransform: Bitmap,
        outWidth: Int,
        outHeight: Int
    ): Bitmap {
        val width = toTransform.width
        val height = toTransform.height
        val result = TransformationUtils.circleCrop(pool, toTransform, width, height)
        lock.withLock {
            val canvas = Canvas(result)

            val paint = Paint().apply {
                isAntiAlias = true
                strokeWidth = borderWidth
                color = borderColor
                isDither = true
                style = Paint.Style.STROKE
            }

            val center = result.width / 2.0f
            val radius = center - borderWidth / 2.0f
            canvas.drawCircle(center, center, radius, paint)

            canvas.setBitmap(null)
        }
        return result
    }

    override fun equals(other: Any?): Boolean {
        return if (other is CircleCropTransformation) {
            other.borderWidth == this.borderWidth && other.borderColor == this.borderColor
        } else {
            false
        }
    }

    override fun hashCode(): Int {
        return ID.hashCode()
    }

    override fun updateDiskCacheKey(messageDigest: MessageDigest) {
        messageDigest.update(ID_BYTES)
        val widthData = ByteBuffer.allocate(4).putFloat(this.borderWidth).array()
        messageDigest.update(widthData)
        val colorData = ByteBuffer.allocate(4).putInt(this.borderColor).array()
        messageDigest.update(colorData)
    }

    companion object {
        private val ID = CircleCropTransformation::class.java.name
        private val ID_BYTES = ID.toByteArray(Key.CHARSET)
    }
}
```

`updateDiskCacheKey`、`equals`和`hashCode`，三者缺一不可。同时如果是使用“多重变换之一”里面提到的方法来进行多重变换，那么每一层都要实现这三个方法，才能达到提高二次加载效率的效果。


## Thumbnail 和 Transformation

很多时候，我们会使用`Thumbnail`来让界面先展示一个模糊一点的图片作为过渡，再最终显示清晰的图片，来提高体验，比如：

```kotlin
Glide.with(this)
    .load("https://blog.ezstudio.cn/assets/img/profile.jpeg")
    .thumbnail(0.3f)//模糊的过渡效果
    .into(image_view)
```    

上面的`thumbnail(0.3f)`表示先加载一个原图`30%`尺寸的过渡图片，比如你期望加载原图的尺寸是`300px * 300px`，则过渡图片的尺寸是`90px * 90px`。

如果我们这时候想要对这个头像做圆角处理，就需要用到前面提到的`RoundedCorners`:

```kotlin
val requestOptions = RequestOptions.bitmapTransform(RoundedCorners(20))
Glide.with(this)
    .load("https://blog.ezstudio.cn/assets/img/profile.jpeg")
    .apply(requestOptions)
    .thumbnail(0.3f)//模糊的过渡效果
    .into(image_view)
        
```

这时候问题就来了，网速慢的时候，或者眼尖的同学就会发现，中间过渡的图片，圆角会比最终图片大很多，过渡的就特别别扭：

|过渡图|清晰图|
|:---:|:---:|
|![](./../assets/img/2018-09-05-glide_transformation/1.png)| ![](./../assets/img/2018-09-05-glide_transformation/2.png)|
可以看到两图圆角不一致。

这是因为同时使用`Thumbnail`和`Transformation`的，中间的过渡图片也会被传递到`transform `方法进行处理。这个时候，传入图片的尺寸是偏小的，而处理圆角的半径是按照原图进行配置的，所以圆角就偏大。

因此在使用的`Transformation`的时候，要么不要使用`Thumbnail`，要么就要在`Transformation`进行区分处理。
