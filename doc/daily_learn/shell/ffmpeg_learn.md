[TOC]

# ffmpeg 基本用法
官方教程请查看[^0]，基本指令参考[^4]
```shell {.line-numbers}
ffmpeg [global_options] {[input_file_options] -i input_url} ... {[output_file_options] output_url} ...

ffmpeg -i [输入文件名] [参数选项] -f [格式] [输出文件] 

参数选项： 
(1) -an: 去掉音频 
(2) -vn: 去掉视频 
(3) -acodec: 设定音频的编码器，未设定时则使用与输入流相同的编解码器。
音频解复用一般后面加copy表示拷贝 
(4) -vcodec: 设定视频的编码器，未设定时则使用与输入流相同的编解码器。
视频解复用一般后面加copy表示拷贝 
(5) –f: 输出格式（视频转码）
(6) -bf: B帧数目控制 
(7) -g: 关键帧间隔控制(视频跳转需要关键帧)
(8) -s: 设定画面的宽和高，分辨率控制(352*278)
(9) -i:  设定输入流
(10) -ss: 指定开始时间（0:0:05）
(11) -t: 指定持续时间（0:05）
(12) -b: 设定视频流量，默认是200Kbit/s
(13) -aspect: 设定画面的比例
(14) -ar: 设定音频采样率
(15) -ac: 设定声音的Channel数
(16)  -r: 提取图像频率（用于视频截图）
(17) -c:v:  输出视频格式
(18) -c:a:  输出音频格式
(18) -y:  输出时覆盖输出目录已存在的同名文件
(19) -map 指定流（0:0）
# 不常见指令
(20) -nr 去噪

# 每隔20秒截一张图
ffmpeg -i out.mp4 -f image2 -vf fps=fps=1/20 out%d.png

# 多张截图
ffmpeg -i out.mp4 -frames 3 -vf \
"select=not(mod(n\,1000)),scale=320:240,tile=2x3" out.png

# 视频转码
ffmpeg -i test.mp4 -vcodec h264 -bf 0 -g 25 -s 352*278 -an -f m4v test.264

# 视频封装
ffmpeg -i video_file -i audio_file -vcodec copy -acodec copy output_file

# 视频录制
ffmpeg -i rtsp://hostname/test -vcodec copy out.avi

# 内容反转
ffmpeg -i input-file.mp4 -vf reverse -af areverse output.mp4
```
---

# 对齐关键帧
`-g`可以指定I帧间隔，但是ffmpeg会根据视频内容，在设置值的基础上进行调整。这会导致I帧间隔与设定不符[^1]。

```shell
key_frame_interval = 48
ffmpeg -i input.mp4 -s 1920x1080 -c:v libx246 -keyint_min ${key_frame_interval} -g \
        ${key_frame_interval} -sc_threshold 0 -an output.mp4
```
* keyint_min 48：keyint表示关键帧（IDR帧）间隔，这个选项表示限制IDR帧间隔最小为48帧，与之前设置的GOP等长
* sc_threshold 0：禁用场景识别，即禁止自动添加IDR帧

IDR帧，I帧概念请参考[^2]，GOP概念请参考[^3]。目测ffmpeg这种自动添加某些内容的手法比较普遍[^8]，需要多注意。


# 视频帧控制
该部分主要参考[^6]
```shell {.line-numbers}
# 获取视频图像数量
file_src=$(ls ./test_data*/*.mp4)
frame_count=$(ffprobe -v error -select_streams v:0 -count_packets -show_entries \
            stream=nb_read_packets -of csv=p=0 ${file_src})
i_frame_count=$(ffprobe -v quiet -show_frames ${file_src} | grep "pict_type=I" | wc -l)

# 获取视频帧率
frame_rate=$(ffprobe -v error -select_streams v:0 -show_entries stream=r_frame_rate \
            -of csv=p=0 ${file_src})

# 从视频保存图像
save_image_reg=test_img_%04d.jpg
ffmpeg  -threads 4 -y -i ${file_src} -vf select='eq(pict_type\,I)' -ss 00:00:00 \
        -frames:v ${frame_count} -f image2 ${save_image_reg}

# 从图像合成视频
file_dst=test.mp4
ffmpeg  -threads 4 -y -r ${frame_rate} -start_number 1 -i ${save_image_reg} -c:v \
 libx264 -vf "fps=10,format=yuv420p" ${file_dst} 2>&1

# 从视频拷贝视频
ffmpeg -ss 00:00:00.012 -t 38.1 -i ${file_src}  -map 0 -codec copy ${file_dst}

# 指定码率/I帧间隔
frame_rate_dst=2000k
iframe_interval=60
ffmpeg -threads 4 -y -r 25 -i ${save_image_reg} -ss 00:00:00 -c:v libx264 \
	-maxrate ${frame_rate_dst} -minrate ${frame_rate_dst} -bf 1 -b_strategy 0 \
	-keyint_min ${iframe_interval} -g ${iframe_interval} -sc_threshold 0 \
    -vf "fps=10,format=yuv420p" ${file_dst} 2>&1
```
如何设置参数得到高质量的H264编码请参考[^7]


