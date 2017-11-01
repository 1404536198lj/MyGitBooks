# Android makefile && hal层到framework的demon 
## 第一章 Android makefile
&emsp;&emsp;makefile是make的输入，定义了编译规则和编译顺序，包括：哪些文件需要编译，怎么编译（gcc还是javac）、哪些文件存在依赖关系、优先编译哪些文件。
<br>&emsp;&emsp;android并没有采用Recursive make的方式进行编译，因为该方式存在“do too little ”和“do too much”的缺陷。针对该缺陷，android基于以下要求来设计android编译系统：
<br>&emsp;&emsp;（1）makefile片段化
<br>&emsp;&emsp;android采用片段化makefile文件的方式进行模块化编译，每个模块下都有一个android.mk文件，该文件的内容是makefile片段，其中定义的是编译规则，另外还有一些配置信息存放于其他的mk文件（例如与具体机型相关的BoardConfig.mk和AndroidProducts.mk）中，最终通过build/core/main.mk 将所有的mk文件汇集为makefile文件。
<br>&emsp;&emsp;（2）自动生成依赖
<br>&emsp;&emsp;&emsp;android自动生成依赖，只需要调用android接口定义编译规则即可，不需要关心依赖关系。
<br>&emsp;&emsp;（3）编译出多个目标
<br>&emsp;&emsp;&emsp;除了Android的最终产物，还能编译出其他的实用工具。
<br>&emsp;&emsp;（4）支持多平台
<br>&emsp;&emsp;&emsp;android能在linux和mac平台编译，编译出的产出也需要支持linux和mac
<br>&emsp;&emsp;.........and so on
<br>&emsp;&emsp;最终的编译产物放在out目录下，该目录下还有/host和/target两个子目录，分别表示PC和手机上的编译产物。
<br>&emsp;&emsp;android.mk详细知识可参阅：[android.mk学习笔记][1]和[Android Makefile 文件讲解][2]



## 第二章 hal层到framework层的demon

### 2.1 HAL层

#### 2.1.1 lijiao.h
&emsp;&emsp;在android源码根目录下的hardware/libhardware/include/文件夹下新建头文件，例如：lijiao.h
```c
#define ANDROID_LIJIAO_INTERFACE_H  
#include <hardware/hardware.h>  
     
__BEGIN_DECLS  
    	 
/*定义模块ID*/  
#define LIJIAO_HARDWARE_MODULE_ID "lijiao" 
    	
/*定义硬件模块结构体*/
struct lijiao_module_t{
    struct hw_module_t m;
};
    	
/*定义硬件模块接口*/
    struct lijiao_device_t{
    	struct hw_device_t d;
    	//int fd;
    	int (*hal_set)(struct lijiao_device_t* dev, int val);
    	int (*hal_get)(struct lijiao_device_t* dev, int * val);
};
    	
__END_DECLS
```
&emsp;&emsp;这里按照Android硬件抽象层规范的要求分别定义了模块ID、硬件模块结构体、硬件模块接口。__BEGIN_DECLS和 __END_DECLS这两个宏，该宏定义包含extern C,使得C和C++能够兼容。

#### 2.1.2 lijiao.c

