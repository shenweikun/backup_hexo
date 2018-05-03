title: framework篇之1.CameraService啓動
categories: Camera系統源碼分析
date: 2018-05-03 14:57:03
tags:
---
Service Manager是系统中一个负责管理所有的服务（如相机，音频）的独立的进程，它的handle是0。<!--more--> 它是整个Binder机制的守护进程，用来管理开发者创建的各种Server，并且向Client提供查询Server远程接口的功能。Android Camera 采用C/S架构，client 与server两个独立的线程之间使用Binder通信，因此其必然需要将它的Service注册到ServiceManager里面，以备后续Client引用。  
*** 
### framework层  
#### 相關文件結構  
service这部分包括以下几个头文件：ICamera.h, ICameraService.h, CameraService.h，对应的实现ICamera.cpp, ICameraService.cpp, CameraService.cpp  
``` 
frameworks\av\camera\cameraserver\main_cameraserver.cpp
frameworks\av\services\camera\libcameraservice\CameraService.h
frameworks\native\include\binder\BinderService.h
frameworks\av\services\camera\libcameraservice\CameraService.cpp 
frameworks\av\services\camera\libcameraservice\common\CameraModule.cpp
```  
#### 啓動流程  
在分析CameraService启动前，先了解一下CameraService类的定义，在frameworks\av\services\camera\libcameraservice\CameraService.h中，如下：  
```  
class CameraService :
    public BinderService<CameraService>,
    public ::android::hardware::BnCameraService,
    public IBinder::DeathRecipient,
    public camera_module_callbacks_t
{
public：
    // Implementation of BinderService<T>
    static char const* getServiceName() { return "media.camera"; }
    ……
}
```  
CameraService多继承于四个父类BinderService、 BnCameraService、 DeathRecipient、 camera_module_callbacks_t  
getServiceName函数用于获取值“media.camera”。  

  
CameraService的启动在frameworks\av\camera\cameraserver\main_cameraserver.cpp的main函数中：  
```  
#define LOG_TAG "cameraserver"
//#define LOG_NDEBUG 0

// from LOCAL_C_INCLUDES
#include "CameraService.h"
#include "RegisterExtensions.h"

using namespace android;

int main(int argc __unused, char** argv __unused)
{
    signal(SIGPIPE, SIG_IGN);

    sp<ProcessState> proc(ProcessState::self());

    sp<IServiceManager> sm = defaultServiceManager(); 
    ALOGI("ServiceManager: %p", sm.get());
    
    //调用CameraService的instantiate方法，来进行CameraService的注册
    CameraService::instantiate();
    registerExtensions();
    ProcessState::self()->startThreadPool();
    IPCThreadState::self()->joinThreadPool();
}
```  
CameraService的注册是通过调用CameraService类的instantiate方法来实现的，instantiate方法来源于其父类BinderService。  
   
   
BinderService类定义在frameworks\native\include\binder\BinderService.h中，如下：  
```  
template<typename SERVICE>
class BinderService
{
public: 
    //这里由于使用了智能指针，所以除了调用CameraService的构造函数之外，还会调用onFirstRef函数
    static status_t publish(bool allowIsolated = false) {
        sp<IServiceManager> sm(defaultServiceManager());
        //3.调用addService函数将CameraService注册到Servicemanager中
        return sm->addService(
                //2.调用getServiceName函数获取值"media.camera";new SERVICE()等价于new CameraService()
                String16(SERVICE::getServiceName()),
                new SERVICE(), allowIsolated);
    }

    //1.单纯调用了publish函数
    static void instantiate() { publish(); }

    ……
};
```  
**a.** instantiate函数调用publish函数  
**b.** publish函数首先构造了一个CameraService，再通过addService函数将它注册到ServiceManager当中，而getServiceName函数获取到的值为“media camera”。这样，Camera就在ServiceManager完成服务注册，提供给client随时使用。这一切都是为了binder通信做准备。  
**c.** 这里使用了c++模版，从上面的CameraService类定义中可以看出，这里的SERVICE等于CameraService，也就是说publish函数中的new SERVICE等于new CameraService。  
**d.** 同时还使用了智能指针，由于智能指针的特性：如果首次调用这个对象的incStrong函数，就会调用一个这个对象的onFirstRef函数。也就是说此处除了调用CameraService的构造函数外，还会调用onFirstRef函数。  
  
  
CameraService的构造函数和onFirstRef函数实现在frameworks\av\services\camera\libcameraservice\CameraService.cpp中，如下：  
```  
CameraService::CameraService() :
        mEventLog(DEFAULT_EVENT_LOG_LENGTH),
        mNumberOfCameras(0), mNumberOfNormalCameras(0),
        mSoundRef(0), mModule(nullptr) {
    ALOGI("CameraService started (pid=%d)", getpid());
    gCameraService = this;

    this->camera_device_status_change = android::camera_device_status_change;
    this->torch_mode_status_change = android::torch_mode_status_change;

    mServiceLockWrapper = std::make_shared<WaitableMutexWrapper>(&mServiceLock);
}

void CameraService::onFirstRef()
{
    ALOGI("CameraService process starting");

    BnCameraService::onFirstRef();

    // Update battery life tracking if service is restarting
    BatteryNotifier& notifier(BatteryNotifier::getInstance());
    notifier.noteResetCamera();
    notifier.noteResetFlashlight();

    camera_module_t *rawModule;
    //通过hw_get_module函数加载了一个hw_module_t模块，这个模块是与hal层对接的接口，
//ID为CAMERA_HARDWARE_MODULE_ID，并将它保存在mModule成员变量中。
    int err = hw_get_module(CAMERA_HARDWARE_MODULE_ID,
            (const hw_module_t **)&rawModule);
    if (err < 0) {
        ALOGE("Could not load camera HAL module: %d (%s)", err, strerror(-err));
        logServiceError("Could not load camera HAL module", err);
        return;
    }

    mModule = new CameraModule(rawModule);
    err = mModule->init();
    if (err != OK) {
        ALOGE("Could not initialize camera HAL module: %d (%s)", err,
            strerror(-err));
        logServiceError("Could not initialize camera HAL module", err);

        delete mModule;
        mModule = nullptr;
        return;
    }
    ALOGI("Loaded \"%s\" camera module", mModule->getModuleName());

    //通过mModule->getNumberOfCameras();函数进入到hal层，获取到了camera的个数。
    //这个函数很重要，对于frameworks层来说只是拿到了camera的个数，
    //但对于hal层和drivers层来说Camera的上电和初始化流程都是从这里开始的
    mNumberOfCameras = mModule->getNumberOfCameras();
    mNumberOfNormalCameras = mNumberOfCameras;

    ……

    CameraService::pingCameraServiceProxy();
}
```  
  


