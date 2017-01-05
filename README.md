# Android-Plugin-Framework

此项目是Android插件开发框架完整源码及示例。用来通过动态加载的方式在宿主程序中运行插件APK。

# 如何使用

    1、新建一个工程, 作为宿主工程
    
    2、在宿主工程的build.gradle文件下添加如下3个配置
         
        android {
            defaultConfig {
                //这个配置不可省略
                applicationId 宿主app包名        
            }
        }
        
        dependencies {
            compile('com.limpoxe.fairy:FairyPlugin:0.0.42-snapshot@aar')
            //optional， 用于支持函数式服务，不使用函数服务不需要添加此依赖
            compile('com.limpoxe.support:android-servicemanager:1.0.5@aar')
        }
        
        apply from: "https://raw.githubusercontent.com/limpoxe/Android-Plugin-Framework/master/FairyPlugin/host.gradle"
        
    3、在宿主工程中新建一个类继承自Application类, 并配置到AndroidManifest.xml中
       重写这个类的2个方法
       
        @Override
        protected void attachBaseContext(Context base) {
            super.attachBaseContext(base);
            PluginLoader.initLoader(this);
            //这个方法是设置首次加载插件时, 定制loading页面的UI, 不传即默认没有loading页
            //在宿主中创建任意一个layout传进去即可
            PluginLoader.setLoadingResId(R.layout.loading);
        }

	    @Override
	    public Context getBaseContext() {
		    return PluginLoader.fixBaseContextForReceiver(super.getBaseContext());
	    }
	    
	4、在宿主工程中通过下面3个方法进行插件操作
	    安装: PluginManagerHelper.installPlugin( SDcard上插件apk的路径 );
	    卸载: PluginManagerHelper.remove( 插件packageName );
	    列表: PluginManagerHelper.getPlugins();
	    
	5、在宿主中打开插件
	
	    //打开插件的Launcher界面
	    Intent launchIntent = getPackageManager().getLaunchIntentForPackage( 插件packageName );			
    	startActivity(launchIntent);
    	
    
    ------------------------------------
    ----下面是插件工程相关的说明
    ------------------------------------
    
    1、新建一个插件工程
    
    2、如果是独立插件,无需任何配置, 一个普通的apk编译出来即可当插件apk安装到宿主中
    
    3、如果是非独立插件, 需要下面2个步骤
     
        3.1、在插件AndroidManifest.xml中manifest节点中增加如下配置:
             
             <manifest
                    android:sharedUserId="这里填写宿主工程包名"/>
             
              这个配置只是作为一个标记符使用，框架通过检查这个配置来判断是否为独立插件，和起原始含义无关
                            
        3.2、在build.gradle中加入如下3个配置
    
            dependencies {
                //这个配置的意思是使插件运行时依赖宿主中的jar、依赖宿主依赖的aar中的jar
                //根据自己的实际情况修改为想要依赖的jar的路径即可
                //这些jar不会被打包到插件中，
                provided files(project(':Samples:PluginMain').getBuildDir().absolutePath + '/xxx/xxx/xxx.jar')
            }
            
            ext {
                //这2个常量在下面的apply的脚本中要用到
                //host_output_dir表示指定宿主工程的编译输出目录
                //host_ap_path表示指定宿主工程编译后的中间产物 .ap_ 文件的路径
                host_output_dir = project(':Samples:PluginMain').getBuildDir().absolutePath + "/outputs"
                
                //注意这里的.ap_文件，是通过编译宿主产生，不同的buildType请更换成不同的名字
                host_ap_path = host_output_dir+ '/PluginMain-resources-debug.ap_'
            }
        
            apply from: "https://raw.githubusercontent.com/limpoxe/Android-Plugin-Framework/master/FairyPlugin/plugin.gradle"
              
        3.3、如果此插件还依赖其他运行时公共插件，比如将网络请求库从宿主中剔除出来作为单独的公共插件使用，参考pluginbase用法
                在插件AndroidManifest.xml中application节点中增加如下配置:
               <uses-library android:name="xx.xx.xx" android:required="true" />
                xx.xx.xx 是其依赖的插件的包名，被依赖的插件不支持携带资源
                
        3.4、如果此插件需要对外提供函数式服务（支持同进程和跨进程）
             插件和外界交互，除了可以使用标准api交互之外，还提供了函数式服务
             插件发布一个函数服务，只需要在AndroidManifest.xml配置<exported-service>节点
             其他插件或者宿主即可调用此服务，具体参考demo
             
        3.5  如果此插件需要对外提供嵌入式Fragment
             如果需要在其他插件，或者宿主的Activity中嵌入插件提供的Fragment
             需要在AndroidManifest.xml配置<exported-fragment>节点，具体参考demo
             
        3.6 如果此插件需要对外提供嵌入式View     
            如果需要在其他插件，或者宿主的Activity中嵌入插件提供的View
            需要在其他插件，或者宿主的的布局文件中嵌入<pluginView> 节点，用法同普通控件，具体参考demo
                
        3.7 如果插件需要使用宿主中定义的主题
            插件中可以直接使用宿主中定义的主题。例如，宿主中定义了一个主题为AppHostTheme，
            那么插件的Manifest文件中可直接使用android:theme="@style/AppHostTheme"来使用宿主主题
             
            如果插件要扩展宿主主题，也可以直接使用。例如，在插件的style.xml中定义一个主题PluginTheme
            <style name="PluginTheme" parent="AppHostTheme" />
             
            以上写法，IDE中会标红，但是不影响编译和运行。
            
        3.8 如果插件需要使用宿主中定义的其他资源
            插件中可以直接使用宿主中定义的资源，但是写法和直接使用插件自己的资源略有不同,
            通常应写为：
                android:label="@string/string_in_plugin"
            但是上面只适用资源在插件中的情况，如果资源是定义在宿主中，应使用下面的写法
                android:label="@*xx.xx.xx:string/string_in_host"
            其中，xx.xx.xx是宿主包名
            
            注意本条与上一条3.7的区别：插件中使用宿主主题，可以直接使用，但是使用宿主资源，需要带上*packageName:前缀
            
        3.9 如果要在插件中获取宿主包名
            插件返回的是插件包名，如果要在插件中获取宿主包名，写法如下：
            //参数为插件的context，例如插件activity或者插件Application
            public String getHostPackageName(ContextWrapper pluginContext) {
                  Context context = pluginContext;
                  while (context instanceof ContextWrapper) {
                    context = ((ContextWrapper) context).getBaseContext();
                  }
                  //到这里context的实际类型应当是ContextImpl类，可以返回宿主packageName
                  return context.getPackageName();
            }
            
        3.10 如果插件中要使用需要appkey的sdk
            插件中使用需要appkey的sdk，例如微信分享和百度地图sdk，参考demo：baidumapsdk，wxsdklibrary
             
    4、如果是非独立插件, 需要先编译宿主, 再编译插件, 因为从如上的配置可以看出非独立插件编译时需要依赖宿主编译时的输出物
       如果是非独立插件, 需要先编译宿主, 再编译插件
       如果是非独立插件, 需要先编译宿主, 再编译插件
       如果是非独立插件, 需要先编译宿主, 再编译插件
       重要的事情讲3遍！遇到编译问题请先编译宿主, 再编译插件
       
    以上所有内容及更多详情可以参考Demo
 
