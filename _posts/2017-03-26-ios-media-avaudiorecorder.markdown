---
layout:     post
title:      "iOS 多媒体－录音"
subtitle:   "ios-media-avaudiorecorder"
date:       2017-03-26
header-img: "img/bg19.jpg"
author:     "CMB"
tags:
    - iOS
    - 音视频
    - AVAudioRecorder
    - media
    - AVFoundation

---

### 介绍

`AVFoundation` 中使用 `AVAudioPlayer` 实现音频播放功能是非常简单的。同样在 `AVAudioRecorder` 实现音频录制功能也是非常简单的。`AVAudioRecorder` 同其用于播放音频一样，构建于 `Audio Queue Services` 之上，是一个功能强大且代码简单易用的 `Objective-C` 接口。我们可以在 `Mac` 机器和 `iOS` 设备上使用这个类来从内置的麦克风录制音频，也可从外部音频设备进行录制，比如数字音频接口或USB麦克风等。

### AVAudioRecorder

`AVAudioRecorder` 支持多种音频格式。与AVAudioPlayer类似，你完全可以将它看成是一个录音机控制类，下面是常用的属性和方法：

属性 | 说明 
-----|-----
@property(readonly, getter=isRecording) BOOL recording; | 是否正在录音，只读
@property(readonly) NSURL *url | 录音文件地址，只读
@property(readonly) NSDictionary *settings | 录音文件设置，只读
@property(readonly) NSTimeInterval currentTime | 录音时长，只读，注意仅仅在录音状态可用
@property(readonly) NSTimeInterval deviceCurrentTime | 输入设置的时间长度，只读，注意此属性一直可访问
@property(getter=isMeteringEnabled) BOOL meteringEnabled; | 是否启用录音测量，如果启用录音测量可以获得录音分贝等数据信息
@property(nonatomic, copy) NSArray *channelAssignments | 当前录音的通道

对象方法 | 说明 
-----|-----
- (instancetype)initWithURL:(NSURL *)url settings:(NSDictionary *)settings error:(NSError **)outError | 录音机对象初始化方法，注意其中的url必须是本地文件url，settings是录音格式、编码等设置
- (BOOL)prepareToRecord	 | 准备录音，主要用于创建缓冲区，如果不手动调用，在调用record录音时也会自动调用
- (BOOL)record | 开始录音
- (BOOL)recordAtTime:(NSTimeInterval)time | 在指定的时间开始录音，一般用于录音暂停再恢复录音
- (BOOL)recordForDuration:(NSTimeInterval) duration | 按指定的时长开始录音
- (BOOL)recordAtTime:(NSTimeInterval)time forDuration:(NSTimeInterval) duration | 在指定的时间开始录音，并指定录音时长
- (void)pause; | 暂停录音
- (void)stop; | 停止录音
- (BOOL)deleteRecording; | 删除录音，注意要删除录音此时录音机必须处于停止状态
- (void)updateMeters; | 更新测量数据，注意只有meteringEnabled为YES此方法才可用
- (float)peakPowerForChannel:(NSUInteger)channelNumber; | 指定通道的测量峰值，注意只有调用完updateMeters才有值
- (float)averagePowerForChannel:(NSUInteger)channelNumber | 指定通道的测量平均值，注意只有调用完updateMeters才有值

代理方法 | 说明
-----|-----
- (void)audioRecorderDidFinishRecording:(AVAudioRecorder *)recorder successfully:(BOOL)flag | 完成录音
- (void)audioRecorderEncodeErrorDidOccur:(AVAudioRecorder *)recorder error:(NSError *)error | 录音编码发生错误

`AVAudioRecorder` 很多属性和方法跟 `AVAudioPlayer` 都是类似的,但是它的创建有所不同，在创建录音机时除了指定路径外还必须指定录音设置信息，因为录音机必须知道录音文件的格式、采样率、通道数、每个采样点的位数等信息，但是也并不是所有的信息都必须设置，通常只需要几个常用设置。

