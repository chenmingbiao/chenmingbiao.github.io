---
layout:     post
title:      "iOS 多媒体－音频队列"
subtitle:   "ios-media-audioqueueservices"
date:       2017-03-29
header-img: "img/bg19.jpg"
author:     "CMB"
tags:
    - iOS
    - 音视频
    - Audio Queue Services
    - media
    - AVFoundation

---

### 介绍

要在 `iOS` 设备上播放和录制音频，苹果推荐我们使用 `AVFoundation` 框架中的 `AVAudioPlayer` 和 `AVAudioRecorder` 类。虽然用法比较简单，但是不支持流式；这就意味着：在播放音频前，必须等到整个音频加载完成后，才能开始播放音频；录音时，也必须等到录音结束后，才能获取到录音数据。这给应用造成了很大的局限性。为了解决这个问题，我们就需要使用 `Audio Queue Services` 来播放和录制音频；为了简化音频文件的处理，这里还需要用到 `Audio File Services` 。

### 工作原理

#### 输入

1. 将音频填入第一个缓冲器中；
2. 当队列中的第一个缓冲器填满时，会自动填充下一个缓冲器。此时，会触发回调；
3. 在回调函数中需要将音频数据流写入磁盘；
4. 然后，需要在回调函数中将该缓冲器重新放入缓冲队列，以便重复使用该缓冲器。重复步骤2。

![](https://camo.githubusercontent.com/432028d661f76159917a67c536ae9448e2306333/687474703a2f2f696d616765732e636e6974626c6f672e636f6d2f626c6f672f36323034362f3230313431322f3236303931333038333237373738312e706e67)

#### 输出

1. 将音频读入到缓存器中。一旦填充满一个缓存器，就会进入缓存队列，此时处于待命状态；
2. 应用程序命令发出指令，要求音频队列开始播放；
3. 音频会从第一个缓存器中取数据，并开始播放；
4. 一旦播放完成，就会触发回调，并开始播放下一个缓存器中的内容；
5. 回调中需要给该缓存器取后面的音频数据，然后重新放入缓存队列中。重复步骤3。

![](https://camo.githubusercontent.com/03a30767c150d54ced0732ab9a1dd69e3e4b7a1a/687474703a2f2f696d616765732e636e6974626c6f672e636f6d2f626c6f672f36323034362f3230313431322f3236303931333132303330333635302e706e67)

> 当然，要明白音频队列服务的原理并不难，问题是如何实现这个自定义的回调函数，这其中我们有大量的工作要做，控制播放状态、处理异常中断、进行音频编码等等。(由于自己实现做的东西还真不少，所以推荐第三方框架： `AudioStreamer`、`FreeStreamer`。)

### 参考

* [Audio Queue Services Programming Guide](https://developer.apple.com/library/content/documentation/MusicAudio/Conceptual/AudioQueueProgrammingGuide/Introduction/Introduction.html#//apple_ref/doc/uid/TP40005343)
* [Audio Queue Services Reference](https://developer.apple.com/reference/audiotoolbox/audio_queue_services#//apple_ref/doc/uid/TP40005117)