# 已支持的功能：
  1、插件apk无需安装，由宿主程序动态加载运行。
  
  2、支持fragment、activity、service、receiver、contentprovider、so、application、notification。
  
  3、支持插件自定义控件、宿主自定控件。
  
  4、开发插件apk和开发普通apk时代码编写方式无区别。对插件apk和宿主程序来说，插件框架完全透明，开发插件apk时无约定、无规范约束。
  
  5、插件中的组件拥有真正生命周期，完全交由系统管理、非反射代理
  
  6、支持插件引用宿主程序的依赖库、插件资源、宿主资源、以及插件依赖插件。
  
  7、支持插件使用宿主主题、系统主题、插件自身主题以及style、轻松支持皮肤切换
  
  8、支持非独立插件和独立插件（非独立插件指自己编译的需要依赖宿主中的公共类和资源的插件，不可独立安装运行。独立插件又分为两种：
     一种是自己编译的不需要依赖宿主中的类和资源的插件，可独立安装运行；一种是第三方发布的apk，如从应用市场下载的apk，可独立安装
     运行，这种只做了简单支持。）
  
  9、支持插件Activity的4个LaunchMode

  10、支持插件资源文件中直接通过@xxx方式引用共享依赖库中的资源

  11、支持插件发送notification时在RemoteViews携带插件自定义的布局资源（只支持5.x、6.x,不支持更高或者更低版本）

  12、支持插件热更新：即在插件模块已经被唤起的情况先安装新版本插件，无需重启进程

  13、支持在插件中注册全局服务：即在某个插件中注册一个服务，在其他插件中通过LocalServiceManager获取这个服务