&emsp;&emsp;接下来在hardware/libhardware/modules下新建目录lijiao，并在lijiao目录下新建c文件：lijiao.c
```c
#define LOG_TAG "LijiaoHal" 
      
#include <stdlib.h>
#include <hardware/hardware.h>  
#include <hardware/lijiao.h>  
#include <fcntl.h>  
#include <errno.h>  
#include <cutils/log.h>  
#include <cutils/atomic.h>  
#include <string.h>

      
#define DEVICE_NAME "/sys/class/leds/lcd-backlight/brightness"  
#define MODULE_NAME "lijiao"  
#define MODULE_AUTHOR "lijiao1@meizu.com"  


/*设备打开和关闭接口*/  
static int lijiao_device_open(const struct hw_module_t* module,const char* name, struct hw_device_t** device);
static int lijiao_device_close(struct hw_device_t* device);  
  
/*设备访问接口*/  
static int lijiao_set_val(struct lijiao_device_t* dev, int val);  
static int lijiao_get_val(struct lijiao_device_t* dev, int* val); 


/*封装lijiao_device_open方法*/
static struct hw_module_methods_t lijiao_module_methods = {  
    .open= lijiao_device_open  
};


/*定义lijiao_device_open方法*/
static int lijiao_device_open(const struct hw_module_t* module,const char* name, struct hw_device_t** device){
	struct lijiao_device_t * dev;
	dev = (struct lijiao_device_t *)malloc(sizeof(struct lijiao_device_t));

	if(!dev) {  
        	ALOGE("lijiao hal: failed to alloc space");
        return -EFAULT;  
	}
	memset(dev, 0, sizeof(struct lijiao_device_t)); 

	dev->d.tag = HARDWARE_DEVICE_TAG;  
    dev->d.version = 0;
    dev->d.module = (hw_module_t*)module;
    dev->d.close = lijiao_device_close;
    dev->hal_set = lijiao_set_val;
	dev->hal_get = lijiao_get_val;
	*device = &(dev->d);  
    return 0;
	}


/*定义lijiao_device_close方法*/
static int lijiao_device_close(struct hw_device_t* device) {  
    struct lijiao_device_t* lijiao_device = (struct lijiao_device_t*)device;  
  
    if(lijiao_device) {  

        free(lijiao_device);  
    }  
      
    return 0;  
}

/*实例化lijiao_module_t*/
struct lijiao_module_t HAL_MODULE_INFO_SYM = {
	.m =  {  
       		.tag=HARDWARE_MODULE_TAG,  
        	.version_major=1,  
       		.version_minor=0,  
        	.id=LIJIAO_HARDWARE_MODULE_ID,  
        	.name=MODULE_NAME,  
        	.author=MODULE_AUTHOR,  
        	.methods= &lijiao_module_methods,  
    	   }  
}; 

/*定义lijiao_set_val方法 */
static int lijiao_set_val(struct lijiao_device_t* dev, int val){

    ALOGI("lijiao hal: lijiao_set_val started");
	int fd = open(DEVICE_NAME, O_RDWR);
    if(fd >= 0)
    {
        ALOGI("lijiao hal: open /sys/class/leds/lcd-backlight/brightness successfully.");
        char buffer[20];
        int bytes = sprintf(buffer, "%d\n", val);
        ssize_t amt = write(fd, buffer, (size_t)bytes);
        if(amt == -1 )
        {
             ALOGI("lijiao hal: write fail.");
             close(fd);
             free(dev);
             return -errno;
        }
        ALOGI("lijiao hal: set value %d to device.", val);
        close(fd);
    	return 0;
    }
    ALOGE("lijiao hal: failed to open /sys/class/leds/lcd-backlight/brightness -- %s.", strerror(errno));
    free(dev);
    return -errno;

}

/*定义lijiao_get_val方法 */
static int lijiao_get_val(struct lijiao_device_t* dev, int* val){
	ALOGI("lijiao hal: lijiao_get_val started");
	if(!val)
	{
        ALOGE("lijiao hal: error val pointer");
        return -errno;
    }
    int fd = open(DEVICE_NAME, O_RDWR);
    if(fd >= 0)
    {
        ALOGI("lijiao hal: open /sys/class/leds/lcd-backlight/brightness successfully.");
        char buffer[20];
        ssize_t amt = read(fd, buffer, 20);
        if(amt == -1)
        {
            ALOGI("lijiao hal: read to device  fail.");
            close(fd);
            free(dev);
            return -errno;
        }
        *val  = atoi(buffer);
        ALOGI("lijiao hal: read  %d to device  success.", *val);
        close(fd);
        return 0;
    }
    ALOGE("lijiao hal: failed to open /sys/class/leds/lcd-backlight/brightness -- %s.", strerror(errno));
    free(dev);
	return -errno;
}

```
&emsp;&emsp;实例化硬件模块时，根据hal层的规范，模块名称规定必须是HAL_MODULE_INFO_SYM，tag的值也必须是HARDWARE_MODULE_TAG。由于DEVICE_NAME是在驱动中创建的，只有root用户可以进行读写，但是如果手机设备是usedebug版本的就行，默认是root权限。

