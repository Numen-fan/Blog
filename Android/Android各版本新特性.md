# Android各个版本新特性

## 引言

>  有时总是记不住Android某个特性是哪个版本引入的，哪些版本会受到影响，每次都需要重新去查一遍，最近开始看《Android进阶之光》一书，第一章就是介绍各版本的新特性，索性做个笔记，并计划不断更新。

## 5.0 版本新特性—2014年（Lollipop）

1. 全新的Material Design设计风格。
2. 支持64位ART虚拟机。
   1. 放弃了之前一直使用的Dalvik虚拟机，改用了ART虚拟机，实现了真正的跨平台编译。（todo：弄懂为何）
      1. https://www.cnblogs.com/ganchuanpu/p/9321682.html
3. 引入RecyclerView（todo：它的优点）。
   1. [Android ListView与RecyclerView对比浅析](https://blog.csdn.net/import_sadaharu/article/details/81323801)
4. 新增悬挂式Notification。
   1. 相较于普通式和折叠式Notification需要拉下通知中心才可以查看的交互，悬挂式直接显示在屏幕上方，并且焦点不变，仍然在用户操作的界面上，不会打断用户的操作，过几秒会消失。
   2. Android 5.0 支持对Notification设置显示等级的能力。
5. 引入更加灵活的Toolbar。

## 6.0 版本新特性—2015年（Marshmallow）

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


## 7.0版本新特性—2016年（Nougat）

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

