title: Android源码笔记——Camera系统架构
tags:
- Android Camera
categories: Android源码笔记

---

Camera的架构与Android系统的整体架构保持一致，如下图所示，本文主要从以下四个方面对其进行说明。

 1. Framework：Camera.java
 2. Android Runtime：android_hardware_Camera.cpp
 3. Library：Camera Client和Camera Service
 4. HAL：CameraHardwareInterface

![](/img/20160401-1.png)
 
----------
## 一、Framework：Camera.java

Camera是应用层软件直接使用的类，涵盖了启动、预览、拍摄及关闭等操作摄像头的全部接口。Camera.java在Android源码中的路径为：framework/base/core/java/android/hardware。为了说明整个Camera系统的架构，这里暂不横向分析Camera.java的功能，下面从open()方法着手：

```
public static Camera open() {
    int numberOfCameras = getNumberOfCameras();
    CameraInfo cameraInfo = new CameraInfo();
    for (int i = 0; i < numberOfCameras; i++) {
        getCameraInfo(i, cameraInfo);
        if (cameraInfo.facing == CameraInfo.CAMERA_FACING_BACK) {
            return new Camera(i);
        }
    }
    return null;
}
```
open()方法需要注意以下几点：

 - getNumberOfCameras为native方法，实现在android_hardware_Camera.cpp中；
 - CameraInfo是Camera定义的静态内部类，包含facing、orientation、canDisableShutterSound；
 - getCameraInfo内部调用native方法_getCameraInfo获取摄像头信息；
 - open()默认启动的是后置摄像头（CAMERA_FACING_BACK）。

```
/** used by Camera#open, Camera#open(int) */
Camera(int cameraId) {
    int err = cameraInitNormal(cameraId);
    if (checkInitErrors(err)) {
        switch(err) {
            case EACCESS:
                throw new RuntimeException("Fail to connect to camera service");
            case ENODEV:
                throw new RuntimeException("Camera initialization failed");
            default:
                // Should never hit this.
                throw new RuntimeException("Unknown camera error");
        }
    }
}
```
Camera构造器的核心实现在cameraInitNormal中，cameraInitNormal调用cameraInitVersion，并传入参数cameraId和CAMERA_HAL_API_VERSION_NORMAL_CONNECT，后者代表HAL的版本。
```
private int cameraInitVersion(int cameraId, int halVersion) {
    ……
    String packageName = ActivityThread.currentPackageName();
    return native_setup(new WeakReference<Camera>(this), cameraId, halVersion, packageName);
}
```
cameraInitNormal调用本地方法native_setup()，由此进入到android_hardware_Camera.cpp中，native_setup()的签名如下：
```
private native final int native_setup(Object camera_this, 
                                int cameraId, int halVersion, String packageName);
```
## 二、Android Runtime：android_hardware_Camera.cpp
native_setup()被动态注册到JNI，通过JNI调用android_hardware_Camera_native_setup()方法。
```
static JNINativeMethod camMethods[] = {
    ……
    { "native_setup",    "(Ljava/lang/Object;ILjava/lang/String;)V",
    (void*)android_hardware_Camera_native_setup }
    ……
};
```
JNI的重点是android_hardware_Camera_native_setup()方法的实现：
```
// connect to camera service
static jint android_hardware_Camera_native_setup(JNIEnv *env, jobject thiz,
    jobject weak_this, jint cameraId, jint halVersion, jstring clientPackageName)
{
    // Convert jstring to String16
    const char16_t *rawClientName = env->GetStringChars(clientPackageName, NULL);
    jsize rawClientNameLen = env->GetStringLength(clientPackageName);
    String16 clientName(rawClientName, rawClientNameLen);
    env->ReleaseStringChars(clientPackageName, rawClientName);

    sp<Camera> camera;
    if (halVersion == CAMERA_HAL_API_VERSION_NORMAL_CONNECT) {
        // Default path: hal version is don't care, do normal camera connect.
        camera = Camera::connect(cameraId, clientName,
                Camera::USE_CALLING_UID);
    } else {
        jint status = Camera::connectLegacy(cameraId, halVersion, clientName,
                Camera::USE_CALLING_UID, camera);
        if (status != NO_ERROR) {
            return status;
        }
    }

    if (camera == NULL) {
        return -EACCES;
    }

    // make sure camera hardware is alive
    if (camera->getStatus() != NO_ERROR) {
        return NO_INIT;
    }

    jclass clazz = env->GetObjectClass(thiz);
    if (clazz == NULL) {
        // This should never happen
        jniThrowRuntimeException(env, "Can't find android/hardware/Camera");
        return INVALID_OPERATION;
    }

    // We use a weak reference so the Camera object can be garbage collected.
    // The reference is only used as a proxy for callbacks.
    sp<JNICameraContext> context = new JNICameraContext(env, weak_this, clazz, camera);
    context->incStrong((void*)android_hardware_Camera_native_setup);
    camera->setListener(context);

    // save context in opaque field
    env->SetLongField(thiz, fields.context, (jlong)context.get());
    return NO_ERROR;
}
```
android_hardware_Camera_native_setup()方法通过调用Camera::connect()方法请求连接CameraService服务。入参中：

 - clientName是通过将clientPackageName从jstring转换为String16格式得到；
 - Camera::USE_CALLING_UID是定义在Camera.h中的枚举类型，其值为ICameraService::USE_CALLING_UID（同样为枚举类型，值为-1）。