#### 2.1.3 创建Android.mk
&emsp;&emsp;继续在lijiao目录下新建Android.mk文件，并写入以下内容：
```shell
LOCAL_PATH := $(call my-dir)
include $(CLEAR_VARS)
LOCAL_MODULE_TAGS := optional
LOCAL_PRELINK_MODULE := false
LOCAL_MODULE_RELATIVE_PATH := hw
LOCAL_SHARED_LIBRARIES := liblog
LOCAL_SRC_FILES := lijiao.c
LOCAL_MODULE := lijiao.default
include $(BUILD_SHARED_LIBRARY)
```
&emsp;&emsp;LOCAL_MODULE在lijiao后加上.default能够确保我们定义的模块能被hal层加载到。

#### 2.1.4 编译
```shell
lijiao@lijiao-virtual-machine:~/WORKING_DIRECTORY/FLYME/1793$ USE_NINJA=false mmm hardware/libhardware/modules/lijiao
```
&emsp;&emsp;编译成功后，能够看到编译结果以下编译结果：
```shell
make:进入目录'/home/lijiao/WORKING_DIRECTORY/FLYME/1793'
target  C: lijiao.default <= hardware/libhardware/modules/lijiao/lijiao.c
hardware/libhardware/modules/lijiao/lijiao.c:34:76: warning: unused parameter 'name' [-Wunused-parameter]
static int lijiao_device_open(const struct hw_module_t* module,const char* name, struct hw_device_t** device){
                                                                           ^
1 warning generated.
target SharedLib: lijiao.default (out/target/product/mz6799_6m_v2_2k_n/obj/SHARED_LIBRARIES/lijiao.default_intermediates/LINKED/lijiao.default.so)
target Pack Relocations: lijiao.default (out/target/product/mz6799_6m_v2_2k_n/obj/SHARED_LIBRARIES/lijiao.default_intermediates/PACKED/lijiao.default.so)
target Symbolic: lijiao.default (out/target/product/mz6799_6m_v2_2k_n/symbols/system/lib64/hw/lijiao.default.so)
target Strip: lijiao.default (out/target/product/mz6799_6m_v2_2k_n/obj/lib/lijiao.default.so)
Install: out/target/product/mz6799_6m_v2_2k_n/system/lib64/hw/lijiao.default.so
target thumb C: lijiao.default_32 <= hardware/libhardware/modules/lijiao/lijiao.c
hardware/libhardware/modules/lijiao/lijiao.c:34:76: warning: unused parameter 'name' [-Wunused-parameter]
static int lijiao_device_open(const struct hw_module_t* module,const char* name, struct hw_device_t** device){
                                                                           ^
1 warning generated.
target SharedLib: lijiao.default_32 (out/target/product/mz6799_6m_v2_2k_n/obj_arm/SHARED_LIBRARIES/lijiao.default_intermediates/LINKED/lijiao.default.so)
target Pack Relocations: lijiao.default_32 (out/target/product/mz6799_6m_v2_2k_n/obj_arm/SHARED_LIBRARIES/lijiao.default_intermediates/PACKED/lijiao.default.so)
target Symbolic: lijiao.default_32 (out/target/product/mz6799_6m_v2_2k_n/symbols/system/lib/hw/lijiao.default.so)
target Strip: lijiao.default_32 (out/target/product/mz6799_6m_v2_2k_n/obj_arm/lib/lijiao.default.so)
Install: out/target/product/mz6799_6m_v2_2k_n/system/lib/hw/lijiao.default.so
make:离开目录“/home/lijiao/WORKING_DIRECTORY/FLYME/1793”

#### make completed successfully (3 seconds) ####

```
&emsp;&emsp;需要特别注意的是其中两行,显示编译结果的存放位置
```shell
Install: out/target/product/mz6799_6m_v2_2k_n/system/lib64/hw/lijiao.default.so
Install: out/target/product/mz6799_6m_v2_2k_n/system/lib/hw/lijiao.default.so
```
&emsp;&emsp;可通过以下方法将编译结果push到手机设备：
```shell
adb root
adb remount
adb push out/target/product/mz6799_6m_v2_2k_n/system/lib64/hw/lijiao.default.so system/lib64/hw/
adb push out/target/product/mz6799_6m_v2_2k_n/system/lib/hw/lijiao.default.so system/lib/hw/
```
&emsp;&emsp;此时我们编译的模块便能在手机设备上生效。但是此时java层仍无法访问hal层法，我们需要编写JNI方法提供java调用hal层的接口。

