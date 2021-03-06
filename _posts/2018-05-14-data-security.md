---
layout: post
title: "数据安全及各种加密算法对比"
date: 2018-05-14
tags: iOS底层
---

平时开发中不仅会遇到各种需要保护用户隐私的情况，而且还有可能需要对公司核心数据进行保护，这时候加密隐私数据就成为了必要。然而市场上存在着各种各样的抓包工具及解密算法，甚至一些公司有专门的逆向部门，这就加大了数据安全的风险，本文将通过以下几个方面对各种加密算法进行分析对比：

* Base64编码（基础）
* 单项散列函数 MD5、SHA1、SHA256、SHA512等
* 消息认证码 HMAC-MD5、HMAC-SHA1
* 对称加密 DES|3DES|AES（高级加密标准）
* 非对称加密 RSA
* 数字签名
* 证书

通常我们对消息进行加解密有两种处理方式：

1. 只需要保存一个值，保证该值得机密性，不需要知道原文（用户登录）
2. 除了保证机密性外还需要对加密后的值进行解密得到原文

## Base64编码
由于我们可能对各种各样的数据进行加密，比如：视频、音频、文本文件等，所以加密之前我们需要统一文件类型，然后再进行加密处理。

* Base64编码

```
// 要编码的字符串
NSString *str = @"haha";

// 转换成二进制文件
NSData *data = [str dataUsingEncoding:NSUTF8StringEncoding];

// 进行base64编码
NSString *dataStr = [data base64EncodedStringWithOptions:kNilOptions];

NSLog(@"%@", dataStr);

```

* Base64解码


```
// 先对数据进行解码
NSData *encData = [[NSData alloc]initWithBase64EncodedString:dataStr options:kNilOptions];
    
// 将二进制数据转换成字符串
NSString *encStr = [[NSString alloc]initWithData:encData encoding:NSUTF8StringEncoding];
    
NSLog(@"%@", encStr);
```

