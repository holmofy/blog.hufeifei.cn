---
title: android文件存储解析
date: 2016-12-02 18:08:32
tags:
    - Android
    - File
categories: Android
description: android文件存储解析
---

安卓中提供了Context中的方法与Environment类来获取文件。

# Context文件操作方法
```java
    public File getFileStreamPath(String name)
    public String[] fileList()
    public File getFilesDir()
    public File getNoBackupFilesDir()
    public File getExternalFilesDir(String type)
    public File[] getExternalFilesDirs(String type)
    public File getObbDir()
    public File[] getObbDirs()
    public File getCacheDir()
    public File getCodeCacheDir()
    public File getExternalCacheDir()
    public File[] getExternalCacheDirs()
    public File[] getExternalMediaDirs()
    public File getDir(String name, int mode)
```
用Log把它们都显示出来
```java
Log.d("context", "context.getFileStreamPath-->" +
	this.getFileStreamPath("test").toString());
Log.d("context", "context.getDir-->" +
	this.getDir("test", Context.MODE_PRIVATE).toString());
Log.d("context", "context.getFilesDir-->" +
	this.getFilesDir().toString());
Log.d("context", "context.getNoBackupFilesDir" +
	this.getNoBackupFilesDir().toString());
Log.d("context", "context.getCacheDir-->" +
	this.getCacheDir().toString());
Log.d("context", "context.getCodeCacheDir" +
	this.getCodeCacheDir().toString());
Log.d("context", "context.getDatabasePath-->" +
	this.getDatabasePath("test").toString());
Log.d("context", "context.getObbDir-->" +
	this.getObbDir().toString());

File[] files1 = this.getObbDirs();
for (File file : files1) {
    Log.d("context", "context.getObbDirs-->" + file.toString());
}
File[] files2 = this.getExternalMediaDirs();
for (File file : files2) {
    Log.d("context", "context.getExternalMediaDirs" + file.toString());
}

Log.d("context", "context.getExternalCacheDir-->" + this.getExternalCacheDir().toString());
File[] files3 = this.getExternalCacheDirs();
for (File file : files3) {
    Log.d("context", "context.getExternalCacheDirs-->" + file.toString());
}

Log.d("context", "context.getExternalFilesDir-->" + this.getExternalFilesDir(Environment.DIRECTORY_ALARMS).toString());

File[] files4 = this.getExternalFilesDirs(Environment.DIRECTORY_ALARMS);
for (File file : files4) {
    Log.d("context", "context.getExternalFilesDirs-->" + file.toString());
}
```
Log输出结果(不同版本的安卓系统，目录可能也不相同，详见[/storage/sdcard/0， /sdcard， /mnt/sdcard ，/storage/emulated/legacy 的区别](http://blog.csdn.net/ouyang_peng/article/details/47173367))

```java
context.getFileStreamPath-->/data/data/cn.hufeifei.environmenttest/files/test
context.getDir-->/data/data/cn.hufeifei.environmenttest/app_test
context.getFilesDir-->/data/data/cn.hufeifei.environmenttest/files
context.getNoBackupFilesDir/data/data/cn.hufeifei.environmenttest/no_backup
context.getCacheDir-->/data/data/cn.hufeifei.environmenttest/cache
context.getCodeCacheDir/data/data/cn.hufeifei.environmenttest/code_cache
context.getDatabasePath-->/data/data/cn.hufeifei.environmenttest/databases/test
context.getObbDir-->/storage/emulated/0/Android/obb/cn.hufeifei.environmenttest
context.getObbDirs-->/storage/emulated/0/Android/obb/cn.hufeifei.environmenttest
context.getExternalMediaDirs/storage/emulated/0/Android/media/cn.hufeifei.environmenttest
context.getExternalCacheDir-->/storage/emulated/0/Android/data/cn.hufeifei.environmenttest/cache
context.getExternalCacheDirs-->/storage/emulated/0/Android/data/cn.hufeifei.environmenttest/cache
context.getExternalFilesDir-->/storage/emulated/0/Android/data/cn.hufeifei.environmenttest/files/Alarms
context.getExternalFilesDirs-->/storage/emulated/0/Android/data/cn.hufeifei.environmenttest/files/Alarms
```
# Environment工具类中提供了以下几个方法：

```java
Environment.getDataDirectory();
Environment.getRootDirectory();
Environment.getDownloadCacheDirectory();
Environment.getExternalStoragePublicDirectory(String type);
Environment.getExternalStorageDirectory();
Environment.getExternalStorageState();
Environment.getExternalStorageState(File path)
Environment.getStorageState();//已被getExternalStorageState取代
```
**1. 前三个方法 **

用Log输出来：
```java
//IS标识内部存储
Log.d("Environment-IS", Environment.getDataDirectory().toString());
Log.d("Environment-IS", Environment.getDownloadCacheDirectory().toString());
Log.d("Environment-IS", Environment.getRootDirectory().toString());
```

输出结果为：
```java
D/Environment-IS: /data
D/Environment-IS: /cache
D/Environment-IS: /system
```

**2. getExternalStoragePublicDirectory方法 **
getExternalStoragePublicDirectory方法用来获取安卓外部存储中系统应用经常用到的公共文件夹，
在Environment中定义了这些文件夹的名字：
```java
Environment.DIRECTORY_MUSIC = "Music"
Environment.DIRECTORY_PODCASTS = "Podcasts"
Environment.DIRECTORY_RINGTONES = "Ringtones"
Environment.DIRECTORY_ALARMS = "Alarms"
Environment.DIRECTORY_NOTIFICATIONS = "Notifications"
Environment.DIRECTORY_PICTURES = "Pictures"
Environment.DIRECTORY_MOVIES = "Movies"
Environment.DIRECTORY_DOWNLOADS = "Download"
Environment.DIRECTORY_DCIM = "DCIM"
Environment.DIRECTORY_DOCUMENTS = "Documents"
```
它们的目录一般在/storage/emulated/0/<dir_name>	(dir_name就是Environment中定义的这些字符串常量)
**3. 最后的三个方法**

最后面三个方法是用来获取挂载点的状态(在Linux中把一些特殊目录称为所谓的挂载点，有点类似于Windows中的分区)：

```java
Environment.MEDIA_REMOVED;//媒体存储已经移除了
Environment.MEDIA_UNMOUNTED;//存储媒体没有挂载
Environment.MEDIA_CHECKING;//正在检查存储媒体
Environment.MEDIA_NOFS;//存储媒体是空白或是不支持的文件系统no_file_system
Environment.MEDIA_MOUNTED;//存储媒体已经挂载，并且挂载点可读/写
Environment.MEDIA_MOUNTED_READ_ONLY;//存储媒体已经挂载，挂载点只读
Environment.MEDIA_SHARED;//存储媒体正在通过USB共享
Environment.MEDIA_BAD_REMOVAL;//在没有挂载前存储媒体已经被移除
Environment.MEDIA_UNMOUNTABLE;//存储媒体无法挂载,可能是文件系统损坏了
Environment.MEDIA_EJECTING;//存储媒体正在移除
Environment.MEDIA_UNKNOWN;//未知的存储状态
```
#总体概括
下面图片大概地概括了上面的方法
![这里写图片描述](http://img.blog.csdn.net/20161202203511889)
* **Context中的方法或得到的路径都与应用包名相关**
* **Environment中的方法与整个系统有关**