### 2.2 JNI方法

#### 2.2.1 com_android_server_LijiaoService.cpp

&emsp;&emsp;在Android源码目录下的framework/base/service/core/jni下创建cpp文件：com_android_server_LijiaoService.cpp，请注意该文件的命名，前缀com_android_server表示的是包名，即在com/android/server下有一个LijiaoService的类，该模块是为LijiaoService类提供jni方法，而LijiaoService是framework层一个提供硬件访问的服务类。
```cpp
#define LOG_TAG "LijiaoService"  
#include "jni.h"  
#include "JNIHelp.h"  
#include "android_runtime/AndroidRuntime.h"  
#include <utils/misc.h>  
#include <utils/Log.h>  
#include <hardware/hardware.h>  
#include <hardware/lijiao.h>  
#include <stdio.h> 

namespace android  
{
	lijiao_device_t * dev = NULL;

	static inline int jni_lijiao_device_open(const hw_module_t* module, struct lijiao_device_t** device)
	{
       		 return module->methods->open(module, LIJIAO_HARDWARE_MODULE_ID,(struct hw_device_t**)device);
   	}

	static jboolean jni_lijiao_init(JNIEnv* env, jclass clazz)
	 {
	    ALOGI("lijiao JNI: jni_lijiao_init start.");
		lijiao_module_t* module; 
		if(hw_get_module(LIJIAO_HARDWARE_MODULE_ID, (const struct hw_module_t**)&module) == 0)
		 {  
            	ALOGI("lijiao JNI: lijiao Stub found.");
	    	if(jni_lijiao_device_open(&(module->m), &dev)== 0)
			{
				ALOGI("lijiao JNI: lijiao device is open.");  
          			return 0;  
			}
			else
			{
				ALOGI("lijiao JNI: fail to open lijiao device .");  
          			return -1; 
			}
		}
		 ALOGE("lijiao JNI: failed to get lijiao stub module.");
        	 return -1; 
	 }

	static jint jni_lijiao_getv(JNIEnv* env, jobject clazz) 
	{
	    ALOGI("lijiao JNI: jni_lijiao_getv start.");
       	int val = 0;
        if(!dev)
		{  
            		ALOGI("lijiao JNI: device is not open.");
            		return val;  
        }
        dev->hal_get(dev, &val);
          
        ALOGI("lijiao JNI: get value %d from device.", val);
      
        return val;
    	}  

	static void jni_lijiao_setv(JNIEnv* env, jobject clazz,jint value) 
	{
	    ALOGI("lijiao JNI: jni_lijiao_setv start.");
		if(!dev) 
		{  
            	ALOGI("lijiao JNI: device is not open.");
            	return;
        }
		 dev->hal_set(dev,value);
	}

	static const JNINativeMethod method_table[] = {  
        {"init_native", "()Z", (void*)jni_lijiao_init},  
        {"setVal_native", "(I)V", (void*)jni_lijiao_setv},  
        {"getVal_native", "()I", (void*)jni_lijiao_getv},  
    	};  

	int register_android_server_LijiaoService(JNIEnv *env) {  
            return jniRegisterNativeMethods(env, "com/android/server/LijiaoService", method_table, NELEM(method_table));  
    }  
};

```
&emsp;&emsp;该模块中主要是：
1. 定义了一个jni_lijiao_init（）方法，该方法中通过hw_get_module（）系统方法来访问hal层，返回hal层的硬件模块结构体实例（该结构体实例中的method成员对应hal层定义的lijiao_device_open（）方法，该方法通过传递一个硬件模块结构体实例来返回一个硬件模块接口，该接口中的成员包括了在hal层定义的访问驱动文件的接口方法）。通过调用模块结构体实例的open方法（open方法指向了lijiao_device_open（）方法））
2. 定义了jni接口方法，在jni接口方法中调用的是jni_lijiao_init（）得到的硬件模块接口的接口方法。
3. 将三个jni方法封装在方法表method_table[]中。
4. 将该方法表通过register_android_server_LijiaoService（）方法注册到系统的jni方法表中

