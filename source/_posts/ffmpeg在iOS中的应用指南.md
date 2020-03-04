---
title: ffmpeg与ijkplayer在iOS中的应用指南
date: 2019-10-16 10:39:28
tags:
    - ffmpeg
    - ijkplayer
    - 音视频开发
categories:
    - ffmpeg
    - ijkplayer
    - 音视频开发
---

# 1 ffmpeg与ijkplayer的介绍
## 1.1 ffmpeg
ffmpeg 理解成一套音视频解决方案，使用 C语言 开发的开源程序，并且免费、开源、跨平台，它提供了录制、转换以及流化音视频，编码，特效，视音频操作等功能，包含了非常先进的音频/视频编解码库. 

可以实现播放歌曲、视频, 甚至通过命令实现对 视频文件的转码、混合、剪辑, 采集等各
种复杂处理.

## 1.2 ijkplayer
ijkplayer是业内很知名的音视频播放框架, 由B站开发并在GitHub开源, 底层使用ffmpeg来进行音视频的编解码处理. 市面上第三方的播放器的播放内核均使用的ijkplayer.

# 2 ffmpeg的使用

