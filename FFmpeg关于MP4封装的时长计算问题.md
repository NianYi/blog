一、影响
1、视频时长异常，导致基于时长计算出的码率不准（码率=文件大小/时长）。
2、视频时长作为统计转码量的一个重要指标，甚至用于计费系统，其准确性十分重要。

二、问题复现
```
a)输入视频时长信息（附件可下载视频）：
Duration 3.05
Video    3.031389
Audio    3.050667

b)转码命令

ffmpeg  -i LlT7Gp2aLrEA.mp4 -vcodec libx264 -vsync 0 output.mp4 或
ffmpeg  -i LlT7Gp2aLrEA.mp4 -vcodec libx264 -vsync 2 output.mp4

c)输出视频时长信息
Duration 5.60
Video    5.597765
Audio    3.050000
```
三、问题分析

3.1、封装层结构
为了弄清楚封装层的时长计算逻辑，对MP4封装进行了一番熟悉和整理，详见另一篇文章：MP4的封装结构。时长信息存储在mdhd类型的box当中，mdhd主要保存了track的duration和timescale信息。对应的ffmpeg实现如下所示，可以看出时长的计算方式是：尾帧dts - 首帧dts + 帧延时，既然时长计算与dts相关，那就从dts入手。
```
libavformat/movenc.c

int ff_mov_write_packet(AVFormatContext *s, AVPacket *pkt)
{
    ....
    if (trk->start_dts == AV_NOPTS_VALUE) {
        trk->start_dts = pkt->dts;
        if (trk->frag_discont) {
            if (mov->use_editlist) {
                trk->frag_start = pkt->pts;
                trk->start_dts  = pkt->dts - pkt->pts;
            } else {
                trk->frag_start = pkt->dts;
                trk->start_dts  = 0;
            }
            trk->frag_discont = 0;
        } else if (pkt->dts && mov->moov_written)
            av_log(s, AV_LOG_WARNING,
                   "Track %d starts with a nonzero dts %"PRId64", while the moov"
                   "already has been written. Set the delay_moov flag to handle "
                   "this case.\n",
                   pkt->stream_index, pkt->dts);
    }
    //时长 = 尾帧dts - 首帧dts + 帧延时
    trk->track_duration = pkt->dts - trk->start_dts + pkt->duration;
    trk->last_sample_is_subtitle_end = 0;
    ....
}
```
 3.2、DTS追踪
打印出了转码过程几个阶段的dts，发现经过了几次时基转换，然而时基转换并不会改变dts值。终于通过对比dts序列，发现问题出在编码这一步，很明显编码前后dts变化并不是时基转换，包含了特殊的dts转换逻辑。经过一番查找，终于在编码器的代码里找到了这部分内容。

输入视频(时基:90000) | 解码后/编码前(时基:179/6) | 编码后(时基:179/6) | 输出视频(时基:11456)
------------- | ------------- | ------------- | -------------
dts=0 dts=236625 dts=239651 dts=242657 dts=245695 dts=248698 dts=251713 dts=254728 dts=257743 dts=260758 dts=263802 dts=266795 dts=269809 | dts=0 dts=78 dts=79 dts=80 dts=81 dts=82 dts=83 dts=84 dts=85 dts=86 dts=87 dts=88 dts=89 | dts=-79 dts=-1 dts=0 dts=78 dts=79 dts=80 dts=81 dts=82 dts=83 dts=84 dts=85 dts=86 dts=87 | dts=-30336 dts=-384 dts=0 dts=29952 dts=30336 dts=30720 dts=31104 dts=31488 dts=31872 dts=32256 dts=32640 dts=33024 dts=33408

