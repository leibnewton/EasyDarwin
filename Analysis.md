1. main.go
main() -> s.Run() -> program.Start()

2. 可正常存储的组合：
ffmpeg -re -i ~/Movies/Asian-LinLi.mp4 -c copy -rtsp_transport tcp -f rtsp rtsp://localhost:554/linli-store
ffmpeg -fflags genpts -rtsp_transport tcp -i rtsp://localhost:554/linli-store -c:v copy -c:a aac -hls_time 6 -hls_list_size 0 linli-store/20230506/out.m3u8