# 不支持的功能：

  1、插件Activity切换动画不支持使用插件自己的资源。
  
  2、不支持插件申请权限，权限必须预埋到宿主中。
  
  3、不支持第三方app试图唤起插件中的组件时直接使用插件app的Intent。即插件app不能认为自己是一个正常安装的app。
     第三方app要唤起插件中的静态组件时必须由宿主程序进行桥接，方法请参看wxsdklibrary工程的用法

  4、不支持android.app.NativeActivity
  
  5、Notification在5.x以下不支持使用插件资源, 在5.x及以上仅支持在RemoteView中使用插件资源
  
  6、插件依赖另一个插件时，被插件依赖的插件暂不支持包含资源

# 目录结构说明：

  1、FairyPlugin工程是插件库核心工程，用于提供对插件功能的支持。

  2、PluginMain是用来测试的宿主程序Demo工程。

  3、PluginShareLib是用来测试非独立插件的公共依赖库Demo工程。
  
  4、PluginTest是用来测试的非独立插件Demo工程。
  
  5、PluginTest1是用来测试的独立插件Demo工程。
  
  6、PluginHelloWorld是用来测试的独立插件Demo工程。

  7、PluginBase是用来测试的被PluginTest插件依赖的插件Demo工程（此插件被PluginTest、wxsdklibrary两个插件依赖），此插件不包含资源。
  
  8、wxsdklibrary是用来测试的非独立插件Demo工程。
  
# demo安装说明：

  1、宿主程序demo工程的assets目录下已包含了编译好的独立插件demo apk和非独立插件demo apk。

  2、宿主程序demo工程根目录下已包含一个已经编译好的宿主demo，可直接安装运行。

  3、宿主程序demo工程源码可直接编译安装运行。
  
  4、插件demo工程：

     1、若使用master分支：
        直接编译即可，无特别要求。

     2、若使用For-gradle-with-aapt分支：
        将openAtlasExtention@github项目提供的BuildTools替换自己的Sdk中相应版本的BuildTools。剩下的步骤照常即可。

     3、若使用For－eclipse－ide分支：
        需要使用ant编译，关注PluginTest工程的ant.properties文件和project.properties文件以及custom_rules.xml,若编译失败，请升级androidSDK。

    4、编译方法

       a）如果是命令行中：
       cd  Android-Plugin-Framework
       ./gradlew clean
       ./gradlew build

       b）如果是studio中：
       打开studio右侧gradle面板区，点clean、点build。不要使用菜单栏的菜单编译。

       重要：
            1、由于编译脚本依赖Task.doLast, 使用其他编译方法（如菜单编译）可能不会触发Task.doLast导致编译失败
            2、必须先编译宿主，再编译非独立插件。（这也是使用菜单栏编译会失败的原因之一）
               原因很简单，既然是非独立插件，肯定是需要引用宿主的类和资源的。所以编译非独立插件时会用到编译宿主时的输出物
       
            所以如果使用其他编译方法，请务必仔细阅读build.gradle，了解编译过程和依赖关系后可以自行调整编译脚本，否则可能会失败。

       待插件编译完成后，插件的编译脚本会自动将插件demo的apk复制到PlugiMain/assets目录下（复制脚本参看插件工程的build.gradle）,然后重新打包安装PluginMain。
       或者也可将插件apk复制到sdcard，然后在宿主程序中调用PluginLoader.installPlugin("插件apk绝对路径")进行安装。
       
    5、需要在android studio中关闭instantRun选项。因为instantRun会替换宿主的application配置导致框架初始化异常


