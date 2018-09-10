# 深入理解JNI

## 概述
JNI，是Java Native Interface的缩写，中文为Java本地调用。通俗地说，JNI是一种技术，通过这种技术可以做到以下两点：

·  Java程序中的函数可以调用Native语言写的函数，Native一般指的是C/C++编写的函数。

·  Native程序中的函数可以调用Java层的函数，也就是在C/C++程序中可以调用Java的函数。

在平台无关的Java中，为什么要创建一个和Native相关的JNI技术呢？这岂不是破坏了Java的平台无关特性吗？本人觉得，JNI技术的推出可能是出于以下几个方面的考虑：

·  承载Java世界的虚拟机是用Native语言写的，而虚拟机又运行在具体平台上，所以虚拟机本身无法做到平台无关。然而，有了JNI技术，就可以对Java层屏蔽具体的虚拟机实现上的差异了。这样，就能实现Java本身的平台无关特性。其实Java一直在使用JNI技术，只是我们平时较少用到罢了。

·  早在Java语言诞生前，很多程序都是用Native语言写的，它们遍布在软件世界的各个角落。Java出世后，它受到了追捧，并迅速得到发展，但仍无法对软件世界彻底改朝换代，于是才有了折中的办法。既然已经有Native模块实现了相关功能，那么在Java中通过JNI技术直接使用它们就行了，免得落下重复制造轮子的坏名声。另外，在一些要求效率和速度的场合还是需要Native语言参与的。


## 步骤
JNI技术也很照顾Java程序员，只要完成下面两项工作就可以使用JNI了，它们是：

- 加载对应的JNI库。

- 声明由关键字native修饰的函数。

## 注册JNI函数

正如代码中注释的那样，native_init函数对应的JNI函数是android_media_MediaScanner_native_init，可是细心的读者可能要问了，你怎么知道native_init函数对应的是这个android_media_MediaScanner_native_init，而不是其他的呢？莫非是根据函数的名字？

大家知道，native_init函数位于android.media这个包中，它的全路径名应该是android.media.MediaScanner.native_init，而JNI层函数的名字是android_media_MediaScanner_native_init。因为在Native语言中，符号“.”有着特殊的意义，所以JNI层需要把“.”换成“_”。也就是通过这种方式，native_init找到了自己JNI层的本家兄弟android.media.MediaScanner.native_init。

上面的问题其实讨论的是JNI函数的注册问题，“注册”之意就是将Java层的native函数和JNI层对应的实现函数关联起来，有了这种关联，调用Java层的native函数时，就能顺利转到JNI层对应的函数执行了。**而JNI函数的注册实际上有两种方法**，下面分别做介绍。

#### (1)静态方法

我们从网上找到的与JNI有的关资料，一般都会介绍如何使用这种方法完成JNI函数的注册，这种方法就是根据函数名来找对应的JNI函数。这种方法需要Java的工具程序javah参与，整体流程如下：

·  先编写Java代码，然后编译生成.class文件。

·  使用Java的工具程序javah，如javah–o output packagename.classname ，这样它会生成一个叫output.h的JNI层头文件。其中packagename.classname是Java代码编译后的class文件，而在生成的output.h文件里，声明了对应的JNI层函数，只要实现里面的函数即可。

这个头文件的名字一般都会使用packagename_class.h的样式



**需解释一下，静态方法中native函数是如何找到对应的JNI函数的。其实，过程非常简单：**

·  当Java层调用native_init函数时，它会从对应的JNI库Java_android_media_MediaScanner_native_linit，如果没有，就会报错。如果找到，则会为这个native_init和Java_android_media_MediaScanner_native_linit建立一个关联关系，其实就是保存JNI层函数的函数指针。以后再调用native_init函数时，直接使用这个函数指针就可以了，当然这项工作是由虚拟机完成的。

**从这里可以看出，静态方法就是根据函数名来建立Java函数和JNI函数之间的关联关系的，它要求JNI层函数的名字必须遵循特定的格式。这种方法也有几个弊端，它们是：**

- 需要编译所有声明了native函数的Java类，每个生成的class文件都得用javah生成一个头文件。

- javah生成的JNI层函数名特别长，书写起来很不方便。

- 初次调用native函数时要根据函数名字搜索对应的JNI层函数来建立关联关系，这样会影响运行效率。

