---
title: Android-Tips
date: 2016-07-04 18:14:06
tags: Android
---

## divier In LinearLayout
如何简洁的在LinearLayout中添加分割线？不需要添加其他代码，在*API>=11*的情况下，直接使用LinearLayout中的属性。

```java
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:divider="@drawable/line_drawable"
    android:showDividers="beginning|middle|end"
    android:orientation="vertical">
```
Notice: *android:divider*不能使用颜色，只能使用drawble

**line_drawable：**

```java
<?xml version="1.0" encoding="utf-8"?>
<shape xmlns:android="http://schemas.android.com/apk/res/android">
    <solid android:color="@color/c_gray_333333"/>
    <size android:height="1px"/>
</shape>
```

