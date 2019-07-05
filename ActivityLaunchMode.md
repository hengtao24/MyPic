# 前言
项目开发时，遇到了跳转到另一Activity时，重复多次，然后退出时，需要多次返回才能退出，理想情况是虽然跳转多次，但只需退出一次即可返回桌面，原因是Activity的启动模式设置有误。
为了给用户良好的体验，界面跳转的启动方式十分重要，与界面跳转相关联的概念是Task（任务）和Back Stack（回退栈）。除了在manifest.xml文件中指定launchMode外，通过设置Intent的
一些标志（以FLAG_ACTIVITY_开头）也可以新Activity的启动模式。
# Activity启动模式
## Task和Back Stack
Task是在程序运行时只针对activity的概念。Task是一组相互关联的activity的集合，它是存在于framework层的一个概念，控制界面的跳转和返回。这个task存在于一个称为back stack(栈)的
结构中，即framework是以栈的形式管理用户开启的activity的。
> 参考：张纪刚 原文：https://blog.csdn.net/zhangjg_blog/article/details/10923643
## 启动模式
默认情况下，当我们多次启动同一个Activity时，系统会创建多个实例并把它们一一放入任务栈中(Back Stack)，每按下back键就会有一个Activity出栈，直到栈空为止。

### standard：标准模式
这是系统默认的模式，每次启动一个新的Activity都会重新创建一个新的实例，不管这个实例已经是否存在。这种模式下，如果A启动了B（标准模式），那么B自动进入A所在的任务栈中。

### singleTop：栈顶复用模式 （登录页面、推送通知栏）
此种模式下，如果新Activity已经位于任务栈的栈顶，那么此Activity不会被重新创建，同时onNewIntent方法会被调用，通过此方法的参数获取当前请求信息。而且，此Activity的
onCreate，onStart不会被调用，因为没有发生改变。

- singleTop模式分3种情况：
1. 当前栈中已有该Activity的实例并且该实例位于栈顶时，不会新建实例，而是复用栈顶的实例，并且会将Intent对象传入，回调onNewIntent方法
2. 当前栈中已有该Activity的实例但是该实例不在栈顶时，其行为和standard启动模式一样，依然会创建一个新的实例
3. 当前栈中不存在该Activity的实例时，其行为同standard启动模式
standard和singleTop启动模式都是在原任务栈中新建Activity实例，不会启动新的Task，即使你指定了taskAffinity属性

### singleTask：栈内复用模式（应用中展示的主页（Home页））
此种模式下，只要Activity在一个栈中存在，那么多次启动此Activity都不会重新创建实例，系统会回调onNewIntent方法。系统会先寻找是否存在A想要的任务栈，如果不存在，就
重建一个任务栈，然后创建A的实例放入新任务栈中；如果存在A想要的任务栈，再查看是否有Activity实例存在，有的话就把该实例调到栈顶，如果实例不存在，则创建A的实例放入
任务栈中。下面举三种例子说明此种模式运行机制：

1. 目前任务栈S1中为ABC，Activity D以singleTask模式请求启动，其需要的任务栈为S1，那么系统会创建D的实例，将D放入S1中
2. 目前任务栈S1中为ABC，Activity D以singleTask模式请求启动，其需要的任务栈为S2，那么系统会创建任务栈S2，再将D放入S2中
3. 目前任务栈S1中为ADBC，Activity D以singleTask模式请求启动，其需要的任务栈为S1，那么系统不会创建D的实例，将D切换到栈顶并调用其onNewIntent方法，同时栈内所有在D上
面的Activity都需要出栈，最终的S1为AD

### singleInstance：单实例模式 （系统Launcher、锁屏键、来电显示等系统应用）
这是一种加强的singleTask模式，除了具有singleTask模式的所有特性外，具有此种模式的Activity只能单独位于一个任务栈中。比如Activity A以singleInstance模式启动，系统会
为其创建一个新的任务栈，然后A独自运行在该任务栈中，后续的请求均不会创建新的Activity。

以singleInstance模式启动的Activity具有独占性，即它会独自占用一个任务，被他开启的任何activity都会运行在其他任务中（官方文档上的描述为，singleInstance模式的Activity
不允许其他Activity和它共存在一个任务中）

# Intent Flags
上面的场景仅仅适用于Activity启动Activity，并且采用的都是默认Intent，没有额外添加任何Flag，下面分析下几个常用FLAG的作用。
## FLAG_ACTIVITY_NEW_TASK
该FlAG的作用可以分为两类情形，一种是Activity启动Activity，另一种是非Activity(如Service)启动Activty。
singleTask和singleInstance这两种启动模式被预设置了Intent.FLAG_ACTIVITY_NEW_TASK，而standard及singletTop则不会设置。
非Activity启动Activity时必须添加此Flag，这个FLAG的关注重点是TASK，设置此flag后，大多数情况下新启动Activity就会被放置到自己taskAffinity的Task中。下面具体总结下：
1. 目标Activity实例或者Task不存在，则一定会新建Activity，并将目标Task移动到前台
2. 目标Task存在，目标Activity不存在，则新建Activity，并将目标Task移动到前台
3. 目标Task存在，目标Activity存在，但是不是根Activity，则新建Activity
4. 目标Task存在，目标Activity存在，且是根Activity，且intent和当前待启动的intent相等，则只要将Task移动至前台即可
## FLAG_ACTIVITY_SINGLE_TOP

## FLAG_ACTIVITY_CLEAR_TOP

## FLAG_ACTIVITY_CLEAR_TASK
https://juejin.im/post/59b0f25551882538cb1ecae1#heading-0