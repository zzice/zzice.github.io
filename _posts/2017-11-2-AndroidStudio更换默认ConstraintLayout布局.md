---
title: Android Studio更换默认ConstraintLayout布局
date: 2017-11-1 14:10:01
categories: 
tags: Android_Studio
---

> Google目前大力推行ConstraintLayout布局，所以Android Studio新建工程的时候默认布局会变 成ConstraintLayout。
> ConstraintLayout使用教程
> 郭霖 [Android新特性介绍，ConstraintLayout完全解析](http://blog.csdn.net/guolin_blog/article/details/53122387)
>
> 鸿洋 [ConstraintLayout 完全解析 快来优化你的布局吧](http://blog.csdn.net/lmj623565791/article/details/78011599)
>
> ......

此文仅介绍如何更换默认布局

<!-- more -->

以下将默认布局由ConstraintLayout改为RelativeLayout，其他举一反三即可。

1. 打开Android Studio安装目录 

   进入到 `\plugins\android\lib\templates\activities\common\root\res\layout`文件夹。

2. Sublime Text(或其他的文本编辑器)打开`simple.xml.ftl`（备份一下，方便日后还原！）

   ```xml
   <?xml version="1.0" encoding="utf-8"?>
   <android.support.constraint.ConstraintLayout
       xmlns:android="http://schemas.android.com/apk/res/android"
       xmlns:tools="http://schemas.android.com/tools"
       xmlns:app="http://schemas.android.com/apk/res-auto"
       android:layout_width="match_parent"
       android:layout_height="match_parent"
   <#if hasAppBar && appBarLayoutName??>
       app:layout_behavior="@string/appbar_scrolling_view_behavior"
       tools:showIn="@layout/${appBarLayoutName}"
   </#if>
       tools:context="${relativePackage}.${activityClass}">

   <#if isNewProject!false>
       <TextView
   <#if includeCppSupport!false>
           android:id="@+id/sample_text"
   </#if>
           android:layout_width="wrap_content"
           android:layout_height="wrap_content"
           android:text="Hello World!"
           app:layout_constraintBottom_toBottomOf="parent"
           app:layout_constraintLeft_toLeftOf="parent"
           app:layout_constraintRight_toRightOf="parent"
           app:layout_constraintTop_toTopOf="parent" />

   </#if>
   </android.support.constraint.ConstraintLayout>
   ```

   ​



3. 直接Ctrl+A全选删除，粘贴以下布局xml，然后保存即可。

    ``` xml
    <?xml version="1.0" encoding="utf-8"?>  
    <RelativeLayout  
        xmlns:android="http://schemas.android.com/apk/res/android"  
        xmlns:app="http://schemas.android.com/apk/res-auto"  
        android:layout_width="match_parent"  
        android:layout_height="match_parent">  
      
        <TextView  
            android:layout_width="wrap_content"  
            android:layout_height="wrap_content"  
            android:text="Hello World!"  
        />  
      
    </RelativeLayout> 
    ```

重新打开Android Studio ，新建工程，默认布局就会变成RelativeLayout。