getNumberOfCamera函数是定义在frameworks\av\services\camera\libcameraservice\common\CameraModule.cpp中，最终调用到get_number_of_cameras  
```  
int CameraModule::getNumberOfCameras() {
    int numCameras;
    ATRACE_BEGIN("camera_module->get_number_of_cameras");
    numCameras = mModule->get_number_of_cameras();
    ATRACE_END();
    return numCameras;
}
```  


### HAL层  
先看一下MTK camera module的定义，
在vendor\mediatek\proprietary\hardware\mtkcam\main\hal\module\module\module.h下：  
```  
static
camera_module
get_camera_module()
{
    //保存在frameworks层CameraService的成员变量mModule里面的就是这个module结构体
    camera_module module = {
        common:{
             tag                    : HARDWARE_MODULE_TAG,
             #if (PLATFORM_SDK_VERSION >= 21)
                #if NEED_MODULE_API_VERSION_2_3
                #warning "NEED_MODULE_API_VERSION_2_3 is: 1, use CAMERA_MODULE_API_VERSION_2_3"
                    module_api_version     : CAMERA_MODULE_API_VERSION_2_3,
                #else
                #warning "NEED_MODULE_API_VERSION_2_3 is: 0, use default CAMERA_MODULE_API_VERSION_2_4"
                    module_api_version     : CAMERA_MODULE_API_VERSION_2_4,
                #endif
             #else
                    module_api_version     : CAMERA_DEVICE_API_VERSION_1_0,
             #endif
             hal_api_version        : HARDWARE_HAL_API_VERSION,
             id                     : CAMERA_HARDWARE_MODULE_ID,
             name                   : "MediaTek Camera Module",
             author                 : "MediaTek",
             methods                : get_module_methods(),
             dso                    : NULL,
             reserved               : {0},
        },
        //当frameworks层调用mModule->get_number_of_cameras函数时，实际就是调用这个结构体的get_number_of_cameras函数
        get_number_of_cameras       : get_number_of_cameras,
        get_camera_info             : get_camera_info,
        set_callbacks               : set_callbacks,
        get_vendor_tag_ops          : get_vendor_tag_ops,
        #if (PLATFORM_SDK_VERSION >= 21)
        open_legacy                 : open_legacy,
        #endif
        set_torch_mode              : set_torch_mode,
        init                        : NULL,
        reserved                    : {0},
    };
    return  module;
};
```  
1.保存在frameworks层CameraService的成员变量mModule里面的就是上面这个module结构体  
2.当frameworks层调用mModule->get_number_of_cameras函数时，实际就是调用上面结构体的get_number_of_cameras函数：  
```  
static
int
get_number_of_cameras(void)
{
    //1.调用getCamDeviceManager获得一个CamDeviceManagerImp
    //2.CamDeviceManagerImp对象的getNumberOfDevices()方法，该方法由其父类CamDeviceManagerBase实现
    return  NSCam::getCamDeviceManager()->getNumberOfDevices();
}
```  
  
  
  
