本文主要介绍腾讯云 TRTC SDK(Android) 的几个最基本功能的使用方法，阅读此文档有助于您对 TRTC 的基本使用流程有一个简单的认识。


## 初始化 SDK

使用 TRTC SDK 的第一步，是先获取 `TRTCCloud` 的单例对象，并注册监听 SDK 事件的回调。

- 先继承`TRTCCloudListener`抽象类并重写你需要监听的事件（eg：用户加入房间、用户退出房间、警告信息、错误信息等）。
- 获取`TRTCCloud`单例对象，调用setListener方法设置`TRTCCloudListener `回调。

```java
import com.tencent.trtc.TRTCCloud;
import com.tencent.trtc.TRTCCloudListener;

// 继承 TRTCCloudListener 回调
static class TRTCCloudListenerImpl extends TRTCCloudListener {
	private WeakReference<TRTCMainActivity> mContext;
    public TRTCCloudListenerImpl(TRTCMainActivity activity) {
        super();
        mContext = new WeakReference<>(activity);
    }
	....
	// 错误通知是要监听的，错误通知意味着 SDK 不能继续运行了
	@Override
	public void onError(int errCode, String errMsg, Bundle extraInfo) {
	    Log.d(TAG, "sdk callback onError");
	    TRTCMainActivity activity = mContext.get();
	    if (activity != null) {
	        Toast.makeText(activity, "onError: " + errMsg + "[" + errCode+ "]" , Toast.LENGTH_SHORT).show();
	        if (errCode == TXLiteAVCode.ERR_ROOM_ENTER_FAIL) {
	            activity.exitRoom();
	        }
	    }
	}
}

// 在activity创建时获取 trtcCloud 单例
@Override
protected void onCreate(Bundle savedInstanceState) {
	super.onCreate(savedInstanceState);
	....
	trtcListener = new TRTCCloudListenerImpl(this);
    trtcCloud = TRTCCloud.sharedInstance(this);
    trtcCloud.setListener(listener);
}

// 销毁 trtcCloud 实例，在不再使用SDK能力时，销毁单例，节省开销
@Override
protected void onDestroy() {
    super.onDestroy();
    //销毁 trtc 实例
    if (trtcCloud != null) {
        trtcCloud.setListener(null);
    }
    trtcCloud = null;
    TRTCCloud.destroySharedInstance();
}

```

## 组装 TRTCParams

TRTCParams 是 SDK 最关键的一个参数，它包含如下四个必填的字段 sdkAppId，userId，userSig 和 roomId。

