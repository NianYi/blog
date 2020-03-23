了解MP4封装结构之前，首先知悉几个概念：
1、sample：video sample简单理解为一帧视频，audio sample为一段连续的音频帧；
2、track：表示一些sample的集合，对于媒体数据来说，track表示一个视频或音频序列；
3、box：MP4的结构是由box组成的，box类型有很多种，不同类型的box作用不同，box之间可以嵌套；
下面罗列了一个MP4文件包含的各类box及主要内容，可见box类型都是4字节的。
```
.MP4的封装结构
│───ftyp 指示文件类型、版本号，比如: mp42、isom类型
│───free 内容无关紧要删除无影响，可以忽略
│───mdat 存储媒体数据
└───moov 存储metadata
    │───mvhd 包含文件的duration、timescale及播放速率（倍速播放）等
    │───trak 可以有多个，描述详细的码流信息
    │   │───tkhd 包含track的duration、width、height（并不是像素宽高）等
    │   │───edts 可选不详
    │   └───mdia 描述真正的码流信息
    │       │───mdhd 包含track的duration、timescale
    │       │───hdlr 包含track的类型，vide、soun、hint分别对应视频、音频和hint track
    │       └───minf 描述具体的媒体信息
    │           │───vmhd 视频合成模式：拷贝原图像、与opencolor合成；
    │           │───smhd 立体声平衡参数
    │           │───dinf 描述媒体定位信息
    │           │   └───dref 暂时不太理解
    │           └───stbl 描述了该track所有sample时间、位置以及编解码信息
    │               │───stsd 包含了视频的编码类型、宽高，音频采样、声道等
    │               │───stts 定义了解码时间与sample序号的关系，可以通过时间找sample
    │               │───stss 说明哪些sample是关键帧
    │               │───ctts 与stts共同决定sample的显示时间
    │               │───stsc 映射了sample到thunk的关系
    │               │───stsz 包含每个sample的大小，这个表较大
    │               └───stco 定义了每个thunk在整个媒体流中的位置
    └───udta
        └───meta
            │───hdlr
            └───ilst
*tips：
1、mvhd中的timescale仅用于mvhd；mdhd中的timescale才是pts或dts的timescale。
2、为了便于理解，以seek某个时间点的帧为例，将一些重要类型串联起来：
首先根据mdhd中的timescale将时间点转换为时间戳，查找stts表找到对应sample，
再通过stsc表找到所属thunk，最后查询stco得到thunk的首地址，进而定位到具体的帧
```
###一、stts/ctts 类型（两种类型格式完全一样，区别是entry含义不同）
时钟是视频处理中经常打交道也是较抽象的概念，时钟信息主要保存在stts和ctts两类box中，stts保存sample的解码时间dts，ctts保存了pts与dts的差值，计算公式为：pts(n) = stts(n) + ctts(n)
a）不存在b帧时，就不存在ctts类型，pts(n) = stts(n) 即 pts等于dts；
b）存在b帧时，pts(n) = stts(n) + ctts(n) 需要结合两个表计算出pts；

|  字段 | 字节数  | 意义  |
| :------------ | :------------ | :------------ |
|   box size|  4 |   box大小|
|   box type|  4 |   box类型|
|   version|  1 |   box版本|
|   flag|  3 |  |
|   entry count|  4 |   entry的数量|
|   entry(stts)|  8 |   包含两个字段(count, duration)，表示连续count个sample具有相同的duration|
|   entry(ctts)|  8 |   包含两个字段(count, offset)，表示连续count个sample的PTS与DTS的差值同为offset，offset可以为负值|

```
实例分析stts类型box：
00 00 00 18 73 74 74 73    00 00 00 00 00 00 00 01    ....stts........
00 00 1C 19 00 00 02 00   
解释：
“00 00 00 18”：box size为24bytes
“73 74 74 73”：是stts类型
“00 00 00 00”：version=0，flag=0
“00 00 00 01”：该box只有1个entry，只有一个entry表示视频为固定帧率
“00 00 1C 19 00 00 02 00”：entry（7193， 512），表示前7193个sample的duration均为512
ctts类型分析省略....
```
###二、mdhd类型

