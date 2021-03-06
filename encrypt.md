## 加密算法比较

| 加密类型 | 代表算法 | 加密方式 | 解密方式 | 优缺点 |适用场景 |
| :- | :- | :- | :- | :- | :- |
| 非对称加密 | RSA(公钥秘钥算法) | 公钥加密 | 私钥解密 | 优点只交换公钥不交换秘钥，减少秘钥被窃取可能性，比较安全；缺点慢 | https前期建立加密信道、git 
| 对称加密 | AES DES | 秘钥（+初始iv）加密 | 秘钥（+初始iv）解密 | 优点快；缺点需要交换秘钥，加大了秘钥被获取的可能 | https安全加密信道
| 哈希 | hash MD5 SHA | 通过算法生成固定长度的值，任何初始数据的改变会导致结果的变化（算法要防碰撞），结果极难逆运算回初始值 | 不可解密 | 解密难道高，数据安全 | 摘要计算、数据完整性校验。密码等数据存储 
| 文字编码 | base64 UTF8 URLEncode | 不算是加密，应该说是编码方式，方便数据传输、存储 | 知道是什么编码方式即可解密 | ** | http请求params