Camera::connect()位于Camera.cpp中，由此进入到Library层。

## 三、Library：Camera Client和Camera Service
如上述架构图中所示，ICameraService.h、ICameraClient.h和ICamera.h三个类定义了Camera的接口和架构，ICameraService.cpp和Camera.cpp两个文件用于Camera架构的实现，Camera的具体功能在下层调用硬件相关的接口来实现。Camera.h是Camera系统对上层的接口。

具体的，Camera类继承模板类CameraBase，Camera::connect()调用了CameraBase.cpp中的connect()方法。

```
sp<Camera> Camera::connect(int cameraId, const String16& clientPackageName,
        int clientUid) {
    return CameraBaseT::connect(cameraId, clientPackageName, clientUid);
}
```
CameraBase实际上又继承了IBinder的DeathRecipient内部类，DeathRecipient虚拟继承自RefBase。RefBase是Android中的引用计数基础类，其中定义了incStrong、decStrong、incWeak和decWeak等涉及sp/wp的指针操作函数，当然这扯远了。
```
template <typename TCam>
struct CameraTraits {
};

template <typename TCam, typename TCamTraits = CameraTraits<TCam> >
class CameraBase : public IBinder::DeathRecipient
{
public:
    
    static sp<TCam> connect(int cameraId,
                            const String16& clientPackageName,
                            int clientUid);
    ……
}
```
```
class DeathRecipient : public virtual RefBase
{
public:
    virtual void binderDied(const wp<IBinder>& who) = 0;
};
```
回到Camera::connect()的实现上，其中，new TCam(cameraId)生成BnCameraClient对象，BnCameraClient定义在ICameraClient.h文件中，继承自模板类BnInterface。getCameraService()方法返回CameraService的服务代理BpCameraService，BpCameraService同样继承自模板类BnInterface。然后通过Binder通信发送CONNECT命令，当BnCameraService收到CONNECT命令后调用CameraService的connect()成员函数来做相应的处理。

```
template <typename TCam, typename TCamTraits>
sp<TCam> CameraBase<TCam, TCamTraits>::connect(int cameraId,
                                               const String16& clientPackageName,
                                               int clientUid)
{
    ALOGV("%s: connect", __FUNCTION__);
    sp<TCam> c = new TCam(cameraId); // BnCameraClient 
    sp<TCamCallbacks> cl = c;
    status_t status = NO_ERROR;
    const sp<ICameraService>& cs = getCameraService(); // BpCameraService

    if (cs != 0) {
        TCamConnectService fnConnectService = TCamTraits::fnConnectService;
        status = (cs.get()->*fnConnectService)(cl, cameraId, clientPackageName, clientUid,
                                             /*out*/ c->mCamera);
    }
    if (status == OK && c->mCamera != 0) {
        c->mCamera->asBinder()->linkToDeath(c);
        c->mStatus = NO_ERROR;
    } else {
        ALOGW("An error occurred while connecting to camera: %d", cameraId);
        c.clear();
    }
    return c;
}
```
```
class BnCameraClient: public BnInterface<ICameraClient>
{
public:
    virtual status_t    onTransact( uint32_t code,
                                    const Parcel& data,
                                    Parcel* reply,
                                    uint32_t flags = 0);
};
```
```
class BpCameraService: public BpInterface<ICameraService>
{
public:
    BpCameraService(const sp<IBinder>& impl)
        : BpInterface<ICameraService>(impl)
    {
    }
    ……
}
```
注：connect()函数在BpCameraService和BnCameraService的父类ICameraService中声明为纯虚函数，在BpCameraService和CameraService中分别给出了实现，BpCameraService作为代理类，提供接口给客户端，真正实现在BnCameraService的子类CameraService中。

