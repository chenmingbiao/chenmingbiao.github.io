---
layout:     post
title:      "iOS 多媒体－音效"
subtitle:   "ios-media-AudioToolbox"
date:       2017-03-23
header-img: "img/bg19.jpg"
author:     "CMB"
tags:
    - iOS
    - 音视频
    - AudioToolbox
    - media

---

## 介绍

iOS 中音频播放其实可以分为 `音效播放` 和 `音乐播放`，`音频播放` 一般指的是很短的音频，对于这类音频不需要进行进度、循环等控制。`音乐播放` 一般指的是较长的音频，通常是主音频，对于这些音频的播放通常需要进行精确的控制。对应的在 `iOS` 中播放这两类音频分别使用 `AudioToolbox` 和 `AVFoundation` 。

## 音效

`AudioToolbox` 是一套基于 `C语言` 的框架，使用它来播放音效其本质是将短音频注册到系统声音服务(System Sound Service)。
`System Sound Service` 是一种简单、底层的声音播放服务，但是它本身也存在着一些限制：

* 音频播放时间不能超过30s
* 数据必须是 `PCM` 或者 `IMA4` 格式
* 音频文件必须打包成.caf、.aif、.wav中的一种（注意这是官方文档的说法，实际测试发现一些.mp3也可以播放）

## 用法

使用 `System Sound Service` 播放音效的步骤如下：

1. 调用 `AudioServicesCreateSystemSoundID(CFURLRef  inFileURL, SystemSoundID* outSystemSoundID)` 函数获得系统声音ID。
2. 如果需要监听播放完成操作，则使用`AudioServicesAddSystemSoundCompletion(SystemSoundID inSystemSoundID,
CFRunLoopRef inRunLoop, CFStringRef inRunLoopMode, AudioServicesSystemSoundCompletionProc  inCompletionRoutine, void* inClientData)` 方法注册回调函数。
3. 调用 `AudioServicesPlaySystemSound(SystemSoundID inSystemSoundID)` 或者 `AudioServicesPlayAlertSound(SystemSoundID inSystemSoundID)` 方法播放音效（后者带有震动效果）。

```objc
- (void)playSoundEffect:(NSString *) name {
    NSString *audioFile = [[NSBundle mainBundle] pathForResource:name ofType:nil];
    NSURL *fileUrl = [NSURL fileURLWithPath:audioFile];
    // 1.获得系统声音ID
    SystemSoundID soundID = 0;
    /**
     * inFileUrl:音频文件url
     * outSystemSoundID:声音id（此函数会将音效文件加入到系统音频服务中并返回一个长整形ID）
     */
    AudioServicesCreateSystemSoundID((__bridge CFURLRef)(fileUrl), &soundID);
    // 如果需要在播放完之后执行某些操作，可以调用如下方法注册一个播放完成回调函数
    AudioServicesAddSystemSoundCompletion(soundID, NULL, NULL, soundCompleteCallback, NULL);
    // 2.播放音频
    AudioServicesPlaySystemSound(soundID);//播放音效
//    AudioServicesPlayAlertSound(soundID);//播放音效并震动
}
```

## 弊端

前文中已经提到利用 `AudioToolbox` 可以播发一些系统声音，那为什么我们大多情况下播放器都不是采用此方法来设计的呢？原因很简单，因为它存在很多的限制。

首先它不能播放一些较大的音频文件，比如时间太长的音频文件它就无能为力。其次对于具有复杂压缩编码方式的音频文件也无法播放。同时对于一些高级的播放器的功能的实现，System Sound Services也存在很多缺陷。比如不能循环播放、无法实时控制音频播放的各种参数信息。这种方法设计的音频播放器对于一些提示音以及震动等效果比较合适。

## 参考

* http://www.cnblogs.com/sunminmin/p/4475710.html