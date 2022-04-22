---
typora-root-url: ../../../static
title: "Android页面布局"
date: 2019-07-11T17:48:36+08:00
lastmod: 2019-07-11T17:48:36+08:00
draft: false
categories: ["Android"]
tags: ["AndroidLayout"]
author: "Pinger"
---

## 介绍
&emsp;&emsp;安卓提供6中基本布局类：帧布局（FrameLayout），线性布局（LinearLayout），相对布局（RelativeLayout），绝对布局（AbsoluteLayout），表格布局（TableLayout）和网格布局（GridLayout）。最常用的是帧布局，线性布局，相对布局和网格布局。

## 帧布局
&emsp;&emsp;帧布局也叫框架布局，在此布局下，所有的视图和控件都将固定在屏幕的左上角显示，**不能指定视图和控件的位置** ，但允许多个视图和控件叠加。

## 线性布局
&emsp;&emsp;线性布局可以让里面的视图垂直或者水平排列。通过指定其android:orientation属性来指定垂直或者水平排列。

测试代码：

	<?xml version="1.0" encoding="utf-8"?>
	<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
	    android:layout_width="match_parent"
	    android:layout_height="match_parent"
	    android:orientation="vertical"
	    >
	    <!--顶部标题-->
	    <TextView
	        android:layout_width="match_parent"
	        android:layout_height="wrap_content"
	        android:textSize="20sp"
	        android:text="@string/title"/>
	
	    <!--用户名-->
	    <LinearLayout
	        android:layout_width="match_parent"
	        android:layout_height="wrap_content"
	        android:orientation="horizontal">
	        <TextView
	            android:layout_width="wrap_content"
	            android:layout_height="wrap_content"
	            android:text="@string/username"
	            android:textSize="15sp"/>
	        <EditText
	            android:layout_width="match_parent"
	            android:layout_height="wrap_content"
	            android:background="@android:drawable/edit_text"
	            android:singleLine="true"/>
	    </LinearLayout>
	
	    <!--密码-->
	    <LinearLayout
	        android:layout_width="match_parent"
	        android:layout_height="wrap_content"
	        android:orientation="horizontal">
	        <TextView
	            android:layout_width="wrap_content"
	            android:layout_height="wrap_content"
	            android:text="@string/password"
	            android:textSize="15sp"/>
	        <EditText
	            android:layout_width="match_parent"
	            android:layout_height="wrap_content"
	            android:inputType="textPassword"/>
	    </LinearLayout>
	
	    <include
	        layout="@layout/zu_one"/>
	    
	</LinearLayout>

预览显示效果：

![效果1][p0]

## 相对布局
&emsp;&emsp;相对布局允许一个视图指定相对于其他视图或父视图的位置（通过视图id属性引用其他视图）。

测试代码：

	<?xml version="1.0" encoding="utf-8"?>
	<RelativeLayout xmlns:android="http://schemas.android.com/apk/res/android"
	    android:layout_width="match_parent"
	    android:layout_height="match_parent"
	    android:padding="10dip"
	    >
	    <!--顶部标题-->
	    <TextView
	        android:id="@+id/welcomeTextView"
	        android:layout_width="match_parent"
	        android:layout_height="wrap_content"
	        android:textSize="20sp"
	        android:text="@string/title"/>
	
	    <!--用户名-->
	    <LinearLayout
	        android:layout_width="match_parent"
	        android:layout_height="wrap_content"
	        android:id="@+id/linearOne"
	        android:layout_below="@id/welcomeTextView"
	        android:orientation="horizontal">
	        <TextView
	            android:layout_width="wrap_content"
	            android:layout_height="wrap_content"
	            android:text="@string/username"
	            android:textSize="15sp"/>
	        <EditText
	            android:layout_width="match_parent"
	            android:layout_height="wrap_content"
	            android:background="@android:drawable/edit_text"
	            android:singleLine="true"/>
	    </LinearLayout>
	
	    <!--密码-->
	    <LinearLayout
	        android:layout_width="match_parent"
	        android:layout_height="wrap_content"
	        android:id="@+id/linearTwo"
	        android:layout_below="@id/linearOne"
	        android:orientation="horizontal">
	        <TextView
	            android:layout_width="wrap_content"
	            android:layout_height="wrap_content"
	            android:text="@string/password"
	            android:textSize="15sp"/>
	        <EditText
	            android:layout_width="match_parent"
	            android:layout_height="wrap_content"
	            android:inputType="textPassword"/>
	    </LinearLayout>
	
	    <include
	        layout="@layout/zu_one"/>
	
	</RelativeLayout>

预览效果和前面的线性布局是一样的。（被include的layout里我设置了其相对于linearOne位于linearOne下方）

## 网格布局
&emsp;&emsp;网格布局是安卓4及其以后的版本退出的。网格布局不会显示行列，单元格的边框线。android:orientation可以指定网格的布局排列方向，android:layout_columnSpan，android:layout_rowSpan可以分别用来合并多列和多行。

测试代码：

	<?xml version="1.0" encoding="utf-8"?>
	<GridLayout xmlns:android="http://schemas.android.com/apk/res/android"
	    android:layout_width="match_parent"
	    android:layout_height="match_parent"
	    android:layout_gravity="center_horizontal"
	    android:orientation="horizontal"
	    android:columnCount="2"
	    android:rowCount="4"
	    >
	    <!--顶部标题-->
	    <TextView
	        android:id="@+id/welcomeTextView"
	        android:layout_width="match_parent"
	        android:layout_height="wrap_content"
	        android:textSize="20sp"
	        android:layout_columnSpan="2"
	        android:layout_gravity="fill"
	        android:text="@string/title"/>
	
	    <!--用户名-->
	    <TextView
	        android:layout_width="wrap_content"
	        android:layout_height="wrap_content"
	        android:text="@string/username"
	        android:textSize="15sp"/>
	
	    <EditText
	        android:layout_width="match_parent"
	        android:layout_height="wrap_content"
	        android:singleLine="true"/>
	
	    <TextView
	        android:layout_width="wrap_content"
	        android:layout_height="wrap_content"
	        android:text="@string/password"
	        android:textSize="15sp"/>
	    <EditText
	        android:layout_width="match_parent"
	        android:layout_height="wrap_content"
	        android:inputType="textPassword"/>
	
	</GridLayout>

预览显示效果：

![效果2][p1]


[p0]:/media/20190812-1.png
[p1]:/media/20190812-2.png