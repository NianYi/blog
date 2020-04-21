FFMpeg 中比较重要的函数以及数据结构如下：
1. 数据结构：

(1) AVFormatContext

(2) AVOutputFormat

(3) AVInputFormat

(4) AVCodecContext

(5) AVCodec

(6) AVFrame
未压缩的帧数据结构，和AVPicture的区别：包含时间戳、宽高等信息，另外AVFrame表示音频时包含多帧音频帧

(7) AVPacket
压缩后的数据结构

(8) AVPicture
图像数据结构

(9) AVStream

2. 初始化函数：

(1) av_register_all()废弃

2018-02-06 - 0694d87024 - lavf 58.9.100 - avformat.h
  Deprecate use of av_register_input_format(), av_register_output_format(),
  av_register_all(), av_iformat_next(), av_oformat_next().
  Add av_demuxer_iterate(), and av_muxer_iterate().

(2) avcodec_open()废弃

2011-07-10 - 3602ad7 / 0b950fe - lavc 53.8.0
  Add avcodec_open2(), deprecate avcodec_open().
  NOTE: this was backported to 0.7

  Add avcodec_alloc_context3. Deprecate avcodec_alloc_context() and
  avcodec_alloc_context2().
  
(3) avcodec_close()

(4) av_open_input_file()废弃

2011-06-16 - 2905e3f / 05e84c9, 2905e3f / 25de595 - lavf 53.4.0 / 53.2.0 - avformat.h
  Add avformat_open_input and avformat_write_header().
  Deprecate av_open_input_stream, av_open_input_file,
  AVFormatParameters and av_write_header.
  
(5) av_find_input_format()
只是根据文件名去匹配查找对应的format

(6) av_find_stream_info()废弃

2011-07-10 - 3602ad7 / a67c061 - lavf 53.6.0
  Add avformat_find_stream_info(), deprecate av_find_stream_info().
  NOTE: this was backported to 0.7
  
(7) av_close_input_file()废弃

2011-12-12 - 8bc7fe4 / 5266045 - lavf 53.25.0 / 53.17.0
  Add avformat_close_input().
  Deprecate av_close_input_file() and av_close_input_stream().
  
3. 音视频编解码函数：

(1) avcodec_find_decoder()

(2) avcodec_alloc_frame()废弃

2013-12-11 - 29c83d2 / b9fb59d,409a143 / 9431356,44967ab / d7b3ee9 - lavc 55.45.101 / 55.28.1 - avcodec.h
  av_frame_alloc(), av_frame_unref() and av_frame_free() now can and should be
  used instead of avcodec_alloc_frame(), avcodec_get_frame_defaults() and
  avcodec_free_frame() respectively. The latter three functions are deprecated.
  
(3) avpicture_get_size()

(4) avpicture_fill()

(5) img_convert()

(6) avcodec_alloc_context()废弃

(7) avcodec_decode_video()废弃

2017-09-26 - b1cf151c4d - lavc 57.106.102 - avcodec.h
  Deprecate AVCodecContext.refcounted_frames. This was useful for deprecated
  API only (avcodec_decode_video2/avcodec_decode_audio4). The new decode APIs
  (avcodec_send_packet/avcodec_receive_frame) always work with reference
  counted frames.
  
(8) av_free_packet()废弃

2015-10-29 - lavc 57.12.100 / 57.8.0 - avcodec.h
  xxxxxx - Deprecate av_free_packet(). Use av_packet_unref() as replacement,
           it resets the packet in a more consistent way.
  xxxxxx - Deprecate av_dup_packet(), it is a no-op for most cases.
           Use av_packet_ref() to make a non-refcounted AVPacket refcounted.
  xxxxxx - Add av_packet_alloc(), av_packet_clone(), av_packet_free().
           They match the AVFrame functions with the same name.
           
(9) av_free()

4. 文件操作：

(1) av_new_stream()

2011-10-19 - d049257 / 569129a - lavf 53.17.0 / 53.10.0
  Add avformat_new_stream(). Deprecate av_new_stream().
  
(2) av_read_frame()

2012-03-20 - 0ebd836 / 3c90cc2 - lavfo 54.2.0
  Deprecate av_read_packet(), use av_read_frame() with
  AVFMT_FLAG_NOPARSE | AVFMT_FLAG_NOFILLIN in AVFormatContext.flags
  
(3) av_write_frame()

2012-01-25 - lavf 53.31.100 / 53.22.0
  3c5fe5b / f1caf01 Allow doing av_write_frame(ctx, NULL) for flushing possible
          buffered data within a muxer. Added AVFMT_ALLOW_FLUSH for
          muxers supporting it (av_write_frame makes sure it is called
          only for muxers with this flag).
          
(4) dump_format()

5. 其他函数：