在BpCameraService中，connect()函数实现如下：
```
// connect to camera service (android.hardware.Camera)
virtual status_t connect(const sp<ICameraClient>& cameraClient, int cameraId,
                         const String16 &clientPackageName, int clientUid,
                         /*out*/
                         sp<ICamera>& device)
{
    Parcel data, reply;
    data.writeInterfaceToken(ICameraService::getInterfaceDescriptor());
    data.writeStrongBinder(cameraClient->asBinder());
    data.writeInt32(cameraId);
    data.writeString16(clientPackageName);
    data.writeInt32(clientUid);
    remote()->transact(BnCameraService::CONNECT, data, &reply); // BpBinder的transact()函数向IPCThreadState实例发送消息，通知其有消息要发送给binder driver        
    if (readExceptionCode(reply)) return -EPROTO;
    status_t status = reply.readInt32();
    if (reply.readInt32() != 0) {
        device = interface_cast<ICamera>(reply.readStrongBinder()); // client端读出server返回的bind
    }
    return status;
}
```
首先将传递过来的Camera对象cameraClient转换成IBinder类型，将调用的参数写到Parcel（可理解为Binder通信的管道）中，通过BpBinder的transact()函数发送消息，然后由BnCameraService去响应该连接，最后就是等待服务端返回，如果成功则生成一个BpCamera实例。

真正的服务端响应实现在BnCameraService的onTransact()函数中，其负责解包收到的Parcel并执行client端的请求的方法。
```
status_t BnCameraService::onTransact(
    uint32_t code, const Parcel& data, Parcel* reply, uint32_t flags)
{
    switch(code) {
        ……
     case CONNECT: {
            CHECK_INTERFACE(ICameraService, data, reply);
            sp<ICameraClient> cameraClient =
                    interface_cast<ICameraClient>(data.readStrongBinder()); // 使用Camera的Binder对象生成Camera客户代理BpCameraClient实例
              int32_t cameraId = data.readInt32();
            const String16 clientName = data.readString16();
            int32_t clientUid = data.readInt32();
            sp<ICamera> camera;
            status_t status = connect(cameraClient, cameraId,
                    clientName, clientUid, /*out*/camera); // 将生成的BpCameraClient对象作为参数传递到CameraService的connect()函数中
              reply->writeNoException();
            reply->writeInt32(status); // 将BpCamera对象以IBinder的形式打包到Parcel中返回
              if (camera != NULL) {
                reply->writeInt32(1);
                reply->writeStrongBinder(camera->asBinder());
            } else {
                reply->writeInt32(0);
            }
            return NO_ERROR;
        } break;
    ……
    }
}
```
主要的处理包括：

 1. 通过data中Camera的Binder对象生成Camera客户代理BpCameraClient实例；
 2. 将生成的BpCameraClient对象作为参数传递到CameraService（/frameworks/av/services/camera /libcameraservice/CameraService.cpp）的connect()函数中，该函数会返回一个BpCamera实例；
 3. 将在上述实例对象以IBinder的形式打包到Parcel中返回。

