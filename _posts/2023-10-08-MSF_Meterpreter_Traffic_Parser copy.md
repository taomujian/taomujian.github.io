---
layout: post
title: "MSF_Meterpreter_Traffic_Parser"
subtitle: 'MSF_Meterpreter_Traffic_Parser'
author: "taomujian"
header-style: text
tags:
  - MSF
  - 后渗透
  - 流量分析
---

## 前言

> 最近在攻防世界刷以往的CTF题目时,有道Misc题目叫做抓住了个黑客,是要分析MSF流量的,解密获得flag,题目给出了RSA私钥,又恰逢最近又在研究Cobalt Strike魔改以及免杀,参考网上公开的已有信息进行整理总结本文章.

> 首先[官方地址](https://github.com/rapid7/metasploit-framework/pull/8625)已经给出了Meterpreter的默认通信协议的流程与大概的组成结构.

## 组成部分

```
Values: [XOR KEY][session guid][encryption flags][packet length][packet type][ .... TLV packets go here .... ]
Size:   [   4   ][    16      ][      4       ][       4     ][     4     ][ ....          N          .... ]
```

### XOR KEY

> 前4个字节是异或用到的密钥,用开始的4字节异或密钥对后续报文进行异或解密,得到32字节的报文头

### session guid

> 会话id,占用16个字节,用来区分会话.在无有效负载的情况下,session guid的值为null,当会话建立时,会生成一个session guid并将其传递给Meterpreter实例.

### encryption flags

> 加密标志位,占用4个字节,用来区分此数据包是否已被加密以及用什么加密.然后如果无加密,包类型之后的数据是TLV报文,否则包类型之后的数据前16位是AES加解密用到的IV,然后才是是TLV报文,这个报文需要用AES(CBC模式,key、iv)解密后才能得到数据.例如,如果加密标志是AES256即值为1,它看起来像这样:

```
Values: ... [packet length][packet type][AES IV][ .... encrypted TLV .... ]
Size:   ... [       4     ][     4     ][  16  ][ ....        N      .... ]
```

> 加密标志位的值共有3种,如下

|  类型   | 值  |   含义
|  ----  | ---- | ----  
|  ENC_FLAG_NONE  | 0 | 不加密 
| ENC_FLAG_AES256  | 1 | AES-256加密 
| ENC_FLAG_AES128  | 2 | AES-128加密

### packet length

> 标志着这个数据包的长度为多少,占用4个字节.

### packet type

> 标志着这个数据包的类型是什么,占用4个字节.

> 数据包的类型如下:

|  类型   | 值  |   含义
|  ----  | ---- | ----  
|  PACKET_TYPE_REQUEST  | 0 | 请求报文
| PACKET_TYPE_RESPONSE  | 1 | 响应保文 
| PACKET_TYPE_PLAIN_REQUEST  | 10 | 明文请求包
| PACKET_TYPE_PLAIN_RESPONSE  | 11 | 明文响应包


### TLV

> 是一个数据结构,一个报文中有多个这样的数据结构,用于传递具体的信息.每个TLV中包含type、length、payload等信息.

> TLV具体的类型的值在源码中并不是直接给出的,而是要和一些常量进行或运算得到,用到的常量如下:

|  类型   | 值    | 含义
|  ----  | ----  | ----             
| TLV_META_TYPE_NONE         | 0         ｜  空
| TLV_META_TYPE_STRING       | 1 << 16   ｜  字符串
| TLV_META_TYPE_UINT         | 1 << 17   ｜  单元
| TLV_META_TYPE_RAW          | 1 << 18   ｜  原始
| TLV_META_TYPE_BOOL         | 1 << 19   ｜  布尔
| TLV_META_TYPE_QWORD        | 1 << 20   ｜  四字
| TTLV_META_TYPE_COMPRESSED  | 1 << 29   ｜  压缩
| TLV_META_TYPE_GROUP        | 1 << 30   ｜  组
| TLV_META_TYPE_COMPLEX      | 1 << 31   ｜  复杂

> TLV具体的类型如下:

|  类型   | 计算公式 |  值   | 含义
|  ----  | ----  | ----  | ----             
| TLV_TYPE_ANY         | TLV_META_TYPE_NONE  \| 0 |0  |  任意值
| TLV_TYPE_COMMAND_ID          | TLV_META_TYPE_UINT \| 1 |1 | 命令id
| TLV_TYPE_REQUEST_ID          | TLV_META_TYPE_STRING \|   2 |2 | 请求id
| TLV_TYPE_EXCEPTION           | TLV_META_TYPE_GROUP  \|   3 |3 | 异常的值
| TLV_TYPE_RESULT              | TLV_META_TYPE_UINT   \|   4 |4 | 结果的值
| TLV_TYPE_STRING              | TLV_META_TYPE_STRING \|  10 |10 | 字符串
| TLV_TYPE_UINT                | TLV_META_TYPE_UINT   \|  11 |11 | 单元
| TLV_TYPE_BOOL                | TLV_META_TYPE_BOOL   \|  12 |12 | 布尔
| TLV_TYPE_LENGTH              | TLV_META_TYPE_UINT   \|  25 |25 | 长度
| TLV_TYPE_DATA                | TLV_META_TYPE_RAW    \|  26 |26 | 任意数据
| TLV_TYPE_FLAGS               | TLV_META_TYPE_UINT   \|  27 |27 | 标志
| TLV_TYPE_CHANNEL_ID          | TLV_META_TYPE_UINT   \|  50 |50 | 渠道id
| TLV_TYPE_CHANNEL_TYPE        | TLV_META_TYPE_STRING \|  51 |51 | 渠道类型
| TLV_TYPE_CHANNEL_DATA        | TLV_META_TYPE_RAW    \|  52 |52 | 渠道数据
| TLV_TYPE_CHANNEL_DATA_GROUP  | TLV_META_TYPE_GROUP  \|  53 |53 | 渠道数据组
| TLV_TYPE_CHANNEL_CLASS       | TLV_META_TYPE_UINT   \|  54 |54 | 渠道类
| TLV_TYPE_CHANNEL_PARENTID    | TLV_META_TYPE_UINT   \|  55 |55 | 渠道类型
| TLV_TYPE_SEEK_WHENCE         | TLV_META_TYPE_UINT   \|  70 |70 | 寻找的开始时间
| TLV_TYPE_SEEK_OFFSET         | TLV_META_TYPE_UINT   \|  71 |71 | 寻找偏移量
| TLV_TYPE_SEEK_POS            | TLV_META_TYPE_UINT   \|  72 |72 | 寻找位置
| TLV_TYPE_EXCEPTION_CODE      | TLV_META_TYPE_UINT   \| 300 |300 | 异常代码
| TLV_TYPE_EXCEPTION_STRING    | TLV_META_TYPE_STRING \| 301 |301 | 异常的值
| TLV_TYPE_LIBRARY_PATH        | TLV_META_TYPE_STRING \| 400 |400 | 加载包的路径
| TLV_TYPE_TARGET_PATH         | TLV_META_TYPE_STRING \| 401 |401 | 目标路径
| TLV_TYPE_MIGRATE_PID         | TLV_META_TYPE_UINT   \| 402 |402 | 迁移目标的进程号
| TLV_TYPE_MIGRATE_PAYLOAD     | TLV_META_TYPE_RAW    \| 404 |404 | 迁移的payload
| TLV_TYPE_MIGRATE_ARCH        | TLV_META_TYPE_UINT   \| 405 |405 | 迁移目标的架构
| TLV_TYPE_MIGRATE_BASE_ADDR   | TLV_META_TYPE_UINT   \| 407 |407 | 迁移目标的基址
| TLV_TYPE_MIGRATE_ENTRY_POINT | TLV_META_TYPE_UINT   \| 408 |408 | 迁移目标的入口点
| TLV_TYPE_MIGRATE_SOCKET_PATH | TLV_META_TYPE_STRING \| 409 |409 | domain套接字路径,在linux上迁移目标中使用
| TLV_TYPE_MIGRATE_STUB        | TLV_META_TYPE_RAW    \| 411 |411 | 迁移存根
| TLV_TYPE_LIB_LOADER_NAME     | TLV_META_TYPE_STRING \| 412 |412 | 反射加载函数路径
| TLV_TYPE_LIB_LOADER_ORDINAL  | TLV_META_TYPE_UINT   \| 413 |413 | 反射加载函数序数
| TLV_TYPE_TRANS_TYPE          | TLV_META_TYPE_UINT   \| 430 |430 | 变换的通信类型
| TLV_TYPE_TRANS_URL           | TLV_META_TYPE_STRING \| 431 |431 | 传输使用的新URL
| TLV_TYPE_TRANS_UA            | TLV_META_TYPE_STRING \| 432 |432 | 传输使用的 user agent
| TLV_TYPE_TRANS_COMM_TIMEOUT  | TLV_META_TYPE_UINT   \| 433 |433 | 通信超时时间
| TLV_TYPE_TRANS_SESSION_EXP   | TLV_META_TYPE_UINT   \| 434 |434 | 回话是否过期
| TLV_TYPE_TRANS_CERT_HASH     | TLV_META_TYPE_RAW    \| 435 |435 | 证书hash
| TLV_TYPE_TRANS_PROXY_HOST    | TLV_META_TYPE_STRING \| 436 |436 | 代理服务器host
| TLV_TYPE_TRANS_PROXY_USER    | TLV_META_TYPE_STRING \| 437 |437 | 代理服务器用户
| TLV_TYPE_TRANS_PROXY_PASS    | TLV_META_TYPE_STRING \| 438 |438 | 代理服务器密码
| TLV_TYPE_TRANS_RETRY_TOTAL   | TLV_META_TYPE_UINT   \| 439 |439 | 重连尝试次数
| TLV_TYPE_TRANS_RETRY_WAIT    | TLV_META_TYPE_UINT   \| 440 |440 | 重连间隔
| TLV_TYPE_TRANS_HEADERS       | TLV_META_TYPE_STRING \| 441 |441 | 请求的自定义头
| TLV_TYPE_TRANS_GROUP         | TLV_META_TYPE_GROUP  \| 442 |442 | 传输组
| TLV_TYPE_MACHINE_ID          | TLV_META_TYPE_STRING \| 460 |460 | 机器id
| TLV_TYPE_UUID                | TLV_META_TYPE_RAW    \| 461 |461 | uuid
| TLV_TYPE_SESSION_GUID        | TLV_META_TYPE_RAW    \| 462 |462 | 会话id
| TLV_TYPE_RSA_PUB_KEY         | TLV_META_TYPE_RAW    \| 550 |550 | RSA公钥
| TLV_TYPE_SYM_KEY_TYPE        | TLV_META_TYPE_UINT   \| 551 |551 | 对称密钥类型
| TLV_TYPE_SYM_KEY             | TLV_META_TYPE_RAW    \| 552 |552 | 对称密钥
| TLV_TYPE_ENC_SYM_KEY         | TLV_META_TYPE_RAW    \| 553 |553 | RSA公钥加密对称密钥后的值
| TLV_TYPE_PIVOT_ID              | TLV_META_TYPE_RAW    \|  650 |650 | pivot监听器id
| TLV_TYPE_PIVOT_STAGE_DATA      | TLV_META_TYPE_RAW    \|  651 |651 | staged在新连接上的数据
| TLV_TYPE_PIVOT_NAMED_PIPE_NAME | TLV_META_TYPE_STRING \|  653 |653 | pipe名


## 交换密钥流程

> 1.用漏洞或者其他方式如钓鱼执行stager

> 2.stager连接到MSF

> 3.MSF将一个(比stager小得多的)metsrv DLL发到stager

> 4.stager下载这个stage(即metsrv DLL),将其复制到RWX内存并调用它

> 5.metsrv开始执行并接管stager的通信

> 6.然后,MSF开始AES密钥协商阶段：

> &emsp; i.MSF生成一个2048位RSA密钥对(公钥和私钥)

> &emsp; ii.公钥以pem格式添加到数据包中,然后使用命令core_negotiate_aes将数据包发送到Meterpreter.

> &emsp; iii.Meterpreter接收数据包.如果AES协商功能不存在,则会像往常一样发生"未找到方法"错误.

> &emsp; iv.
如果支持该命令并且支持公钥处理(此时支持Windows Meterpreter),则：

> &emsp; &emsp; a.提取并解析公钥.

> &emsp; &emsp; b.使用加密安全随机数生成器生成256位AES密钥.

> &emsp; &emsp; c.AES密钥与会话数据存储在Meterpreter的存储器中.

> &emsp; &emsp; d.用给出的公钥对AES密钥进行加密,这个加密信息作为初始消息的一部分.

> &emsp; &emsp; e.Meterpreter用包含加密后的AES-256密钥的数据包响应MSF.

> &emsp; v.如果支持该命令但不支持公钥处理(可能不是这种情况,但可能是其他Meterpreter)：

> &emsp; &emsp; a.生成AES密钥并与Meterpreter会话数据一起存储.

> &emsp; &emsp; b.AES密钥可以以明文形式返回.

> &emsp; vi.MSF接收命令的结果：

> &emsp; &emsp; a.如果消息不受支持,则没有AES密钥与会话相关联

> &emsp; &emsp; b.如果消息完全受支持,则MSF使用与公钥一起生成的私钥解密AES密钥,并且AES密钥与会话处理程序一起存储.

> &emsp; &emsp; c.如果消息部分受支持,则MSF复制明文的AES密钥并将其与会话处理程序一起存储.

> &emsp; &emsp; d.在所有情况下,生成的RSA密钥对都会被销毁.

> 7.如果密钥协商成功,则使用共享的AES密钥加密后面所有的数据包,并且在数据包上设置加密标志.

> 8.如果密钥协商不成功,后面的的数据包将以纯文本发送,就像平常一样(与此同时仍然会XOR).


## 解析代码

```

import zlib
from Crypto.Cipher import AES
from Crypto.PublicKey import RSA
from Crypto.Cipher import PKCS1_v1_5
from Crypto.Util.number import long_to_bytes

def xor_decrypt(xor_key, data):
    
    """
    进行异或解密
    
    :param str xor_key: 异或用到的key
    :param str data: 要解密的数据
    
    :return str result: 解密后的数据
    
    """
    
    result = b''
    for i in range(0, len(data)):
        result += (data[i] ^ xor_key[i % 4]).to_bytes(1, 'big')

    return result

def get_aes_key(private_key, payload):
    
    """
    利用RSA私钥获得AES KEY
    
    :param str private_key: RSA私钥
    :param str payload: AES被加密后的数据
    
    :return str aes_key: 获取到的AES KEY
    
    """
    
    rsa_key = RSA.import_key(private_key)
    cipher = PKCS1_v1_5.new(rsa_key)
    aes_key = cipher.decrypt(payload, None)
    return aes_key

def parse_packet_header(data):
    
    """
    解析数据包头部信息
    
    :param str data: 数据包头部数据
   
    :return dict packet_header: 解析后的数据
    
    """
    
    packet_header = {}
    packet_header['xor_key'] = data[0:4]
    data = xor_decrypt(packet_header['xor_key'], data[4:32])
    packet_header['session_guid'] = data[0:16].hex()
    packet_header['enc_flags'] = int.from_bytes(data[16:20], 'big')
    packet_header['pkt_length'] = int.from_bytes(data[20:24], 'big')
    packet_header['pkt_type'] = int.from_bytes(data[24:28], 'big')
    return packet_header

def parse_tlv_unit(data):
    
    """
    解析TLV数据结构
    
    :param str data: TLV数据
   
    :return dict tlv_unit: 解析后的数据
    
    """
    
    tlv_unit = {}
    data_header = data[0:8]
    tlv_unit['type'] = int.from_bytes(data_header[4:8], 'big')
    tlv_unit['length'] = int.from_bytes(data_header[0:4], 'big')
    tlv_unit['payload'] = data[8:tlv_unit['length']]
    if tlv_unit['type'] & (1 << 29) == (1 << 29):
        tlv_unit['payload'] = zlib.decompress(tlv_unit['payload'])
        tlv_unit['type'] = tlv_unit['type'] ^ (1 << 29)
    
    return tlv_unit

def run(data, aes_key = b""):
    
    """
    函数主入口
    
    :param int data: 数据包信息
    :param bytes aes_key: aes key
   
    :return dict result: 解析后的数据
    
    """
    
    packet_header = parse_packet_header(data[0:32])
    data_raw = xor_decrypt(packet_header['xor_key'], data[32:])
    # Decrypt packet if enc_flags are set
    if packet_header['enc_flags'] != 0:
        if len(aes_key) != 32:
            raise "Invalid AES Key or AES Key was not set"
        iv = data_raw[0:16]
        cipher = AES.new(aes_key, AES.MODE_CBC, iv)
        data_raw = cipher.decrypt(data_raw[16:])
        
    raw = data_raw
    tlv_units = []
    while len(raw) != 0:
        tlv_unit = parse_tlv_unit(raw)
        raw = raw[tlv_unit['length']:]
        tlv_units.append(tlv_unit)

    result = {
        'xor_key': packet_header['xor_key'], 
        'session_guid': packet_header['session_guid'], 
        'enc_flags': packet_header['enc_flags'], 
        'pkt_length': packet_header['pkt_length'], 
        'pkt_type': packet_header['pkt_type'],
        'tlv_units': tlv_units
    }
    
    return result

if __name__ == '__main__':

    data = 0x2926993adf8c06b90319d8059f51ec99b401f9b42926993a292698b52926993b2926991c2927993b4a49eb5f7648fc5d4652f05b5d43c64e4550c65f4745eb435952f0554726993a290f993b2924ac0e181fac0f1110a90d1f10ab031e13ac081b16aa0f1a16a0031813af0a1f15993a2926953a2b24be3a2926983a2927913a2d24b039ec75e27c93f3e0167fb28dd2c98ea0fd2ff42887d961f4e205a619311cf4b8dd93f189d24b8eb45af277bc76cc0ea8d5a0b3332812e6c8f73cced9f1d32189311dc13c30b80ebf944228979fd974a6ce23aacf386ebb5dbc1ac414a49678e6425944edf4a56f263a9c9e2a80e4d1884b35fc8da6e68bff64d33265350bf3e2b72dc5b0f0580cc12925074fc5505c9cd250b279150b9701cf5b2666ae9d0a0722f8b72aed8ffaa51f43594ebebe13b5bcb75e8a33c8cd396445c0c9da3d5a5b862bde38f62d13afd5387a62abce70e3d4192e3e96996ceaa0d1b0c7796028bdeabfbd8eddd13576f23135e1e7b06a0057a4b07bc786394ba1cd691547fad5db3a2926953a2b269d3a2926993a2926813a2d27542832d93a2fb12d08349729256baeb187
    p = run(long_to_bytes(data))

    for i in p['tlv_units']:
        if i['type'] & 551 == 551:
            # 0x01 => AES256
            sym_key_type = int.from_bytes(i['payload'], 'big')
        if i['type'] & 552 == 552:
            sym_key_value_encrypted = i['payload']

    with open('key.pem', "r") as key_file:
        rsa_private_key = key_file.read()
        
    aes_key = get_aes_key(rsa_private_key, sym_key_value_encrypted)
    print(aes_key)

    print(run(long_to_bytes(0x7e30af14889a3097540fee2bc847dab7e317cf9a7e30af157e30af7c7e30af144a1999b733e68b894ee6b64581a31e4f800f1e2413c6ffabc4690df0ffdf5ad592f3d3de710b43f85611c113dff2d03a4a91fc6cf046cafc37d823f986dee137e05368e717be6acbd31e87435f8bc7601dae969be7b5888050246f2f01b9cab8), aes_key))
    print(run(long_to_bytes(0x2ab4ef55dc1e70d6008bae6a9cc39af6b7938fdb2ab4ef542ab4effd2ab4ef54b835d361e2e784b4e2e9168aa4cd074c5b434009388fb2550a98ee6d49ea108c50b1f1b6c4c17a6fa0b01cd5e04f5fe5ba536da008841f20cef4324fe752f92923ea05e1d683bf991306b6878e301a68c933579c58c47482afc576545cc3cc2610d3f2ee70a3a36404996ff877b3a7756efe57bd043ad32f04ca9a137846cfeb6f53c817f32986117c0f26fc30b60b70a1f072987c80eda747fcc5ae85da2ed3), aes_key))
    print(run(long_to_bytes(0x2926993adf8c06b90319d8059f51ec99b401f9b42926993b292699b22926993b0438eb2dc77a917a03839c6bb9f6afb8192bbf8baf251e45d7f08cb799975ab8c4661cee35f7d4c9c63b7d1607fabdd602313a66d04c4fe8570ddd650eb2e9595358f04c0e781a347ddb62027dea0a0e899709fa3eeaa238600e4abb48d28fce74817d9ce4a627d36b5ee9e9d59719e4ae246ef1d39f8468dbe5ca236dbee597), aes_key))
    print(run(long_to_bytes(0x2ab4ef55dc1e70d6008bae6a9cc39af6b7938fdb2ab4ef542ab4eddd2ab4ef54a64307198e637c0fd0dfc74aad3b1fbcd26e62d8113e05e6af14d5cf78b82c11a0761ba6e95a984da59db055e44b9d5830cf6ec34a0e7ff0686fef9472546c93cd169eebf86e933f424f8a6f116bfa447a286bed01168019eb59eb83f5d455be5bf7fe55c49db5e7752b9b3135d1a4ccb6da5ca2c37af0c9311e6e335bc57642debf54952abd91b589527436429128a062f587edf20a215bf4161253898cc89a146c92c77cc6b34dd782f97007444f674dba89901758b8a7e55f1c82c7f0f0af78b41bae90d0d008518668d7fadf570da991365e09048b70d162b0e0b281cc28df2b64e759e480fa3dab1bb8635f85237205c66d6926fd85c829921956403b454598d57a7185ec80ea7ca1a8494d3c6ed7b7580dfbaaa585f018ca8c8efd5dd15b38e9281c3fc0385312544d7f24cb0fd3e8392fce9f07501028be0b6fa656dccb9247ab9b99fd3ff8cc04199d26398ea7fb4fac7be96769c8fa5f7e0d63cc524f0699104e89c91411d5a2168bbd7b59237b65143f81107667fddc2ba0e5c5382cf4dd672950da2ee088ba8e74039e872d89e69aeca904dac87463bb691075cd888a93d07c9ca792ffc53bd25a8487389a1776ffdfc4232028ea08d5f555d7738574ddd942d95d7cc5d0b086798c991ed140c72257b039a7aa8b61727976c6cd939b3815a4e04e94bafe8d5cb69f9dc11930431207ddc03517ccfd5e4499609b87914c40be1481a5ef174a45ce0fa70e8f6a683a2975a1b6b1905613444dac3ba12e8902d41f6ba2e7c940e347a6fdade78c6560e79f2f2313a90b74af7c1137e0bb3cd9f721af2a13deade2551a4be241b095992a608fb0e5aa063a10207414c2f2c18f7272bcffe0ecac0cfa8b4c2bf2bb3232044cccecaf91ed310d07ca1c), aes_key))
    print(run(long_to_bytes(0x2ab4ef55dc1e70d6008bae6a9cc39af6b7938fdb2ab4ef542ab4ef9d2ab4ef5415d984862f64b0e891b131ecc5d2a06c8ec29332954025fa1b2807cb007edcb73bba65e5c12cb31a40db6b26a4fa05a8d3e5a67d3871228a8cd6b6f6ce75a263823fecc17be818e1bd27a4cc9d512372e5f2f58f0aea4a11bfb631ceee791e0f5682f4dd94fc8b1aa59514037a8d76ff8b8d72ca32c2a299049f9089f430e403b9c64419d1eb8b6577c440bec9a16a5c51081ad4040fa32759ef37740c20a1a3893f511b08d37926ae67abedfa4084d1c3cb97f47a130dea915cd000fb51873c), aes_key))
    print(run(long_to_bytes(0x2ab4ef55dc1e70d6008bae6a9cc39af6b7938fdb2ab4ef542ab4effd2ab4ef549568c9bca4f5201e93dbdb4852d51967a2e9b23cc1a88ce4b2a37950450a8586cde6043906b7803994a8ce6db44ede0258b2b5d29318145c26075ddb963991ad85007119e0ad4de637818aa78e8fbe14631c113894ff87aefc336a427eb6c90cfc4e8b2efb088d63c1a301381cf90270e2b7c96371530800724ed4762f963d17b9ce23ab12cc4d656fbfc999d144f5f7cedee8eb9cc56200d51bede00a242bea), aes_key))
    print(run(long_to_bytes(0x2ab4ef55dc1e70d6008bae6a9cc39af6b7938fdb2ab4ef542ab4eddd2ab4ef54e1a628a81569018cd0f5bf315d7b320e5b77b0f8b8345c79e0fc4594a74a390315f368d3c474883c2aa91a6e0573ff936f5bb5aba31790051bb71dc50bba249affef1b06d4fa9d297abbc5142c40a90d031af7e4cf885d6ad3e09bc0338a8d0e8675b7f332797ec9d87c48b37e8a2e3a75c8f80d78c3de96029eb298203090029a32a858b55714fb5a805dcd52c02734d04d04285f729285ba731207c73ee0c7c2ef52ecb3e973e787cce4fd720c83a3a221f8a98de6180c939785c52fec692acb34f3cf2a36de23a850928dd405afd0abd376c4dac3582153ea8de6e3cdf19fd4706688782e9a48b8b05c591abcc892555e2a061601f73a3498bdcbd1f1baf142a6471065d42312b48ded576689a1bb5bcdb877e8160995be495603a4405e2a2892f2b650760640286b1ad93461909518fd8392929e251157b0c5461825badfa065a42bb12b60bb7bc0fdbd62bc376b188f705e76820e832c0226f536b0c2620ab917dbebb346c9bf01c2a0c81832fed64e6f64407d3d7f051c6c8460900c50d2d8c361f57afb5958888c1bf983bd314ea54697d87ce70b9230837a98398b4d1cba073b86615712078079362f6fb1b0c80e2b25af230ce8016a0b0115607f730195f0811d21db64cc422e5b0f73d6fbcc88a0843fbdbed51db87851c940c99e4a0d72d3fa7604caf585b40b962f93cbd252373b78bbecf891975d8e152358534d999fe0008453b9f6952acccdce89d2f31021f67b5d4ae2099ff7b38d92a9f2f7541e9fcf9a89d9b8146dfbee579bfbd933ca07bb7ba04efda4c24cf5e9ca43afdd5bdaea05677fd293bcd74c31b37b24196f8e0d0be9e79831f7794ea91a52db03e8f35230ed75adbb5a0974380905b510883af3be066a03b984af5effc060), aes_key))
    print(run(long_to_bytes(0x2ab4ef55dc1e70d6008bae6a9cc39af6b7938fdb2ab4ef542ab4ef9d2ab4ef54a74abf08695adaad8be818da14b4dadc981f6157553f5d955e82e35ff422847f969300e94f38c7ec84dcc4fef465f3e62b0fbc142e3761c036cdfe4c5e7a880445ad31a51ee1112a89fec6e8693852d739fbd91d05327c0d6cda8cb26a17c3ae9b43f28774f991afb2545a69210f47d82339c219d7f7bfbd5d76c2a4a82e376ef1d2ff1142a0713b1e055bd6c6c78866e4da63c65cb6d161da9d55222c90984a6295e5b68bf4dce2ce684b9d067c02352b8f3fa457783c911a340f2b9c552fa6), aes_key))
    print(run(long_to_bytes(0x2ab4ef55dc1e70d6008bae6a9cc39af6b7938fdb2ab4ef542ab4effd2ab4ef545042629ac50792c2d9f387304e12245dcb7bc137fcf865d9548f52c1f4b4d7662a2e5b4a04ce32d51bdb99b725807534a9b288fdcce5512a4801b71d7a97de1624de6175b884a4ef080e2b8b39683a98d86cb9ff8ce0e1c1c7300fc582f2ed5d8f8a2b23b77ddb01e65e059738478f9aab61ec7e93f8f17eb1903786ae1c654edaf7ca60cd5c4d7012bffea32c54f5091952d4ddf7573efe77bda7d974bc4bb3), aes_key))
    print(run(long_to_bytes(0x2ab4ef55dc1e70d6008bae6a9cc39af6b7938fdb2ab4ef542ab4eddd2ab4ef54a0c32aa9adca2454808fa09531321b03dade8023acab14d49f4bd36589277683be8ad1f5483cff7411c3db741baca849b7647a94650614d5ca05acadffd7ead06a172d74f279bbcf94a7c1365a1f6c624ae42f39644511bcecdddd516ba7e64280b6ee9300837628b207057b3c1d67abace40bb66ea250c438daac90110725da12c7b4fdca846a84ff3c842355eb5ae9c5814a83f0266e8e304b7c18c99ee2e6dddba999ef6c16bd459f7e5e0133a89f93c139887816477b133e70e333fd2c8187c39b504d983dded4b8d2ed57124dd564ecf36212a8208864dbaa756317d084d85a54066fb1179efb7e1d2ae41d49e5f79691f83676cd6d9d9f7e69ef5be859398c00bbed2dae001f80f7e784ed3ecd37e860a1a0b7eada9cae6cd34a78fd383ffb006597de7a427cf3fe2109edbbc9bbd8a4e58b7e9137c836b0ee899e8228c536e78664cf6f1626a298b635250d1e3c8b67dc33ce96ad33d8562e7bbc2eac3b0268aaec137377fa29542af2ffb940ad168fa6bc28a24d671c71432ac557026f7bd6cb5c4a3d6295ba03f9732138b74c6c0190c6d03cc65ec7ff90dbbd6a033d3d8f464af5523ddc1eec87f4a8920204bea066b62463f7055fc3a3443eb6831e068ff7c7884534c8c961b59ab638cc2c68f996c1e04938e6b36982940470775a8d67842d62db2a8fa297057816e3c1a8494ab8d8605cd363e7b02398f9462b29d4c6ae6d054e34df2de2960da4d2ecdc03da04a8b8c0f0429f5b6d69f654f895d39d538d772a54aa99e6ff4c9f2553a8d562e67de80bb6ce6f7b64f4a15006a97f3e356201b0ec1f35cb01d8e3121658f2f07db46174c1332dcda43e11ea9aab1d6da405ddeea61ebaca8db07aab89dbc971a107144f77d1e34fa5d162da5a), aes_key))
    print(run(long_to_bytes(0x2ab4ef55dc1e70d6008bae6a9cc39af6b7938fdb2ab4ef542ab4ef9d2ab4ef547eeb277e493e9d2dad9be2f9bb47f0b6355a47223f26e74ea01c55dbd02f3a256bebb806798caae2813f140c39b000a2b8d366c28b88bf92e71ba5760993884f3661f258cac484858353a5e430f20ab1ea10e2b9d58a92d395804ac305c8620cc25a0e41c33543723871e32dd2551b740e083d9524d8ab912d7d33ebc574ac103aa3b6d960d6812a40eb371be855151846d027dac6694e01d3ce26bcb648309c721316d0baf1c167a1ff4f21181a4f45fd1bb8b4219047d7cdcbbc0befff021c), aes_key))
    print(run(long_to_bytes(0x2ab4ef55dc1e70d6008bae6a9cc39af6b7938fdb2ab4ef542ab4effd2ab4ef5416d7e52b7aafb3d98d01695ad4dad9c844c16f1f5ab9c6524bd888fa10316d0f8cc027e58ca6abac24105f2b006c5c1687ee80da56344e9915c81c88195897056dbe672a7d5ed6060ca3b079edb8bd7c7137ba8f78160c16fec5ce685439cebd31761f47527f04cf840da0cdb430e5f71f13c6bd45ee04e8aa6670c2f3265b8663c1c9b600f68afd7b943ae566f9e9bf8eab2ea9714bf1fdce89e89e522af060), aes_key))
    print(run(long_to_bytes(0x2ab4ef55dc1e70d6008bae6a9cc39af6b7938fdb2ab4ef542ab4eddd2ab4ef549ed176f02b7f872de8b1a1c09035e4c445e4eefd55f8b21766398cebdbbbac51638200564e211698faade01639890646b5a7df93dbfe2ab6fed342d53fe139f6d698075298003c873a3b394d52d5fa2db04ca3ced01e26f505a9dae196e528cf229390e572fd3c4f49712d4552f182e903e4cab21880065813920d8eb27d40a57b691a8cef7e83abaaec5b6d27b653942481771236a9e1fa96de9fa7013539f4f51a632c7d5377087c1e5ff04ece37e31be0201d8acb15f3ffea099ba7161a8b099fe436babe9ec972f5acd7b1b13c8fc63531a0e48613fe0828f70263c21af801c1cdbae91dc7f05dfd348e9120230dbbe52dfbc7d6f4ff4281a26e04d64ac17be5ffdd46c38bfd4d5f23e536bb976210e764922a9fa1a4a0d32676c2a3ae77376b0fdee2e484877cca94e36d38b7fe4370ab2cd24dd1c1daf37a3025b82c4cdd43808997fbab7ae1bbfb38039f225f238a05c7a43d75d325964cc8709d06e4a0201760bd978ae7719af4f44bde5e55bcfd4cb5e20f71fb9c2ac3bf71eb7b4552f1ea466e11218b291d2311344e72232c71286fc1b41464aff44975bda745c78ae7309c1c6cafdc22833de0585e879d8fa1257421c6372cf5d6159eb927a64b32b4a26f93e119983e1b75dcb51588d666ce3f66078a9f9ed1b843bd82a0d8ba2020869fd49ef9434610961c7d95ce2e43cdf3b4d8cf1a1eaeadcfc0deb5651a6af86cecce1ab15172160eb25c17421e65d34e4efbdd6917860610360b662aa4d7e3fa62157ab07d1e2a2af88f1387fd82df72155f0602ece0ce617215c5bef59a393e0d7cd2572d848510c5c09c78916d6240778f2d2efead1dd1795096554a8100d61bcf1a7d51e5fd22e9633d0673268745990827da0027c7812fbdefb0d1), aes_key))
    print(run(long_to_bytes(0x2ab4ef55dc1e70d6008bae6a9cc39af6b7938fdb2ab4ef542ab4ef9d2ab4ef54f6cf111c29995e8ddf9cfc68568b392a827cbd823b57a5b72e8a5befc5581afc9a3d62dc240ec020910740d3d782ec0d25232da13f1f1634569d749562e8e9c65de0016af28faa98409554e21b7932eb5587670935ddc66138139828832adb339b6e390090432eb4ea1f292ad9918345903aae300c1ece3e8e04b19d3a95856d908975f370b4e4dbb054f7d2b62325e95578df134e23e15478855f5015318f78d6e5cf9ea0808207153432f12f6c916dba4d73a5b603abd5944ff81873278e0c), aes_key))
    print(run(long_to_bytes(0x2ab4ef55dc1e70d6008bae6a9cc39af6b7938fdb2ab4ef542ab4effd2ab4ef548a457a00a50fdd743b833ace6e884f5e1ee7d93e1d5465013006b47661b78a31cb121fddc9cad7539a91d2f876775063efb3517bcadb658160ec961b4482cd1ef4ac797936a9061a48318c67ff7e5d2055f8d2b0e7e5fbca0fef43935e23afe3daaee2c0400d950f96c2a41e3a36dc82b6f02f8ce2031d95fee1c4129e3b11e352afa470bc219fd9ebc0f70cec328071cd9ee9c423559debc72a4b82997bcf72), aes_key))
    print(run(long_to_bytes(0x2ab4ef55dc1e70d6008bae6a9cc39af6b7938fdb2ab4ef542ab4eddd2ab4ef5417935c499443c593861b309077d9bde14d10e835529bf1ae2d48cf5eb8732fe500780aef921e1e4c9d42400ef3aa0d8642e9a53a17cc4115093bdfe64b15af6ddb9ba8a928e8aee3c693712a25cc9776956dd2bd88e2eb1b340e49c7c06004025d4018b5cb963edb72d36335c0e46599684bbacba9be975919d374cf36f8036bc85dc7b6eb28f2db271d89f2a6884850fba6a4223a607cfc6ea27ad5967fbf111193c80dc5651e5875bdde2590d3759c6229012c82f571bac7e9c34278a2c6510c5440a563282078f75437a6b83a2795628747c8a42cd1962d5892a301f240fa0ab2ff2ffdc37fd9c31681c7be58bb211fa2a65656612936aee9610785fba11a8fb5eeffecffd28da7257a06147b8b3f7b4840b9e14ff58a62de5d895a72192db8da9bc8eab0018dba287c2259a2424ef2c07320ecf8333bad402ed073fdfec3a9074e4ac483adffaae6827363292969b0a8b667ce87608c280eea0d588636862e7c685a1098c4cfe5be6ced796dcb68a13dc8d627c0a1507f29118c873538eca45775ff552df3caeeffd83dda2b690c577a4ac2a6b2ee1fb6292f30b92f653817b1347a2511f4ce6b16b8b326561788207153b8e34abe2c01f18084c39d8d112f30abba79c18fd41364dc6569700ba8e50bfd3698c364c980185e41983b5b8a2123e05c8d2d20e8ff344d45eba3a696c168a41828e89dfd52b933ae2fda939115e33388f8d001f222968cdff85014df2a13012e8ed2de7b7e344016a31fe040c4f499b11405331825d792889f6ff88de901bb22c63b3b2ed03230f466dcfd266b9de98a9555e55612be2488cd4725d8dbc082821d5de0579c71755d2f1546e88eb20a0f4bfd15d6ee97a560e73fd9a428af958e2d8f4359ddb89de5d02e387a), aes_key))
    print(run(long_to_bytes(0x2ab4ef55dc1e70d6008bae6a9cc39af6b7938fdb2ab4ef542ab4ef9d2ab4ef54673f20ea63ed7dcee5ad41191624f11654116ea6e529188198f3d8c99fb8c7a4cd02d95c775fcd228293c4e07f457fff20f4dd86beecd707729f7e320557c541568e499880bd3d60edd8d517ef847314fa92add61d1ebad74bb38fe874defc674d5c0a185089963ec494879a56af9fd814a476e41e057c4c1a00a9d1f69e1b7c45bbb769791636ae9a42514c366eee52f9f5346c92c6fa9138845f0c3eaf3a30f3f71c048ee4f9fd3e5dec452cde0bf13794ad43c7c7334c1b841294ed61d3bc), aes_key))
    print(run(long_to_bytes(0x2ab4ef55dc1e70d6008bae6a9cc39af6b7938fdb2ab4ef542ab4effd2ab4ef54bf7b08dd116835468ca87c273505a310dab1059bfd1ba816574d1e3f35f8429788dccf3848b0872bfc5b5d149ca192532ee30b2291bb57fca70ed1fb4f7653445f1c373a74f0e324ea23efdf3c3be17aa51821c4fac4a2b26f73f7ac306ad49b7cbaf5edbdd2e9962a1ee0d3127ae049a5823e8f2ab88879dbb679f37323e1497fdea2d807af51c94a7f8a9a27dbffcc65ee8778848a5315a994b9a7c71c914d), aes_key))
    print(run(long_to_bytes(0x2ab4ef55dc1e70d6008bae6a9cc39af6b7938fdb2ab4ef542ab4efcd2ab4ef543db8b8d5acda5a1235b3ad7d07db722102cd9e53750cef7e31131b2d59afa2206d5adc9c877d8ea88bef37dfdff1ffe46e42443d29429793ab306042ee0b2173de2b24e750a0ec61eae0fb8dc5ba0c9961330cfc253aa550022cbf03120d8561e96d54ff94e08ca352d1c069065caceb101e16845fe69db34893f8cb9f3c1e7749b573c1e3f4f2caf2486620cab399fd), aes_key))
    print(run(long_to_bytes(0x2ab4ef55dc1e70d6008bae6a9cc39af6b7938fdb2ab4ef542ab4efdd2ab4ef54e28b751ce10cff077f8445dfe8ddc32363ccef83833a6e9c72f75026116d0253b77e884b483db4b28ab58a9397911ed8afd892a938b4eaee9806d96c070f1f1e587bca16e8e8e16e2b2af5a530e84cc95221f13ce3c66b391318c11f6740bb16d7216dbf70004cbdf4e451b13cb70223235f7e63a33d32c631f512bd5ca7511b), aes_key))
    print(run(long_to_bytes(0x2ab4ef55dc1e70d6008bae6a9cc39af6b7938fdb2ab4ef542ab4ef8d2ab4ef549df15c8d63e084ea41796e1080c422c10c4af26c0b5563cb3058bd6e4aa3f2cc22de1c6edf7e6468ad7e1ff8fa11bd0d2955218906c89bc9c1aca68485ba41d6c1363d26e1ca77e379eede5843a78cf2890a4a4d48d532564d081638df298d10b5b8e57b4714d29993407e26b857a054f18cbba0769a9d7a3ab9b2fc503b9b4e0f13e3e0e4086d9def861eb75b54df80426839c540b6836702d3585c1d11b1fcc1f56d6a2a58bee99fa22ea3792bd764075cdd0bfe11d2a7953c9174210b35afc3a38ab0ff13abebf06b74046bc902df), aes_key))
```

## 总结

> 以上就是MSF与Meterpreter通信的全流程,以及通信的数据包组成结构.总结看下来,并没有多复杂,加密数据是用RSA+AES结合的方式来进行加密的,关键是二者之间交换AES密钥,同时又为了防止MITM,采用了RSA来交换AES KEY.

## 参考

> https://blog.sparta-en.org/2021/10/10/Meterpreter%E9%80%9A%E8%AE%AF%E5%88%86%E6%9E%90/

> https://github.com/rapid7/metasploit-framework/pull/8625