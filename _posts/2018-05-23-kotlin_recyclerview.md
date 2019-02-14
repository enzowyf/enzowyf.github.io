---
layout: post
title:  "Kotlin DSL快速创建RecyclerView.Adapter，ViewHolder再见！"
date:   2018-05-23 16:43:42 +0800
categories: Android开发
tags: kotlin android recyclerview
---

在Android开发中使用RecyclerView，要继承重写Adapter、创建ViewHolder，非常繁琐。这里实现了一个CommonRecyclerAdapter，通过Kotlin DSL语法，可以快速的完成RecyclerView的生成：

```kotlin
downloadedRecycleView.apply {
    layoutManager = LinearLayoutManager(context)
    adapter = CommonRecyclerAdapter<DownloadInfo> {
        onCount { downloadedList.size }
        onItem { downloadedList[it] }
        onLayout { R.layout.app_download_item }
        onBind { _, _, _, item ->
            appNameView.text = item.appName
            appSizeView.text = Formatter.formatFileSize(context, item.fileSize ?: 0)
            appIconView.loadCenterCropImage(item.iconUrl?:"")
        }
    }
}
```
 
CommonRecyclerAdapter源码如下：

```kotlin
import android.support.v7.widget.RecyclerView
import android.view.LayoutInflater
import android.view.View
import android.view.ViewGroup

/**
 * Created by enzowei on 2018/2/9.
 */
class CommonRecyclerAdapter<T>(build: CommonRecyclerAdapter<T>.() -> Unit) :
    RecyclerView.Adapter<CommonViewHolder>() {
    private var onLayout: ((viewType: Int) -> Int)? = null
    private var onCreateView: ((viewType: Int) -> View)?= null
    private lateinit var onItem: (position: Int) -> T
    private lateinit var onCount: () -> Int
    private lateinit var onBind: View.(holder: CommonViewHolder, position: Int, viewType:Int, item: T) -> Unit
    private var onItemViewType: (position: Int) -> Int = { 0 }
    private var onViewRecycle: View.() -> Unit = { }

    init {
        build()
    }

    fun onLayout(onLayout: (viewType: Int) -> Int) {
        this.onLayout = onLayout
    }

    fun onCreateView(onCreateView: (viewType: Int) -> View) {
        this.onCreateView = onCreateView
    }

    fun onItem(onItem: (position: Int) -> T) {
        this.onItem = onItem
    }

    fun onBind(onBind: View.(holder: CommonViewHolder, position: Int, viewType:Int, item: T) -> Unit) {
        this.onBind = onBind
    }

    fun onCount(onCount: () -> Int) {
        this.onCount = onCount
    }

    fun onItemViewType(onItemViewType: (position: Int) -> Int) {
        this.onItemViewType = onItemViewType
    }

    fun onViewRecycle(onViewRecycle:View.() -> Unit) {
        this.onViewRecycle = onViewRecycle
    }

    override fun onCreateViewHolder(parent: ViewGroup, viewType: Int): CommonViewHolder {
        val view = when {
            onCreateView != null -> onCreateView?.invoke(viewType)
            onLayout != null -> LayoutInflater.from(parent.context).inflate(
                onLayout!!.invoke(viewType), parent, false
            )
            else -> throw IllegalArgumentException("itemView may not be null")
        }
        return CommonViewHolder(view, viewType)
    }

    override fun onBindViewHolder(holder: CommonViewHolder, position: Int) {
        holder.bind(position, onItem(position), onBind)
    }

    override fun getItemCount(): Int = onCount()

    override fun getItemViewType(position: Int): Int = onItemViewType(position)

    override fun onViewRecycled(holder: CommonViewHolder?) {
        holder?.itemView?.onViewRecycle()
    }

}

class CommonViewHolder(itemView: View?, val viewType: Int) : RecyclerView.ViewHolder(itemView) {
    inline fun <T> bind(position: Int, item: T, onBind: View.(CommonViewHolder, Int, Int, T) -> Unit) {
        itemView.onBind(this, position, viewType, item)
    }

}
```