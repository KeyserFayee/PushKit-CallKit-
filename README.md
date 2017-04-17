# PushKit-CallKit-

1、PushKit分析：
    简介：pushkit中的voippush,可以帮助我们提升voip应用的体验，优化voip应用的开发实现，降低voip应用的电量消耗，它需要我们重新规划和设计我们的voip应用,从而得到更好的体验(voip push可以说是准实时的，实侧延时1秒左右)；苹果的目的是提供这样一种能力，可以让我们抛弃后台长连接的方案，也就是说应用程序通常不用维持和voip服务器的连接，在呼叫或者收到呼叫时，完成voip服务器的注册；当程序被杀死或者手机重启动时，都可以收到对方的来电，正常开展voip的业务。也就是说，我们当前可以利用它来优化voip的体验，增加接通率；条件成熟时我们就可以完全放弃后台的长连接，走到苹果为我们规划的道路上。
pushkit与传统的apns的区别：

1、高优先级别，更大的payload
2、当后台或kill状态下接收到notification后，激活app并展示local notification，当用户滑动时，进入app并准备好通话。
3、pushkit的voip推送没有任何ui效果，当收到推送激活app时，你必须手动处理消息(例如添加本地推送)，下表所示apns推送和pushkit的诧异。

Summary of differences:
 	Regular Push	VoIP Push
Getting Device Token	application.registerForRemoteNotifications()	set PKPushRegistry.desiredPushTypes
Handle Registration	application:didRegisterForRemoteNotificationsWithDeviceToken:	pushRegistry:didUpdatePushCredentials
Handle Received Notification	application:didReceiveRemoteNotification	pushRegistry:didReceiveIncomingPushWithPayload
Payload Size	2048 bytes	4096 bytes
Certificate Type	iOS Push Services	VoIP Services
Requires User Consent	Yes	No*
2、CallKit分析
    从iOS10开始，CallKit 这个开发框架，能够让语音或视讯电话的开发者将 UI 界面整合在 iPhone 原生的电话 App 中.将允许开发者将通讯 App 的功能内建在电话 App 的“常用联系人”，以及“通话记录”，方便用户透过原生电话 App，就能直接取用这些第三方功能;允许用户在通知中心就能直接浏览并回覆来电，来电的画面也将整合在 iOS 原生的 UI 里
    总体来说，等于让 iOS 原本单纯用来打电信电话的“电话”功能，能够结合众多第三方语音通讯软件，具备更完整的数码电话潜力。CallKit 也拓展了在 iOS 8 就出现的 App Extensions 功能，可以让用户在接收来电时，在原生电话 App 中就透过第三方 App 辨识骚扰电话）.总结为以下几点：
    1、网络视频电话类app被叫时可以调用系统电话界面，像这样：

    2、电话界面可以加入用户action，例如进入app或是与其他app相互作用：
    3、可以从系统通讯录，通话记录中打开电话app：
    
    

3、应用案例
1、Tribe
2、skype
3、微信电话本
3、PushKit+CallKit使用介绍
pushkit和callkit结合使用可以实现像tribe那样的来电提醒效果，下面是实现步骤
一、PushKit证书创建

    证书创建以及利用local notification处理消息的方法请详见：http://www.jianshu.com/p/5939dcb5fcd2，这里不过多赘述。
二、利用CallKit处理PushKit消息(仅限ios10及以上)：

Receiving Incoming Calls(接电话)

1、在AppDelegate中创建CXProvider全局对象，CXProvider对象通过CXCallUpdate想系统传递信息
2、在pushkit的通知的处理方法中获取payload的UUID(通话标识)、handle(呼叫者)、identifier、hasVideo等标识


1
2
3
4
5
6
7
8
9
10
11
funcpushRegistry(_registry:PKPushRegistry,didReceiveIncomingPushWithpayload:PKPushPayload
    ,forTypetype:PKPushType){
guardtype==.voIPelse{return}
ifletuuidString=payload.dictionaryPayload["UUID"]as?String,
lethandle=payload.dictionaryPayload["handle"]as?String,
lethasVideo=payload.dictionaryPayload["hasVideo"]as?Bool,
letuuid=UUID(uuidString:uuidString)
{
displayIncomingCall(uuid:uuid,handle:handle,hasVideo:hasVideo)
}
}
3、再创建CXCallUpdate对象update，将hasVideo、identifier等信息传入update中，并调用reportNewIncomingCallWithUUID:update:completion:方法通知系统唤起系统电话(通知系统显示Incoming Call的全屏UI)


1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
funcreportIncomingCall(uuid:UUID,handle:String,hasVideo:Bool=false,completion:
    ((NSError?)->Void)?=nil){
//ConstructaCXCallUpdatedescribingtheincomingcall,includingthecaller.
letupdate=CXCallUpdate()
update.remoteHandle=CXHandle(type:.phoneNumber,value:handle)
update.hasVideo=hasVideo
//Reporttheincomingcalltothesystem
provider.reportNewIncomingCall(with:uuid,update:update){errorin
/*
Onlyaddincomingcalltotheapp'slistofcallsifthecallwasallowed(i.e
    .therewasnoerror)
sincecallsmaybe"denied"forvariouslegitimatereasons
    .SeeCXErrorCodeIncomingCallError.
*/
iferror==nil{
letcall=SpeakerboxCall(uuid:uuid)
call.handle=handle
self.callManager.addCall(call)
}
completion?(erroras?NSError)
}
}
   
Making Outgoing Calls(打Voip电话)
1、创建CXCallController对象callController,创建UUID对象uuid，创建需要拨打的电话handle(被呼叫者)，通过CXStratCallAction初始化将上述参数传入，组成一个Action，然后创建CXTransaction对象传入这个Action，最后通过callController的.request(transaction)方法拨打电话。


1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
  func startCall(handle: String, video: Bool = false) {
    let handle = CXHandle(type: .phoneNumber, value: handle)
    let startCallAction = CXStartCallAction(call: UUID(), handle: handle)
    startCallAction.isVideo = video
    let transaction = CXTransaction()
    transaction.addAction(startCallAction)
    requestTransaction(transaction)
  }
  
      private func requestTransaction(_ transaction: CXTransaction) {
        callController.request(transaction) { error in
            if let error = error {
                print("Error requesting transaction: \(error)")
            } else {
                print("Requested transaction successfully")
            }
        }
    }

CXProvider的代理方法在接打电话时是必须实现的
func providerDidReset(_ provider: CXProvider)   provider被重置
func provider(_ provider: CXProvider, perform action: CXStartCallAction)  拨打电话
func provider(_ provider: CXProvider, perform action: CXAnswerCallAction) 电话接通时
func provider(_ provider: CXProvider, perform action: CXEndCallAction) 电话挂断时
func provider(_ provider: CXProvider, perform action: CXSetHeldCallAction)   电话被保留时
func provider(_ provider: CXProvider, timedOutPerforming action: CXAction) 链接超时
func provider(_ provider: CXProvider, didActivate audioSession: AVAudioSession)  会话被激活时
func provider(_ provider: CXProvider, didDeactivate audioSession: AVAudioSession) 会话被挂断时
 
参考链接
https://developer.apple.com/reference/callkit?language=objc
http://www.jianshu.com/p/2bf4f186dfd9
参考Demo
https://github.com/Lax/iOS-Swift-Demos
