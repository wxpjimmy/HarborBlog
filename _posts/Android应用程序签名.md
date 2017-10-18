title: Android应用程序签名
date: 2015-11-01 16:00:48
tags: [android, 签名]
---
最近工作中需要封装一个客户端的sdk包，写了一个简单的示例程序，准备安装到手机的时候提示程序没有签名，安装失败，特查了一下android包的签名方式，在此记录一下，以备后用。  
# 签名的意义    
为了保证每个应用程序开发商合法ID，防止部分开放商可能通过使用相同的Package Name来混淆替换已经安装的程序，我们需要对我们发布的APK文件进行唯一签名，保证我们每次发布的版本的一致性(如自动更新不会因为版本不一致而无法安装)。

# APK签名步骤  
1. 生成keystore
2. 用jarsigner签名apk
3. zipalign对齐优化apk文件（可选）  
下面分别介绍一下这三部分。 
 
# 生成keystore
这一步需要使用keytool这个工具，这个工具是跟随JDK一起安装的，所以只要安装完java设置好路径后一般都是可以直接在命令行使用的。打开命令行，执行下面命令：
> keytool -genkey -alias abc.keystore -keyalg RSA -validity 20000 -keystore abc.keystore  

* -genkey 产生密钥  
* -alias abc.keystore 别名 abc.keystore  (abc只是示例所用的名字，可以根据需要自己重新命名的）  
* -keyalg RSA 使用RSA算法对签名加密  
* -validity 20000 有效期限20000天  
* -keystore abc.keystore  （一般要跟alias保持一致）  
执行完这个命令后会要求输入一堆信息，示例如下：  
{% asset_img sample.png [input sample] %}  
执行成功的话会在当前目录下看到生成的abc.keystore文件  

# 用jarsigner签名apk  
jarsigner这个tool也是跟随JDK一起安装的，命令行可以直接使用。命令如下:    
< jarsigner -verbose -keystore abc.keystore -signedjar demo_signed.apk demo.apk abc.keystore  

* -verbose 输出签名的详细信息
* -keystore  abc.keystore 密钥库位置
* -signedjar demo_signed.apk demo.apk abc.keystore 正式签名，三个参数中依次为签名后产生的文件demo_signed，要签名的文件demo.apk和密钥库abc.keystore.  
提示输入密码的时候记得输入生成keystore文件时设置的密码  

# zipalign优化生成的apk文件  
未签名的apk不能使用，也不能优化。签名之后的apk谷歌推荐使用zipalign.exe(位于下载的android-sdk包中)工具对其优化，命令如下：  
> zipalign -v 4 demo_signed.apk final.apk  

zipalign能够使apk文件中未压缩的数据在4个字节边界上对齐（4个字节是一个性能很好的值），这样android系统就可以使用mmap()(请自行查阅这个函数的用途)函数读取文件，可以在读取资源上获得较高的性能，在4个字节边界上对齐的意思就是，一般来说，是指编译器吧4个字节作为一个单位来进行读取的结果，这样的话，CPU能够对变量进行高效、快速的访问（较之前不对齐）。对齐的根源是android系统中的Davlik虚拟机使用自己专有的格式DEX，DEX的结构是紧凑的，为了让运行时的性能更好，可以进一步用"对齐"进一步优化，但是大小一般会有所增加。  

经过以上三个步骤，最终生成的final.apk文件就可以通过adb install成功安装到手机上了。

#  签名对APP的影响  
你不可能只做一个APP，你可能有一个宏伟的战略工程，想要在生活，服务，游戏，系统各个领域都想插足的话，你不可能只做一个APP，谷歌建议你把你所有的APP都使用同一个签名证书。  
使用你自己的同一个签名证书，就没有人能够覆盖你的应用程序，即使包名相同，所以影响有:  
1. App升级。 使用相同签名的升级软件可以正常覆盖老版本的软件，否则系统比较发现新版本的签名证书和老版本的签名证书不一致，不会允许新版本安装成功的。
2. App模块化。android系统允许具有相同的App运行在同一个进程中，如果运行在同一个进程中，则他们相当于同一个App，但是你可以单独对他们升级更新，这是一种App级别的模块化思路。
3. 允许代码和数据共享。android中提供了一个基于签名的Permission标签。通过允许的设置，我们可以实现对不同App之间的访问和共享，如下：
> AndroidManifest.xml：&lt;permission android:protectionLevel="normal" />

其中protectionLevel标签有4种值：normal(缺省值),dangerous, signature,signatureOrSystem。简单来说，normal是低风险的，所有的App不能访问和共享此App。dangerous是高风险的，所有的App都能访问和共享此App。signature是指具有相同签名的App可以访问和共享此App。signatureOrSystem是指系统image中App和具有相同签名的App可以访问和共享此App，谷歌建议不要使用这个选项，因为签名就足够了，一般这个许可会被用在在一个image中需要共享一些特定的功能的情况下。

最后，请一定要记得保管好你的签名证书的密码。