接下来分析一下Base64的[编码过程](https://zh.wikipedia.org/wiki/Base64)，参考维基百科：

![](http://otogtitz7.bkt.clouddn.com/2018-05-14-15260869886055.jpg)

> 如果要编码的字节数不能被3整除，最后会多出1个或2个字节，那么可以使用下面的方法进行处理：先使用0字节值在末尾补足，使其能够被3整除，然后再进行Base64的编码。在编码后的Base64文本后加上一个或两个=号，代表补足的字节数。也就是说，当最后剩余两个八位字节（2个byte）时，最后一个6位的Base64字节块有四位是0值，最后附加上两个等号；如果最后剩余一个八位字节（1个byte）时，最后一个6位的base字节块有两位是0值，最后附加一个等号。 参考下表：

![](http://otogtitz7.bkt.clouddn.com/2018-05-14-15260870722695.jpg)

* Base64编码原理

1. 将所有字符串转换成ASCII码
2. 将ASCII码转换成8位二进制
3. 将二进制三位归成一组（不足三位在后边补0），再按每组6位，拆成若干组
4. 统一在6位二进制后不足8位的补0
5. 将补0后的二进制转换成十进制
6. 从Base64编码表取出十进制对应的Base64编码

若原数据长度不是3的倍数时且剩下1个输入数据，则在编码结果后加2个=；若剩下2个输入数据，则在编码结果后加1个= 

如上面的例子：

原数据为`A`，数据长度为1，1 % 3 = 1   后面加两个==

原数据为`bc`，数据长度为2，2 % 3 = 2  后面加一个=

* Base64编码的特点

1. 可以将任意的二进制数据进行Base64编码。
2. 所有的数据都能被编码为并只用65个字符就能表示的文本文件。
3. 编码后的65个字符包括A~Z,a~z,0~9,+,/,=
4. 对文件或字符串进行Base64编码后将比原始大小增加33%。
5. 能够逆运算
6. 不够安全，但却被很多加密算法作为编码方式

## 单项散列函数

单向散列函数也称为消息摘要函数、哈希函数或者杂凑函数。
单向散列函数输出的散列值又称为消息摘要或者指纹

特点：
1. 对任意长度的消息散列得到散列值是定长的
2. 散列计算速度快，非常高效
3. 消息不同，则散列值一定不同
4. 消息相同，则散列值一定相同
5. 具备单向性，无法逆推计算

经典算法：

* MD4、MD5、SHA1、SHA256、SHA512等

安全性：

* md5解密网站：http://www.cmd5.com
* MD5的强抗碰撞性已经被证实攻破，即对于重要数据不应该再继续使用MD5加密。

**疑问一：单项散列函数为什么不可逆？？**

原来好多同学知识知道md5加密是不可逆的，却不知道是为什么，其实散列函数可以将任意长度的输入经过变化得到不同的输出，如果存在两个不同的输入得到了相同的散列值，我们称之为这是一个碰撞，因为使用的`hash`算法，在计算过程中原文的部分信息是丢失了的，一个MD5理论上可以对应多个原文，因为MD5是有限多个，而原文是无限多个的。

网上看到一个形象的例子：2 + 5 = 7，但是根据 7 的结果，却并不能推算出是由 2 + 5计算得来的

**疑问二：为什么有些网站可以解密MD5后的数据？？**

MD5[解密网站](http://www.cmd5.com/)，并不是对加密后的数据进行解密，而是数据库中存在大量的加密后的数据，对用户输入的数据进行匹配（也叫暴力碰撞），匹配到与之对应的数据就会输出，并没有对应的解密算法。

#### MD5改进

由以上信息可以知道，MD5加密后的数据也并不是特别安全的，其实并没有绝对的安全策略，我们可以对MD5进行改进，加大破解的难度，典型的加大解密难度的方式有一下几种：

1. 加盐（Salt）：在明文的固定位置插入随机串，然后再进行MD5
2. 先加密，后乱序：先对明文进行MD5，然后对加密得到的MD5串的字符进行乱序
3. 先乱序，后加密：先对明文字符串进行乱序处理，然后对得到的串进行加密
4. 先乱序，再加盐，再MD5等
5. HMac消息认证码

也可以进行多次的md5运算，总之就是要加大破解的难度。

#### Hmac消息认证码（对MD5的改进）

**原理：**

1. 消息的发送者和接收者有一个共享密钥
2. 发送者使用共享密钥对消息加密计算得到MAC值（消息认证码）
3. 消息接收者使用共享密钥对消息加密计算得到MAC值
4. 比较两个MAC值是否一致

**使用：**

1. 客户端需要在发送的时候把（消息）+（消息·HMAC）一起发送给服务器
2. 服务器接收到数据后，对拿到的消息用共享的KEY进行HMAC，比较是否一致，如果一致则信任

![](http://otogtitz7.bkt.clouddn.com/2018-05-14-15261103539157.jpg)



## 对称加密算法

对称加密的特点：

* 加密/解密使用相同的密钥
* 是可逆的

经典算法：

* DES  数据加密标准
* 3DES  使用3个密钥，对消息进行（密钥1·加密）+（密钥2·解密）+（密钥3·加密）
* AES  高级加密标准

密码算法可以分为分组密码和流密码两种：

* 分组密码：每次只能处理特定长度的一zu数据的一类密码算法。一个分组的比特数量就称之为分组长度。
* 流密码：对数据流进行连续处理的一类算法。流密码中一般以1比特、8比特或者是32比特等作为单位俩进行加密和解密。

分组模式：主要有两种

* ECB模式(又称电子密码本模式)
    * 使用ECB模式加密的时候，相同的明文分组会被转换为相同的密文分组。
    * 类似于一个巨大的明文分组 -> 密文分组的对照表。

![](http://otogtitz7.bkt.clouddn.com/2018-05-14-15261268137714.jpg)

**某一块分组被修改，不影响后面的加密结果**

* CBC模式(又称电子密码链条)
    * 在CBC模式中，首先将明文分组与前一个密文分组进行XOR(异或)运算，然后再进行加密。
    * 每一个分组的加密结果依赖需要与前一个进行异或运算，由于第一个分组没有前一个分组，所以需要提供一个初始向量`iv`

    ![](http://otogtitz7.bkt.clouddn.com/2018-05-14-15261269484225.jpg)

**某一块分组被修改，影响后面的加密结果**
    
#### 代码演示两种分组模式

* AES - ECB模式

加密：
 
```
    /**
     *  加密字符串并返回base64编码字符串
     *
     *  @param string    要加密的字符串
     *  @param keyString 加密密钥
     *  @param iv        初始化向量(8个字节)
     *
     *  @return 返回加密后的base64编码字符串
     */
    NSLog(@"%@", [[EncryptionTools sharedEncryptionTools] encryptString:@"haha" keyString:@"abc" iv:nil]);
    
    // 输出 MIoAu+xUEpQZSUmkZUW6JQ==
```

解密：

```
NSLog(@"%@", [[EncryptionTools sharedEncryptionTools] decryptString:@"MIoAu+xUEpQZSUmkZUW6JQ==" keyString:@"abc" iv:nil]);
    
// 输出 haha
```

* AES - CBC模式

加密：

```
uint8_t iv[8] = {1,2,3,4,5,6,7,8};

NSData *data = [[NSData alloc] initWithBytes:iv length:sizeof(iv)];

NSLog(@"%@", [[EncryptionTools sharedEncryptionTools] encryptString:@"haha" keyString:@"abc" iv:data]);
    
// 输出 E/wWqUTiw/E+1DThAzV39A==
```

解密：


```
NSLog(@"%@", [[EncryptionTools sharedEncryptionTools] decryptString:@"E/wWqUTiw/E+1DThAzV39A==" keyString:@"abc" iv:data]);
    
// 输出 haha
```

**对称加密存在的问题？？**

很明显，对称加密主要取决于秘钥的安全性，数据传输的过程中，如果秘钥被别人破解的话，以后的加解密就将失去意义

![](http://otogtitz7.bkt.clouddn.com/2018-05-14-15261292485801.jpg)

其实有点儿类似于我们平常看的谍战类的电视剧，地下党将情报发送给后方，通常需要一个中间人将密码本传输给后方，如果中间人被抓交出密码本，那么将来所有的情报都将失去意义，由此可见情报工作多么的重要！！！

对称密码体制中只有一种密钥，并且是非公开的，如果要解密就得让对方知道密钥。所以保证其安全性就是保证密钥的安全，而非对称密钥体制有两种密钥，其中一个是公开的，这样就可以不需要像对称密码那样传输对方的密钥了

## 非对称加密

鉴于对称加密存在的风险，非对称加密应运而生

特点：

* 使用公钥加密，使用私钥解密
* 公钥是公开的，私钥保密
* 加密处理安全，但是性能极差

非对称密码体制的特点：算法强度复杂、安全性依赖于算法与密钥，但是由于其算法复杂，而使得加密解密速度没有对称加密解密的速度快


经典算法：

* RSA

###### RSA算法原理
    * 求N，准备两个质数p和q,N = p x q
    * 求L,L是p-1和q-1的最小公倍数。L = lcm（p-1,q-1）
    * 求E，E和L的最大公约数为1（E和L互质）
    * 求D，E x D mode L = 1
###### RSA加密实践
    * p = 17,q = 19 =>N = 323
    * lcm（p-1,q-1）=>lcm（16，18）=>L= 144
    * gcd（E,L）=1 =>E=5
    * E乘以几可以mode L =1? D=29可以满足
    * 得到公钥为：E=5,N=323
    * 得到私钥为：D=29,N=323
    * 加密 明文的E次方 mod N = 123的5次方 mod 323 = 225（密文）
    * 解密 密文的D次方 mod N = 225的29次方 mod 323 = 123（明文）

###### openssl生成密钥命令

* 生成强度是 512 的 RSA 私钥：$ openssl genrsa -out private.pem 512
* 以明文输出私钥内容：$ openssl rsa -in private.pem -text -out private.txt
* 校验私钥文件：$ openssl rsa -in private.pem -check
* 从私钥中提取公钥：$ openssl rsa -in private.pem -out public.pem -outform PEM -pubout
* 以明文输出公钥内容：$ openssl rsa -in public.pem -out public.txt -pubin -pubout -text
* 使用公钥加密小文件：$ openssl rsautl -encrypt -pubin -inkey public.pem -in msg.txt -out msg.bin
* 使用私钥解密小文件：$ openssl rsautl -decrypt -inkey private.pem -in msg.bin -out a.txt
* 将私钥转换成 DER 格式：$ openssl rsa -in private.pem -out private.der -outform der
* 将公钥转换成 DER 格式：$ openssl rsa -in public.pem -out public.der -pubin -outform der

###### 非对称加密存在的安全问题

原理上看非对称加密非常安全，客户端用公钥进行加密，服务端用私钥进行解密，数据传输的只是公钥，原则上看，就算公钥被人截获，也没有什么用，因为公钥只是用来加密的，那还存在什么问题呢？？那就是经典的**中间人攻击**

![](http://otogtitz7.bkt.clouddn.com/2018-05-14-15261322915101.jpg)

废了半天劲画的图，太low了，我还是自己总结一遍吧！！！

**中间人攻击详细步骤：**

1. 客户端向服务器请求公钥信息
2. 服务端返回给客户端公钥被中间人截获
3. 中间人将截获的公钥存起来
4. 中间人自己伪造一套自己的公钥和私钥
5. 中间人将自己伪造的公钥发送给客户端
6. 客户端将重要信息利用伪造的公钥进行加密
7. 中间人获取到自己公钥加密的重要信息
8. 中间人利用自己的私钥对重要信息进行解密
9. 中间人篡改重要信息（将给客户端转账改为向自己转账）
10. 中间人将篡改后的重要信息利用原来截获的公钥进行加密，发送给服务器
11. 服务器收到错误的重要信息（给中间人转账）

**疑问一：为什么会造成中间人攻击？？**

造成中间人攻击的直接原因就是客户端没办法判断公钥信息的正确性。

**疑问二：怎么解决中间人攻击？？**

需要对公钥进行**数字签名**。就像古代书信传递，家人之所以知道这封信是你写的，是因为信上有你的签名、印章等证明你身份的信息。

数字签名需要严格验证发送发的身份信息！！！

## 数字证书

数字证书包含：

* 公钥
* 认证机构的数字签名（权威机构CA）

数字证书可以自己生成，也可以从权威机构购买，但是注意，自己生成的证书，只能自己认可，别人都不认可.

权威机构签名的证书：
以[GitHub官网](https://github.com/)为例，Chrome浏览器打开网址，地址栏有一个小绿锁，点击，内容如下：

#### 权威机构认证的证书：

![](http://otogtitz7.bkt.clouddn.com/2018-05-14-15261345080716.jpg)

可以看到连接是安全的，点击证书可以看到详细信息
![](http://otogtitz7.bkt.clouddn.com/2018-05-14-15261345685235.jpg)

这是由权威机构认证的证书，但是是需要花钱的，一年至少得一两千，所以也有些公司用自己签名的证书，自己签名的证书不被信任，可能会提示用户有风险，比如原来的12306网站，现在大多数网站也都采用了CA签名数字证书进行签名，自己做签名的也不好找了！！！12306都改成CA认证的了。。。

#### 自己生成一个证书



* 生成私钥 

```
openssl genrsa -out private.pem 1024
```

* 创建证书请求 

```
openssl req -new -key private.pem -out rsacert.csr
```

* 生成证书并签名，有效期10年 

```
openssl x509 -req -days 3650 -in rsacert.csr -signkey private.pem -out rsacert.crt
```

* 将 PEM 格式文件转换成 DER 格式

```
openssl x509 -outform der -in rsacert.crt -out rsacert.der
```

* 导出P12文件

```
openssl pkcs12 -export -out p.p12 -inkey private.pem -in rsacert.crt
```
 
 ![](http://otogtitz7.bkt.clouddn.com/2018-05-14-15263070093426.jpg)

 
**注意：**

 * 在iOS开发中，不能直接使用 PEM 格式的证书，因为其内部进行了Base64编码，应该使用的是DER的证书，是二进制格式的
 * OpenSSL默认生成的都是PEM格式的证书

代码演示：

```
// p12 是私钥
    // .der 是公钥
    // 非对称加密，使用公钥加密，私钥解密
    
    // 加载公钥
    [[RSACryptor sharedRSACryptor] loadPublicKey:[[NSBundle mainBundle] pathForResource:@"rsacert.der" ofType:nil]];
    
    // 对数据加密
    NSData *data = [[RSACryptor sharedRSACryptor] encryptData:[@"hahaha" dataUsingEncoding:NSUTF8StringEncoding]];
    
    // 对加密得到的密文进行base64编码打印
    NSLog(@"%@", [data base64EncodedStringWithOptions:kNilOptions]);
    
    // 输出结果：PflhCgTVNegcQXrb39RJOoxCRRIHuZ3LN0/hoxTDFBbC+8yKjp0m+/hxVUWBVsTo28WnNFCAFfrQ2of5SkqttD51a5eLb21R7bQSQRxg/gVZ5hePcE3vh7Slfcxm2qJM+J8hRWDP/MF4BiDLXI9ZqTpLCSS5mjJtmUBf2wNvI1Y=
    
    // 私钥解密
    
    // 加载私钥
    [[RSACryptor sharedRSACryptor] loadPrivateKey:[[NSBundle mainBundle] pathForResource:@"p.p12" ofType:nil] password:@"123456"];

    // 解密
    NSData *decryDeta = [[RSACryptor sharedRSACryptor] decryptData:data];
    
    NSLog(@"%@", [[NSString alloc] initWithData:decryDeta encoding:NSUTF8StringEncoding]);
    
    // 输出：hahaha
```

# [代码地址](https://github.com/czjwarrior/DailyPractice)

## 总结

至此数据安全和加解密相关结算完毕，写博客的过程，也是学习的过程，断断续续的写了三四天，总算写完了，同时也对原来一些模糊的概念有了更清晰的认识，写的这篇文章看了[文顶顶老师](https://www.jianshu.com/u/c5703017b9f5)的视频，受益匪浅，十分感谢，最近学习发现，学的越多，感觉会的越少，时间十分的不够用。同时渴望遇到一些希望进步、不甘平凡的同行！！！共勉！！！







