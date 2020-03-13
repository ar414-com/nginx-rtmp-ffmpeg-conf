### :two_men_holding_hands:前言
> 最近帮朋友的公司部署了一套分流+水印的直播系统
>
> 顺手打包成docker镜像，方便大家需要用到的时候开箱即用，不需要百度一些零碎的文章
> 也可做简单的直播服务，只需调整配置文件便可达到你的需求.
>
> 需求：将直播流分流到两个云厂商的直播云，一个有水印，一个无水印。使用hls播放
>
> 简单的拓扑示意图：
>
> ![ar414-nginx-rtmp](https://cdn.ar414.com/ar414-nginx-rtmp.png)

# :whale:[docker-nginx-rtmp-ffmpeg](https://hub.docker.com/repository/docker/ar414/nginx-rtmp-ffmpeg)
基于[docker-nginx-rtmp](https://github.com/alfg/docker-nginx-rtmp)进行配置部署，这篇文章的意义是实现直播分流及直播画面水印.
* Nginx 1.16.1（从源代码编译的稳定版本）
* nginx-rtmp-module 1.2.1（从源代码编译）
* ffmpeg 4.2.1（从源代码编译）
* 已配置好的[nginx.conf](https://github.com/ar414-com/nginx-rtmp-ffmpeg-conf/blob/master/nginx.conf)
    * 只支持1920*1080（如需支持其他分辨率可参考[nginx.conf](https://github.com/alfg/docker-nginx-rtmp/blob/master/nginx.conf)）
    * 实现两路分流
        * 本机
        * 直播云（例：阿里云、腾讯云、ucloud）
    * 实现直播水印效果
        * 水印图片存放位置（容器内）：/opt/images/logo.png

## :computer:部署运行
### 服务器
* 安装docker(Centos7,其他系统请发挥你的搜索功能)
```bash
$ yum -y install docker #安装docker
$ systemctl enable docker #配置开机启动
$ systemctl start docker #启动docker服务
```
* 拉取docker镜像并运行
```bash
#如果速度慢可使用阿里云：docker pull registry.cn-shenzhen.aliyuncs.com/ar414/nginx-rtmp-ffmpeg:v1
$ docker pull ar414/nginx-rtmp-ffmpeg
$ docker run -it -d -p 1935:1935 -p 8080:80 --rm ar414/nginx-rtmp-ffmpeg
```
* 推流地址（Stream live content to）：
```
rtmp://<server ip>:1935/stream/$STREAM_NAME
```
* SSL证书

将证书复制到容器内，并在容器内修改nginx.conf配置文件，然后重新commit（操作容器内的文件都需要重新commit才会生效）
```
#/etc/nginx/nginx.conf
listen 443 ssl;
ssl_certificate     /opt/certs/example.com.crt;
ssl_certificate_key /opt/certs/example.com.key;
```

### OBS配置
* Stream Type: `Custom Streaming Server`
* URL: `rtmp://<server ip>:1935/stream`
* Stream Key：ar414
![obs-config](https://cdn.ar414.com/obs-config.png)

### 观看测试
> HLS播放测试工具：http://player.alicdn.com/aliplayer/setting/setting.html （如果配置了证书则使用https）

* HLS播放地址
    * 有水印：http://\<server ip>:8080/hls/ar414_wm.m3u8
    ![ar414-hls-wm](https://cdn.ar414.com/ar414-hls-wm.png)
    * 无水印：http://\<server ip>:8080/hls/ar414.m3u8
    ![ar414-hls](https://cdn.ar414.com/ar414-hls.png)
    
> RTMP测试工具：[PotPlayer](https://daumpotplayer.com/download/)

* RTMP播放地址
    * 无水印：rtmp://\<server ip>:1935/stream/ar414
    ![ar414-rtmp](https://cdn.ar414.com/ar414-rtmp.png)
    * 有水印：需要分流到其他服务器上
    
## :page_facing_up:配置文件简解（分流、水印及水印位置）
> [完整配置文件](https://github.com/ar414-com/nginx-rtmp-ffmpeg-conf/blob/master/nginx.conf)

* RTMP配置
```bash
rtmp {
    server {
        listen 1935; #端口
        chunk_size 4000;
        #RTMP 直播流配置
        application stream {
            live on;
            #添加水印及分流，这次方便测试直接分流到当前服务器hls
            #实际生产一般都分流到直播云（腾讯云、阿里云、ucloud）
            #只需把需要分流的地址替换即可
            #有水印：rtmp://localhost:1935/hls/$name_wm
            #无水印：rtmp://localhost:1935/hls/$name
            exec ffmpeg -i rtmp://localhost:1935/stream/$name -i /opt/images/ar414.png
              -filter_complex "overlay=10:10,split=1[ar414]"
              -map '[ar414]' -map 0:a -s 1920x1080 -c:v libx264 -c:a aac -g 30 -r 30 -tune zerolatency -preset veryfast -crf 23 -f flv rtmp://localhost:1935/hls/$name_wm
              -c:a libfdk_aac -b:a 128k -c:v libx264 -b:v 2500k -f flv -g 30 -r 30 -s 1920x1080 -preset superfast -profile:v baseline rtmp://localhost:1935/hls/$name;
        }

        application hls {
            live on;
            hls on;
            hls_fragment 5;
            hls_path /opt/data/hls;
        }
    }
}
```
* 如果需要推多个直播云则复制多个 exec ffmpeg即可 如下：
```bash
application stream {
    live on;
    #分流至本机hls           
    exec ffmpeg -i rtmp://localhost:1935/stream/$name -i /opt/images/ar414.png
      -filter_complex "overlay=10:10,split=1[ar414]"
      -map '[ar414]' -map 0:a -s 1920x1080 -c:v libx264 -c:a aac -g 30 -r 30 -tune zerolatency -preset veryfast -crf 23 -f flv rtmp://localhost:1935/hls/$name_wm
      -c:a libfdk_aac -b:a 128k -c:v libx264 -b:v 2500k -f flv -g 30 -r 30 -s 1920x1080 -preset superfast -profile:v baseline rtmp://localhost:1935/hls/$name;
    
    #分流至腾讯云
    exec ffmpeg -i rtmp://localhost:1935/stream/$name -i /opt/images/ar414.png
      -filter_complex "overlay=10:10,split=1[ar414]"
      -map '[ar414]' -map 0:a -s 1920x1080 -c:v libx264 -c:a aac -g 30 -r 30 -tune zerolatency -preset veryfast -crf 23 -f flv rtmp://live-push.tencent.com/stream/$name_wm
      -c:a libfdk_aac -b:a 128k -c:v libx264 -b:v 2500k -f flv -g 30 -r 30 -s 1920x1080 -preset superfast -profile:v baseline rtmp://live-push.tencent.com/stream/$name;

    #分流至阿里云
    exec ffmpeg -i rtmp://localhost:1935/stream/$name -i /opt/images/ar414.png
      -filter_complex "overlay=10:10,split=1[ar414]"
      -map '[ar414]' -map 0:a -s 1920x1080 -c:v libx264 -c:a aac -g 30 -r 30 -tune zerolatency -preset veryfast -crf 23 -f flv rtmp://live-push.aliyun.com/stream/$name_wm
      -c:a libfdk_aac -b:a 128k -c:v libx264 -b:v 2500k -f flv -g 30 -r 30 -s 1920x1080 -preset superfast -profile:v baseline rtmp://live-push.aliyun.com/stream/$name;
}
```  

* 水印位置
    * 水印位置
    
        |  水印图片位置   | overlay值  |
        |  ----   | ----  |
        | 左上角  | 10:10 |
        | 右上角  | main_w-overlay_w-10:10 |
        | 左下角  | 10:main_h-overlay_h-10 |
        | 右下角  | main_w-overlay_w-10 : main_h-overlay_h-10 |
    * overlay参数
    
        |  参数   | 说明  |
        |  ----   | ----  |
        | main_w  | 视频单帧图像宽度（当前配置文件1920） |
        | main_h  | 视频单帧图像高度（当前配置文件1080） |
        | overlay_w  | 水印图片的宽度 |
        | overlay_h  | 水印图片的高度 |

     
## :octocat:结语
* 如果觉得对你有帮助请给我一个[start](https://github.com/ar414-com/nginx-rtmp-ffmpeg-conf)