AVCodecContext *pAVCodecCtx = avcodec_alloc_context3(pCodec);

pAVCodecCtx->thread_count = 8;