|  字段 | 字节数  | 意义  |
| :------------ | :------------ | :------------ |
|   box size|  4 |   box大小|
|   box type|  4 |   box类型|
|   version|  1 |   box版本|
|   flag|  3 |  |
|   creation time|  4 |  创建时间 |
|   modification time|  4 |   修改时间|
|   time scale|  4 |   时基|
|   duration|  4 |   时长|
|   language|  2 |   媒体语言码|
|   pre-defined|  2 |   .|

###三、stss类型

|  字段 | 字节数  | 意义  |
| :------------ | :------------ | :------------ |
|   box size|  4 |   box大小|
|   box type|  4 |   box类型|
|   version|  1 |   box版本|
|   flag|  3 |  |
|   entry count|  4 |   entry的数量|
|   entry|  4 |   关键帧的sample序号|

```
实例分析stts类型box：
00 00 01 0C 73 74 73 73    00 00 00 00 00 00 00 3F    ....stss........
00 00 00 01 00 00 00 5F    00 00 00 C5 00 00 01 75    ...............u
00 00 02 6F 00 00 02 9E    00 00 03 51 00 00 04 25    ...o............
00 00 04 AD 00 00 04 DD    00 00 05 9C 00 00 05 E2    ................
00 00 06 D4 00 00 07 28    00 00 07 6A 00 00 07 91    ...........j....
00 00 07 BD 00 00 07 F4    00 00 08 BF 00 00 09 50    ................
00 00 09 A8 00 00 0A 51    00 00 0B 41 00 00 0C 2F    ................
00 00 0C 7F 00 00 0C CE    00 00 0D 03 00 00 0D 47    ................
00 00 0D 7F 00 00 0D B4    00 00 0D F3 00 00 0E 38    ................
00 00 0E 5C 00 00 0E 7F    00 00 0F 21 00 00 0F 5F    ................
00 00 0F A2 00 00 0F D0    00 00 0F F2 00 00 10 1E    ................
00 00 10 5B 00 00 10 8B    00 00 10 F6 00 00 11 2C    ................
00 00 11 C4 00 00 12 2E    00 00 12 8A 00 00 12 F8    ................
00 00 13 4B 00 00 13 FD    00 00 14 2D 00 00 14 77    ...............w
00 00 15 6E 00 00 15 F1    00 00 16 73 00 00 17 15    ...n.......s....
00 00 17 87 00 00 18 3B    00 00 18 D6 00 00 19 8A    ................
00 00 1A 84
解释：
“00 00 01 0C”：box size为268bytes
“73 74 73 73”：是stss类型
“00 00 00 00”：version=0，flag=0
“00 00 00 3F”：有63个关键帧，太多了这里只分析几个
“00 00 00 01”：第一个关键帧的sample序号1
“00 00 00 5F”：第二个关键帧的sample序号95
....
“00 00 1A 84”：最后一个关键帧的sample序号6788
```
###四、stsd类型

|  字段 | 字节数  | 意义  |
| :------------ | :------------ | :------------ |
|   box size|  4 |   box大小|
|   box type|  4 |   box类型|
|   version|  1 |   box版本|
|   flag|  3 |  |
|   entry count|  4 |   entry的数量|
|   entry|  不固定 |   比如avc、mp4a等，这是另一个类型|
avc结构

|  字段 | 字节数  | 意义  |
| :------------ | :------------ | :------------ |
|   avc1 size|  4 |   大小|
|   avc1 type|  4 |   类型|
|   reserve|  6 |   保留|
|   data_reference_index|  2 |  |
|reserve |16 ||
|Width | 2| 宽|
|Height | 2| 高|
|.... | | |
|NALU 长度|2||
|SPS个数，低五位有效|2||
|SPS长度 |2 | | 
|SPS内容|12||
|PPS个数|1||
|PPS长度|2||
|PPS内容|4| .|
