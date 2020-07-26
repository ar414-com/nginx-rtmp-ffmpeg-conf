## :whale:[docker-nginx-rtmp-ffmpeg](https://hub.docker.com/repository/docker/ar414/nginx-rtmp-ffmpeg)  
[简体中文](./CN-README.md)      

Based on the configuration and deployment of [docker-nginx-rtmp](https://github.com/alfg/docker-nginx-rtmp)the meaning of this article is to realize live streaming and live screen watermarking.
* Nginx 1.16.1（Stable version compiled from source code）
* nginx-rtmp-module 1.2.1（Compile from source）
* ffmpeg 4.2.1（Compile from source）
* Configured [nginx.conf](https://github.com/ar414-com/nginx-rtmp-ffmpeg-conf/blob/master/nginx.conf)
    * Only support 1920*1080 (if you need to support other resolutions, please refer to [nginx.conf](https://github.com/alfg/docker-nginx-rtmp/blob/master/nginx.conf)）
    * Realize two-way shunt
        * localhost
        * Live cloud provider（例：AWS）
    * Realize live watermark effect
        * Watermark image storage location (in the container)：/opt/images/logo.png

## :computer:Deploy and run
### Server
* Install docker (Centos7, other systems please use your search function)
```bash
$ yum -y install docker #install docker
$ systemctl enable docker #Configure startup
$ systemctl start docker #Start the docker service
```
* Pull the docker image and run
```bash
$ docker pull ar414/nginx-rtmp-ffmpeg
$ docker run -it -d -p 1935:1935 -p 8080:80 --rm ar414/nginx-rtmp-ffmpeg
```
* Push stream address（Stream live content to）：
```
rtmp://<server ip>:1935/stream/$STREAM_NAME
```
* SSL certificate

Copy the certificate to the container, and modify the nginx.conf configuration file in the container, and then re-commit (all the files in the container need to be re-committed to take effect)
```
#/etc/nginx/nginx.conf
listen 443 ssl;
ssl_certificate     /opt/certs/example.com.crt;
ssl_certificate_key /opt/certs/example.com.key;
```

### OBS configuration
* Stream Type: `Custom Streaming Server`
* URL: `rtmp://<server ip>:1935/stream`
* Stream Key：ar414
![obs-config](https://cdn.ar414.com/obs-config.png)

### Watch test
> HLS playback test tool：http://player.alicdn.com/aliplayer/setting/setting.html （If a certificate is configured, use https）

* HLS playback address
    * With watermark：http://\<server ip>:8080/hls/ar414_wm.m3u8
    ![ar414-hls-wm](https://cdn.ar414.com/ar414-hls-wm.png)
    * No watermark：http://\<server ip>:8080/hls/ar414.m3u8
    ![ar414-hls](https://cdn.ar414.com/ar414-hls.png)
    
> RTMP test tool：[PotPlayer](https://daumpotplayer.com/download/)

* RTMP playback address
    * No watermark：rtmp://\<server ip>:1935/stream/ar414
    ![ar414-rtmp](https://cdn.ar414.com/ar414-rtmp.png)
    * With watermark：Need to be diverted to other servers
    
## :page_facing_up:Brief explanation of configuration file (diversion, watermark and watermark position)
> [Full configuration file](https://github.com/ar414-com/nginx-rtmp-ffmpeg-conf/blob/master/nginx.conf)

* RTMP configuration
```bash
rtmp {
    server {
        listen 1935; #port
        chunk_size 4000;
        #RTMP Live streaming configuration
        application stream {
            live on;
            # Add watermark and diversion, this time it is convenient to test diversion directly to the current server hls
            # Actual production is generally diverted to the live cloud（AWS）
            # Just replace the address that needs to be diverted
            # With watermark：rtmp://localhost:1935/hls/$name_wm
            # No watermark：rtmp://localhost:1935/hls/$name
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
* If you need to push multiple live clouds, copy multiple exec ffmpeg as follows：
```bash
application stream {
    live on;
    #Shunt to local hls           
    exec ffmpeg -i rtmp://localhost:1935/stream/$name -i /opt/images/ar414.png
      -filter_complex "overlay=10:10,split=1[ar414]"
      -map '[ar414]' -map 0:a -s 1920x1080 -c:v libx264 -c:a aac -g 30 -r 30 -tune zerolatency -preset veryfast -crf 23 -f flv rtmp://localhost:1935/hls/$name_wm
      -c:a libfdk_aac -b:a 128k -c:v libx264 -b:v 2500k -f flv -g 30 -r 30 -s 1920x1080 -preset superfast -profile:v baseline rtmp://localhost:1935/hls/$name;
    
    #Diverted to Tencent Cloud
    exec ffmpeg -i rtmp://localhost:1935/stream/$name -i /opt/images/ar414.png
      -filter_complex "overlay=10:10,split=1[ar414]"
      -map '[ar414]' -map 0:a -s 1920x1080 -c:v libx264 -c:a aac -g 30 -r 30 -tune zerolatency -preset veryfast -crf 23 -f flv rtmp://live-push.tencent.com/stream/$name_wm
      -c:a libfdk_aac -b:a 128k -c:v libx264 -b:v 2500k -f flv -g 30 -r 30 -s 1920x1080 -preset superfast -profile:v baseline rtmp://live-push.tencent.com/stream/$name;

    #Diverted to Alibaba Cloud
    exec ffmpeg -i rtmp://localhost:1935/stream/$name -i /opt/images/ar414.png
      -filter_complex "overlay=10:10,split=1[ar414]"
      -map '[ar414]' -map 0:a -s 1920x1080 -c:v libx264 -c:a aac -g 30 -r 30 -tune zerolatency -preset veryfast -crf 23 -f flv rtmp://live-push.aliyun.com/stream/$name_wm
      -c:a libfdk_aac -b:a 128k -c:v libx264 -b:v 2500k -f flv -g 30 -r 30 -s 1920x1080 -preset superfast -profile:v baseline rtmp://live-push.aliyun.com/stream/$name;
}
```  

* Watermark position
    * Watermark position
    
        |  Watermark image location   | overlay value  |
        |  ----   | ----  |
        | Upper left corner  | 10:10 |
        | Upper right corner  | main_w-overlay_w-10:10 |
        | Bottom left  | 10:main_h-overlay_h-10 |
        | Bottom right corner  | main_w-overlay_w-10 : main_h-overlay_h-10 |
    * overlay param
    
        |  param   | description  |
        |  ----   | ----  |
        | main_w  | Video single frame image width (current profile 1920) |
        | main_h  | Video single frame image height (current profile 1080) |
        | overlay_w  | The width of the watermark image |
        | overlay_h  | The height of the watermark image |     