有什么办法可以克服上面三种弊端吗？根据上面的介绍，Java native函数是通过函数指针来和JNI层函数建立关联关系的。如果直接让native函数知道JNI层对应函数的函数指针，不就万事大吉了吗？这就是下面要介绍的第二种方法：动态注册法。

#### (2)动态方法

既然Java native函数数和JNI函数是一一对应的，那么是不是会有一个结构来保存这种关联关系呢？答案是肯定的。在JNI技术中，用来记录这种一一对应关系的，是一个叫JNINativeMethod的结构，其定义如下：



```C
typedef struct {

    constchar* name;   //Java中native函数的名字，不用携带包的路径。例如“native_init“。

 

    const char* signature;//Java函数的签名信息，用字符串表示，是参数类型和返回值类型的组合。

    

    void* fnPtr;  //JNI层对应函数的函数指针，注意它是void*类型。

} JNINativeMethod;
```


**其实动态注册的工作，只用两个函数就能完成。总结如下：**


```
/*

env指向一个JNIEnv结构体，它非常重要，后面会讨论它。classname为对应的Java类名，由于

JNINativeMethod中使用的函数名并非全路径名，所以要指明是哪个类。

*/

jclass clazz =  (*env)->FindClass(env, className);

//调用JNIEnv的RegisterNatives函数，注册关联关系。

(*env)->RegisterNatives(env, clazz, gMethods,numMethods);
```

所以，在自己的JNI层代码中使用这种方法，就可以完成动态注册了。这里还有一个很棘手的问题：这些动态注册的函数在什么时候、什么地方被谁调用呢？好了，不卖关子了，直接给出该问题的答案：

·  **当Java层通过System.loadLibrary加载完JNI动态库后，紧接着会查找该库中一个叫JNI_OnLoad的函数，如果有，就调用它，而动态注册的工作就是在这里完成的。**

所以，如果想使用动态注册方法，就必须要实现JNI_OnLoad函数，只有在这个函数中，才有机会完成动态注册的工作。静态注册则没有这个要求，可我建议读者也实现这个JNI_OnLoad函数，因为有一些初始化工作是可以在这里做的。

## JNIEnv介绍
JNIEnv是一个和线程相关的，代表JNI环境的结构体

从上图可知，JNIEnv实际上就是提供了一些JNI系统函数。通过这些函数可以做到：

- 调用Java的函数。

- 操作jobject对象等很多事情。


上面提到说JNIEnv，是一个和线程有关的变量。也就是说，线程A有一个JNIEnv，线程B有一个JNIEnv。**由于线程相关，所以不能在线程B中使用线程A的JNIEnv结构体**。读者可能会问，JNIEnv不都是native函数转换成JNI层函数后由虚拟机传进来的吗？使用传进来的这个JNIEnv总不会错吧？是的，在这种情况下使用当然不会出错。**不过当后台线程收到一个网络消息，而又需要由Native层函数主动回调Java层函数时，JNIEnv是从何而来呢？**根据前面的介绍可知，我们不能保存另外一个线程的JNIEnv结构体，然后把它放到后台线程中来用。这该如何是好？

还记得前面介绍的那个JNI_OnLoad函数吗？**它的第一个参数是JavaVM**，它是虚拟机在JNI层的代表，代码如下所示：

//全进程只有一个JavaVM对象，所以可以保存，任何地方使用都没有问题。

jint JNI_OnLoad(JavaVM* vm, void* reserved)

正如上面代码所说，**不论进程中有多少个线程，JavaVM却是独此一份，所以在任何地方都可以使用它。**那么，JavaVM和JNIEnv又有什么关系呢？答案如下：

**调用JavaVM的AttachCurrentThread函数，就可得到这个线程的JNIEnv结构体。这样就可以在后台线程中回调Java函数了。**

**另外，后台线程退出前，需要调用JavaVM的DetachCurrentThread函数来释放对应的资源。**

## 通过JNIEnv操作jobject

前面提到过一个问题，即Java的引用类型除了少数几个外，最终在JNI层都用jobject来表示对象的数据类型，那么该如何操作这个jobject呢？

从另外一个角度来解释这个问题。一个Java对象是由什么组成的？当然是它的成员变量和成员函数了。那么，操作jobject的本质就应当是操作这些对象的成员变量和成员函数。所以应先来看与成员变量及成员函数有关的内容。

#### （1）jfieldID 和jmethodID的介绍
我们知道，成员变量和成员函数是由类定义的，它是类的属性，所以在JNI规则中，用jfieldID 和jmethodID 来表示Java类的成员变量和成员函数，它们通过JNIEnv的下面两个函数可以得到：


