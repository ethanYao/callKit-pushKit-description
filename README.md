# CallKit
## About CallKit
CallKit是一个由苹果提供的开发框架，这个框架能够让第三方应用获得系统通话的权限以及体验，在iOS 10发布以来，各大音频移动引用都陆续支持这个功能。以下是这个框架的特性：

	1. 提高VoIP电话的权限，避免通话过程中被系统来电直接打断的情况。
	2. 将VoIP电话融入系统电话界面UI，包括锁屏界面直接接听 / 保留电话通话记录并可回拨 / 呼叫联系人，贴合用户接听和拨打电话习惯。
	3. 无需打开App，使用Siri即可唤起进行VoIP电话。
	4. 来电识别，用户可在接听前发现骚扰电话。
它可以分为三大模块：`VoIP`，`CallCenter`和`来电屏蔽`，而目前我们仅仅需要关注VoIP这一块就可以了。

## CallKit API & Description
### <a> 主要的类 __CXProvider和CXCallController__ </a>
> __简介__
> 
> 0. <mark> __CXProvider__ </mark>可以理解为处理系统电话界面有关的逻辑，比如来电呼起系统电话界面或者将用户在系统电话界面上的操作通知给App。
> 1. <mark> __CXProviderDelegate__ </mark>使用CallKit代接收来电状态的VoIP应用都需要一个初始化一个CXProvider，比如有来电时通过provider告知系统帮我接听、要呼出电话时告知系统这条电话的基本信息、以及各种操作和状态的更新信息会通过协议代理传回应用；并需要设置一个代理类来接受处理CXProviderDelegate 代理任务操作（各种CXAction：接听、挂断、DTMF、免提等本地操作）。使用之前，通过 CXProviderConfiguration来配置app的具体信息（你的VoIP 自己的显示名称、是否要支持视频通话、最大会话分组数、logo、来电提示音等信息），以便在系统通话界面显示。
> 2. <mark> __CXCallController__ </mark>则是将用户在App界面上的操作通知给系统。
> 3. <mark> __CXCall__ </mark>电话信息基类，有一个唯一识别符UUID，是系统用以区分一个唯一来电信息的东西、通过这个ID可以定位到特定时刻的特定号码的来电信息。
> 4. <mark> __CXCallUpdate__ </mark>通话过程就是信息和状态的变化过程，CallKit的作用只是在于将通话状态和信息在系统接听界面和应用之间传递 ，通过provider请求进行处理；
> 5. <mark> __CXAction/CXCallAction__ </mark>电话操作载体类，细分包括（电话开始:CXStartCallAction、 接听:CXAnswerCallAction、暂停：CXSetHeldCallAction、静音：CXSetMutedCallAction、群组电话CXSetGroupCallAction、双频多音功能：CXPlayDTMFCallAction、挂断或拒接：CXEndCallAction  ）。
> 6. <mark> __CXTransaction__ </mark>操作执行类；CXCallController：话务控制器，每种action在配置好CXTransaction后都需要控制器CXCallController去向系统发起请求来响应操作。
> 7. <mark> __CXCallObserver__ </mark>可以设置一个代理来随时捕获电话信息的更新。


#### 工作原理
##### 先来看一下CallKit的大致流程分工：

