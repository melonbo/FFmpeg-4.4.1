`bitstream_filter.c`主要负责实现比特流过滤器(Bitstream Filter)的相关功能。以下是该文件的主要作用和功能详解：

## 1. 比特流过滤器的注册与管理

`bitstream_filter.c`文件包含了所有内置比特流过滤器的注册代码，例如：

- `h264_mp4toannexb`：将H.264码流从MP4封装格式转换为Annex B格式
- `hevc_mp4toannexb`：对HEVC/H.265码流执行类似转换
- `trace_headers`：用于调试NAL单元头信息

这些过滤器通过全局数组`bitstream_filters`进行管理，`av_bsf_iterate()`函数可以遍历所有已注册的过滤器。

## 2. 比特流过滤器的核心架构实现

该文件实现了比特流过滤器的核心架构，包括：

- ​**旧版接口**​：基于`AVBitStreamFilterContext`的实现(FFmpeg 4.4之前)
- ​**新版接口**​：FFmpeg 4.4引入的`AVBSFContext`架构，解耦了过滤器上下文和过滤器逻辑

新版架构采用`AVBSFContext + AVBitStreamFilter`模式，使得比特流过滤器流水线更加独立和灵活。

## 3. 提供公共API函数

`bitstream_filter.c`实现了以下关键API函数：

- `av_bsf_get_by_name()`：通过名称查找比特流过滤器
- `av_bsf_iterate()`：内部使用的过滤器迭代函数
- `av_bsf_alloc()`：分配过滤器上下文
- `av_bitstream_filter_filter()`：应用过滤器处理数据(旧版API)

## 4. 处理不同封装格式转换

该文件中的过滤器主要处理不同封装格式间的转换，特别是：

- ​**MP4格式(AVCC/AVC1)​**​：使用长度前缀而非起始码分隔NAL单元，SPS/PPS存储在文件头
- ​**Annex B格式**​：使用`0x000001`或`0x00000001`起始码分隔NAL单元，SPS/PPS内嵌在码流中

许多解码器只支持Annex B格式，因此需要这些过滤器进行实时转换。

## 5. 实现比特流级修改

比特流过滤器可以在不解码的情况下直接修改编码数据流，常见操作包括：

- 添加/删除起始码
- 重新组织SPS/PPS参数集
- 修改NAL单元头信息
- 修复或故意引入错误数据(如用于测试)

---
### H264封装格式

Annex B格式的全称是**H.264 Annex B Byte Stream Format**​（H.264附录B字节流格式），它是H.264/AVC视频编码标准中定义的一种码流封装格式。
通过起始码（Start Code，如`0x000001`或`0x00000001`）分隔NALU

AVCC（AVC Configuration）格式的核心特征是通过**长度前缀**来标识每个NALU（网络抽象层单元）的边界，而非使用起始码。
- 示例：若前缀为`00 00 00 1A`（十六进制），表示后续NALU长度为26字节

| 特性           | AVCC                               | Annex B                  |
| ------------ | ---------------------------------- | ------------------------ |
| ​**边界标识**​   | 长度前缀（1/2/4字节）                      | 起始码（0x000001/0x00000001） |
| ​**参数集存储**​  | 集中存储在extradata                     | 内嵌在码流中周期性重复              |
| ​**随机访问**​   | 支持快速跳转                             | 需扫描起始码                   |
| ​**适用场景**​   | 文件存储、点播，如MP4/MKV，HLS/DASH等协议的MP4切片 | 实时传输（RTSP/RTP）           |
| ​**数据冗余**​   | 较低（参数集不重复）                         | 较高（关键帧前带SPS/PPS）         |
| ​**同步恢复能力**​ | 依赖容器索引                             | 起始码提供自同步能力               |

---

av_bitstream_filter_filter函数可以将AVCC格式转为Annex B格式，代码如下
```c
    AVBitStreamFilterContext* h264bsfc =  av_bitstream_filter_init("h264_mp4toannexb");   
    while(av_read_frame(ifmt_ctx, &pkt)>=0){  
        if(pkt.stream_index==videoindex){  
            av_bitstream_filter_filter(h264bsfc, ifmt_ctx->streams[videoindex]->codec, NULL, &pkt.data, &pkt.size, pkt.data, pkt.size, 0);  
            fwrite(pkt.data,1,pkt.size,fp_video);  
            //...  
        }  
        av_free_packet(&pkt);  
    }  
    av_bitstream_filter_close(h264bsfc);
```

处理后的字节流如下
```
处理前
00 00 00 21 67 4d 04 2a e8 a0 5a 09 37 fe 00 02 00 02 84 00 04 05 00 00 03 00 01 00 00 03 00 32 8f 08 84 65 80 00 00 00 04 68 ee 3c 80 00 00 8d dc 65 
处理后
00 00 00 01 67 4d 04 2a e8 a0 5a 09 37 fe 00 02 00 02 84 00 04 05 00 00 03 00 01 00 00 03 00 32 8f 08 84 65 80 00 00 00 01 68 ee 3c 80 00 00 01 65 88
```

此时没有插入SPS、PPS帧，需要重建packet
```c
    AVPacket *new_pkt = av_packet_alloc();
    uint8_t *new_data = av_malloc(prefix_size + pkt->size);
    
    memcpy(new_data, prefix_data, prefix_size);
    memcpy(new_data + prefix_size, pkt->data, pkt->size);
    
    av_packet_from_data(new_pkt, new_data, prefix_size + pkt->size);
    av_packet_copy_props(new_pkt, pkt);
    
    av_packet_unref(pkt);
    *pkt = *new_pkt;
    new_pkt->buf = NULL; // 防止双重释放
    av_packet_free(&new_pkt);
```