module.h中的get_number_of_cameras函数首先调用了getCamDeviceManager()函数，获得一个CamDeviceManagerImp对象：
getCamDeviceManager()函数定义在  
vendor\mediatek\proprietary\hardware\mtkcam\main\hal\module\depend\CamDeviceManagerImp.cpp  
```  
CamDeviceManagerImp gCamDeviceManager;

ICamDeviceManager*
getCamDeviceManager()
{
    return  &gCamDeviceManager;
}
```  
   
   
CamDeviceManagerImp继承了CamDeviceManagerBase，这里的getNumberOfDevices方法将由父类CamDeviceManagerBase实现:  
vendor\mediatek\proprietary\hardware\mtkcam\main\hal\module\devicemgr\CamDeviceManagerBase.cpp  
```  
int32_t
CamDeviceManagerBase::
getNumberOfDevices()
{
    RWLock::AutoWLock _l(mRWLock);
    //
    if  ( 0 != mi4DeviceNum )
    {
#if MTKCAM_HAVE_CAMERAMOUNT
        mi4DeviceNum = mEnumMap.size() + mExternalDeviceInfoMap.size() - 1;
        MY_LOGD("#devices:%d #remote:%u mEnumMap:%u", mi4DeviceNum, mExternalDeviceInfoMap.size(), mEnumMap.size());
#else
        MY_LOGD("#devices:%d", mi4DeviceNum);
#endif
    }
    else
    {
        Utils::CamProfile _profile(__FUNCTION__, "CamDeviceManagerBase");
        //获得camera的个数
        mi4DeviceNum = enumDeviceLocked();
        _profile.print("");
    }
    //
    return  mi4DeviceNum;
```  

