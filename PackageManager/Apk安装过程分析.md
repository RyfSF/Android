# APK安装过程分析

## APK的安装方式以及存储路径
+ 安装方式

1.系统应用安装————开机是完成，没有安装界面
2.网络下载应用安装————通过market应用完成，没有安装界面
3.ADB工具安装————没有安装界面
4.第三方应用安装————通过SdCard里面的APK文件进行安装，有安装界面

+ 存储路径

1./system/app 系统应用，无法删除
2./data/app 用户程序安装目录，可以删除。安装的时候吧apk文件复制到此目录
3./data/data 存放用户程序的数据
4./data/dalvik-cache 将APK中的dex文件安装到该目录下，在三星手机中这里面存放了/system/app，system/framework,/system/priv-app 的dex文件.
> dex 文件和 odex文件的区别?
## APK安装过程解析
APK安装的接口为 PackageManager中的抽象接口 
``` java
pm.installPackage(mPackageURI,observer,installFlags);
```
>[抽象类与接口的区别](http://www.importnew.com/18780.html)
``` java
@Deprecated
public abstract void installPackage(
    Uri packageURI,
    IPackageInstallObserver observer,
    @InstallFlags int flags,
    String installerPackageName);
```
发现这个地方只是一个抽象的方法，那么真正的方法是在哪里实现的？
其实是在ContextImpl.java文件中。
``` java
@Override
public PackageManager getPackageManager() {
    if (mPackageManager != null) {
        return mPackageManager;
    }

    IPackageManager pm = ActivityThread.getPackageManager();
    if (pm != null) {
        // Doesn't matter if we make more than one instance.
        return (mPackageManager = new ApplicationPackageManager(this, pm));
    }

    return null;
}
```
ApplicationPackageManager继承了PackageManager，这个类实现了所有的抽象方法，如下面所需要的
``` java
public void installPackage(Uri packageURI, IPackageInstallObserver observer, int flags,String installerPackageName) {
    try {
        mPM.installPackage(packageURI, observer, flags, installerPackageName);
    } catch (RemoteException e) {
        // Should never happen!
    }
}
```
这样就完成了对PackageManagerService.java中installPackage 的调用。
``` java
public void installPackage(final Uri packageURI, final IPackageInstallObserver observer, final int flags,final String installerPackageName) {
        mContext.enforceCallingOrSelfPermission(android.Manifest.permission.INSTALL_PACKAGES, null);
        Message msg = mHandler.obtainMessage(INIT_COPY);
        msg.obj = new InstallParams(packageURI, observer, flags,installerPackageName);
        mHandler.sendMessage(msg);
}
```
通过sendMessage将消息发送给PackageHandler中的HandleMessage（Message msg）.
``` java
通过sendMessage将消息发送给PaackageHandler中的handleMessage(Message msg)接收.
``` java
void doHandleMessage(Message msg) {
    switch (msg.what) {
        case INIT_COPY: {
            HandlerParams params = (HandlerParams) msg.obj;
            int idx = mPendingInstalls.size();
            if (DEBUG_INSTALL) Slog.i(TAG, "init_copy idx=" + idx + ": " + params);
            // If a bind was already initiated we dont really
            // need to do anything. The pending install
            // will be processed later on.
            if (!mBound) {
                // If this is the only one pending we might
                // have to bind to the service again.
                if (!connectToService()) {
                    Slog.e(TAG, "Failed to bind to media container service");
                    params.serviceError();
                    return;
                } else {
                    // Once we bind to the service, the first
                    // pending request will be processed.
                    mPendingInstalls.add(idx, params);
                }
            } else {
                mPendingInstalls.add(idx, params);
                // Already bound to the service. Just make
                // sure we trigger off processing the first request.
                if (idx == 0) {
                    mHandler.sendEmptyMessage(MCS_BOUND);
                }
            }
            break;
        }
        ...
    }
    ...
}
```
这里发送消息以后就进入到了消息处理INIT_COPY的逻辑,首先检查是否已经和IMediaContainerService完成服务绑定，如果没有则与该服务尝试连接，负责将接收到的安装包添加到mPendingInstalls变量里边，但是mPendingInstalls这里边的请求在什么时候被处理？
那么来看看onServiceConnected()的实现.
``` java
public void onServiceConnected(ComponentName name, IBinder service) {
    if (DEBUG_SD_INSTALL) Log.i(TAG, "onServiceConnected");
    IMediaContainerService imcs =
        IMediaContainerService.Stub.asInterface(service);
    mHandler.sendMessage(mHandler.obtainMessage(MCS_BOUND, imcs));
}

```
其实在第一次绑定该服务成功后就发送了MCS_BOUND消息，消息调用到HandlerParams的startCopy方法准备安装
``` java
    else if (mPendingInstalls.size() > 0) {
        HandlerParams params = mPendingInstalls.get(0);
        if (params != null) {
            params.startCopy();
    }
```
接下来在该方法中调用mHandler.sendEmptyMessage(MCS_UNBIND),PackageHandler接收到MSC_UNBIND消息以后会进入一下逻辑块。
``` java
if (mPendingInstalls.size() == 0) {
    if (mBound) {
        disconnectService();
    }
    } else {
        // There are more pending requests in queue.
        // Just post MCS_BOUND message to trigger processing
        // of next pending install.
        mHandler.sendEmptyMessage(MCS_BOUND);
    }
}
```
如果PendingInstalls还有未处理的请求继续重复上述步骤，否则断开与IMediaContainerService建立的服务。

抽象类HandlerParams中实现了方法，startCopy(),而handlerStartCopy()和HandleReturnCode两个方法的实现则是在HandklerParams的自雷InstallParams里面。
``` java
final void startCopy() {
    try {
        if (DEBUG_SD_INSTALL) Log.i(TAG, "startCopy");
            retry++;
            if (retry > MAX_RETRIES) {
                Slog.w(TAG, "Failed to invoke remote methods on default container service. Giving up");
                mHandler.sendEmptyMessage(MCS_GIVE_UP);
                handleServiceError();
                return;
            } else {
                handleStartCopy();
                if (DEBUG_SD_INSTALL) Log.i(TAG, "Posting install MCS_UNBIND");
                mHandler.sendEmptyMessage(MCS_UNBIND);
            }
    } catch (RemoteException e) {
        if (DEBUG_SD_INSTALL) Log.i(TAG, "Posting install MCS_RECONNECT");
         mHandler.sendEmptyMessage(MCS_RECONNECT);
    }
    handleReturnCode();
}
```
在InstallParams里边handleStartCopy()主要实现的功能是获取安装位置信息以及复制apk到指定位置。
``` java
if (ret == PackageManager.INSTALL_SUCCEEDED) {
    // Create copy only if we are not in an erroneous state.
    // Remote call to initiate copy using temporary file
    ret = mArgs.copyApk(mContainerService, true);
}
```
抽行类InstallArgs中的copyApk负责复制apk文件，具体是现在子类FileInstallArgs和SdInstallArgs里面。
其中FileInstallArgs中实现该方法的关键部分。
``` java

```