![CallKit](http://chuantu.biz/t6/34/1504510397x2728278847.png)

<font color=#000000 size=3 face="微软雅黑"> CallKit有两个主要的类 __CXProvider__ 和 __CXCallController__。CXProvider可以将一些外来事件通知给系统，而CXCallController可以让系统收到App的一些Request，用户的action，内部的事件。
</font>
#### 具体描述
当caller需要唤起一个通话（也就是在app内的操作），需要把用户start的`action`放进去CXTransaction里面，由CXCallController（这个是处理用户在app的操作的控制器）通过这个trasaction载体告诉系统，要打电话了。一旦发起这个请求，那么就会有一个成功或者失败的回调（注意，这里只是对应发起通话的<mark> __请求__ </mark>结果而不是是否接通的结果，后面会用一幅图详细解释），当系统收到这个外来事件后会调用自身的功能，并且会让CXProvider作为他的代理，告诉provider有什么action，其中如果provider想要和系统交互，需要用CXCallUpdate和系统交互的，例如一个来电，某个点触发了来电需要唤起系统的界面，那么需要让provider通过callupdate这个载体告诉系统执行相关事件，系统处理完会有一个回调告诉provider到底完成的怎样！一个完整的拨打&接听的流程就在下图：


![CallKit](http://chuantu.biz/t6/34/1504510342x1022828786.png)

## Use CallKit

<font color=#000000 size=3 face="微软雅黑"> __以下这部分可能比较无聊，也繁琐，但这是使用的技巧__ </font>

1. 首先需要配置好provider，例如是否支持video，最大通话数，是电话类型还是email，设置好provider代理，这个是为了系统做了某些操作告诉provider的。另外也需要在这里配置初始化好CXCallController，把callObserver代理设置好（这个可以监听一切通话状态发生变化）。

```
    NSString *localizedName = @"优分期";    //本地显示的应用名字
    CXProviderConfiguration *configuration = [[CXProviderConfiguration alloc] initWithLocalizedName:localizedName];
    configuration.supportsVideo = NO;
    configuration.maximumCallsPerCallGroup = 1;
    configuration.supportedHandleTypes = [NSSet setWithObjects:[NSNumber numberWithInteger:CXHandleTypePhoneNumber], nil];  // 配置类型
    configuration.iconTemplateImageData = UIImagePNGRepresentation([UIImage imageNamed:@"logo.png"]);   // 显示头像
//    configuration.ringtoneSound = @"test.wav";//如果没有音频文件 就用系统的
    self.provider = [[CXProvider alloc] initWithConfiguration:configuration];
    [self.provider setDelegate:self queue:self.completionQueue ? self.completionQueue : dispatch_get_main_queue()];
    self.callController = [[CXCallController alloc] initWithQueue:dispatch_get_main_queue()];
    [self.callController.callObserver setDelegate:self queue:dispatch_get_main_queue()];
```
2.如果需要拨打电话，由callCtroller通过每一个action放到trasaction载体告诉system，要进行什么操作，这里是需要自己create一个UUID：

```
	CXHandle* handle=[[CXHandle alloc]initWithType:CXHandleTypePhoneNumber value:contact.phoneNumber];
    _callUUID = [NSUUID UUID];
    CXStartCallAction *action = [[CXStartCallAction alloc] initWithCallUUID:_callUUID handle: handle];
    action.contactIdentifier = [contact uniqueIdentifier];
    CXTransaction * transaction = [CXTransaction transactionWithActions:@[action]];
    [_callController requestTransaction:transaction completion:^( NSError *_Nullable error){
        if (error !=nil) {
            NSLog(@"Error requesting transaction: %@", error);
        }else{
            NSLog(@"Requested transaction successfully");
        }
    }];

```
一旦这个request发出去，系统会返回一个是否成功的结果，如果成功了的话，那么callObserver的代理机会响应。

3.当电话来的时候，需要让provider通知callUpdate这个载体告诉系统有电话来需要调用系统的来电功能和页面：

```
	NSString * number = contact.phoneNumber;
    CXHandle* handle=[[CXHandle alloc]initWithType:CXHandleTypePhoneNumber value:number];
    NSUUID *callUUID = [NSUUID UUID];	// 这部分可能是不需要的，来电应该是不需要自己生成的UUID
    _currentCallUUID=callUUID;
    
    CXCallUpdate *callUpdate = [[CXCallUpdate alloc] init];
    callUpdate.remoteHandle = handle;
    callUpdate.localizedCallerName = contact.displayName;
    [self.provider reportNewIncomingCallWithUUID:callUUID update:callUpdate completion:completion];
    [self.provider reportCallWithUUID:_currentCallUUID updated:callUpdate];
```

那么一旦告诉系统之后，系统机会通过之前设置好的provider代理告诉你现在到底什么情况，代理以及说明会在下面一一列出，请先不用着急！！请注意看注释，都标出来了：

```
	// 如果重拨是会调用此方法的,当然需要先暂停之前app内的所有音视频操作
	- (void)providerDidReset:(CXProvider *)provider
	
	// 接电话的才会调用这个方法
	- (void)provider:(CXProvider *)provider performAnswerCallAction:(nonnull CXAnswerCallAction *)action
	
	// 接通电话的情况下：无论是接听者还是呼叫者挂了电话都会调用这个方法(电话未接通的时候，被叫方挂电话会调用此方法，主叫方不会调用)
	- (void)provider:(CXProvider *)provider performEndCallAction:(nonnull CXEndCallAction *)action
	
	// 拨打电话的只要一旦呼出就会调用这个方法
	- (void)provider:(CXProvider *)provider performStartCallAction:(nonnull CXStartCallAction *)action
	
	// 这个点击了静音会调用此方法
	- (void)provider:(CXProvider *)provider performSetMutedCallAction:(nonnull CXSetMutedCallAction *)action
```

另外还有其他的代理可以自行查阅苹果的官方API，根据自己的需要加上就是。此外再加上一个代理回调的方法说明：

	// 通话状态发生变化会调起这个接口
	- (void)callObserver:(CXCallObserver *)callObserver callChanged:(CXCall *)call

<font color=#000000 size=3 face="微软雅黑">大家就可以根据自己的需要去使用这个代理了，很方便有没有！！</font>

## pushKit
callKit配合pushKit使用就是完全模仿来电的效果，在锁屏状态下也会调起callKit，使用也不难：

	1. 在appdelegate的方法里面先进行pushKit的注册，设置好代理，那么在	代理方法就会返回一个token，我们需要把这个token和我们已经生成的voip证	书一起给服务器，由服务器统一支配，把这些资料交给苹果服务器，那么苹果服	务器就会类似于push消息一样通知指定的设备了；
	2. 苹果服务器一旦把这个push推给指定的设备机会激活pushkit的另一个代理	方法，我们需要在这个代理方法里面去主动呼起系统的来电页面就可以了，另外	后台的这个推送服务也不难，php和java都是可以的，不仅限这两个语言，而且	推送的代码和原理和push消息是非常相似的，可参考push消息。

## Attention

<font color=#000000 size=4 face="黑体">另外iOS10已经警告VoIP功能的应用去使用<a>PushKit</a>来接受来电推送，以往的VoIP后台申请不受支持。</font>

CallKit只是一个调起系统来电的页面，提高VoIP的通话权限，并 <mark> __不具备VoIP__ </mark>， <mark> __不具备VoIP__ </mark>， <mark> __不具备VoIP__ </mark> 的功能。

无论打电话还是接电话，都必须要先对provider进行预先配置，这个设置配置可以放在APPDelegate里面，只执行一次即可。

## Additional & Diseases
目前暂时存在一个问题，就是在未接通电话话的状态下，caller或者callee主动去挂断，未能通知另一端。暂时是猜想是和服务端的推送有关系。另一个就是UUID的问题，我们已经知道caller这边outgoing是会主动生成一个UUID，但是callee是否需要自动生成，还存在疑惑，官方的的demo是只生成一次的UUID，用pushkit作为传输。造成前一个问题的关键很可能和这个有关。