# 开发注意事项

    1、非独立插件开发需要解决插件资源id和宿主资源id重复产生的冲突问题。

        解决冲突的方式有如下两种：

        a）通过在宿主中添加一个public.xml文件来解决资源id冲突（master分支采用的方案）

        b）通过定制过的aapt在编译插件时指定id范围来解决冲突（For-gradle-with-aapt分支采用的方案）
           此方案需要替换sdk原生的aapt，且要区分多平台，buildTools版本更新后需同步升级aapt。
           定制的aapt由 openAtlasExtention@github 项目提供，目前的版本是基于22.0.1，将项目中的BuildTools替换
           到本地Android Sdk中相应版本的BuildTools中，
           并指定gradle的buildTools version为对应版本即可。

    2、非独立插件中的class不能同时存在于宿主和插件程序中，因此其引用的公共库仅参与编译，不参与打包，参看demo中的gradle脚本。
    
    3、若插件中包含so，则需要在宿主中添加一个占位的so文件。占位so可随意创建，随意命名，关键在于so所在的cpuarch目录要正确。
       在pluginMain工程。pluginMain工程中的libstub.so其实只是一个txt文件。

       需要占位so的原因是，宿主在安装时，系统会扫描宿主中的so的（根据文件夹判断）类型，决定宿主在哪种cpu模式下运行、并保持到系统设置里面。
       (系统源码可查看com.android.server.pm.PackageManagerService.setBundledAppAbisAndRoots()方法)
       例如32、64、还是x86等等。如果宿主中不包含so，系统默认会选择一个最适合当前设备的模式。
       那么问题来了，如果系统默认选择的模式，和将来下载安装的插件中的so支持的模式不匹配，则会出现so异常。

       因此需要提前通知系统，宿主需要在哪些cpu模式下运行。提前通知的方式即内置占位so。

    4、插件默认是在插件进程中运行，如需切到宿主进程，仅需将core工程的Manifest中配置的所有组件去都去掉掉process属性即可。
       PluginMain工程下有几个插件进程的demo也需要去掉process属性

    5、将配置插件为非独立插件、为插件配置依赖插件的方法。
       插件框架识别一个插件是否为独立插件，是根据插件的Manifest文件中的android:sharedUserId属性。
       将android:sharedUserId属性设置为宿主的packageName，则表示为非独立插件。不配置或配置为其他值表示为独立插件

       插件如果依赖其他基础插件，需要在插件Manifest中配置如下信息
       <uses-library android:name="XX.XX.XX" android:required="true" />
       name是被依赖的插件的packageName

    6、框架中对非独立插件做了签名校验。如果宿主是release模式，要求插件的签名和宿主的签名一致才允许安装。
       这是为了验证插件来源合法性

# 需要注意的问题

   1、项目插件化后，特别需要注意的是宿主程序混淆问题。
      若公共库混淆后，可能会导致非独立插件程序运行时出现classnotfound，原因很好理解。
      所以公共库一定要排除混淆或者使用稳定的mapping混淆。
      
      具体方法：
            1、开启混淆编译宿主，保留mapping文件
            2、将插件的build.gradle文件中的provided配置换成compile， 因为provided方式提供的包不会被混淆
            3、在插件的混淆配置中apply编译宿主时产生的mapping文件。
            4、接着在插件编译脚本中开启multdex编译。并配置multdex的mainlist，使得原先所有provided的包的class被打入到副dex中。
               这样插件编译完成后，会有2个dex，1个是插件自己需要的代码，1个是原先provided后来改成了compile的那些包。
            5、再将这个原provided的包形成的dex，也就是副dex从apk中删除，再对插件apk重新签名。
            
            上述方法作者也未试过，理论上可以解决公共库混淆问题。
            gradle插件在1.5版本以后去除了指定mainlist的功能，因此在高于这个版本时指定multidex分包需要使用其他分包插件。
            可使用这个项目：https://github.com/ceabie/DexKnifePlugin
      
      若需要混淆core工程的代码，请参考PluginMain工程下的混淆配置

   2、插件编译问题。
  
     如果插件和宿主共享依赖库，常见的如supportv4，那么编译插件的时候不可将共享库编译到插件当中，
     包括共享库的代码以及R文件，只需在编译时添加到classpath中，且插件中如果要使用共享依赖库中的资源，
     需要使用共享库的R文件来进行引用。这几点在PluginTest示例工程中有体现。

     更新：已接入gradle，通过provided方式即可，具体可参考PluginShareLib和PluginTest的build.gradle文件
  
  3、插件Fragment
    插件UI可通过fragment或者activity来实现        
    
    如果是fragment实现的插件，又分为3种：
    1、fragment运行在宿主中的普通Activity中
    2、fragment运行在宿主中的特定Activity中
    3、fragment运行在插件中的Activity中

    对第2种和第3种，fragmet的开发方式和正常开发方式没有任何区别
    
    对第1种，fragmeng中凡是要使用context的地方，都需要使用通过PluginLoader.getDefaultPluginContext(FragmentClass)或者
    通过context.createPackageContext(插件包名)获取的插件context，
    那么这种fragment对其运行容器没有特殊要求
    
    第1种Activity和第2种Activity，两者在代码上没有任何区别。主要是插件框架在运行时需要区分注入的Context的类型。
    
    demo中都有例子。
   
   4、android sdk中的build tools版本较低时也无法编译public.xml文件，因此如果采用public.xml的方式，应使用较新版本的buildtools。

   5、For-gradle-with-aapt分支使用的是aapt分组id方法。master也支持切换分组id的方法
   
   6、本项目除master分支外，其他分支不会更新维护。