#### 2.2.2 onload.cpp
&emsp;&emsp;要想jni文件中register_android_server_LijiaoService（）生效，我们需要修改com_android_server_LijiaoService.cpp同目录下的onload.cpp文件,首先在namespace android中增加register_android_server_LijiaoService声明：
```cpp
  namespace android {
      ...................................................................

      int register_android_server_LijiaoService(JNIEnv *env);// 添加this

      };

```
&emsp;&emsp;然后在JNI_onLoad方法中增加register_android_server_LijiaoService(env)调用
```cpp
  extern "C" jint JNI_onLoad(JavaVM* vm, void* reserved)

      {
       .................................................................................................

       register_android_server_LijiaoService(env);

       .................................................................................................

      }
```
&emsp;&emsp;这样，系统初始化的时候就能自动加载我们定义的jni方法表了。

#### 2.2.3 修改Android.mk
&emsp;&emsp;修改同目录下的Android.mk文件，在LOCAL_SRC_FILES变量中增加一行：
```shell
 LOCAL_SRC_FILES:= \

      com_android_server_AlarmManagerService.cpp \

      com_android_server_BatteryService.cpp \

      com_android_server_InputManager.cpp \

      com_android_server_LightsService.cpp \

      com_android_server_PowerManagerService.cpp \

      com_android_server_SystemServer.cpp \

      com_android_server_UsbService.cpp \

      com_android_server_VibratorService.cpp \

      com_android_server_location_GpsLocationProvider.cpp \

      com_android_server_LijiaoService.cpp \          //添加this

      onload.cpp
```

#### 2.2.4 编译
```shell
lijiao@lijiao-virtual-machine:~/WORKING_DIRECTORY/FLYME/1793$ USE_NINJA=false mmm frameworks/base/services/core/jni
```
&emsp;&emsp;push 方法同2.1.4。

### 2.3 创建服务类

#### 2.3.1 创建aidl脚本

&emsp;&emsp;硬件服务一般是运行在独立的进程中为各种应用进程提供服务，进程间需要使用binder来进行通信。在Android源码目录下的framewors/base/core/java/android/os创建ILijiaoService.aidl文件，并添加以下内容：
```java
package android.os;  

interface ILijiaoService{
    void setVal(int val);  
    int getVal();  
}

```
#### 2.3.2 修改Android.mk
&emsp;&emsp;修改frameworks/base下的Android.mk文件，修改LOCAL_SRC_FILES变量的值
```shell
  LOCAL_SRC_FILES += /

   ....................................................................

   core/java/android/os/IVibratorService.aidl /

   core/java/android/os/ILijiaoService.aidl /            //添加this

   core/java/android/service/urlrenderer/IUrlRendererService.aidl /

   .....................................................................


```
#### 2.3.3 编译
```shell
lijiao@lijiao-virtual-machine:~/WORKING_DIRECTORY/FLYME/1793$ USE_NINJA=false mmm frameworks/base
```
&emsp;&emsp;Android会基于aidl脚本生成相应的类。

#### 2.3.4 创建service
&emsp;&emsp;在framewors/base/services/java/com/android/server下添加类文件：LijiaoService.java，并添加以下内容：
```java
package com.android.server;  
import android.content.Context;  
import android.os.ILijiaoService;  
import android.util.Slog;

public class LijiaoService extends ILijiaoService.Stub{

	 static native boolean init_native();
	 static native void setVal_native(int val);
	 static native int getVal_native();

	public LijiaoService(){
		init_native();
	}

	public void setVal(int val){
		setVal_native(val);
	}
	public int getVal(){
		return getVal_native();
	} 

} 
```
&emsp;&emsp;修改同目录下的SystemServer.java文件，在ServerThread::run函数中添加以下代码：
```java
  @Override

     public void run() {

     ....................................................................................

            try {

                  Slog.i(TAG, "DiskStats Service");

                  ServiceManager.addService("diskstats", new DiskStatsService(context));

            } catch (Throwable e) {

                  Slog.e(TAG, "Failure starting DiskStats Service", e);

            }

            try {

                  Slog.i(TAG, "Lijiao Service");

                  ServiceManager.addService("lijiao", new LijiaoService());

           	 } catch (Throwable e) {

                  Slog.e(TAG, "Failure starting Lijiao Service", e);

           }//添加this

     ......................................................................................

     }      

```
#### 2.3.5 编译
```shell
lijiao@lijiao-virtual-machine:~/WORKING_DIRECTORY/FLYME/1793$ USE_NINJA=false mmm frameworks/base/services
```
push方法同2.1.4。