```
jfieldID GetFieldID(jclass clazz,const char*name, const char *sig);

jmethodID GetMethodID(jclass clazz, const char*name,const char *sig);
```


其中，jclass代表Java类，name表示成员函数或成员变量的名字，sig为这个函数和变量的签名信息。如前所示，成员函数和成员变量都是类的信息，这两个函数的第一个参数都是jclass。

#### （2）使用jfieldID和jmethodID


```
virtua lbool scanFile(const char* path, long long lastModified,

long long fileSize)

    {

       jstring pathStr;

        if((pathStr = mEnv->NewStringUTF(path)) == NULL) return false;

       

/*

调用JNIEnv的CallVoidMethod函数，注意CallVoidMethod的参数：

第一个是代表MediaScannerClient的jobject对象，

第二个参数是函数scanFile的jmethodID，后面是Java中scanFile的参数。

*/

       mEnv->CallVoidMethod(mClient, mScanFileMethodID, pathStr,

lastModified, fileSize);

 

       mEnv->DeleteLocalRef(pathStr);

       return (!mEnv->ExceptionCheck());

}
```

白了，通过JNIEnv输出的CallVoidMethod，再把jobject、jMethodID和对应参数传进去，JNI层就能够调用Java对象的函数了！

实际上JNIEnv输出了一系列类似CallVoidMethod的函数，形式如下：


```
NativeType Call<type>Method(JNIEnv *env,jobject obj,jmethodID methodID, ...)。
```


其中type是对应Java函数的返回值类型，例如CallIntMethod、CallVoidMethod等。

上面是针对非static函数的，如果想调用Java中的static函数，则用JNIEnv输出的CallStatic<Type>Method系列函数。

现在，我们已了解了如何通过JNIEnv操作jobject的成员函数，那么怎么通过jfieldID操作jobject的成员变量呢？这里，直接给出整体解决方案，如下所示：


```
//获得fieldID后，可调用Get<type>Field系列函数获取jobject对应成员变量的值。

NativeType Get<type>Field(JNIEnv *env,jobject obj,jfieldID fieldID)

//或者调用Set<type>Field系列函数来设置jobject对应成员变量的值。

void Set<type>Field(JNIEnv *env,jobject obj,jfieldID fieldID,NativeType value)
```

//下面我们列出一些参加的Get/Set函数。

GetObjectField()         SetObjectField()

GetBooleanField()         SetBooleanField()

GetByteField()           SetByteField()

GetCharField()           SetCharField()

GetShortField()          SetShortField()

GetIntField()            SetIntField()

GetLongField()           SetLongField()

GetFloatField()          SetFloatField()

GetDoubleField()                  SetDoubleField()


通过本节的介绍，相信读者已了解jfieldID和jmethodID的作用，也知道如何通过JNIEnv的函数来操作jobject了。虽然jobject是透明的，但有了JNIEnv的帮助，还是能轻松操作jobject背后的实际对象了。


## JNI中的异常处理

JNI中也有异常，不过它和C++、Java的异常不太一样。当调用JNIEnv的某些函数出错后，会产生一个异常，但这个异常不会中断本地函数的执行，直到从JNI层返回到Java层后，虚拟机才会抛出这个异常。虽然在JNI层中产生的异常不会中断本地函数的运行，但一旦产生异常后，就只能做一些资源清理工作了（例如释放全局引用，或者ReleaseStringChars）。如果这时调用除上面所说函数之外的其他JNIEnv函数，则会导致程序死掉。


来看一个和异常处理有关的例子，代码如下所示：


```
virtual bool scanFile(const char* path, long long lastModified,

long long fileSize)

 {

       jstring pathStr;

       //NewStringUTF调用失败后，直接返回，不能再干别的事情了。

        if((pathStr = mEnv->NewStringUTF(path)) == NULL) return false;

       ......

}
```


```
JNI层函数可以在代码中截获和修改这些异常，JNIEnv提供了三个函数进行帮助：

ExceptionOccured函数，用来判断是否发生异常。

ExceptionClear函数，用来清理当前JNI层中发生的异常。

ThrowNew函数，用来向Java层抛出异常。
```

## 参考资料

[[深入理解Android卷一 全文-第二章]深入理解JNI](https://blog.csdn.net/innost/article/details/47204557)