最后，BpCamera实例是通过CameraService::connect()函数返回的。CameraService::connect()实现的核心是调用connectHelperLocked()函数根据HAL不同API的版本创建不同的client实例（早期版本中好像没有connectHelperLocked()这个函数，但功能基本相似）。
```
status_t CameraService::connectHelperLocked(
        /*out*/
        sp<Client>& client,
        /*in*/
        const sp<ICameraClient>& cameraClient,
        int cameraId,
        const String16& clientPackageName,
        int clientUid,
        int callingPid,
        int halVersion,
        bool legacyMode) {

    int facing = -1;
    int deviceVersion = getDeviceVersion(cameraId, &facing);

    if (halVersion < 0 || halVersion == deviceVersion) {
        // Default path: HAL version is unspecified by caller, create CameraClient
        // based on device version reported by the HAL.
        switch(deviceVersion) {
          case CAMERA_DEVICE_API_VERSION_1_0:
            client = new CameraClient(this, cameraClient,
                    clientPackageName, cameraId,
                    facing, callingPid, clientUid, getpid(), legacyMode);
            break;
          case CAMERA_DEVICE_API_VERSION_2_0:
          case CAMERA_DEVICE_API_VERSION_2_1:
          case CAMERA_DEVICE_API_VERSION_3_0:
          case CAMERA_DEVICE_API_VERSION_3_1:
          case CAMERA_DEVICE_API_VERSION_3_2:
            client = new Camera2Client(this, cameraClient,
                    clientPackageName, cameraId,
                    facing, callingPid, clientUid, getpid(), legacyMode);
            break;
          case -1:
            ALOGE("Invalid camera id %d", cameraId);
            return BAD_VALUE;
          default:
            ALOGE("Unknown camera device HAL version: %d", deviceVersion);
            return INVALID_OPERATION;
        }
    } else {
        // A particular HAL version is requested by caller. Create CameraClient
        // based on the requested HAL version.
        if (deviceVersion > CAMERA_DEVICE_API_VERSION_1_0 &&
            halVersion == CAMERA_DEVICE_API_VERSION_1_0) {
            // Only support higher HAL version device opened as HAL1.0 device.
            client = new CameraClient(this, cameraClient,
                    clientPackageName, cameraId,
                    facing, callingPid, clientUid, getpid(), legacyMode);
        } else {
            // Other combinations (e.g. HAL3.x open as HAL2.x) are not supported yet.
            ALOGE("Invalid camera HAL version %x: HAL %x device can only be"
                    " opened as HAL %x device", halVersion, deviceVersion,
                    CAMERA_DEVICE_API_VERSION_1_0);
            return INVALID_OPERATION;
        }
    }

    status_t status = connectFinishUnsafe(client, client->getRemote());
    if (status != OK) {
        // this is probably not recoverable.. maybe the client can try again
        return status;
    }

    mClient[cameraId] = client;
    LOG1("CameraService::connect X (id %d, this pid is %d)", cameraId,
         getpid());

    return OK;
}
```
可见，在CAMERA_DEVICE_API_VERSION_2_0之前使用CameraClient进行实例化，之后则采用Camera2Client进行实例化。以CameraClient为例，其initialize()函数如下：
```
status_t CameraClient::initialize(camera_module_t *module) {
    int callingPid = getCallingPid();
    status_t res;

    LOG1("CameraClient::initialize E (pid %d, id %d)", callingPid, mCameraId);

    // Verify ops permissions
    res = startCameraOps();
    if (res != OK) {
        return res;
    }

    char camera_device_name[10];
    snprintf(camera_device_name, sizeof(camera_device_name), "%d", mCameraId);

    mHardware = new CameraHardwareInterface(camera_device_name);
    res = mHardware->initialize(&module->common);
    if (res != OK) {
        ALOGE("%s: Camera %d: unable to initialize device: %s (%d)",
                __FUNCTION__, mCameraId, strerror(-res), res);
        mHardware.clear();
        return res;
    }

    mHardware->setCallbacks(notifyCallback,
            dataCallback,
            dataCallbackTimestamp,
            (void *)(uintptr_t)mCameraId);

    // Enable zoom, error, focus, and metadata messages by default
    enableMsgType(CAMERA_MSG_ERROR | CAMERA_MSG_ZOOM | CAMERA_MSG_FOCUS |
                  CAMERA_MSG_PREVIEW_METADATA | CAMERA_MSG_FOCUS_MOVE);

    LOG1("CameraClient::initialize X (pid %d, id %d)", callingPid, mCameraId);
    return OK;
}
```
上述函数中，主要注意以下流程：

 1. 加粗的代码CameraHardwareInterface新建了了一个Camera硬件接口，当然，camera_device_name为摄像头设备名；
 2. mHardware->initialize(&module->common)调用底层硬件的初始化方法；
 3. mHardware->setCallbacks将CamerService处的回调函数注册到HAL处。

CameraHardwareInterface定义了Camera的硬件抽象特征，由此进入到HAL。

