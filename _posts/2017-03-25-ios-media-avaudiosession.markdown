---
layout:     post
title:      "iOS 多媒体－音频会话"
subtitle:   "ios-media-avaudiosession"
date:       2017-03-25
header-img: "img/bg19.jpg"
author:     "CMB"
tags:
    - iOS
    - 音视频
    - AVAudioSession
    - media
    - AVFoundation

---

## 介绍

事实上上面的播放器还存在一些问题，例如通常我们看到的播放器即使退出到后台也是可以播放的，而这个播放器如果退出到后台它会自动暂停。如果要支持后台播放需要做下面几件事情：

1.设置后台运行模式：在 `plist` 文件中添加 `Required background modes` ，并且设置 `item 0=App plays audio or streams audio/video using AirPlay`（其实可以直接通过 Xcode 在 `Project Targets-Capabilities-Background Modes` 中设置）

![](https://camo.githubusercontent.com/0072dfffc0a67f95c644275d196bf7f9b9bb15a6/687474703a2f2f696d616765732e636e6974626c6f672e636f6d2f626c6f672f36323034362f3230313431322f3236303931333030343833383634332e706e67)

2.设置 `AVAudioSession` 的类型为 `AVAudioSessionCategoryPlayback` 并且调用 `setActive::` 方法启动会话。

```objc
 AVAudioSession *audioSession=[AVAudioSession sharedInstance];
    [audioSession setCategory:AVAudioSessionCategoryPlayback error:nil];
    [audioSession setActive:YES error:nil];
```

3.为了能够让应用退到后台之后支持耳机控制，建议添加远程控制事件（这一步不是后台播放必须的）

### 会话模式

前两步是后台播放所必须设置的，第三步主要用于接收远程事件，这部分内容之前的文章中有详细介绍，如果这一步不设置虽让也能够在后台播放，但是无法获得音频控制权（如果在使用当前应用之前使用其他播放器播放音乐的话，此时如果按耳机播放键或者控制中心的播放按钮则会播放前一个应用的音频），并且不能使用耳机进行音频控制。第一步操作相信大家都很容易理解，如果应用程序要允许运行到后台必须设置，正常情况下应用如果进入后台会被挂起，通过该设置可以上应用程序继续在后台运行。但是第二步使用的 `AVAudioSession` 有必要进行一下详细的说明。

在 `iOS` 中每个应用都有一个音频会话，这个会话就通过 `AVAudioSession` 来表示。 `AVAudioSession` 同样存在于 `AVFoundation` 框架中，它是单例模式设计，通过 `sharedInstance` 进行访问。在使用 `Apple` 设备时大家会发现有些应用只要打开其他音频播放就会终止，而有些应用却可以和其他应用同时播放，在多种音频环境中如何去控制播放的方式就是通过音频会话来完成的。

### AVAudioPlayer

会话类型 | 说明 | 是否要求输入	| 是否要求输出 | 是否遵从静音键
--------| ----|------------|-----------|---------
AVAudioSessionCategoryAmbient | 混音播放，可以与其他音频应用同时播放 | 否 | 是 | 是
AVAudioSessionCategorySoloAmbient	独占播放 | 否 | 是 | 是
AVAudioSessionCategoryPlayback | 后台播放，也是独占的 | 否 | 是 | 否
AVAudioSessionCategoryRecord | 录音模式，用于录音时使用 | 是 | 否	 | 否
AVAudioSessionCategoryPlayAndRecord | 播放和录音，此时可以录音也可以播放 | 是 | 是 | 否
AVAudioSessionCategoryAudioProcessing | 硬件解码音频，此时不能播放和录制 | 否 | 否 | 否
AVAudioSessionCategoryMultiRoute | 多种输入输出，例如可以耳机、USB设备同时播放 | 是 | 是 | 否

> 注意：是否遵循静音键表示在播放过程中如果用户通过硬件设置为静音是否能关闭声音。

### 实现

根据前面对音频会话的理解，相信大家开发出能够在后台播放的音频播放器并不难，但是注意一下，在前面的代码中也提到设置完音频会话类型之后需要调用 `setActive::` 方法将会话激活才能起作用。类似的，如果一个应用已经在播放音频，打开我们的应用之后设置了在后台播放的会话类型，此时其他应用的音频会停止而播放我们的音频，如果希望我们的程序音频播放完之后（关闭或退出到后台之后）能够继续播放其他应用的音频的话则可以调用 `setActive::` 方法关闭会话。代码如下：

```objc
#import <AVFoundation/AVFoundation.h>
#define kMusicFile @"刘若英 - 原来你也在这里.mp3"
#define kMusicSinger @"刘若英"
#define kMusicTitle @"原来你也在这里"

@interface ViewController ()<AVAudioPlayerDelegate>

@property (nonatomic,strong) AVAudioPlayer *audioPlayer;//播放器
@property (weak, nonatomic) IBOutlet UILabel *controlPanel; //控制面板
@property (weak, nonatomic) IBOutlet UIProgressView *playProgress;//播放进度
@property (weak, nonatomic) IBOutlet UILabel *musicSinger; //演唱者
@property (weak, nonatomic) IBOutlet UIButton *playOrPause; //播放/暂停按钮(如果tag为0认为是暂停状态，1是播放状态)

@property (weak ,nonatomic) NSTimer *timer;//进度更新定时器

@end

@implementation ViewController

- (void)viewDidLoad {
    [super viewDidLoad];

    [self setupUI];
    
}

/**
 *  显示当面视图控制器时注册远程事件
 *
 *  @param animated 是否以动画的形式显示
 */
-(void)viewWillAppear:(BOOL)animated{
    [super viewWillAppear:animated];
    //开启远程控制
    [[UIApplication sharedApplication] beginReceivingRemoteControlEvents];
    //作为第一响应者
    //[self becomeFirstResponder];
}
/**
 *  当前控制器视图不显示时取消远程控制
 *
 *  @param animated 是否以动画的形式消失
 */
-(void)viewWillDisappear:(BOOL)animated{
    [super viewWillDisappear:animated];
    [[UIApplication sharedApplication] endReceivingRemoteControlEvents];
    //[self resignFirstResponder];
}

/**
 *  初始化UI
 */
-(void)setupUI{
    self.title=kMusicTitle;
    self.musicSinger.text=kMusicSinger;
}

-(NSTimer *)timer{
    if (!_timer) {
        _timer=[NSTimer scheduledTimerWithTimeInterval:0.5 target:self selector:@selector(updateProgress) userInfo:nil repeats:true];
    }
    return _timer;
}

/**
 *  创建播放器
 *
 *  @return 音频播放器
 */
-(AVAudioPlayer *)audioPlayer{
    if (!_audioPlayer) {
        NSString *urlStr=[[NSBundle mainBundle]pathForResource:kMusicFile ofType:nil];
        NSURL *url=[NSURL fileURLWithPath:urlStr];
        NSError *error=nil;
        //初始化播放器，注意这里的Url参数只能时文件路径，不支持HTTP Url
        _audioPlayer=[[AVAudioPlayer alloc]initWithContentsOfURL:url error:&error];
        //设置播放器属性
        _audioPlayer.numberOfLoops=0;//设置为0不循环
        _audioPlayer.delegate=self;
        [_audioPlayer prepareToPlay];//加载音频文件到缓存
        if(error){
            NSLog(@"初始化播放器过程发生错误,错误信息:%@",error.localizedDescription);
            return nil;
        }
        //设置后台播放模式
        AVAudioSession *audioSession=[AVAudioSession sharedInstance];
        [audioSession setCategory:AVAudioSessionCategoryPlayback error:nil];
//        [audioSession setCategory:AVAudioSessionCategoryPlayback withOptions:AVAudioSessionCategoryOptionAllowBluetooth error:nil];
        [audioSession setActive:YES error:nil];
        //添加通知，拔出耳机后暂停播放
        [[NSNotificationCenter defaultCenter] addObserver:self selector:@selector(routeChange:) name:AVAudioSessionRouteChangeNotification object:nil];
    }
    return _audioPlayer;
}

/**
 *  播放音频
 */
-(void)play{
    if (![self.audioPlayer isPlaying]) {
        [self.audioPlayer play];
        self.timer.fireDate=[NSDate distantPast];//恢复定时器
    }
}

/**
 *  暂停播放
 */
-(void)pause{
    if ([self.audioPlayer isPlaying]) {
        [self.audioPlayer pause];
        self.timer.fireDate=[NSDate distantFuture];//暂停定时器，注意不能调用invalidate方法，此方法会取消，之后无法恢复
        
    }
}

/**
 *  点击播放/暂停按钮
 *
 *  @param sender 播放/暂停按钮
 */
- (IBAction)playClick:(UIButton *)sender {
    if(sender.tag){
        sender.tag=0;
        [sender setImage:[UIImage imageNamed:@"playing_btn_play_n"] forState:UIControlStateNormal];
        [sender setImage:[UIImage imageNamed:@"playing_btn_play_h"] forState:UIControlStateHighlighted];
        [self pause];
    }else{
        sender.tag=1;
        [sender setImage:[UIImage imageNamed:@"playing_btn_pause_n"] forState:UIControlStateNormal];
        [sender setImage:[UIImage imageNamed:@"playing_btn_pause_h"] forState:UIControlStateHighlighted];
        [self play];
    }
}

/**
 *  更新播放进度
 */
-(void)updateProgress{
    float progress= self.audioPlayer.currentTime /self.audioPlayer.duration;
    [self.playProgress setProgress:progress animated:true];
}

/**
 *  一旦输出改变则执行此方法
 *
 *  @param notification 输出改变通知对象
 */
-(void)routeChange:(NSNotification *)notification{
    NSDictionary *dic=notification.userInfo;
    int changeReason= [dic[AVAudioSessionRouteChangeReasonKey] intValue];
    //等于AVAudioSessionRouteChangeReasonOldDeviceUnavailable表示旧输出不可用
    if (changeReason==AVAudioSessionRouteChangeReasonOldDeviceUnavailable) {
        AVAudioSessionRouteDescription *routeDescription=dic[AVAudioSessionRouteChangePreviousRouteKey];
        AVAudioSessionPortDescription *portDescription= [routeDescription.outputs firstObject];
        //原设备为耳机则暂停
        if ([portDescription.portType isEqualToString:@"Headphones"]) {
            [self pause];
        }
    }
    
//    [dic enumerateKeysAndObjectsUsingBlock:^(id key, id obj, BOOL *stop) {
//        NSLog(@"%@:%@",key,obj);
//    }];
}

-(void)dealloc{
    [[NSNotificationCenter defaultCenter] removeObserver:self name:AVAudioSessionRouteChangeNotification object:nil];
}

#pragma mark - 播放器代理方法
-(void)audioPlayerDidFinishPlaying:(AVAudioPlayer *)player successfully:(BOOL)flag{
    NSLog(@"音乐播放完成...");
    //根据实际情况播放完成可以将会话关闭，其他音频应用继续播放
    [[AVAudioSession sharedInstance]setActive:NO error:nil];
}

@end
```

> 在上面的代码中还实现了拔出耳机暂停音乐播放的功能，这也是一个比较常见的功能。在 iOS7 及以后的版本中可以通过通知获得输出改变的通知，然后拿到通知对象后根据 userInfo 获得是何种改变类型，进而根据情况对音乐进行暂停操作。

### 扩展 - 播放音乐库中的音乐

众所周知音乐是 `iOS` 的重要组成播放，无论是 `iPod`、`iTouch`、`iPhone` 还是 `iPad` 都可以在 `iTunes` 购买音乐或添加本地音乐到音乐库中同步到你的 `iOS` 设备。在 `MediaPlayer` 中有一个 `MPMusicPlayerController` 用于播放音乐库中的音乐。

下面先来看一下 `MPMusicPlayerController` 的常用属性和方法：


属性 | 说明
----|----
@property (nonatomic, readonly) MPMusicPlaybackState playbackState	 | 播放器状态
@property (nonatomic) MPMusicRepeatMode repeatMode | 重复模式
@property (nonatomic) MPMusicShuffleMode shuffleMode | 随机播放模式
@property (nonatomic, copy) MPMediaItem *nowPlayingItem | 正在播放的音乐项
@property (nonatomic, readonly) NSUInteger indexOfNowPlayingItem | 当前正在播放的音乐在播放队列中的索引
@property(nonatomic, readonly) BOOL isPreparedToPlay | 是否准好播放准备
@property(nonatomic) NSTimeInterval currentPlaybackTime | 当前已播放时间，单位：秒
@property(nonatomic) float currentPlaybackRate | 当前播放速度，是一个播放速度倍率，0表示暂停播放，1代表正常速度

类方法 | 说明
----|----
+ (MPMusicPlayerController *)applicationMusicPlayer; | 获取应用播放器，注意此类播放器无法在后台播放
+ (MPMusicPlayerController *)systemMusicPlayer | 获取系统播放器，支持后台播放

对象方法 | 说明
----|----
- (void)setQueueWithQuery:(MPMediaQuery *)query | 使用媒体队列设置播放源媒体队列
- (void)setQueueWithItemCollection:(MPMediaItemCollection *)itemCollection | 使用媒体项集合设置播放源媒体队列
- (void)skipToNextItem | 下一曲
- (void)skipToBeginning	 | 从起始位置播放
- (void)skipToPreviousItem | 上一曲
- (void)beginGeneratingPlaybackNotifications | 开启播放通知，注意不同于其他播放器，MPMusicPlayerController要想获得通知必须首先开启，默认情况无法获得通知
- (void)endGeneratingPlaybackNotifications | 关闭播放通知
- (void)prepareToPlay | 做好播放准备（加载音频到缓冲区），在使用play方法播放时如果没有做好准备回自动调用该方法
- (void)play | 开始播放
- (void)pause | 暂停播放
- (void)stop | 停止播放
- (void)beginSeekingForward | 开始向前查找（快进）
- (void)beginSeekingBackward | 开始向后查找（快退）
- (void)endSeeking | 结束查找

通知 | 说明（注意：要想获得MPMusicPlayerController通知必须首先调用beginGeneratingPlaybackNotifications开启通知）
----|----
MPMusicPlayerControllerPlaybackStateDidChangeNotification | 播放状态改变
MPMusicPlayerControllerNowPlayingItemDidChangeNotification | 当前播放音频改变
MPMusicPlayerControllerVolumeDidChangeNotification | 声音大小改变
MPMediaPlaybackIsPreparedToPlayDidChangeNotification | 准备好播放


* `MPMusicPlayerController` 有两种播放器： `applicationMusicPlayer` 和 `systemMusicPlayer`，前者在应用退出后音乐播放会自动停止，后者在应用停止后不会退出播放状态。
* `MPMusicPlayerController` 加载音乐不同于前面的 `AVAudioPlayer` 是通过一个文件路径来加载，而是需要一个播放队列。在 `MPMusicPlayerController` 中提供了两个方法来加载播放队列： `- (void)setQueueWithQuery:(MPMediaQuery *)query`和 `- (void)setQueueWithItemCollection:(MPMediaItemCollection *)itemCollection` ，正是由于它的播放音频来源是一个队列，因此 `MPMusicPlayerController` 支持上一曲、下一曲等操作。

那么接下来的问题就是如何获取 `MPMediaQueue` 或者 `MPMediaItemCollection`？ 

`MPMediaQueue` 对象有一系列的类方法来获得媒体队列：

* `+ (MPMediaQuery *)albumsQuery;`
* `+ (MPMediaQuery *)artistsQuery;`
* `+ (MPMediaQuery *)songsQuery;`
* `+ (MPMediaQuery *)playlistsQuery;`
* `+ (MPMediaQuery *)podcastsQuery;`
* `+ (MPMediaQuery *)audiobooksQuery;`
* `+ (MPMediaQuery *)compilationsQuery;`
* `+ (MPMediaQuery *)composersQuery;`
* `+ (MPMediaQuery *)genresQuery;`

有了这些方法，就可以很容易获到歌曲、播放列表、专辑媒体等媒体队列了，这样就可以通过： `- (void)setQueueWithQuery:(MPMediaQuery *)query` 方法设置音乐来源了。又或者得到 `MPMediaQueue` 之后创建 `MPMediaItemCollection` ，使用- `(void)setQueueWithItemCollection:(MPMediaItemCollection *)itemCollection` 设置音乐来源。

有时候可能希望用户自己来选择要播放的音乐，这时可以使用 `MPMediaPickerController` ，它是一个视图控制器，类似于 `UIImagePickerController`，选择完播放来源后可以在其代理方法中获得MPMediaItemCollection对象。

无论是通过哪种方式获得 `MPMusicPlayerController` 的媒体源，可能都希望将每个媒体的信息显示出来，这时候可以通过MPMediaItem对象获得。一个 `MPMediaItem` 代表一个媒体文件，通过它可以访问媒体标题、专辑名称、专辑封面、音乐时长等等。无论是 `MPMediaQueue` 还是 `MPMediaItemCollection` 都有一个 `items` 属性，它是 `MPMediaItem` 数组，通过这个属性可以获得 `MPMediaItem` 对象。

下面就简单看一下 `MPMusicPlayerController` 的使用，在下面的例子中简单演示了音乐的选择、播放、暂停、通知、下一曲、上一曲功能，相信有了上面的概念，代码读起来并不复杂（示例中是直接通过 `MPMeidaPicker` 进行音乐选择的，但是仍然提供了两个方法 `getLocalMediaQuery` 和 `getLocalMediaItemCollection` 来演示如何直接通过 `MPMediaQueue` 获得媒体队列或媒体集合）：

```objc
#import <MediaPlayer/MediaPlayer.h>

@interface ViewController ()<MPMediaPickerControllerDelegate>

@property (nonatomic,strong) MPMediaPickerController *mediaPicker;//媒体选择控制器
@property (nonatomic,strong) MPMusicPlayerController *musicPlayer; //音乐播放器

@end

@implementation ViewController

- (void)viewDidLoad {
    [super viewDidLoad];
}

-(void)dealloc{
    [self.musicPlayer endGeneratingPlaybackNotifications];
}

/**
 *  获得音乐播放器
 *
 *  @return 音乐播放器
 */
-(MPMusicPlayerController *)musicPlayer{
    if (!_musicPlayer) {
        _musicPlayer=[MPMusicPlayerController systemMusicPlayer];
        [_musicPlayer beginGeneratingPlaybackNotifications];//开启通知，否则监控不到MPMusicPlayerController的通知
        [self addNotification];//添加通知
        //如果不使用MPMediaPickerController可以使用如下方法获得音乐库媒体队列
        //[_musicPlayer setQueueWithItemCollection:[self getLocalMediaItemCollection]];
    }
    return _musicPlayer;
}

/**
 *  创建媒体选择器
 *
 *  @return 媒体选择器
 */
-(MPMediaPickerController *)mediaPicker{
    if (!_mediaPicker) {
        //初始化媒体选择器，这里设置媒体类型为音乐，其实这里也可以选择视频、广播等
//        _mediaPicker=[[MPMediaPickerController alloc]initWithMediaTypes:MPMediaTypeMusic];
        _mediaPicker=[[MPMediaPickerController alloc]initWithMediaTypes:MPMediaTypeAny];
        _mediaPicker.allowsPickingMultipleItems=YES;//允许多选
//        _mediaPicker.showsCloudItems=YES;//显示icloud选项
        _mediaPicker.prompt=@"请选择要播放的音乐";
        _mediaPicker.delegate=self;//设置选择器代理
    }
    return _mediaPicker;
}

/**
 *  取得媒体队列
 *
 *  @return 媒体队列
 */
-(MPMediaQuery *)getLocalMediaQuery{
    MPMediaQuery *mediaQueue=[MPMediaQuery songsQuery];
    for (MPMediaItem *item in mediaQueue.items) {
        NSLog(@"标题：%@,%@",item.title,item.albumTitle);
    }
    return mediaQueue;
}

/**
 *  取得媒体集合
 *
 *  @return 媒体集合
 */
-(MPMediaItemCollection *)getLocalMediaItemCollection{
    MPMediaQuery *mediaQueue=[MPMediaQuery songsQuery];
    NSMutableArray *array=[NSMutableArray array];
    for (MPMediaItem *item in mediaQueue.items) {
        [array addObject:item];
        NSLog(@"标题：%@,%@",item.title,item.albumTitle);
    }
    MPMediaItemCollection *mediaItemCollection=[[MPMediaItemCollection alloc]initWithItems:[array copy]];
    return mediaItemCollection;
}

#pragma mark - MPMediaPickerController代理方法
//选择完成
-(void)mediaPicker:(MPMediaPickerController *)mediaPicker didPickMediaItems:(MPMediaItemCollection *)mediaItemCollection{
    MPMediaItem *mediaItem=[mediaItemCollection.items firstObject];//第一个播放音乐
    //注意很多音乐信息如标题、专辑、表演者、封面、时长等信息都可以通过MPMediaItem的valueForKey:方法得到,但是从iOS7开始都有对应的属性可以直接访问
//    NSString *title= [mediaItem valueForKey:MPMediaItemPropertyAlbumTitle];
//    NSString *artist= [mediaItem valueForKey:MPMediaItemPropertyAlbumArtist];
//    MPMediaItemArtwork *artwork= [mediaItem valueForKey:MPMediaItemPropertyArtwork];
    //UIImage *image=[artwork imageWithSize:CGSizeMake(100, 100)];//专辑图片
    NSLog(@"标题：%@,表演者：%@,专辑：%@",mediaItem.title ,mediaItem.artist,mediaItem.albumTitle);
    [self.musicPlayer setQueueWithItemCollection:mediaItemCollection];
    [self dismissViewControllerAnimated:YES completion:nil];
}
//取消选择
-(void)mediaPickerDidCancel:(MPMediaPickerController *)mediaPicker{
    [self dismissViewControllerAnimated:YES completion:nil];
}

#pragma mark - 通知
/**
 *  添加通知
 */
-(void)addNotification{
    NSNotificationCenter *notificationCenter=[NSNotificationCenter defaultCenter];
    [notificationCenter addObserver:self selector:@selector(playbackStateChange:) name:MPMusicPlayerControllerPlaybackStateDidChangeNotification object:self.musicPlayer];
}

/**
 *  播放状态改变通知
 *
 *  @param notification 通知对象
 */
-(void)playbackStateChange:(NSNotification *)notification{
    switch (self.musicPlayer.playbackState) {
        case MPMusicPlaybackStatePlaying:
            NSLog(@"正在播放...");
            break;
        case MPMusicPlaybackStatePaused:
            NSLog(@"播放暂停.");
            break;
        case MPMusicPlaybackStateStopped:
            NSLog(@"播放停止.");
            break;
        default:
            break;
    }
}

#pragma mark - UI事件
- (IBAction)selectClick:(UIButton *)sender {
    [self presentViewController:self.mediaPicker animated:YES completion:nil];
}

- (IBAction)playClick:(UIButton *)sender {
    [self.musicPlayer play];
}

- (IBAction)puaseClick:(UIButton *)sender {
    [self.musicPlayer pause];
}

- (IBAction)stopClick:(UIButton *)sender {
    [self.musicPlayer stop];
}

- (IBAction)nextClick:(UIButton *)sender {
    [self.musicPlayer skipToNextItem];
}

- (IBAction)prevClick:(UIButton *)sender {
    [self.musicPlayer skipToPreviousItem];
}

@end
```

## 参考

* [iOS音频播放 (二)：AudioSession](http://msching.github.io/blog/2014/07/08/audio-in-ios-2/)