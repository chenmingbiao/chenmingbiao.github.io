---
layout:     post
title:      "iOS 多媒体－音乐"
subtitle:   "ios-media-avaudioplayer"
date:       2017-03-24
header-img: "img/bg19.jpg"
author:     "CMB"
tags:
    - iOS
    - 音视频
    - AVAudioPlayer
    - media
    - AVFoundation

---

## 介绍

基于 `System Sound Services` 的弊端，如果我们想播放比较大的音频就需要用到 `AVFoundation` 来实现了，`AVFoundation` 中使用 `AVAudioPlayer` 来实现音频播放，`AVAudioPlayer` 可以当成是一个播放器，可用它来控制进度、音量、播放速度等参数。

### AVAudioPlayer

属性 | 说明
------------- | -------------
@property (readonly, getter=isPlaying) BOOL playing  | 是否正在播放，只读
@property (readonly) NSUInteger numberOfChannels  | 音频声道数，只读
@property (readonly) NSTimeInterval duration | 音频时长
@property (readonly) NSURL *url | 音频文件路径，只读
@property (readonly) NSData *data | 音频数据，只读
@property float pan | 立体声平衡，如果为-1.0则完全左声道，如果0.0则左右声道平衡，如果为1.0则完全为右声道
@property float volume | 音量大小，范围0-1.0
@property BOOL enableRate | 是否允许改变播放速率
@property float rate | 播放速率，范围0.5-2.0，如果为1.0则正常播放，如果要修改播放速率则必须设置enableRate为YES
@property NSTimeInterval currentTime | 当前播放时长
@property (readonly) NSTimeInterval deviceCurrentTime | 输出设备播放音频的时间，注意如果播放中被暂停此时间也会继续累加
@property NSInteger numberOfLoops | 	循环播放次数，如果为0则不循环，如果小于0则无限循环，大于0则表示循环次数
@property (readonly) NSDictionary *settings | 音频播放设置信息，只读
@property (getter=isMeteringEnabled) BOOL meteringEnabled | 是否启用音频测量，默认为NO，一旦启用音频测量可以通过updateMeters方法更新测量值 
@property(nonatomic, copy) NSArray *channelAssignments | 获得或设置播放声道

对象方法 | 说明
------------- | -------------
- (instancetype)initWithContentsOfURL:(NSURL *)url error:(NSError **)outError | 使用文件URL初始化播放器，注意这个URL不能是HTTP URL，AVAudioPlayer不支持加载网络媒体流，只能播放本地文件
- (instancetype)initWithData:(NSData *)data error:(NSError **)outError | 使用NSData初始化播放器，注意使用此方法时必须文件格式和文件后缀一致，否则出错，所以相比此方法更推荐使用上述方法或- (instancetype)initWithData:(NSData *)data fileTypeHint:(NSString *)utiString error:(NSError **)outError方法进行初始化
- (BOOL)prepareToPlay; | 加载音频文件到缓冲区，注意即使在播放之前音频文件没有加载到缓冲区程序也会隐式调用此方法。
- (BOOL)play; | 播放音频文件
- (BOOL)playAtTime:(NSTimeInterval)time | 在指定的时间开始播放音频
- (void)pause; | 暂停播放
- (void)stop; | 停止播放
- (void)updateMeters | 更新音频测量值，注意如果要更新音频测量值必须设置meteringEnabled为YES，通过音频测量值可以即时获得音频分贝等信息
- (float)peakPowerForChannel:(NSUInteger)channelNumber; | 获得指定声道的分贝峰值，注意如果要获得分贝峰值必须在此之前调用updateMeters方法
- (float)averagePowerForChannel:(NSUInteger)channelNumber | 	获得指定声道的分贝平均值，注意如果要获得分贝平均值必须在此之前调用updateMeters方法

代理方法 | 说明
------------- | -------------
- (void)audioPlayerDidFinishPlaying:(AVAudioPlayer *)player successfully:(BOOL)flag | 音频播放完成
- (void)audioPlayerDecodeErrorDidOccur:(AVAudioPlayer *)player error:(NSError *)error | 音频解码发生错误

## 使用步骤

* 初始化 `AVAudioPlayer` 对象，此时通常指定本地文件路径
* 设置播放器属性，例如重复次数、音量大小等
* 调用 `play` 方法播放

## 用法

当然由于 `AVAudioPlayer` 一次只能播放一个音频文件，所有上一曲、下一曲其实可以通过创建多个播放器对象来完成，这里暂不实现。播放进度的实现主要依靠一个定时器实时计算当前播放时长和音频总时长的比例，另外为了演示委托方法，下面的代码中也实现了播放完成委托方法，通常如果有下一曲功能的话播放完可以触发下一曲音乐播放。下面是主要代码：

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

#pragma mark - 播放器代理方法
-(void)audioPlayerDidFinishPlaying:(AVAudioPlayer *)player successfully:(BOOL)flag{
    NSLog(@"音乐播放完成...");
}

@end
```

## 参考

* [Tutorial: Playing Audio with AVAudioPlayer](http://iosdeveloperzone.com/2012/10/01/tutorial-playing-audio-with-avaudioplayer/)