##其他
1. [原理简介](https://github.com/limpoxe/Android-Plugin-Framework/wiki/%E5%8E%9F%E7%90%86%E7%AE%80%E4%BB%8B)
2. [使用Public.xml实现插件和宿主资源id分组需要注意的问题](https://github.com/limpoxe/Android-Plugin-Framework/wiki/%E4%BD%BF%E7%94%A8Public.xml%E5%AE%9E%E7%8E%B0%E6%8F%92%E4%BB%B6%E5%92%8C%E5%AE%BF%E4%B8%BB%E8%B5%84%E6%BA%90id%E5%88%86%E7%BB%84%E9%9C%80%E8%A6%81%E6%B3%A8%E6%84%8F%E7%9A%84%E9%97%AE%E9%A2%98).
       
      
# 更新记录：

    2016-12-21： 修复Android7.1.1下webview相关问题

    2016-12-5：  1、修复一些bug和兼容问题
                 2、添加对非独立插件res使用declare-styleable的支持

    2016-10-22： 支持Android7.0，代码优化、重构
    
    2016-10-12： 框架目录结构重构
    
    2016-09-28： 1、添加对插件Context的getPackageName返回插件自身PackageName的支持
                 2、剥离LocalServiceManager
                 3、若干优化;修改若干bug和兼容性问题、
                 4、增加插件API版本检查
                 5、编译脚本优化
               
    2016-05-18： 添加对函数式服务localservice的跨进程支持

    2016-05-07： 分离插件data目录、优化编译脚本、修复若干bug

    2016-04-08： 修复若干bug，优化

    2016-02-24： 1、添加插件进程
                 2、添加插件MultiDex支持

    2016-02-18： 增加对插件使用宿主主题的支持，例如supportV7主题

    2016-01-27： 1、修复几个bug
                 2、增加对插件Activity透明主题支持

    2016-01-16： 对系统服务增加了一层Proxy，以支持拦截系统服务的方法调用

    2016-01-01： 1、添加对插件依赖插件的支持
                 2、添加localservice

    2015-12-27： 添加控件插件支持。可在宿主或插件布局文件中直接嵌入其他插件中定义的控件

    2015-12-05： 1、修复插件so在多cpu平台下模式选择错误的问题
                 2、添加对基于主题style和自定义属性的换肤功能

    2015-11-22： 1、gradle插件1.3.0以上版本不支持public.xml文件也无法识别public-padding节点的文件的问题已解决，
                    因此master分支切回到利用public.xml分组的实现
                 2、支持插件资源文件直接通过@package:type/name方式引用宿主资源

    2015-05-04： add project

联系作者：
  Q：15871365851，添加时请注明插件开发

  Q群：116993004、207397154(已满)，重要：添加前请务必仔细阅读此ReadMe！请务必仔细阅读Demo！