# 旋转
该部分主要参考[^5]
```shell {.line-numbers}
# 视频+音频倒放
ffmpeg -i test.mp4 -vf reverse -af areverse out.mp4
# 顺时针旋转画面90度
ffmpeg -i test.mp4 -vf "transpose=1" out.mp4
# 逆时针旋转画面90度
ffmpeg -i test.mp4 -vf "transpose=2" out.mp4 
# 顺时针旋转画面90度再水平翻转
ffmpeg -i test.mp4 -vf "transpose=3" out.mp4 
# 逆时针旋转画面90度水平翻转
ffmpeg -i test.mp4 -vf "transpose=0" out.mp4 
# 水平翻转视频画面
ffmpeg -i test.mp4 -vf hflip out.mp4 
# 垂直翻转视频画面
ffmpeg -i test.mp4 -vf vflip out.mp4
# 其他旋转，暂时觉得不安全，不建议用
ffmpeg -i input.mp4 -metadata:s:v rotate="90" -codec copy test_output.mp4
```


# 视频加减速
该部分主要参考[^9]，对音频或水印感兴趣也可参考该链接。
```shell
ffmpeg -i input.mp4 -an -vf "setpts=5.0*PTS" test_speed_down.mp4
ffmpeg -i input.mp4 -an -vf "setpts=0.2*PTS" test_speed_up.mp4
ffmpeg -i input.mp4 -filter_complex [0:v]setpts=%.2f*PTS[v];[0:a]\
atempo=%.2f[a] -map [v] -map [a] output.mp4
```

---
以下示例均去掉了音频：

# 视频去重
```shell
# 这将生成一个控制台读数，显示过滤器认为哪些帧重复
ffmpeg -i input.mp4 -an -vf mpdecimate -loglevel debug -f null -
# 要生成删除了重复项的视频
ffmpeg -i input.mp4 -an -vf mpdecimate,setpts=N/FRAME_RATE/TB out.mp4
```

# 视频多段截取
```shell
# 截取start-5 1--15 20-end
ffmpeg -i 4.mp4 -an -filter_complex \
"[0:v]trim=duration=5[a]; \
 [0:v]trim=start=10:end=15,setpts=PTS-STARTPTS[b]; \
 [a][b]concat[c]; \
 [0:v]trim=start=20,setpts=PTS-STARTPTS[d]; \
 [c][d]concat[out1]" -map [out1] out.mp4
```

# 视频压缩
```shell
#!/bin/bash
# 1:输入;2:码率;3.分辨率;4.纵横比;5.输出

set -e
echo y|./ffmpeg -i $1 -vcodec libx264 -vprofile high -preset slow -b:v $2 \
-maxrate $2 -bufsize 4000k -vf scale=$3 -aspect $4 -threads 0 -pass 1 -an \
-f mp4 -y /dev/null
echo -----
echo y|./ffmpeg -i $1 -vcodec libx264 -vprofile high -preset slow -b:v $2 \
 -maxrate $2 -bufsize 4000k -vf scale=$3 -aspect $4 -threads 0 -pass 2  $5 \
 -f mp4
set +e

bash ./transform.sh "stream.avi" "1024k" "-1:720" "16:9" output.mp4 
```

---

# 参考
[^0]: [ffmpeg官方手册](http://ffmpeg.org/ffmpeg-all.html)
[^1]: [FFmpeg的GOP（I帧）对齐问题](https://blog.csdn.net/LvGreat/article/details/103540007)
[^2]: [I帧 B帧 p帧 IDR帧的区别](https://blog.csdn.net/sphone89/article/details/8086071)
[^3]: [I帧和IDR帧](https://blog.csdn.net/qq_32245927/article/details/80029949?utm_medium=distribute.pc_aggpage_search_result.none-task-blog-2)
[^4]: [FFmpeg常用命令](https://www.jianshu.com/p/91727ab25227)
[^5]: [用ffmpeg将一段mp4视频旋转90度](https://www.zhihu.com/question/20207331/answer/480910560)
[^6]: [使用ffmpeg编码时，如何设置恒定码率，并控制好关键帧I帧间隔](https://www.cnblogs.com/blackhumour2018/p/9427665.html)
[^7]: [Video Encoding Settings for H.264 Excellence](https://www.lighterra.com/papers/videoencodingh264/)
[^8]: [ffmpeg中时间戳调整参数setpts](http://ericliu.cn/2018/08/15/ffmpeg-setpts/)
[^9]: [FFmpeg_command_line](https://github.com/xufuji456/FFmpegAndroid/blob/master/doc/FFmpeg_command_line.md)
