1. main.go
main() -> s.Run() -> program.Start()

2. 可正常存储的组合：
ffmpeg -re -i ~/Movies/Asian-LinLi.mp4 -c copy -rtsp_transport tcp -f rtsp rtsp://localhost:554/linli-store
ffmpeg -fflags genpts -rtsp_transport tcp -i rtsp://localhost:554/linli-store -c:v copy -c:a aac -hls_time 6 -hls_list_size 0 linli-store/20230506/out.m3u8

3. 启动一个http server，一个rtsp server
程序以系统服务(system service)的方式运行，由github.com/MeloQi/service(源项目：https://github.com/kardianos/service)封装了跨平台的系统服务运行能力。
可以直接运行，也可以install/start/stop/uninstall等方式控制以服务的方式运行

http server采用gin框架

4. 使用sqlite保存数据，使用gorm操作sqlite
保存的数据有用户信息、流信息、Session信息。

5. 推流、拉流
推流：外部RTSP客户端连接EasyDarwin服务器，发送RTSP流。
拉流：EasyDarwin内RTSP客户端，连接外部RTSP服务器，获得流后转发到EasyDarwin服务器。

所有的流都会发到EasyDarwin RTSP服务器（对应Pusher），只是直接发与转发之别。

6. RTSP Server
为TCP Server，Accept连接后，包装为一个Session管理数据接收和发送。
- Session接收数据，分为接收二进制的RTP包（RTPPack，$(0x24)开头），和接收RTSP文本命令（Request）
- 每个Session的角色为Pusher或Player（二选一），根据发送的命令确定角色
  - 发送了ANNOUNCE命令(声明流信息-sdp)的为Pusher，会推送音视频数据；
  - 发送了DESCRIBE命令(查询流信息-sdp)的为Player，会接收音视频数据；
- 一个Pusher Session可绑定0~n个Player Session，将音视频数据转发
- Session收到RTP包后，转送到Pusher的队列，Pusher再广播到多个Player的队列，Player负责真正发送到客户端

Pusher：转发器，有两种，负责转送RTP包到Player的队列，以及进行gop缓存
- RTSP Server收到流后，负责转发的角色，一个Session对应一个Pusher，一个Pusher对应多个Player（客户端，收流处理）
  - 每个Path对应一个Session(Server接收)，对应0~n个Player(Server发送)
- 拉流转发时，内部RTSP Client收到流后，负责转发给RTSP Server。
- 所有Pusher按路径注册在RTSP Server中，从而可为Player Session查找匹配的Pusher。也会确保一个Path只有一个Pusher。

Player：发送器
- Player Session在执行Play/Record命令后，才会绑定到Pusher，并开启发送循环

7. TCP/UDP
- 如果音视频数据选择TCP传输，则复用同一个TCP连接，通过数据包内的channel id信息区分包类型 （tcp-rtsp消耗资源更少，且对防火墙更友好；但码流上限较低）
- 如果音视频数据选择UDP传输，则对Pusher Session会创建4个udp server，用于audio data/control和video data/control。
  - 4个udp server连接由UDPServer对象管理，绑定到Pusher对象。因为Pusher可用于Session或RTSPClient。（但实际在RTSPClient中独立绑定了UDPServer）
- 如果音视频数据选择UDP传输，则对Player Session会创建4个**udp client**，用于audio data/control和video data/control
  - 4个udp client连接由UDPClient对象管理，绑定到Session对象。（个人认为绑定到Player对象也可以）
- 对RTP/RTCP，采用UDP传输时，音视频流的接收方是Server，发送方是Client！

8. RTSP Client
- 默认选择TCP传输RTP/RTCP，可指定用UDP
- 与HTTP Client不同，RTSP Client是有状态的，建立一个TCP连接后，保持长连接发送各命令。
- **RTSP Client无需再连接RTSP Server，其只是绑定一个Pusher，然后该Pusher加到Server的Pusher列表，为匹配的Player Session提供数据！**

9. 用url.Parse解析rtsp地址，可得scheme、username、password、host、port、path等信息

------
已知问题：
1. Pusher的queue采用slice，头部节点消费后有效期往后推移。可能导致大内存被引用得不到释放。
宜选用circular buffer，或list链表
2. Session接收rtp数据，有大量小数组分配，宜使用sync.Pool
3. Session.SendRTP代码较为冗余。
4. RTSPClient.requestStream中，发送OPTIONS和DESCRIBE后都检查是否有401错误，并加上Auth信息重新发送，此逻辑可以放在RTSPClient.Request内复用。
5. RTSPClient额外绑定了UDPServer对象，没有必要，可以使用绑定的Pusher的UDPServer对象

------
待验证：
1. 拉流中断后恢复，是否能接上？
2. 推流中断后恢复，是否能接上？
