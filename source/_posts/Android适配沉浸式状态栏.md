---
title: Android适配沉浸式状态栏
date: 2024-03-06 19:54:38
tags: [Android]
categories: [Android]
---
Android系统提供了两种显示模式：明亮模式与暗黑模式
- 亮色模式（Light Model）：整体偏亮，即背景亮色，文字等内容暗色
- 暗黑模式（Dark Model）：整体偏暗，即背景暗色，文字等内容亮色。

将decorView的fitSystemWindows属性设置为false
```kotlin
WindowCompat.setDecorFitsSystemWindows(window, false)
```
设置状态栏颜色为透明
```kotlin
window.statusBarColor = Color.TRANSPARENT
```
是否需要改变状态栏上的 图标、字体 的颜色
```kotlin
val controller = WindowCompat.getInsetsController(window, window.decorView)
```
mask：遮罩 默认是false 
```kotlin
var mask = true
// mask = true 状态栏字体颜色为黑色，一般在状态栏下面的背景色为浅色时使用
// mask = false 状态栏字体颜色为白色，一般在状态栏下面的背景色为深色时使用
controller.isAppearanceLightStatusBars = mask
```
android Q+ 去掉虚拟导航键 的灰色半透明遮罩
```kotlin
if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.Q) {
    window.isNavigationBarContrastEnforced = false
}
```
设置虚拟导航键的 背景色为透明
```kotlin
if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
    //8.0+ 虚拟导航键图标颜色可以修改，所以背景可以用透明
    window.navigationBarColor = Color.TRANSPARENT
} else {
    //低版本因为导航键图标颜色无法修改，建议用黑色，不要透明
    window.navigationBarColor = Color.BLACK
}
```
是否需要修改导航键的颜色，mask 同上面状态栏的一样
```kotlin
controller.isAppearanceLightNavigationBars = mask
```
在带有刘海或者挖孔屏上，横屏时刘海或者挖孔的那条边会有黑边，解决方法是：
给APP的主题v27加上
```xml
<item name="android:windowLayoutInDisplayCutoutMode">shortEdges</item>
```
