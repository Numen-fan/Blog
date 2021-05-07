![](https://i.loli.net/2020/06/17/e9SXUWTyrPiVMN5.png)
<!-- more -->
### 1. 前言
> `AndroidX replaces the original support library APIs with packages in the androidx namespace. Only the package and Maven artifact names changed; class, method, and field names did not change.`，Google不再对`android support`库进行维护，`android support`中的API由命名空间`AndroidX`下的软件包进行替换，即相应的`包名`和`Maven工件名`发生改变。


### 2. 迁移AndroidX
#### 2.1 迁移之前的准备
- 原有项目的`support`库版本升级至28(Android 9)，这也是`support library`的最后版本，SDK 28 和AndroidX 1.0 是等效的。`This is because AndroidX artifacts with version 1.0.0 are binary equivalent to the Support Library 28.0.0 artifacts.`，

```
compileSdkVersion 28
```

- 建议使用Android studio 3.2或更高版本，(当前最新版已经到了4.0)。
- `gradle-wrapper.properties`中Gradle插件版本不低于4.6。

```
distributionUrl=https\://services.gradle.org/distributions/gradle-5.4.1-all.zip
```

- 如果代码在版本控制器中，建议在单独的分支中迁移。
#### 2.2 执行迁移
1. 在gradle.properties文件中添加下列项。

```
# Android 插件会使用对应的 AndroidX 库而非支持库。
android.useAndroidX=true
# Android 插件会通过重写现有第三方库的二进制文件，自动将这些库迁移为使用 AndroidX，但并不完全自动。
android.enableJetifier=true
```

2. 如果是AS 3.2或更高版本，则提供了一键迁移，选择菜单`Refactor-> Migrate to AndroidX`，会提示备份当前工程，勾选`Backup project as Zip file`，可以自动帮你备份。
![](https://i.loli.net/2020/06/17/pODBjRtxs327Ggw.png)
3. 左下角提示，点击`Do Refactor`
![](https://i.loli.net/2020/06/17/sWOgrdKR4PycnQt.png)

### 3 迁移结果
在一键迁移之后，gradle文件中implementation的所有support库被androidx替换，比如
```
implementation 'com.android.support:appcompat-v7:28.0.0' 
变为
implementation 'androidx.appcompat:appcompat:1.0.0'
```
相应类名也会发生改变
```
import android.support.v7.app.AppCompatActivity;
变为
import androidx.appcompat.app.AppCompatActivity;
```
> 所以，可以先看看上面两项结果，如果没有替换成功，可手动替换，相应替换可查阅官方提供的CSV格式的[依赖库映射文件](https://developer.android.google.cn/topic/libraries/support-library/downloads/androidx-artifact-mapping.csv)和[类映射文件](https://developer.android.google.cn/topic/libraries/support-library/downloads/androidx-class-mapping.csv)。

`rebuild project`，如果编译通过，那么恭喜你了，我反正是失败了。
### 4 迁移出错
#### 4.1 可手动纠正的错
1. 有的文件中没能替换掉，需要按照上述两项映射手动替换。
2. 检查gradle中通过`implementation`引入的库，比如`implementation androidx.recyclerview:recyclerview:1.0.0'`，则一键迁移后导入的类为`import androidx.appcompat.widget.RecyclerView;`，需要替换为`import androidx.recyclerview.widget.RecyclerView;`，猜测只是全局替换掉`support`字样。因为`类似`还有`GridLayoutManager`、`FragmentTransaction`;
`等。

#### 4.2 第三方库冲突
`support`库和`androidx`是不能共存的，
- 情况1 ：当迁移结束之后，理论上讲自己的项目使用的是`androidx`，但是老项目中导入了许多第三方的库，这些旧版本的库使用的是`support`。
- 情况2：这种情况发生在未进行迁移的项目中，由于导入了最新版的第三方库，而该库使用了`androidx`，也会报错。

> 解决方法：

- 情况1，更新第三方库到最新版本或使用`androidx`的版本，如果这个库没有使用`androidx`的版本，那就要找其他的方案代替吧（不知道是否是正确的解决方案）。
- 情况2：使用旧版本的第三方库。

> 总之，就是多build，根据异常信息解决问题。

### 5 参考资料
[AndroidX预览](https://developer.android.google.cn/jetpack/androidx)
[官方迁移教程](https://developer.android.google.cn/jetpack/androidx/migrate)
[谷歌开发者-是时候迁移至 AndroidX 了](https://www.jianshu.com/p/04389c42792d)

--- 
> 本文若有出入，请指正！
> 我是小小范同学。