### 2.4 apk调用
&emsp;&emsp;我们可以先在android studio中创建apk，在把apk工程拷贝到Android源码目录下的packages/apps下，
apk项目拷贝时只需要留下res、src目录和AndroidManifest.xml文件。
#### 2.4.1 布局文件
```xml
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:orientation="vertical"
    android:layout_width="fill_parent"
    android:layout_height="fill_parent">
    <LinearLayout
        android:layout_width="fill_parent"
        android:layout_height="wrap_content"
        android:orientation="vertical"
        android:gravity="center">
        <TextView
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:text="@string/value">
        </TextView>
        <EditText
            android:layout_width="fill_parent"
            android:layout_height="wrap_content"
            android:id="@+id/edit_value"
            android:hint="@string/hint">
        </EditText>
    </LinearLayout>
    <LinearLayout
        android:layout_width="fill_parent"
        android:layout_height="wrap_content"
        android:orientation="horizontal"
        android:gravity="center">
        <Button
            android:id="@+id/button_read"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:text="@string/read">
        </Button>
        <Button
            android:id="@+id/button_write"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:text="@string/write">
        </Button>
        <Button
            android:id="@+id/button_clear"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:text="@string/clear">
        </Button>
    </LinearLayout>
</LinearLayout>
```
#### 2.4.2 字符串文件
&emsp;&emsp;values/strings.xml
```xml
<?xml version="1.0" encoding="utf-8"?>
<resources>
	<string name="app_name">LiJiaoFW</string>
	<string name="value">Value</string>
	<string name="hint">Please input a value...</string>
	<string name="read">Read</string>
	<string name="write">Write</string>
	<string name="clear">Clear</string>
</resources>
```

#### 2.4.3 主程序

```java
package lijiao.android.com.lijiaofw;

import android.os.RemoteException;
import android.os.Bundle;
import android.util.Log;
import android.view.View;
import android.widget.Button;
import android.widget.EditText;

import android.app.Activity;
import android.os.ServiceManager;
import android.os.Bundle;
import android.view.View.OnClickListener;

import android.os.ILijiaoService;

public class MainActivity extends Activity implements View.OnClickListener {

    private EditText valueText = null;
    private Button readButton = null;
    private Button writeButton = null;
    private Button clearButton = null;
    ILijiaoService lijiaoService = null;
    String LOG_TAG = "lijiaoMain";

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        valueText = (EditText) findViewById(R.id.edit_value);
        readButton = (Button) findViewById(R.id.button_read);
        writeButton = (Button) findViewById(R.id.button_write);
        clearButton = (Button) findViewById(R.id.button_clear);

        readButton.setOnClickListener(this);
        writeButton.setOnClickListener(this);
        clearButton.setOnClickListener(this);

        lijiaoService = ILijiaoService.Stub.asInterface(ServiceManager.getService("lijiao"));
    }

    public void onClick(View v) {
        switch (v.getId())
        {
            case R.id.button_read:
            {
                try {
                    int val = lijiaoService.getVal();
                    String text = String.valueOf(val);
                    valueText.setText(text);
                } catch (RemoteException e) {
                    Log.e(LOG_TAG, "Remote Exception while reading value from device.");
                }
                break;
            }
            case R.id.button_write:
            {
                try{
                    String text = valueText.getText().toString();
                    int val = Integer.parseInt(text);
                    lijiaoService.setVal(val);
                } catch (RemoteException e) {
                    Log.e(LOG_TAG, "Remote Exception while writing value to device.");
                }
                 break;
            }
            case R.id.button_clear:
            {
                valueText.setText("");
                break;
            }
        }
    }
}
```

## 第三章 总结

### 3.1 思路总结

