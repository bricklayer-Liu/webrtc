#云服务器搭建Janus Server & chrome直播树莓派摄像头采集的视频

###开放端口
    TCP: 443, 3478, 8080, 8089, 8188, 8989
    UDP: 3478, 30000-60000


###1、安装依赖
```shell script
    yum install libmicrohttpd-devel jansson-devel \
       openssl-devel libsrtp-devel sofia-sip-devel glib2-devel \
       opus-devel libogg-devel libcurl-devel pkgconfig gengetopt \
       libconfig-devel libtool autoconf automake
       
    yum install gnutls gnutls-deve

```
###2、安装WebSocket
```shell script
    git clone https://github.com/warmcat/libwebsockets.git
    
    cd libwebsockets
    
    git branch -a 查看选择最新的稳定版本，目前的是remotes/origin/v3.2-stable
    
    git checkout v3.2-stable 切换到最新稳定版本
    
    mkdir build
    
    cd build
    
    cmake -DCMAKE_INSTALL_PREFIX:PATH=/usr -DCMAKE_C_FLAGS="-fpic" ..
    
    make && sudo make install
```
###3、安装 libsrtp
```shell script
    wget https://github.com/cisco/libsrtp/archive/v2.2.0.tar.gz
    
    tar xfv v2.2.0.tar.gz
    
    cd libsrtp-2.2.0
    
    ./configure --prefix=/usr --enable-openssl
    
    make shared_library && sudo make install
```
###4、安装libusrsctp(支持--enable-data-channels)
```shell script
    git clone https://github.com/Kurento/libusrsctp.git
    
    cd libusrsctp
    
    ./bootstrap
    
    ./configure
    
    make
    
    sudo make install
```
###5、安装libmicrohttpd(支持--enable-rest)
```shell script
    wget https://ftp.gnu.org/gnu/libmicrohttpd/libmicrohttpd-0.9.71.tar.gz
    
    tar zxf libmicrohttpd-0.9.71.tar.gz
    
    cd libmicrohttpd-0.9.71/
    
    ./configure
    
    make
    
    sudo make install
 ```   
###5 安装libnice
```shell script
    先安装python3环境,
     pip3 install --user meson

    git clone https://gitlab.freedesktop.org/libnice/libnice
    cd libnice
    meson --prefix=/usr build && ninja -C build && sudo ninja -C build install

```
###6、编译 Janus
```shell script
    git clone https://github.com/meetecho/janus-gateway.git
    
    git tag 查看当前的 tag，选择最新稳定的版本v0.10.4
    
    git  checkout v0.10.4
    
    sh autogen.sh
    
    ./configure --prefix=/opt/janus --enable-websockets --enable-post-processing --enable-docs --enable-rest --enable-data-channels
    
    make
    
    sudo make install
```
###7、安装nginx
```shell script
    wget http://nginx.org/download/nginx-1.15.8.tar.gz
    
    tar xvzf nginx-1.15.8.tar.gz
    
    cd nginx-1.15.8/
    
    ./configure --with-http_ssl_module  (支持https)
    
    make
    
    sudo make install 
```
###8、生成证书
```shell script
    mkdir -p ~/cert
    cd ~/cert
    # CA私钥
    openssl genrsa -out key.pem 2048
    # 自签名证书
    openssl req -new -x509 -key key.pem -out cert.pem -days 1095 


###9、修改nginx配置文件
    路径:/usr/local/nginx/conf/nginx.conf


    #
    server {
        listen       443 ssl;
        server_name  localhost;
				# 配置相应的key
        ssl_certificate      /root/cert/cert.pem;
        ssl_certificate_key  /root/cert/key.pem;

        ssl_session_cache    shared:SSL:1m;
        ssl_session_timeout  5m;

        ssl_ciphers  HIGH:!aNULL:!MD5;
        ssl_prefer_server_ciphers  on;
				# 指向janus demo所在目录
        location / {
            root   /opt/janus/share/janus/demos;
            index  index.html index.htm;
        }
    }
```
###10、安装coturn
```shell script
    #git clone https://github.com/coturn/coturn 
    #cd coturn
    # 提供另一种安装方式turnserver是coturn的升级版本
    wget http://coturn.net/turnserver/v4.5.0.7/turnserver-4.5.0.7.tar.gz
    tar xfz turnserver-4.5.0.7.tar.gz
    cd turnserver-4.5.0.7
     
    ./configure 
    make 
    sudo make install
```    