> 关于录音设置详见帮助文档中的 [AV Foundation Audio Settings Constants](https://developer.apple.com/reference/avfoundation/audio_systems/av_foundation_audio_settings_constants)。

### 实现

在这个示例中将实行一个完整的录音控制，包括录音、暂停、恢复、停止，同时还会实时展示用户录音的声音波动，当用户点击完停止按钮还会自动播放录音文件。程序的构建主要分为以下几步：

* 设置音频会话类型为 `AVAudioSessionCategoryPlayAndRecord` ，因为程序中牵扯到录音和播放操作。
* 创建录音机 `AVAudioRecorder` ，指定录音保存的路径并且设置录音属性，注意对于一般的录音文件要求的采样率、位数并不高，需要适当设置以保证录音文件的大小和效果。
* 设置录音机代理以便在录音完成后播放录音，打开录音测量保证能够实时获得录音时的声音强度。（注意声音强度范围 `-160` 到 `0` ， `0`代表最大输入）
* 创建音频播放器 `AVAudioPlayer` ，用于在录音完成之后播放录音。
创建一个定时器以便实时刷新录音测量值并更新录音强度到 `UIProgressView` 中显示。
* 添加录音、暂停、恢复、停止操作，需要注意录音的恢复操作其实是有音频会话管理的，恢复时只要再次调用 `record` 方法即可，无需手动管理恢复时间等。

```objc
#import <AVFoundation/AVFoundation.h>
#define kRecordAudioFile @"myRecord.caf"

@interface ViewController ()<AVAudioRecorderDelegate>

@property (nonatomic,strong) AVAudioRecorder *audioRecorder;//音频录音机
@property (nonatomic,strong) AVAudioPlayer *audioPlayer;//音频播放器，用于播放录音文件
@property (nonatomic,strong) NSTimer *timer;//录音声波监控（注意这里暂时不对播放进行监控）

@property (weak, nonatomic) IBOutlet UIButton *record;//开始录音
@property (weak, nonatomic) IBOutlet UIButton *pause;//暂停录音
@property (weak, nonatomic) IBOutlet UIButton *resume;//恢复录音
@property (weak, nonatomic) IBOutlet UIButton *stop;//停止录音
@property (weak, nonatomic) IBOutlet UIProgressView *audioPower;//音频波动

@end

@implementation ViewController

#pragma mark - 控制器视图方法
- (void)viewDidLoad {
    [super viewDidLoad];
    
    [self setAudioSession];
}

#pragma mark - 私有方法
/**
 *  设置音频会话
 */
-(void)setAudioSession{
    AVAudioSession *audioSession=[AVAudioSession sharedInstance];
    //设置为播放和录音状态，以便可以在录制完之后播放录音
    [audioSession setCategory:AVAudioSessionCategoryPlayAndRecord error:nil];
    [audioSession setActive:YES error:nil];
}

/**
 *  取得录音文件保存路径
 *
 *  @return 录音文件路径
 */
-(NSURL *)getSavePath{
    NSString *urlStr=[NSSearchPathForDirectoriesInDomains(NSDocumentDirectory, NSUserDomainMask, YES) lastObject];
    urlStr=[urlStr stringByAppendingPathComponent:kRecordAudioFile];
    NSLog(@"file path:%@",urlStr);
    NSURL *url=[NSURL fileURLWithPath:urlStr];
    return url;
}

/**
 *  取得录音文件设置
 *
 *  @return 录音设置
 */
-(NSDictionary *)getAudioSetting{
    NSMutableDictionary *dicM=[NSMutableDictionary dictionary];
    //设置录音格式
    [dicM setObject:@(kAudioFormatLinearPCM) forKey:AVFormatIDKey];
    //设置录音采样率，8000是电话采样率，对于一般录音已经够了
    [dicM setObject:@(8000) forKey:AVSampleRateKey];
    //设置通道,这里采用单声道
    [dicM setObject:@(1) forKey:AVNumberOfChannelsKey];
    //每个采样点位数,分为8、16、24、32
    [dicM setObject:@(8) forKey:AVLinearPCMBitDepthKey];
    //是否使用浮点数采样
    [dicM setObject:@(YES) forKey:AVLinearPCMIsFloatKey];
    //....其他设置等
    return dicM;
}

/**
 *  获得录音机对象
 *
 *  @return 录音机对象
 */
-(AVAudioRecorder *)audioRecorder{
    if (!_audioRecorder) {
        //创建录音文件保存路径
        NSURL *url=[self getSavePath];
        //创建录音格式设置
        NSDictionary *setting=[self getAudioSetting];
        //创建录音机
        NSError *error=nil;
        _audioRecorder=[[AVAudioRecorder alloc]initWithURL:url settings:setting error:&error];
        _audioRecorder.delegate=self;
        _audioRecorder.meteringEnabled=YES;//如果要监控声波则必须设置为YES
        if (error) {
            NSLog(@"创建录音机对象时发生错误，错误信息：%@",error.localizedDescription);
            return nil;
        }
    }
    return _audioRecorder;
}

/**
 *  创建播放器
 *
 *  @return 播放器
 */
-(AVAudioPlayer *)audioPlayer{
    if (!_audioPlayer) {
        NSURL *url=[self getSavePath];
        NSError *error=nil;
        _audioPlayer=[[AVAudioPlayer alloc]initWithContentsOfURL:url error:&error];
        _audioPlayer.numberOfLoops=0;
        [_audioPlayer prepareToPlay];
        if (error) {
            NSLog(@"创建播放器过程中发生错误，错误信息：%@",error.localizedDescription);
            return nil;
        }
    }
    return _audioPlayer;
}

/**
 *  录音声波监控定制器
 *
 *  @return 定时器
 */
-(NSTimer *)timer{
    if (!_timer) {
        _timer=[NSTimer scheduledTimerWithTimeInterval:0.1f target:self selector:@selector(audioPowerChange) userInfo:nil repeats:YES];
    }
    return _timer;
}

/**
 *  录音声波状态设置
 */
-(void)audioPowerChange{
    [self.audioRecorder updateMeters];//更新测量值
    float power= [self.audioRecorder averagePowerForChannel:0];//取得第一个通道的音频，注意音频强度范围时-160到0
    CGFloat progress=(1.0/160.0)*(power+160.0);
    [self.audioPower setProgress:progress];
}
#pragma mark - UI事件
/**
 *  点击录音按钮
 *
 *  @param sender 录音按钮
 */
- (IBAction)recordClick:(UIButton *)sender {
    if (![self.audioRecorder isRecording]) {
        [self.audioRecorder record];//首次使用应用时如果调用record方法会询问用户是否允许使用麦克风
        self.timer.fireDate=[NSDate distantPast];
    }
}

/**
 *  点击暂定按钮
 *
 *  @param sender 暂停按钮
 */
- (IBAction)pauseClick:(UIButton *)sender {
    if ([self.audioRecorder isRecording]) {
        [self.audioRecorder pause];
        self.timer.fireDate=[NSDate distantFuture];
    }
}

/**
 *  点击恢复按钮
 *  恢复录音只需要再次调用record，AVAudioSession会帮助你记录上次录音位置并追加录音
 *
 *  @param sender 恢复按钮
 */
- (IBAction)resumeClick:(UIButton *)sender {
    [self recordClick:sender];
}

/**
 *  点击停止按钮
 *
 *  @param sender 停止按钮
 */
- (IBAction)stopClick:(UIButton *)sender {
    [self.audioRecorder stop];
    self.timer.fireDate=[NSDate distantFuture];
    self.audioPower.progress=0.0;
}

#pragma mark - 录音机代理方法
/**
 *  录音完成，录音完成后播放录音
 *
 *  @param recorder 录音机对象
 *  @param flag     是否成功
 */
-(void)audioRecorderDidFinishRecording:(AVAudioRecorder *)recorder successfully:(BOOL)flag{
    if (![self.audioPlayer isPlaying]) {
        [self.audioPlayer play];
    }
    NSLog(@"录音完成!");
}

@end
```

### 参考

* [iOS Programming 101: Record and Play Audio using AVFoundation Framework](https://www.appcoda.com/ios-avfoundation-framework-tutorial/)