## 四、HAL：CameraHardwareInterface
CameraHardwareInterface的作用在于链接Camera Server和V4L2，通过实现CameraHardwareInterface可以屏蔽不同的driver对Camera Server的影响。CameraHardwareInterface同样虚拟继承自RefBase。
```
class CameraHardwareInterface : public virtual RefBase {
public:
    CameraHardwareInterface(const char *name)
    {
        mDevice = 0;
        mName = name;
    }
    ……
}
```
CameraHardwareInterface中包含了控制通道和数据通道，控制通道用于处理预览和视频获取的开始/停止、拍摄照片、自动对焦等功能，数据通道通过回调函数来获得预览、视频录制、自动对焦等数据。当需要支持新的硬件时就需要继承于CameraHardwareInterface ，来实现对应的功能。CameraHardwareInterface提供的public方法如下：
![](/img/20160401-2.png)
在前一节中，initialize()函数调用了mHardware->initialize和mHardware->setCallbacks，下面来看下CameraHardwareInterface.h对其的实现。
```
status_t initialize(hw_module_t *module)
{
    ALOGI("Opening camera %s", mName.string());
    camera_module_t *cameraModule = reinterpret_cast<camera_module_t *>(module);
    camera_info info;
    status_t res = cameraModule->get_camera_info(atoi(mName.string()), &info);
    if (res != OK) return res;

    int rc = OK;
    if (module->module_api_version >= CAMERA_MODULE_API_VERSION_2_3 &&
        info.device_version > CAMERA_DEVICE_API_VERSION_1_0) {
        // Open higher version camera device as HAL1.0 device.
        rc = cameraModule->open_legacy(module, mName.string(),
                                           CAMERA_DEVICE_API_VERSION_1_0,
                                           (hw_device_t **)&mDevice);
    } else {
        rc = CameraService::filterOpenErrorCode(module->methods->open(
            module, mName.string(), (hw_device_t **)&mDevice));
    }
    if (rc != OK) {
        ALOGE("Could not open camera %s: %d", mName.string(), rc);
        return rc;
    }
    initHalPreviewWindow();
    return rc;
}
```
在initialize()方法中，通过cameraModule->open_legacy打开摄像头模组，initHalPreviewWindow()用于初始化Preview的相关流opspreview_stream_ops，初始化hal的预览窗口。
```
void initHalPreviewWindow()
{
    mHalPreviewWindow.nw.cancel_buffer = __cancel_buffer;
    mHalPreviewWindow.nw.lock_buffer = __lock_buffer;
    mHalPreviewWindow.nw.dequeue_buffer = __dequeue_buffer;
    mHalPreviewWindow.nw.enqueue_buffer = __enqueue_buffer;
    mHalPreviewWindow.nw.set_buffer_count = __set_buffer_count;
    mHalPreviewWindow.nw.set_buffers_geometry = __set_buffers_geometry;
    mHalPreviewWindow.nw.set_crop = __set_crop;
    mHalPreviewWindow.nw.set_timestamp = __set_timestamp;
    mHalPreviewWindow.nw.set_usage = __set_usage;
    mHalPreviewWindow.nw.set_swap_interval = __set_swap_interval;

    mHalPreviewWindow.nw.get_min_undequeued_buffer_count =
            __get_min_undequeued_buffer_count;
}
```
```
/** Set the notification and data callbacks */
void setCallbacks(notify_callback notify_cb,
                  data_callback data_cb,
                  data_callback_timestamp data_cb_timestamp,
                  void* user)
{
    mNotifyCb = notify_cb;
    mDataCb = data_cb;
    mDataCbTimestamp = data_cb_timestamp;
    mCbUser = user;

    ALOGV("%s(%s)", __FUNCTION__, mName.string());

    if (mDevice->ops->set_callbacks) {
        mDevice->ops->set_callbacks(mDevice,
                               __notify_cb,
                               __data_cb,
                               __data_cb_timestamp,
                               __get_memory,
                               this);
    }
}
```
set_callbacks中，__notify_cb、__data_cb、__data_cb_timestamp和__get_memory分别消息回调，数据回调，时间戳回调，以及内存相关操作的回调。
 

以上通过简略分析应用层调用Camera.open()之后在Framework、ART、Library以及HAL层的响应，来说明Android中Camera系统的整体架构，希望对读者能有一定的帮助，后续将在理解Camera整体架构的基础，探索更加高效的Preview方式，敬请期待！

 