CamDeviceManagerBase::getNumberOfDevices()函数主要调用了enumDeviceLocked()函数，并返回camera的个数，接着来看enumDeviceLocked()函数的实现，其定义在：  
vendor\mediatek\proprietary\hardware\mtkcam\main\hal\module\depend\CamDeviceManagerImp.cpp  
```  
int32_t
CamDeviceManagerImp::
enumDeviceLocked()
{
    status_t status = OK;
    int32_t i4DeviceNum = 0;
    //
#if '1'==MTKCAM_HAVE_METADATA
    NSMetadataProviderManager::clear();
#endif
    mEnumMap.clear();
//---------------------------------------------------  ---------------------------
#if '1'==MTKCAM_HAVE_SENSOR_HAL
    //
    IHalSensorList*const pHalSensorList = MAKE_HalSensorList();
    //返回sensor的个数
    size_t const sensorNum = pHalSensorList->searchSensors();
    CAM_LOGD("pHalSensorList:%p searchSensors:%zu queryNumberOfSensors:%d", pHalSensorList, sensorNum, pHalSensorList->queryNumberOfSensors());
     
    ……    

    //
    return  i4DeviceNum;
}
```  
   
  
enumDeviceLocked()函数重点关注pHalSensorList->searchSensors()，searchSensors()定义在：   
vendor\mediatek\proprietary\hardware\mtkcam\drv\src\sensor\mt6757\HalSensorList.cpp  
```  
MUINT
HalSensorList::
searchSensors()
{
    Mutex::Autolock _l(mEnumSensorMutex);
    //
    MY_LOGD("searchSensors");
    //调用enumerateSensor_Locked返回sensor的个数
    return  enumerateSensor_Locked();
}
```  


   
enumerateSensor_Locked定义在 vendor\mediatek\proprietary\hardware\mtkcam\drv\src\sensor\mt6757\HalSensorList.enumList.cpp：  
```  
MUINT
HalSensorList::
enumerateSensor_Locked()
{
    int ret_count = 0;
    int ret = 0;

    //
    #warning "[WARN] Simulation for enumerateSensor_Locked()"

    MUINT halSensorDev = SENSOR_DEV_NONE;
    NSFeature::SensorInfoBase* pSensorInfo ;

    SensorDrv *const pSensorDrv = SensorDrv::get();
    SeninfDrv *const pSeninfDrv = SeninfDrv::createInstance();
    if(!pSeninfDrv) {
        MY_LOGE("pSeninfDrv == NULL");
                return 0;
    }


    ret = pSeninfDrv->init();
    if(ret < 0) {
        MY_LOGE("pSeninfDrv->init() fail");
                return 0;
    }
    /*search sensor using 8mA driving current*/
    pSeninfDrv->setMclk1IODrivingCurrent(3);// [6:5] = 0:2mA, 1:4mA, 2:6mA, 3:8mA
    pSeninfDrv->setMclk2IODrivingCurrent(3);// [6:5] = 0:2mA, 1:4mA, 2:6mA, 3:8mA
    pSeninfDrv->setMclk1(1, 1, 1, 0, 1, 0, 0, 0);
    pSeninfDrv->setMclk2(1, 1, 1, 0, 1, 0, 0, 0);
    //pSeninfDrv->setMclk3(1, 1, 1, 0, 1, 0, 0);  /* No main2 */

    //重点关注impSearchSensor，它的返回值将决定enumerateSensor_Locked()的返回值
    int const iSensorsList = pSensorDrv->impSearchSensor(NULL);

     ……     

    return  ret_count;
}
```  


   
下面着重分析一下ImgSensorDrv::impSearchSensor(pfExIdChk pExIdChkCbf)，它定义在：
 vendor\mediatek\proprietary\hardware\mtkcam\drv\src\sensor\mt6757\Imgsensor_drv.cpp  
