<!--
{
  "title": "Android Application",
  "date": "2017-05-18T12:28:03+09:00",
  "category": "",
  "tags": ["android"],
  "draft": true
}
-->


# TODO

- Application process main function
  - ActivityThread
  - follow Zygote steps (ZygoteConnection.Arguments)
- IPC architecture (Binder)
- WindowManager, SurfaceFlinger
- who/when launches SystemServer app ?


```
[ Data structure (App) ]
ActivityThread
'-' ApplicationThread (< ApplicationThreadNative < Binder, IApplicationThread)
'-' Looper

[ Data structure (system service) ]
ActivityManagerService::Lifecycle (< SystemService)
'-' ActivityManagerService

ActivityManagerService (< ActivityManagerNative < Binder)
'-' SystemServiceManager
'-' ProcessRecord mHomeProcess
'-' ActivityThread mSystemThread
'-' WindowManagerService
'-' UiHandler (e.g. handles SHOW_NOT_RESPONDING_UI_MSG)
'-' MainHandler
'-' Handler mBgHandler

ServiceManagerProxy < IServiceManager
'-' BpBinder (cpp) < IBinder, < IServiceManager

[ Data structure (around IPC) ]
Binder < IBinder
BpBinder < IBinder
ProcessState
IPCThreadState


[ SystemServer main code path ]
- SystemServer.main =>
  - new SystemServer
  - SystemServer.run =>
    - Looper.prepareMainLooper
    - createSystemContext =>
      - ActivityThread.systemMain =>
        - new ActivityThread
        - ActivityThread.attach (different path from normal app) =>
          - ActivityThread.getSystemContext =>
            - ContextImpl.createSystemContext =>
              - new LoadedApk
              - new ContextImpl
          - ContextImpl context = ContextImpl.createAppContext => new ContextImpl
          - context.mPackageInfo.makeApplication => newApplication ...
          - Application.onCreate => ?
    - System.loadLibrary("android_servers") (see base/services/Android.mk) =>
      - JNI_OnLoad =>
        - register_android_server_ActivityManagerService
        - ... ?
    - new SystemServiceManager
    - LocalServices.addService(SystemServiceManager.class ...)
    - startBootstrapServices =>
      - SystemServiceManager.startService(Installer.class) => ?
      - SystemServiceManager.startService(ActivityManagerService.Lifecycle.class) =>
        - (some reflection e.g. Class.forName and Class.getConstructor)
          new ActivityManagerService::Lifecycle =>
          - new ActivityManagerService(context) =>
            - new ServiceThread and start
            - new MainHandler
            - new UiHandler
            - new IntentFirewall
            - new ActivityStackSupervisor
            - new ActivityStarter
            - ...
        - ActivityManagerService::Lifecycle.onStart =>
          - ActivityManagerService.start =>
            - LocalServices.addService(ActivityManagerInternal.class, new LocalService())
      - ActivityManagerService.setSystemServiceManager
      - SystemServiceManager.startService(DisplayManagerService.class) => ?
      - PackageManagerService.main => ?
      - SystemServiceManager.startService(UserManagerService.LifeCycle.class) => ?
      - ActivityManagerService.setSystemProcess =>
        - ServiceManager.addService(Context.ACTIVITY_SERVICE (= "activity") ..) =>
          - (initialize central Binder and a lot more happens ...)
        - ...
    - startCoreServices => BatteryService, UsageStatsService, ...
    - startOtherServices =>
      - WindowManagerService.main => ?
      - ServiceManager.addService(Context.WINDOW_SERVICE, wm)
      - ActivityManagerService.setWindowManager
      - DisplayManagerService.windowManagerAndInputReady => ?
      - WindowManagerService.displayReady => ?
      - SystemServiceManager.startService(MOUNT_SERVICE_CLASS)
      - new StatusBarManagerService
      - .. and much more services (e.g. telephony, wifi, ...)
      - SystemServiceManager.startBootPhase(SystemService.PHASE_LOCK_SETTINGS_READY)
      - SystemServiceManager.startBootPhase(SystemService.PHASE_SYSTEM_SERVICES_READY)
      - WindowManagerService.systemReady => ?
      - DisplayManagerService.systemReady => ?
      - ActivityManagerService.systemReady => ?
    - Looper.loop




[ Application main code path ]
- ((static init ?))
- ActivityThread.main =>
  - Looper.prepareMainLooper =>
    - prepare => sThreadLocal.set(new Looper)
    - sMainLooper = myLooper
  - new ActivityThread =>
    - ((non-static init))
    - mAppThread = new ApplicationThread() =>
      - ApplicationThreadNative => Binder.attachInterface(this, "android.app.IApplicationThread")
    - mLooper = Looper.myLooper()
  - ActivityThread.attach(false) =>
    - RuntimeInit.setApplicationObject(mAppThread.asBinder())
    - ActivityManagerNative.getDefault => Singleton<IActivityManager>.get => create =>
      - IBinder b = ServiceManager.getService("activity") =>
        - getIServiceManager =>
          - BinderInternal.getContextObject =>
            - (cpp) android_os_BinderInternal_getContextObject =>
              - ProcessState::self => new ProcessState =>
                - open_driver => open("/dev/binder" ..) and mmap
              - ProcessState::getContextObject =>
                - getStrongProxyForHandle => new BpBinder (as IBinder)
              - javaObjectForIBinder => ??
          - ServiceManagerNative.asInterface =>
            - BpBinder.queryLocalInterface ?? or
            - new ServiceManagerProxy => mRemote = (BpBinder)
        - ServiceManagerProxy.getService("activity") =>
          - mRemote.transact(GET_SERVICE_TRANSACTION, ..) (TODO: what is "activity" remote process ?)=>
            - (cpp) BpBinder.transact =>
              - IPCThreadState::transact =>
                - writeTransactionData => ...
                - waitForResponse =>
                  - while(1) talkWithDriver =>
                    - do while ioctl(mProcess->mDriverFD, BINDER_WRITE_READ, &bwr)
      - IActivityManager am = asInterface(b) => new ActivityManagerProxy(b) => mRemote = b
    - ActivityManagerProxy.attachApplication =>
      - mRemote.transact(ATTACH_APPLICATION_TRANSACTION, ..) => (IPC ?) =>
        - ActivityManagerNative.onTransact (case ATTACH_APPLICATION_TRANSACTION) =>
          - ActivityManagerService.attachApplication => attachApplicationLocked(IApplicationThread thread, int pid) =>
            - ProcessRecord app = mPidsSelfLocked.get(pid)
            - ProcessRecord.makeActive =>
              - baseProcessTracker = ProcessStatsService.getProcessStateLocked =>
                - ProcessStats.getProcessStateLocked => ?? ProcessState ..?
              - baseProcessTracker.makeActive => ?
            - updateProcessForegroundLocked ?
            - thread.bindApplication => (IPC ?) => ApplicationThread.bindApplication
              - mRemote.transact(BIND_APPLICATION_TRANSACTION, )
            - mStackSupervisor.attachApplicationLocked => ?
            - mServices.attachApplicationLocked => ?
  - Looper.loop =>
    - for (;;)
      - Message msg = queue.next
      - msg.target.dispatchMessage => ...
```