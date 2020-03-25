```
transcode_step()
----choose_output()
----process_input()
--------get_input_packet()
------------av_read_frame()->read_frame_internal()->ff_read_packet()->mov_read_packet()
--------process_input_packet()
------------decode_video->decode()
----------------avcodec_send_packet()
----------------avcodec_receive_frame()
----reap_filters()
--------do_video_out
------------avcodec_send_frame->X264_frame
------------avcodec_receive_packet
manuellog transcode() transcode_step()
manuellog transcode() choose_output()返回NULL表示文件到头
选择输出流OutputStream：查找流列表中最小dts的流
cur_dts is invalid st:0 (0) [init:1 i_done:0 finish:0] (this is harmless if it occurs once at the start per stream)
manuellog transcode() transcode_from_filter
滤镜先忽略

manuellog transcode() process_input
获取输入流中的一帧数据，解封装、解码
manuellog transcode() get_input_packet()
调用libavformat/utils.c里av_read_frame()->read_frame_internal()->ff_read_packet()读取一帧
ff_read_packet()函数里调用了输入封装的read_packet函数指针：s->iformat->read_packet()即mov.c中的mov_read_packet()函数
manuelmov mov_read_packet()
[mov,mp4,m4a,3gp,3g2,mj2 @ 0x3ceacc0] stream 0, sample 6, dts 160000
manuelmov mov_current_sample_inc()
manuelmov avio_seek()
manuelmov av_get_packet()
manuelmov av_get_packet() end
打印信息
[mov,mp4,m4a,3gp,3g2,mj2 @ 0x3ceacc0] ff_read_packet stream=0, pts=2560, dts=2048, size=26, duration=0, flags=0
[mov,mp4,m4a,3gp,3g2,mj2 @ 0x3ceacc0] IN delayed:1 pts:2560, dts:2048 cur_dts:1536 st:0 pc:(nil) duration:512 delay:2 onein_oneout:0
[mov,mp4,m4a,3gp,3g2,mj2 @ 0x3ceacc0] OUTdelayed:1/2 pts:2560, dts:2048 cur_dts:2048 st:0 (1)
[mov,mp4,m4a,3gp,3g2,mj2 @ 0x3ceacc0] read_frame_internal stream=0, pts=2560, dts=2048, size=26, duration=512, flags=0
manuellog transcode() get_input_packet() end ret: 0

打印信息
demuxer -> ist_index:0 type:video next_dts:160000 next_dts_time:0.16 next_pts:80000 next_pts_time:0.08 pkt_pts:2560 pkt_pts_time:0.2 pkt_dts:2048 pkt_dts_time:0.16 off:0 off_time:0
demuxer+ffmpeg -> ist_index:0 type:video pkt_pts:2560 pkt_pts_time:0.2 pkt_dts:2048 pkt_dts_time:0.16 off:0 off_time:0

manuellog process_input_packet decode_video()
manuellog decode_video() decode()
manuellog decode() avcodec_send_packet()
定义与decode.c中，调用了decode_receive_frame_internal()->decode_simple_receive_frame() ->decode_simple_internal() 
当中调用ff_decode_get_packet()获取AvPacket，再调用avctx->codec->decode解码出AvFrame赋值给avctx->internal->buffer_frame（264解码位于libavcodec/h264dec.c）

manuellog decode() avcodec_receive_frame()
由于解码再avcodec_send_packet()中执行过了，这里直接用avctx->internal->buffer_frame数据处理后返回（如果为buffer_frame空则执行decode_receive_frame_internal（）获取并解码）

打印信息
decoder -> ist_index:0 type:video frame_pts:1024 frame_pts_time:0.08 best_effort_ts:1024 best_effort_ts_time:0.08 keyframe:0 frame_type:3 time_base:1/12800
[h264 @ 0x3cf0340] nal_unit_type: 1(Coded slice of a non-IDR picture), nal_ref_idc: 0
需要继续解码，还不理解为啥有两次decode_video(),这一边只调用了avcodec_receive_frame，未调用avcodec_send_packet
manuellog process_input_packet decode_video()
manuellog decode_video() decode()
manuellog decode() avcodec_receive_frame()

manuellog transcode() reap_filters
manuellog reap_filters() av_buffersink_get_frame_flags
filter -> pts:2 pts_time:0.08 exact:2.000008 time_base:1/25
manuellog reap_filters() do_video_out
encoder <- type:video frame_pts:2 frame_pts_time:0.08 time_base:1/25
manuellog do_video_out() avcodec_send_frame
manuellog avcodec_encode_video2() encode2->avcode_send_frame()
manuellog X264_frame()
manuellog avcodec_encode_video2() encode2->avcode_send_frame()end
manuellog do_video_out() avcodec_receive_packet()
manuellog avcodec_receive_packet()
manuellog avcodec_receive_packet() point1
manuellog reap_filters() av_buffersink_get_frame_flags

```

AVCodecContext *pAVCodecCtx = avcodec_alloc_context3(pCodec);

pAVCodecCtx->thread_count = 8;


测试环境

组内测试机，4核，1.8GHZ，内存2G

测试过程

默认线程设置，ffmpeg的AVCodecContext中thread_count为1

测试结果：cpu占用100%，转码时间9分48秒



线程设置：thread_count为4，thread_type为FRAME类型

测试结果：cpu占用250%，4个cpu平均使用率50%多，转码时间4分28秒

结果总结

x264多线程可以缩减编码时间，4倍线程数实际达不到节省4倍的转码时间（可能由由于转码包含解码亦损耗时间的因素），线程数越大，实际编码性能提升呈递减。