&emsp;&emsp;（1）hal层。头文件(hardware/libhardware/include)中主要定义了硬件模块结构体、硬件模块接口、硬件模块id。硬件模块结构体中包含一个结构体成员hw_module_t m，在硬件接口中包含一个hw_device_t d的结构体成员以及函数指针。
<br>&emsp;&emsp;在相应的c文件(hardware/libhardware/module/lijiao)中，实例化了硬件模块结构体，根据hal层的规范，该硬件结构体实例的名称必须为HAL_MODULE_INFO_SYM，成员tag的值必须为HARDWARE_MODULE_TAG。成员id被赋予了头文件定义的硬件模块id。Android框架的上层通过hw_get_module（）方法根据硬件模块id来返回一个hal层存在的硬件模块结构体，为了使上层能够使用hal层的更多方法，硬件模块结构体中包含一个结构体成员method，method中有一个函数指针open，我们使该open指向我们编写的lijiao_device_open（）方法，传递进一个硬件模块实例，并实例化、返回硬件模块接口，这个硬件模块接口在实例化时其函数指针成员指向了我们在hal层定义的硬件访问方法。

<br>&emsp;&emsp;（2）jni层。jni层的方法(frameworks/base/services/core/jni)主要是定义一系列有jni标志的jni方法，并将这些方法放到JNINativeMethod类型方法数组中，并通过register_android_server_LijiaoService（）方法注册到jni方法表中。jni文件的命名是有讲究的，其中com_android_server_LijiaoService.cpp表示在com/android/server包下有一个LijiaoService服务类，该cpp文件是为LijiaoService服务类提供jni方法的。

<br>&emsp;&emsp;（3）服务类。硬件访问服务一般是运行在单独的进程中，为其他应用进程提供硬件访问服务，因此硬件访问服务和应用进程的通信需要通过binder实现。
首先需要在framworks/core/java/android/os/下创建binder接口脚本ILijiaoService.aidl,通过编译能产生对应的java代码。其次在frameworks/base/services/java/com/android/server下添加类文件：LijiaoService.java，该类继承ILijiaoService.Stub，声明了jni方法，并在实现的binder接口方法中调用jni方法。

<br>&emsp;&emsp;（4）apk调用。apk层通过ServiceManager.getService("")方法来获得远程服务的binder引用，并通过ILijiaoService.Stub.asInterface（）将binder服务的
引用转为binder代理，通过该代理可以使用SystemServer中的服务中的方法（方法中调用了jni方法，jni方法实际上调用了hal层的硬件访问接口函数）。

### 3.2 问题与分析
#### 3.2.1 -errno
<br>&emsp;&emsp;errno 是记录系统的最后一次错误代码。代码是一个int型的值，在errno.h中定义。查看错误代码errno是调试程序的一个重要方法。当linux C api函数发生异常时,一般会将errno变量(需include errno.h)赋一个整数值,不同的值表示不同的含义,可以通过查看该值推测出错的原因。在实际编程中用这一招解决了不少原本看来莫名其妙的问题。

#### 3.2.2 mk文件
<br>&emsp;&emsp;为什么创建的源码文件，有的需要在当前目录下创建Android.mk文件，有的需要在上级目录中修改Android.mk文件，而有的均不需要？
<br>&emsp;&emsp;详细描述：
> &emsp;&emsp;hardware/libhardware/module/lijiao下的 hal层代码是在当前目录下创建mk文件，
<br>&emsp;&emsp;frameworks/base/services/core/jni下的cpp源码文件是修改当前目录下的mk，
<br>&emsp;&emsp;frameworks/base/core/java/android/os/下的aidl源文件是在frameworks/base下修改mk,
<br>&emsp;&emsp;frameworks/base/services/java/com/android/server/下的service java源文件并不需要添加或者修改mk文件，
<br>&emsp;&emsp;packages/apps/下的java源代码是在apk根目录下创建mk文件。

&emsp;&emsp;why？

<br>&emsp;&emsp;我们先一个个来看：
<br>&emsp;&emsp;（1）lijiao.c
>&emsp;&emsp; 首先hardware下并没有mk文件，因此mk需要hardware下的模块自己编写，其次，hardware/libhardware下的mk文件中的LOCAL_SRC_FILES += hardware.c，即该规则指定只编译当前目录下的hardware.c文件，然而libhardware下仍有不少模块，因此libhardware下面的模块mk也需要自己编写，我们再来看hardware/libhardware/include，include下全是头文件，头文件被其他源文件include后会自动编译，因此include下没有也不需要mk文件，接下来看hardware/libhardware/module，module下的mk文件中没有LOCAL_SRC_FILES这一行，意味着该mk并没有指明要编译哪些文件，因此module下的各个模块需要自己定义mk，module下的模块已经是最后一级目录，因此当我们在module下创建lijiao模块时，需要创建mk文件，并指明lijiao下需要编译哪些模块。

