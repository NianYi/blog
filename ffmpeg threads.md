AVCodecContext *pAVCodecCtx = avcodec_alloc_context3(pCodec);

pAVCodecCtx->thread_count = 8;


测试环境

组内测试机，4核，1.8GHZ，内存2G

测试过程

默认线程设置，ffmpeg的AVCodecContext中thread_count为1

测试结果：cpu占用100%，转码时间9分48秒



线程设置：thread_count为4，thread_type为FRAME类型

测试结果：cpu占用250%，4个cpu平均使用率20%多，转码时间4分28秒

结果总结

x264多线程可以缩减编码时间，4倍线程数实际达不到节省4倍的转码时间（可能由由于转码包含解码亦损耗时间的因素），线程数越大，实际编码性能提升呈递减。
————————————————
版权声明：本文为CSDN博主「galen6」的原创文章，遵循 CC 4.0 BY-SA 版权协议，转载请附上原文出处链接及本声明。
原文链接：https://blog.csdn.net/zhengbin6072/article/details/78865941