###11、修改janus配置文件

```shell script
路径:/opt/janus/etc/janus
#拷贝文件
sudo cp janus.jcfg.sample janus.jcfg
sudo cp janus.transport.http.jcfg.sample janus.transport.http.jcfg
sudo cp janus.transport.websockets.jcfg.sample janus.transport.websockets.jcfg
sudo cp janus.plugin.videoroom.jcfg.sample janus.plugin.videoroom.jcfg
sudo cp janus.transport.pfunix.jcfg.sample janus.transport.pfunix.jcfg
sudo cp janus.plugin.streaming.jcfg.sample janus.plugin.streaming.jcfg
sudo cp janus.plugin.recordplay.jcfg.sample janus.plugin.recordplay.jcfg
sudo cp janus.plugin.voicemail.jcfg.sample janus.plugin.voicemail.jcfg
sudo cp janus.plugin.sip.jcfg.sample janus.plugin.sip.jcfg
sudo cp janus.plugin.nosip.jcfg.sample janus.plugin.nosip.jcfg
sudo cp janus.plugin.textroom.jcfg.sample  janus.plugin.textroom.jcfg
sudo cp janus.plugin.echotest.jcfg.sample janus.plugin.echotest.jcfg

    
    配置janus.jcfg
    
    # 大概237行
    stun_server = "1.15.156.106"
        stun_port = 3478
        nice_debug = false
    
    #大概274行
    # credentials to authenticate...
        turn_server = "1.15.156.106"
        turn_port = 3478
        turn_type = "udp"
        turn_user = "lyx"
        turn_pwd = "123456"
        
     
     配置janus.transport.http.jcfg
     
        #取消以下两行注释,支持https
        https = true                            
        secure_port = 8089  
        
        #结尾处取消注释并更改为自己的证书路径
        cert_pem = "/root/cert/cert.pem"
        cert_key = "/root/cert/key.pem"
        
        
      配置janus.transport.websockets.jcfg
      
        #取消以下两行注释
        wss = true                                            
        wss_port = 8989 
        
        #结尾处取消注释并更改为自己的证书路径
        cert_pem = "/root/cert/cert.pem"
        cert_key = "/root/cert/key.pem"
 ```   
###12、运行服务器
```shell script
    /usr/local/nginx/sbin/nginx
    nohup turnserver -L 0.0.0.0 --min-port 30000 --max-port 60000  -a -u lyx:123456 -v -f -r nort.gov &
    /opt/janus/bin/janus --debug-level=5
 ```   
  
    
##修改官方源码
    #安装包
    pip3 install aiohttp aiortc opencv-python websockets
    
    #连接云服务器
    session = JanusSession("http://1.15.156.106:8088/janus")
    
    #调用树莓派摄像头
     player = MediaPlayer('/dev/video0', format='v4l2', options={
       'video_size': '320x240'
    })
    
##修改janus video-room demo
    修改 videoroomtest.js文件
    第406行关闭订阅者视频发送  media: { audioRecv: false, videoRecv: false, audioSend: useAudio, videoSend: false}
    第576行增加 muted="muted",设置为静音
    $('#videoremote'+remoteFeed.rfindex).append('<video class="rounded centered relative hide" id="remotevideo' + remoteFeed.rfindex + '" width="100%" height="100%" autoplay playsinline muted="muted"/>');
    
    
    修改videoroomtest.html文件
    在第94,127,137,145,153行加入style="display: none;隐藏5个订阅者窗口
    设置窗口自适应,在第93行添加tyle="margin-left:10px;margin-right:10px;margin-top:10px;margin-bottom:10px;


    
    
    

   
   