<br>&emsp;&emsp;（2）jni方法
>&emsp;&emsp; 首先我们来看frameworks下是没有mk的，再看frameworks/base，base下的mk中的LOCAL_SRC_FILES是包含清一色aidl文件，显然jni方法需要在下层的模块中定义编译规则，我们再看frameworks/base/services，services下的mk文件中LOCAL_SRC_FILES := $(call all-java-files-under,java)，该语句的含义是编译frameworks/base/services/java目录下的所有java源文件，并没有涉及同目录下的frameworks/base/services/core下的jni方法，因此我们继续查看frameworks/base/services/core，其下的mk文件
```shell
     LOCAL_SRC_FILES += \
        $(call all-java-files-under,java) \
        java/com/android/server/EventLogTags.logtags \
        java/com/android/server/am/EventLogTags.logtags \
```
>只指明了编译frameworks/base/services/core/java下的所有java源文件，以及两个logtags文件，还是没有提及frameworks/base/services/core/jni下的任何文件，因此该jni下的模块需要自己定义编译规则，当我们进入frameworks/base/services/core/jni目录下，会发现已经存在一个mk文件，我们只需要编辑好cpp源文件，再在mk中添加文件名即可。

<br>&emsp;&emsp;（3）aidl
>&emsp;&emsp; 我们先看frameworks下是没有mk的，再看frameworks/base,base下的mk 的LOCAL_SRC_FILES包含了（frameworks/base）core/java/android/os/下的很多aidl文件，而我们的aidl源文件是放在frameworks/base/core/java/android/os/下的，因此，我们的aidl文件编辑好之后，是需要在frameworks/base下mk文件中添加文件名。

<br>&emsp;&emsp;（4）service的java源文件
>&emsp;&emsp; 我们查看frameworks/base下的mk文件可以发现，其中LOCAL_SRC_FILES包含的是清一色的aidl文件，而我们的service是一个java文件，不宜在此mk中添加文件名，再看frameworks/base/servies下的mk，mk文件中的LOCAL_SRC_FILES := $(call all-java-files-under,java)，意味着该规则是要编译frameworks/base/servies/java下的所有java源文件，而我们的service源文件正处于frameworks/base/servies/java下，因此我们编辑好service的java源文件之后并不需要添加或者修改mk文件了。

<br>&emsp;&emsp;（5）apk源文件
> &emsp;&emsp;首先来看packages，此下并没有mk文件，再看packages/apps，此下也没有mk文件，再往下就是各个apk的工程目录了，因此我们需要将apk工程LijiaoFW放到packages/apps下后，在packages/apps/LijiaoFW创建mk文件，并编写编译规则。

#### 3.2.3 硬件访问服务的放置位置
&emsp;&emsp;为什么服务类放在了SystemServer.java同目录（frameworks/base/servies/java）下？
>&emsp;&emsp; 观察了frameworks/base/servies/core下的jni和java目录，发现frameworks/base/servies/core/jni下的cpp所对应的service是放在frameworks/base/servies/core/java目录下的，例如：SearchEngineManagerService。目前只是一个demon将我们的service放在了frameworks/base/servies/java下，遵循规范应该是要把service放在frameworks/base/servies/core/java下的。

#### 3.2.4 jni放置位置
&emsp;&emsp;为什么jni方法不放在frameworks/base/core/jni下，而是放在frameworks/base/serviees/core/jni下？
> &emsp;&emsp;因为这个jni是为我们的service写的（被我们的LijiaoService调用），那么该jni就应该和我们的servies同在一个模块下，LijiaoService所属模块为frameworks/base/services,因此jni方法也应该在frameworks/base/services下，而且jni方法是写在core/jni下的，因此我们的jni方法必须写在frameworks/base/services/core/jni下。




[1]:http://www.cnblogs.com/cy568searchx/p/4135102.html
[2]:http://blog.csdn.net/yangzhiloveyou/article/details/8627969


