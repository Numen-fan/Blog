# Android各个版本的新特性概要

## 引言

>  1. 有时总是记不住Android某个特性是哪个版本引入的，哪些版本会受到影响，每次都需要重新去查一遍，最近开始看《Android进阶之光》一书，第一章就是介绍各版本的新特性，索性做个笔记，并计划不断更新。
>
>  2. 其实说新特性有点不准确，分析一个全新版本的新特性，应该从版本新功能和接口变动两个方面分析。
>     1. 新特性是指只在此版本上的功能，低于此版本的设备无法使用。
>     2. 接口变动指的是我们的APP运行到Android新版本上需要做出的调整（版本适配）。

## 版本变更记录

> 1. v1.0，2022年3月6日，笔记记录《Android进阶之光》中对5.0~10.0版本新特性介绍。
> 1. v2.0，2022年3月12日，查阅资料补齐11~12版本新特效介绍。

## 5.0 新特性—2014年（Lollipop）

1. 全新的Material Design设计风格。
2. 支持64位ART虚拟机。
   1. 放弃了之前一直使用的Dalvik虚拟机，改用了ART虚拟机，实现了真正的跨平台编译。（todo：弄懂为何）
      1. https://www.cnblogs.com/ganchuanpu/p/9321682.html
3. 引入RecyclerView（todo：它的优点）。
   1. [Android ListView与RecyclerView对比浅析](https://blog.csdn.net/import_sadaharu/article/details/81323801)
4. 新增悬挂式Notification。
   1. 相较于普通式和折叠式Notification需要拉下通知中心才可以查看的交互，悬挂式直接显示在屏幕上方，并且焦点不变，仍然在用户操作的界面上，不会打断用户的操作，过几秒会消失。
   2. Android 5.0 支持对Notification设置显示等级的能力。
5. 引入更加灵活的Toolbar，取代ActionBar。

## 6.0 新特性—2015年（Marshmallow）

1. 统一支付标准Android Pay。
1. 指纹支持。
3. Doze电量管理。
   1. 手机静止不动一段时间后，会进入Doze电量管理模式，提高续航时间。

4. APP Links。
   1. 加强了软件间的关联，支持点击链接跳转到对应的App（todo：scheme调起？？？）

5. Now on Tap
   1. 长按Home键激活Now on Tap，他会识别当前屏幕上的内容并创建Now卡片。

6. **【重点】运行时权限管理**。
   1. targetSdkVersion >= 23。
   2. 分位Normal Permissions和Dangerous Permissions。
   3. ActivityCompat.checkSelfPermissions()请求，低于6.0的版本，次方法默认返回值为PackManager.PERMISSION_GRANTED。
   4. onRequestPermissionsResult()回调结果。
   5. 如果用户选择了『不在询问』，下次则不会弹框，而是直接处理拒绝后的逻辑。


## 7.0 新特性—2016年（Nougat）

1. 多窗口模式（分屏模式）
   1. 进入多窗口的Activity生命周期变化，会先onDestroy销毁，随后重建，停在onPause状态。
   2. 推出多窗口的Activity生命周期变化，接着上面onPause->onDestroy，随后正常重建。
   3. 禁用多窗口模式：在manifest.xml中配置`android:resizeableActivity="false"`
2. Data Server
   1. 一种流量保护机制，启用Data Server后，系统将拦截后台应用的数据使用。
3. 改进的Java8语言支持。
   1. 支持java8，可以使用lambda表达式等。
4. 自定义壁纸
   1. 设置壁纸时，可以选择是设置桌面还是锁屏壁纸。
5. 快捷回复
   1. 在通知中快捷回复。
6. 快速设置
   1. 下拉通知栏顶部，有edit按钮，可以对菜单进行自定义添加、删除、拖动排序。
7. 其它：Daydream VR、后台省点、Unicode 9支持和全新的emoji表情符号、Google Assistant。

## 8.0 新特性—2017年（Oreo）

1. **【重点】通知中心**
   1. 所有通知都必须分到一个渠道，即新增NotificationChannel。
2. 画中画（PIP）支持
   1. 一种特殊的多窗口模式，常用于视频播放。
3. 自适应启动器图标
   1. 桌面icon在不同的设备型号上显示为不同的形状。
4. 后台执行限制
   1. 后台service限制。
   2. 广播限制：除了有限的例外情况，应用无法使用清单注册隐式广播。
5. 后台位置信息限制
   1. 为降低耗电量，后台应用检索用户当前位置信息的频率会得到限制。
6. 其它：自动填充框架、自动调整TextView的大小、WebView API、多显示器支持

## 9.0 新特性—2018年（Pie）

1. 全面支持全面屏
   1. 通过DisplayCutout类可以确定非功能区域的位置和形状，这些区域不应显示内容。
2. 动画
   1. 引入AnimatedImageDrawable类，用于显示GIF和WebP动画图像。
3. 利用Wi-Fi RTT进行室内定位。
4. 隐私变更
   1. 限制后台访问设备传感器，限制通过WiFi扫描检索到的信息等。
5. 其它：机器学习，HDR VP9视频、HEIF图像压缩和Media API、对使用非SDK接口的限制。

## 10.0 新特性—2019年（Q）

1. 5G支持。
2. 支持可折叠设备。
3. **【重点】暗黑主题**。
4. 手势导航。
   1. 全面屏手势操作。
5. 智能回复。
   1. 通过机器学习预测你在回复消息时可能会说些什么。
6. 用户隐私。给用户更多应用程序控制权。
   1. 提供仅这一次、应用使用时授权等选择。
7. ART优化，
   1. 添加了一种垃圾回收机制，节省垃圾回收的时间，帮助在低版本设备上顺畅运行。
8. 机器学习更新。

## 11.0 新特性—2020年（R）

1. 短信 更新改进，提供更加友好的交互。
2. 权限和隐私
   1. 在Android10的用户隐私基础上，新增了位置、麦克风和摄像头的一次性权限许可。
3. 内置屏幕录制。
4. 适配不同设备。
   1. 折叠屏支持优化，增加铰链角度传感器API等。
   2. 高刷新率支持。
5. 网络优化。
   1. 新增『动态计量API』，如果检测到连接到无限5G信号，将可以访问最高质量的视频和图片。



## 12.0 新特性—2021年

1. 原生的ImageDecoder支持GIF和WebP格式。
2. 支持圆角。
   1. `Display.getRounderCorner()`获取屏幕圆角的详细信息。
3. 更易用的模糊、色彩滤镜等特效。
   1. `View.setRenderEffect(RenderEffect)` 将特效直接应用于视图
4. 限制对MAC地址的访问。
5. 应用覆盖控制。
   1. 可以控制是否允许在自己的内容上显示这些覆盖图层，调用`Window#setHideOverlayWindows()`，表明不允许`TYPE_APPLICATION_OVERLAY`的窗口显示。
6. 应用无法关闭系统对话框。
   1. 弃用了 `ACTION_CLOSE_SYSTEM_DIALOGS` intent 操作。
7. Activity/BroadcastReciver/Service 声明了Filter，则必须显示设置`android:exported`属性。
8. 必须为每个PendingIntent设置可变性。
9. 后台应用无法再启动前台服务。



## 参考资料

1. 《Android进阶之光-第2版》刘望舒。
2. [Android11新特性。](https://blog.csdn.net/xiangzhihong8/article/details/105962075)
3. [Android11功能变更](https://developer.android.google.cn/about/versions/11)
4. [Android 12 功能和变更列表](https://developer.android.google.cn/about/versions/12/summary)
5. [Android 12 新特性 - 基于预览版2](https://www.jianshu.com/p/b1dd33fde956)