```  
MINT32
ImgSensorDrv::impSearchSensor(pfExIdChk pExIdChkCbf)
{
    MUINT32 SensorEnum = (MUINT32) DUAL_CAMERA_MAIN_SENSOR;
    MUINT32 i,id[KDIMGSENSOR_MAX_INVOKE_DRIVERS] = {0,0};
    MBOOL SensorConnect=TRUE;
    UCHAR cBuf[64];
    MINT32 err = SENSOR_NO_ERROR;
    MINT32 err2 = SENSOR_NO_ERROR;
    ACDK_SENSOR_INFO_STRUCT SensorInfo;
    ACDK_SENSOR_CONFIG_STRUCT SensorConfigData;
    ACDK_SENSOR_RESOLUTION_INFO_STRUCT SensorResolution;
    MINT32 sensorDevs = SENSOR_NONE;
    IMAGE_SENSOR_TYPE sensorType = IMAGE_SENSOR_TYPE_UNKNOWN;
    IMGSENSOR_SOCKET_POSITION_ENUM socketPos = IMGSENSOR_SOCKET_POS_NONE;


    //! If imp sensor search process already done before,
    //! only need to return the sensorDevs, not need to
    //! search again.
    if (SENSOR_DOES_NOT_EXIST != m_mainSensorId) {
        //been processed.
        LOG_MSG("[impSearchSensor] Already processed \n");
        if (BAD_SENSOR_INDEX != m_mainSensorIdx) {
            sensorDevs |= SENSOR_MAIN;
        }
        if (BAD_SENSOR_INDEX != m_main2SensorIdx) {
            sensorDevs |= SENSOR_MAIN_2;
        }
        if (BAD_SENSOR_INDEX != m_subSensorIdx) {
            sensorDevs |= SENSOR_SUB;
        }
#ifdef MTK_SUB2_IMGSENSOR
        if (BAD_SENSOR_INDEX != m_sub2SensorIdx) {
            sensorDevs |= SENSOR_SUB_2;
        }
#endif

        #ifdef  ATVCHIP_MTK_ENABLE

            sensorDevs |= SENSOR_ATV;

        #endif


        return sensorDevs;
    }

    //调用GetSensorInitFuncList函数来获取hal层的sersors列表，并把它保存在m_pstSensorInitFunc变量中
    GetSensorInitFuncList(&m_pstSensorInitFunc);

    LOG_MSG("SENSOR search start \n");

    if (-1 != m_fdSensor) {
        ::close(m_fdSensor);
        m_fdSensor = -1;
    }
    sprintf(cBuf,"/dev/%s",CAMERA_HW_DEVNAME);
    //通过系统调用open函数打开camera的设备节点，后面会通过这个节点来进入到kernel层
    m_fdSensor = ::open(cBuf, O_RDWR);
    if (m_fdSensor < 0) {
         LOG_ERR("[impSearchSensor]: error opening %s: %s \n", cBuf, strerror(errno));
        return sensorDevs;
    }

    // search main/main_2/sub 3 sockets
#ifdef MTK_SUB2_IMGSENSOR
    for (SensorEnum = DUAL_CAMERA_MAIN_SENSOR; SensorEnum <= DUAL_CAMERA_SUB_2_SENSOR; SensorEnum <<= 1)  
    {
       LOG_MSG("impSearchSensor search to sub2\n");

#else
#ifdef MTK_MAIN2_IMGSENSOR
    for (SensorEnum = DUAL_CAMERA_MAIN_SENSOR; SensorEnum <= DUAL_CAMERA_MAIN_2_SENSOR; SensorEnum <<= 1)  
    {
        LOG_MSG("impSearchSensor search to main_2\n");
#else
   #ifdef MTK_SUB_IMGSENSOR
    for (SensorEnum = DUAL_CAMERA_MAIN_SENSOR; SensorEnum <= DUAL_CAMERA_SUB_SENSOR; SensorEnum <<= 1)  
    {
        LOG_MSG("impSearchSensor search to sub\n");
   #else
    for (SensorEnum = DUAL_CAMERA_MAIN_SENSOR; SensorEnum < DUAL_CAMERA_SUB_SENSOR; SensorEnum <<= 1)  
    {
        LOG_MSG("impSearchSensor search to main\n");
   #endif
   #endif
#endif


        //
        for (i = 0; i < MAX_NUM_OF_SUPPORT_SENSOR; i++) {
            //end of driver list
            if (m_pstSensorInitFunc[i].getCameraDefault == NULL) {
                LOG_MSG("m_pstSensorInitFunc[i].getCameraDefault is NULL: %d \n", i);
                break;
            }
                //set sensor driver
            id[KDIMGSENSOR_INVOKE_DRIVER_0] = (SensorEnum << KDIMGSENSOR_DUAL_SHIFT) | i;
            LOG_MSG("set sensor driver id =%x\n", id[KDIMGSENSOR_INVOKE_DRIVER_0]);
            //通过ioctl下达setDriver指令，并下传正在遍历的sensorlist中的ID。
            //Driver层根据这个ID，挂载Driver层sensorlist中对应的操作接口
            err = ioctl(m_fdSensor, KDIMGSENSORIOC_X_SET_DRIVER,&id[KDIMGSENSOR_INVOKE_DRIVER_0] );
                if (err < 0) {
                    LOG_ERR("ERROR:KDCAMERAHWIOC_X_SET_DRIVER\n");
                }


                //err = open();
                //通过ioctl下达check ID指令，Driver层为对应sensor上电，通过I2C读取预存在寄存器中的sensor id。
                //然后比较读取结果，如果不匹配return error后继续遍历
                err = ioctl(m_fdSensor, KDIMGSENSORIOC_T_CHECK_IS_ALIVE);


                if (err < 0) {
                    LOG_MSG("[impSearchSensor] Err-ctrlCode (%s) \n", strerror(errno));
                }
            //
     
     ……

    return sensorDevs;
}//
```  

### 總結
![](CameraService.jpg) 