3.3、编码器的DTS策略
核心策略就是预测b帧需要参考帧，允许存在b帧时要保证在dts=0时有足够的缓存帧供b帧参考。先介绍两个概念：i_bframe_delay表示预测b帧需要的缓存帧数量，i_bframe_delay_time是对应的缓存时间；
a）如果 i_bframe_delay等于0，说明不存在b帧，则不做任何转换dts就等于pts；
b）如果 i_bframe_delay不等于0 且 当前帧序号 <= i_bframe_delay，说明存在b帧且当前帧是缓存帧，pts减去缓存时间 i_bframe_delay_time后再赋值给dts；
c）如果 i_bframe_delay不等于0 且 当前帧序号 > i_bframe_delay，说明存在b帧且当前帧不是缓存帧，dts 从prev_recordered_pts[ ] 数组循环地取值（这块逻辑可以对照代码和上面dts表格来理解）；
```
encoder/encoder.c

int     x264_encoder_encode( x264_t *h,
                             x264_nal_t **pp_nal, int *pi_nal,
                             x264_picture_t *pic_in,
                             x264_picture_t *pic_out )
{
    ....
    /* ------------------- part1 -------------------- */
    if( pic_in != NULL )
    {
        fenc->i_frame = h->frames.i_input++;
        if( fenc->i_frame == 0 )
            h->frames.i_first_pts = fenc->i_pts;
	//i_bframe_delay是预测b帧需要缓存的帧数
        //i_bframe_delay_time就是前i_bframe_delay帧对应的时间
        if( h->frames.i_bframe_delay && fenc->i_frame == h->frames.i_bframe_delay )
            h->frames.i_bframe_delay_time = fenc->i_pts - h->frames.i_first_pts;
        ....
    }
    /* ------------------- part2 -------------------- */
    h->fdec->i_pts = h->fenc->i_pts;
    if( h->frames.i_bframe_delay )
    {
        int64_t *prev_reordered_pts = thread_current->frames.i_prev_reordered_pts;
	//i_dts是当前帧的解码时间
	//i_frame是当前帧序号、i_reordered_pts是当前帧的播放时间
	//i_bframe_delay和i_bframe_delay_time是预测b帧需要的缓存帧数量和时间
	//prev_reordered_pts[]数组是计算dts的辅助数组
        h->fdec->i_dts = h->i_frame > h->frames.i_bframe_delay
                ? prev_reordered_pts[ (h->i_frame - h->frames.i_bframe_delay) % h->frames.i_bframe_delay ]
                : h->fenc->i_reordered_pts - h->frames.i_bframe_delay_time;
        prev_reordered_pts[ h->i_frame % h->frames.i_bframe_delay ] = h->fenc->i_reordered_pts;
    }
    else{
        h->fdec->i_dts = h->fenc->i_reordered_pts;
    }
}
```
再说下 i_bframe_delay 的计算过程：i_bframe表示 i帧和p帧之间允许的b帧数量，i_bframe_pyramid表示b帧能否作为参考帧（i_bframes可以通过-x264-params bframes=number来强制指定）
a）如果 i_bframe等于0，表示不存在b帧，i_bframe_delay自然也等于0；
b）如果 i_bframe不等于0，i_bframe_delay 的值由 i_bframe_pyramid 决定，具体看代码；
```
encoder/encoder.c

x264_t *x264_encoder_open( x264_param_t *param )
{
    x264_t *h;
    char buf[1000], *p;
    int i_slicetype_length;
    ....
    /*
    i_bframe:i和p之间最大b帧数量
    i_bframe_pyramid: 是否允许b帧作为参考帧，
                      =0表示不允许
                      =1表示1个gop只允许1个b参考帧
                      =2表示随意
    */
    h->frames.i_bframe_delay = h->param.i_bframe 
                            ? (h->param.i_bframe_pyramid ? 2 : 1) : 0;
}
```
四、小结

到现在为止，我们已经找到了时长异常发生的原因：当允许存在b帧且当前帧是缓存帧时，该帧dts就等于pts减去缓存时间 ，而缓存时间包含了pts跳跃的部分，导致首帧dts异常小，进而导致视频时长计算偏大。

ffmpeg  -i LlT7Gp2aLrEA.mp4 -vcodec libx264 -vsync 1 output.mp4
可以通过上面命令进行验证：-vsync 1 参数表示通过插帧或丢帧来达到固定码率，由于补齐了pts跳跃部分的帧，输出视频的时长就是正确的。那怎么解决其他情况下的时长计算问题呢？用于时长计算的指标只有dts和pts，由上可知dts缺陷明显，pts不存在这个问题，而且pts表示显示时间用来计算视频时长更符合人类的认知，所以我提出使用pts计算MP4封装时长，代码就不再解释了，有兴趣的可以和我私下探讨。

查看代码点击这里https://github.com/FFmpeg/FFmpeg/commit/2989d7fe11e61dbc4dfb8c408abfe5adfd13c8de