- **sdkAppId**
进入腾讯云实时音视频[控制台](https://console.cloud.tencent.com/rav)，如果您还没有应用，请创建一个，即可看到 sdkAppId。
![](https://main.qcloudimg.com/raw/832b48f444e86c00097d3f9f322a3439.png)

- **userId**
您可以随意指定，由于是字符串类型，可以直接跟您现有的账号体系保持一致，但请注意，**同一个音视频房间里不应该有两个同名的 userId**。

- **userSig**
基于 sdkAppId 和 userId 可以计算出 userSig，计算方法请参考 [如何计算UserSig](https://cloud.tencent.com/document/product/647/17275)。

- **roomId**
房间号是数字类型，您可以随意指定，但请注意，**同一个应用里的两个音视频房间不能分配同一个 roomId**。


## 进入(或创建)房间

调用 `enterRoom` 函数进入房间时，除了需要 TRTCParams 参数，还需要一个叫 **appScene** 的参数，该参数是指定应用场景用的。

- **VideoCall** 对应视频通话场景，即绝大多数时间都是两人或两人以上视频通话的场景，内部编码器和网络协议优化侧重流畅性，降低通话延迟和卡顿率。
- **LIVE** 对应直播场景，即绝大多数时间都是一人直播，偶尔有多人视频互动的场景，内部编码器和网络协议优化侧重性能和兼容性，性能和清晰度表现更加。
- 如进入房间，SDK 会回调 `onEnterRoom` 接口，参数：`elapsed`代表进入耗时，单位ms。
- 如进房失败，SDK 会回调 `onError` 接口，参数：`errCode`（错误码`ERR_ROOM_ENTER_FAIL`，错误码可参考`TXLiteAVCode.h`）、`errMsg`（错误原因）、`extraInfo`（保留参数）。
- 如果已在房间中，则必须调用 `exitRoom` 方法退出当前房间，才能进入下一个房间。

```java
void enterRoom() {
	//TRTCParams 定义参考头文件TRTCCloudDef.java
	trtcParams = new TRTCCloudDef.TRTCParams();
	trtcParams.sdkAppId = sdkappid;
	trtcParams.userId   = userid;
	trtcParams.userSig  = usersig;
	trtcParams.roomId   = 908; //输入你想进入的房间
	trtcCloud.enterRoom(trtcParams, TRTC_APP_SCENE_VIDEOCALL);
}

...

@Override
public void onError(int errCode, String errMsg, Bundle extraInfo) {
    Log.d(TAG, "sdk callback onError");
    TRTCMainActivity activity = mContext.get();
    if (activity != null) {
        Toast.makeText(activity, "onError: " + errMsg + "[" + errCode+ "]" , Toast.LENGTH_SHORT).show();
        if (errCode == TXLiteAVCode.ERR_ROOM_ENTER_FAIL) {
            activity.exitRoom();
        }
    }
}

@Override
public void onEnterRoom(long elapsed) {
    TRTCMainActivity activity = mContext.get();
    if (activity != null) {
        Toast.makeText(activity, "加入房间成功", Toast.LENGTH_SHORT).show();
    }
}
```

>!请根据应用场景选择合适的 scene 参数，使用错误可能会导致卡顿率或画面清晰度不达预期。

## 收听远端音频流
- TRTC SDK 会默认接收远端的音频流，您无需为此编写额外的代码。
- 如果您不希望收听某一个 userid 的音频流，可以使用 `muteRemoteAudio`将其静音。

## 观看远端视频流
TRTC SDK 并不会默认拉取远端的视频流，您可以通过调用`startRemoteView`方法来实现这个目标：

- 当有远端用户加入或离开房间时，SDK 会通过`onUserEnter`和 `onUserExit`通知到您。
- 当收到 `onUserEnter`回调后，可以调用`startRemoteView`方法来观看新进 userId 的视频画面。
- 当收到 `onUserExit`回调后，可以调用`stopRemoteView`停止观看。
- 通过 `setRemoteViewFillMode` 可以指定视频显示模式为`Fill`或`Fit`模式。两种模式下视频尺寸都是等比缩放，区别在于：
- `Fill` 模式：优先保证视窗被填满。如果缩放后的视频尺寸与显示视窗尺寸不一致，多出的视频将被截掉。
- `Fit`   模式：优先保证视频内容全部显示。如果缩放后的视频尺寸与显示视窗尺寸不一致，未被填满的视窗区域将使用黑色填充。

```java
@Override
public void onUserEnter(String userId) {
	// 设置remoteView
	TXCloudVideoView remoteView = new TXCloudVideoView(mContext);
	mParentView.add(remoteView);

	trtcCloud.setRemoteViewFillMode(userId, TRTCCloudDef.TRTC_VIDEO_RENDER_MODE_FIT);
	trtcCloud.startRemoteView(userId, remoteView);
}

@Override
public void onUserExit(String userId, int reason) {
    //停止观看画面
    trtcCloud.stopRemoteView(userId);
}

```

## 开启（或关闭）本地声音采集
TRTC SDK 并不会默认打开本地的麦克风采集，`startLocalAudio`可以开启本地的声音采集并将音视频数据广播出去，`stopLocalAudio`则会关闭之。

- `startLocalAudio` 会检查麦克风使用权限，如果没有麦克风权限，SDK 会向用户申请开启。
- 您可以在 `startLocalPreview` 之后紧接着调用 `startLocalAudio`。

## 开启（或关闭）本地视频采集

TRTC SDK 并不会默认打开本地的摄像头采集，`startLocalPreview` 可以开启本地的摄像头并显示预览画面，`stopLocalPreview` 则会关闭之。

- 启动本地预览前，可调用`setLocalViewFillMode`指定视频显示模式为`Fill`或 `Fit` 模式。两种模式下视频尺寸都是等比缩放，区别在于：  
	- `Fill` 模式：优先保证视窗被填满。如果缩放后的视频尺寸与显示视窗尺寸不一致，多出的视频将被截掉。  
	- `Fit`   模式：优先保证视频内容全部显示。如果缩放后的视频尺寸与显示视窗尺寸不一致，未被填满的视窗区域将使用黑色填充。

- 调用`startLocalPreview`，参数：`frontCamera`（true：前置摄像头 false：后置摄像头）、`view`（TXCloudVideoView SDK自定义渲染控件）。

```java
/** 打开本地摄像头预览画面 */
void startLocalPreview(boolean frontCamera, TXCloudVideoView localVideoView) {
	trtcCloud.setLocalViewFillMode(TRTCCloudDef.TRTC_VIDEO_RENDER_MODE_FIT);
    trtcCloud.startLocalPreview(frontCamera, localVideoView);
}
```

## 屏蔽音视频数据流

- **屏蔽本地视频数据**
  如果用户在通话过程中，出于隐私目的希望屏蔽本地的视频数据，让房间里的其他用户暂时无法看到您的画面，可以调用 `muteLocalVideo`。
  
- **屏蔽本地音频数据**
  如果用户在通话过程中，出于隐私目的希望屏蔽本地的音频数据，让房间里的其他用户暂时无法听到您的声音，可以调用 `muteLocalAudio`。
  
- **屏蔽远程视频数据**
  通过 `stopRemoteView` 可以屏蔽某一个 userid 的视频数据。
  通过 `stopAllRemoteView` 可以屏蔽所有远端用户的视频数据。
  
- **屏蔽远程音频数据**
  通过 `muteRemoteAudio` 可以屏蔽某一个 userid 的音频数据。
  通过 `muteAllRemoteAudio` 可以屏蔽所有远端用户的音频数据。


## 退出房间

调用`exitRoom`方法退出房间。不论当前是否还在通话中，调用该方法会把视频通话相关的所有资源释放掉。

- 在您调用`exitRoom`之后，SDK 会进入一个复杂的退房握手流程，当 SDK 回调 `onExitRoom` 方法时才算真正完成资源的释放。

```java
...
private void exitRoom() {
    if (trtcCloud != null) {
        trtcCloud.exitRoom();
    }
}
...
@Override
public void onExitRoom(int reason) {
    TRTCMainActivity activity = mContext.get();
    if (activity != null) {
        activity.finishActivity();
    }
}

```

