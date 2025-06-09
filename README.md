
> [上篇文章](https://blog.csdn.net/qq_22218713/article/details/143383700)记录了unimrcp和funasr构建呼叫中心语音识别的mrcp服务端，现在记录一下unimrcp和大厂免费流式TTS引擎的结合，可私有化部署，实现mrcp文本转语音

>至于如何进行unimrcp模块开发，可参考[智能客服搭建(2) - MRCP Server ASR插件开发
](https://blog.csdn.net/initiallht/article/details/119280960)

1.在websocket_tts_channel_speak的时候，创建websocket client去连接TTS服务端
```
static apt_bool_t websocket_tts_channel_speak(mrcp_engine_channel_t *channel, mrcp_message_t *request, mrcp_message_t *response)
{   
    /*
    other code
    */
    /* Start WebSocket connection and send text */
    if (!tts_channel->connected && tts_channel->tts_engine->ws_context) {
        apt_log(APT_LOG_MARK, APT_PRIO_INFO, "[websocket tts] Attempting to connect to WebSocket server");
        /* Connect to server */
        struct lws_client_connect_info conn_info;
        memset(&conn_info, 0, sizeof(conn_info));
        conn_info.context = tts_channel->tts_engine->ws_context;
        conn_info.address = "127.0.0.1";
        conn_info.port = 20013;
        conn_info.path = "/streamTTS";
        conn_info.host = conn_info.address;
        conn_info.origin = conn_info.address;
        conn_info.protocol = "tts-protocol";
        conn_info.userdata = tts_channel;
        tts_channel->wsi = lws_client_connect_via_info(&conn_info);
        if (!tts_channel->wsi) {
            apt_log(APT_LOG_MARK, APT_PRIO_ERROR, "[websocket tts] Failed to connect to WebSocket server");
            return FALSE;
        }
        apt_log(APT_LOG_MARK, APT_PRIO_INFO, "[websocket tts] WebSocket connection initiated");
    } else {
        apt_log(APT_LOG_MARK, APT_PRIO_INFO, "[websocket tts] WebSocket already connected or context not available");
    }

    return TRUE;
}
```
2.将流式TTS引擎返回的音频文件暂存
```
case LWS_CALLBACK_CLIENT_RECEIVE:
	/*other code*/
    /* process audio data */
	apt_bool_t result = mpf_buffer_audio_write(tts_channel->audio_buffer, (void*)processed_audio, processed_len);
	break;
	
```
3.读取音频帧，播放
```
/** Callback is called from MPF engine context to read/get new frame */
static apt_bool_t websocket_tts_stream_read(mpf_audio_stream_t *stream, mpf_frame_t *frame)
{
    websocket_tts_channel_t *tts_channel = stream->obj;

    // Handle stop request
    if(tts_channel->stop_response) {
        apt_log(APT_LOG_MARK, APT_PRIO_DEBUG, "[websocket tts] Processing stop response");
        mrcp_engine_channel_message_send(tts_channel->channel,tts_channel->stop_response);
        tts_channel->stop_response = NULL;
        tts_channel->speak_request = NULL;
        tts_channel->paused = FALSE;
        return TRUE;
    }

    // Read audio data if we have an active speak request and not paused
    if(tts_channel->speak_request && tts_channel->paused == FALSE) {
        // Check buffer size before reading
        size_t buffer_size = mpf_buffer_get_size(tts_channel->audio_buffer);
        mpf_buffer_frame_read(tts_channel->audio_buffer, frame);
        // Check for completion event
    } 

    return TRUE;
}
```
4.unimrcp server处理音频，并返回给freeswitch
![处理音频](https://i-blog.csdnimg.cn/direct/ee26b61b6d8248b1a858e966200ca821.png)
5.Streaming-TTS 服务端日志
![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/be290dd0380d41c4b6be919b4603426a.png)
可以看到首包响应在0.19s

vx: wxh_assistant




