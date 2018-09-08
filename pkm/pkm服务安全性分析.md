# pkm服务安全性分析

## 前言

通过分析pkm的安装部署环境和结合源代码走读的方式来分析pkm服务的安全性。

pkm安装部署环境通过跳板机访问。

pkm源代码位于项目[github/ofgp/pkm](https://github.com/ofgp/pkm)。

## 分析

### 本地信息收集

通过跳板机登录到安装pkm服务的linux服务器上面，然后 ` ps -ef ` 查看服务器上面启了哪些进程，看到一个进程也许和pkm服务相关：

```
admin    14077 14000  4 Aug22 ?        15:45:29 java -Xmx1024m -jar target/pkm.jar --spring.config.location=conf/application.properties --password=${PASSWORD}
```
但是通过 ` env ` 并且没有看到有PASSWORD的环境变量。基于项目组的服务一般都是采用docker的方式部署，使用 ` docker ps ` 看到起来了一个docker容器：

```

CONTAINER ID        IMAGE                                    COMMAND                  CREATED             STATUS              PORTS                    NAMES
11b476c1a34c        hub.ibitcome.com/ofgp/pkmserver:master   "/entrypoint.sh supe…"   2 weeks ago         Up 2 weeks          0.0.0.0:8976->8976/tcp   pkm_keystore_1

```
OK，那么我们进入到容器里面 ` ps -ef ` 看看启动了哪些进程，然后也看到一条和pkm有关进程：

```
[root@11b476c1a34c /app]# 
[root@11b476c1a34c /app]# ps -ef
UID        PID  PPID  C STIME TTY          TIME CMD
worker       1     0  0 Aug22 ?        00:04:15 /usr/bin/python /usr/bin/supervisord
worker      16     1  0 Aug22 ?        00:00:00 python /app/bin/kill_supervisor.py
worker      17     1  4 Aug22 ?        15:45:59 java -Xmx1024m -jar target/pkm.jar --spring.config.location=conf/application.properties --password=${PASSWORD}
root     17884     0  0 18:00 pts/0    00:00:00 /bin/bash
root     17904 17884  0 18:00 pts/0    00:00:00 ps -ef
[root@11b476c1a34c /app]# 
[root@11b476c1a34c /app]# 

```
然后我们再看一下环境变量：

```
[root@11b476c1a34c /app]# 
[root@11b476c1a34c /app]# env
HOSTNAME=11b476c1a34c
TERM=xterm-256color
HISTSIZE=10000
LC_ALL=zh_CN.UTF-8
HISTFILESIZE=10000
PASSWORD=T2ZncDE4QGJpdG1haW4K
PAGER=less
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
PWD=/app
JAVA_HOME=/usr/lib/jvm/default-jvm
EDITOR=vim
LANG=zh_CN.UTF-8
PS1=\[\e[36m\][\[\e[36m\]\u\[\e[31m\]@\h \[\e[32m\]\w\[\e[36m\]]\[\e[0m\]\$ 
SHLVL=1
HOME=/root
PROMPT_COMMAND=history -a
CHARSET=UTF-8
HISTTIMEFORMAT=%F %T 11b476c1a34c:root 
HISTFILE=/var/log/history/root/201809061800.hist
_=/usr/bin/env
[root@11b476c1a34c /app]# 
[root@11b476c1a34c /app]# 
```
好了，这里看到了变量PASSWORD=T2ZncDE4QGJpdG1haW4K，不过目前为止我也不知道启动这个服务的时候呆这个参数是拿来干嘛的，后续继续分析。

对于进程启动加载的参数application.properties，我们可以看到：

```
[root@11b476c1a34c /app]# cd conf/
[root@11b476c1a34c /app/conf]# ls
application.properties  supervisord.conf
[root@11b476c1a34c /app/conf]# cat application.properties 
#http服务端口
server.port = 8976
#数据持久化目录，请确保keystore对此目录具有文件读写权限
pkm.data-path=/data/pkm/
[root@11b476c1a34c /app/conf]# 
[root@11b476c1a34c /app/conf]#
```
两个变量很好理解，并且这里看到启动的是一个http协议的服务的话，很可能是一个web进程。

好了，信息收集到这里就差不多了。

下面继续分析源代码。

## 源码分析

将github上面下载到源码导入到eclipse，然后加载各种依赖。

从源码可以看到是一个spring boot启动到spring mvc到web服务。找到整个程序的入口com.rst.pkm.PkmApplication，为何断定这里是程序的入口，因为这个类上面被注解了 `  @SpringBootApplication ` 啊。从main函数可以看到：

```
	public static void main(String[] args) {
	    String password = null;
	    if (args != null) {
            for (int i = 0; i < args.length; i++) {
                if (args[i].startsWith("--password=")) {
                    password = args[i].substring("--password=".length(), args[i].length());
                    break;
                } else if (args[i].startsWith("-password=")) {
                    password = args[i].substring("-password=".length(), args[i].length());
                    break;
                }
            }
        }

		if (StringUtils.isEmpty(password)) {
			logger.error("You must startup with your keystore password!");
			System.exit(-1);
		} else {
	        //for load encrypted data
            Constant.ADMIN_KEY = AESUtil.aesKeyFrom(password);
        }

		SpringApplication.run(PkmApplication.class, args);
	}
```
这里看到提取出变量password，并且判断是否为空。在不为空的情况下通过aesKeyFrom函数生成ADMIN_KEY，类型为字节数组。当然目前还不知道ADMIN_KEY是拿来干嘛的，待后面继续分析。

前面讲过，这个服务本质是一个spring mvc的web工程，spring mvc一般都有Controller，Service，Repository对应于MVC。

当然本工程大量采用了注解的方式，这样给我们提供了源码分析很多方便。

我们先找到Controller的包com.rst.pkm.controller，里面有两个Controller：GodController和KeyController。两个controller分别是用于管理和对外提供服务。那么我们先分析管理的controller。

### GodController分析

GodController的路径是 ` /god ` 。有三个接口分别是` /createService ` ，` /config/{op}/{serviceId}/{ip}v ` ，` /createSignature ` 。

#### /createService 

` /createService `对应的实现方法是：

```
public String generateService() {
    logger.info("/createService");
    ResGenerateService resGenerateService = godService.generateService();
    return "serviceId: " + resGenerateService.getServiceId() + "\n"
            + "privateKey: " + resGenerateService.getPrivateKey() + "\n";
}
```

这里我们只需要重点关注

```
ResGenerateService resGenerateService = godService.generateService();
```
跟踪函数 `generateService()`的实现，在com.rst.pkm.service.impl.GodServiceImpl类里面实现了`generateService()`函数：

```
public ResGenerateService generateService() {
    String serviceId = UUID.randomUUID().toString();
    byte[] privateKey = secureRandom.nextBytes(32);
    BigInteger privateIntValue = new BigInteger(1, privateKey);
    ECPoint multiply = Constant.CURVE.getG().multiply(privateIntValue);
    byte[] publicKey = multiply.getEncoded(false);
    byte[] aesKey = secureRandom.nextBytes(16);

    ServiceProfile serviceProfile = new ServiceProfile();
    serviceProfile.setAesHex(Converter.byteArrayToHexString(aesKey));
    serviceProfile.setPrivateKey(Converter.byteArrayToHexString(privateKey));
    serviceProfile.setPublicKey(Converter.byteArrayToHexString(publicKey));
    serviceProfile.setServiceId(serviceId);
    serviceProfileDao.save(serviceProfile);

    ResGenerateService res = new ResGenerateService();
    res.setPrivateKey(Converter.byteArrayToHexString(privateKey));
    res.setServiceId(serviceId);

    return res;
}
```
从上面的代码可以看到。

- serviceid由UUID生成，这个肯定是唯一并且是随机化的字符串。
- 私钥是随机生成的一个长度为32的字节数组。
- 公钥由私钥根据椭圆曲线算法导出。
- 这里还生成了一个长度为16的字节数组aesKey。

>注意：
>这里的secureRandom是自己继承了java库的SecureRandom并且从新实现了种子的选取和随机数的生成函数。这里的种子选取自 ` /dev/urandom ` 。同时随机数生成函数大量依赖于时间戳。

实例化了ServiceProfile对象，把aesKey、私钥、公钥、serviceid四个变量封装到ServiceProfile对象里面，调用DAO层的ServiceProfileDao类的save函数保存ServiceProfile对象。

ServiceProfileDao类的save函数为：

```
public boolean save(ServiceProfile profile) {
    byte[] randomKey = FileUtil.loadRandomKey(savePath);
    byte[] encryptAesHexBytes = AESUtil.aesEncrypt(profile.getAesHex().getBytes(), randomKey);
    profile.setAesHex(Converter.byteArrayToHexString(encryptAesHexBytes));

    serviceProfileMap.put(profile.getServiceId(), profile);
    FileData data = new FileData();
    data.setRandomKey(Converter.byteArrayToHexString(randomKey));
    data.getServiceProfiles().addAll(serviceProfileMap.values());
    boolean success;
    byte[] content = AESUtil.aesEncrypt(new Gson().toJson(data).getBytes(), Constant.ADMIN_KEY);
    success = FileUtil.save(Converter.byteArrayToHexString(content), savePath + FILE_NAME);
    if (success) {
        success = FileUtil.backup(savePath + FILE_NAME, savePath + BACKUP_FILE_NAME);
    }

    AESUtil.wipe(randomKey);
    return success;
}
```
从上面代码可以看到，实现使用loadRandomKey函数获取一个叫randomKey的字节数组，这里的savePath为

```
@Value("${pkm.data-path}")
private String savePath;
```
通过注解的方式去获取 ` pkm.data-path ` 变量的值。前面分析 ` application.properties ` 的时候说过，里面有一个变量就是 ` pkm.data-path=/data/pkm/ ` 。loadRandomKey函数的代码如下，

```
public static byte[] loadRandomKey(String path) {
    byte[] aesKey;
    if (!hasCreatedRandomKey) {
        synchronized (hasCreatedRandomKey) {
            BtRandom secureRandom = new BtRandom();
            aesKey = secureRandom.nextBytes(16);
            save(Converter.byteArrayToHexString(aesKey), path + randomKeyFileName);
            hasCreatedRandomKey = true;
        }
    }

    File file = new File(path + randomKeyFileName);
    if (file.exists()) {
        byte[] hexBytes = load(path + randomKeyFileName);
        return Converter.hexStringToByteArray(new String(hexBytes));
    }

    BtRandom secureRandom = new BtRandom();
    aesKey = secureRandom.nextBytes(16);
    save(Converter.byteArrayToHexString(aesKey), path + randomKeyFileName);
    return aesKey;
}
```

loadRandomKey函数的主要逻辑是判断是hasCreatedRandomKey变量是否已经创建，如果没有就随机生成一个长度为16的字节数组，并且在创建成功以后把hasCreatedRandomKey变量设置为true。并且通过save函数将生成的数据保存到硬盘的文件里面。文件名为random-key。

然后读取此文件并且把结果返回。

回过头来看ServiceProfileDao类的save函数，在获取到randomKey以后，将此作为aes的加密输入密钥对前面生成的aesKey进行加密，并且将加密后了的aesKey更新到输入的ServiceProfile这个对象里面。

下一步有一个Map为

```
Map<String, ServiceProfile> serviceProfileMap
```
将ServiceProfile对象的serviceid作为key，ServiceProfile对象为value保存到serviceProfileMap里面。

然后这里有一个FileDate对象定义如下：

```
private class FileData {
    private String randomKey;
    private List<ServiceProfile> serviceProfiles = new ArrayList<>();
}
```
从代码里面看到有两个变量。

回到save函数，将randomKey保存到FileData对象，将ServiceProfile保存serviceProfiles这个类型为ServiceProfile的List里面。然后将FileData对象转换为一个json格式的数据并且使用Constant.ADMIN_KEY作为输入密码进行aes加密。Constant.ADMIN_KEY在前面信息收集阶段已经分析过是怎么来的了。现在知道了Constant.ADMIN_KEY是拿来干嘛的。

然后调用`FileUtil.save(Converter.byteArrayToHexString(content), savePath + FILE_NAME);`把数据持久化的保存到硬盘上面，这里的savePath已经分析过了，` service-profile ` 为 `service-profile ` 。

在数据保存成功以后，还要进行备份一份。

最后函数generateService把serviceid和私钥封装到ResGenerateService对象里面，因为前面的接口实现的函数要返回servicid和私钥给接口调用者。

基本上 ` /generate ` 接口的流程走完了。

>这里需要注意的是，在ServiceProfileDao里面有一个init的函数，并且被注解为@PostConstruct。对于这个注解详细的定义可以自行百度或者是google，简单来说就是一个spring扫描javabean的时候会初始化。
>我们来看看这个init函数是拿来干嘛的，通过分析init函数源码主要实现了几个功能：
>- 初始化前面分析到的serviceProfileMap这个HashMap。
>- 在service-profile和备份的service-profile文件不为空的情况下提取出来数据，并且在经过相关的解密后从新把数据封装到serviceProfile和serviceProfileMap。至于具体为什么要这样做，在分析了签名接口后，就明白了。签名需要使用到的私钥这些，并不会从service-profile文件里面去取，而是直接从前面这两个对象去取值。我猜测是这里的初始化就是防止掉电之类导致服务重启后内存里面这些数据是空的，因此在加载Javabean的时候就进行初始化。


至于这个接口生成的公私钥对以及aesKey拿来做什么的，可以从ofgp网关的配置上看得出来

- 这个接口生成的私钥是拿来配置到网关，网关发送到pkm数据都要使用此私钥来签名，然后pkm接收到了以后要验签。
- aesKey在分析了KeyController以后，才知道是拿来做作为aes加密的输入密钥对 ` /generate ` 接口生成的私钥加密的。

#### /config/{op}/{serviceId}/{ip}

此接口的实现函数是：

```
public String allowIp(
        @ApiParam(value = "添加/删除白名单ip", required = true) @PathVariable("op") int op,
        @ApiParam(value = "添加/删除白名单ip", required = true) @PathVariable("serviceId") String serviceId,
        @ApiParam(value = "添加/删除白名单ip") @PathVariable(value = "ip") String ip
) {
    logger.info("/config");
    if (op == 0) {
        godService.addValidIp(serviceId, ip);
    } else if (op == 1) {
        godService.delValidIp(serviceId, ip);
    } else if (op == 2) {
        return godService.getService(serviceId) + "\n";
    } else if (op == 3) {
        godService.lockService(serviceId);
    } else if (op == 4) {
        godService.unLockService(serviceId);
    } else  {
        return "unsupported op\n";
    }

    return "op success!\n";
}
```
从代码来看就是很简单了，对白名单的ip地址进行增删以及锁定与否。白名单位于ServiceProfile对象里面的变量  ` private Set<String> allowIps; ` 。是否锁定也是在此对象里面。




#### /createSignature

此接口的实现函数是：

```
public CommonResult<String> generateSignature(
        @ApiParam(value = "请求server的serviceId", required = true) @RequestParam(value = "serviceId") String serviceId,
        @ApiParam(value = "待签名数据", required = true) @RequestParam(value = "input") String input
) {
    logger.info("/createSignature");
    return CommonResult.make(godService.generateSignature(input, serviceId));
}

```
函数很简单，我们具体分析 ` godService.generateSignature(input, serviceId) `，generateSignature函数在GodServiceImpl里面实现了。

```
public String generateSignature(String input, String serviceId) {
    ServiceProfile serviceProfile = profileFrom(serviceId);

    ECDSASignature signature = ECDSAUtil.sign(
            Converter.sha256(input.getBytes()),
            Converter.hexStringToByteArray(serviceProfile.getPrivateKey()),
            0);

    return Converter.byteArrayToHexString(signature.encodeToDER());
}
```
从代码来看逻辑很简单，根据serviceid查看ServiceProfile对象，然后获取到获取到的私钥和待签数据以及类型一起扔给sign函数进行签名。

首先我们看` ServiceProfile serviceProfile = profileFrom(serviceId);`里面的 profileFrom函数：

```
private ServiceProfile profileFrom(String serviceId) {
    ServiceProfile serviceProfile = serviceProfileDao.findByServiceId(serviceId);
    if (serviceProfile == null) {
        CustomException.response(Error.SID_INVALID);
    }
    return serviceProfile;
}
```
其实就是根据serviceid从serviceProfileDao对象里面获取ServiceProfile对象，findByServiceId函数的实现在：

```
public ServiceProfile findByServiceId(String serviceId) {
    ServiceProfile profile = ServiceProfile.from(serviceProfileMap.get(serviceId));

    if (profile != null && !StringUtils.isEmpty(profile.getAesHex())) {
        byte[] randomKey = FileUtil.loadRandomKey(savePath);
        byte[] aesHexBytes = AESUtil.aesDecrypt(Converter.hexStringToByteArray(profile.getAesHex()), randomKey);
        AESUtil.wipe(randomKey);
        profile.setAesHex(new String(aesHexBytes));
    }

    return profile;
}
```
好了就是根据serviceId去serviceProfileMap这个HashMap里面查找ServiceProfile对象，在`/createService `里面讲过，生成的ServiceProfile对象都是通过serviceId为key，ServiceProfile为值存入到了serviceProfileMap这个HashMap里面了。查找出来以后，需要把aesKey用randomKey进行aes解码然后重新封装到ServiceProfile对象里面。

回到generateSignature函数，继续调用ECDSAUtil的sign函数进行签名，sign函数为：

```
public static ECDSASignature sign(byte[] hash, byte[] privateKey, int type) {
    if (type == Constant.SIGNATURE_TYPE_ETH) {
        return signEth(hash, privateKey);
    } else if (type == Constant.SIGNATURE_TYPE_EOS) {
        return signEos(hash, privateKey);
    } else {
        return generateSignature(hash, privateKey, 0);
    }
}
```
从sign的代码可以看到，根据传入的type进行不同类型的签名，最终生成签名的r和s值，具体签名算法实现就不再这里做具体分析了。


好了，GodController的所有接口就分析完了，下面分析这里我们分析KeyController的接口。


### KeyController分析

这里我们分析KeyController：

我们从KeyController这个类里面看到@RestController，说明这是一个restful风格的controller。有三个url分别是/generate，/sign和/verify，那么我们来一个个的分析。

#### /generate

首先我们分析/generate实现了，它对应的方法是：

```
public CommonResult<ResGenerateKey> generateKey(@Validated @RequestBody ReqGenerateKey body) {
    ResGenerateKey res = new ResGenerateKey();

    for (int i = 0; i < body.getCount(); i++) {
        res.getKeys().add(new ResGenerateKey.Key(
                keyService.generateKey(body.getServiceId())));
    }

    return CommonResult.make(res);
}
```

代码比较简单，首先从body里面获取到一个int类型的变量count来决定生成多少对公私钥对，然后添加到ResGenerateKey里面的Key类型的List里面。这里的Key对象的定义是在ResGenerateKey类里面的：

```
public static class Key {
    @ApiModelProperty(value = "非压缩公钥，16进制字符串")
    public String pubHex;
    @ApiModelProperty(value = "非压缩公钥的hash，16进制字符串")
    public String pubHashHex;

    public Key(byte[] pub) {
        this.pubHex = Converter.byteArrayToHexString(pub);
        this.pubHashHex = Converter.byteArrayToHexString(Converter.ripemd160(Converter.sha256(pub)));
    }

    public Key(String pubHex) {
        this.pubHex = pubHex;
        this.pubHashHex = Converter.byteArrayToHexString(
                Converter.ripemd160(
                        Converter.sha256(
                                Converter.hexStringToByteArray(pubHex))));
    }
}
```
很简单，公钥和公钥哈希。

我们重点分析 `keyService.generateKey(body.getServiceId()))); `。在com.rst.pkm.service.impl.KeyServiceImpl类里面实现了generateKey方法。

```
public byte[] generateKey(String serviceId) {
    byte[] privateKey = secureRandom.nextBytes(32);
    byte[] publicKey = pubFromPriv(privateKey);
    keystoreService.save(serviceId, privateKey, publicKey);

    return publicKey;
}
```
看上面代码大致就是：

1.生成一个随机长度为32的字节数组作为私钥。

2.把第一步生成的私钥根据曲线为 ` secp256k1 ` 的椭圆曲线算法生成公钥。

3.把前面生成的公私钥和传入过来的serviceid一起通过函数save保存起来。

我们再分析一下save函数是怎么保存公私钥和serviceid的。com.rst.pkm.service.impl.KeystoreServiceImpl类实现了save方法，代码如下：

```
public void save(String serviceId, byte[] privateKey, byte[] publicKey) {
    ServiceProfile sp = spRepository.findByServiceId(serviceId);

    byte[] aesKey = Converter.hexStringToByteArray(sp.getAesHex());
    byte[] privEncrypt = AESUtil.aesEncrypt(privateKey, aesKey);

    ServiceKey serviceKey = new ServiceKey();
    serviceKey.setPrivKey(Converter.byteArrayToHexString(privEncrypt));
    serviceKey.setPubKey(Converter.byteArrayToHexString(publicKey));
    String pubHashHex = Converter.byteArrayToHexString(
            Converter.ripemd160(Converter.sha256(publicKey)));
    serviceKey.setPubKeyHash(pubHashHex);
    serviceKey.setServiceId(serviceId);

    skDao.save(serviceKey);
}
```

第一步

```
ServiceProfile sp = spRepository.findByServiceId(serviceId);
```
findByServiceId的方法实现：
```
public ServiceProfile findByServiceId(String serviceId) {
    ServiceProfile profile = ServiceProfile.from(serviceProfileMap.get(serviceId));

    if (profile != null && !StringUtils.isEmpty(profile.getAesHex())) {
        byte[] randomKey = FileUtil.loadRandomKey(savePath);
        byte[] aesHexBytes = AESUtil.aesDecrypt(Converter.hexStringToByteArray(profile.getAesHex()), randomKey);
        AESUtil.wipe(randomKey);
        profile.setAesHex(new String(aesHexBytes));
    }

    return profile;
}

```
从叫serviceProfileMap的HashMap根据serviceid提出来ServiceProfile对象并且扔给类型为ServiceProfile的profile对象，然后profile通过get方法获取到AesHex，这里的AesHex加密了的，需要通过

```
 byte[] randomKey = FileUtil.loadRandomKey(savePath);
```
获取到randomKey。这里的savePath为

```

@Value("${pkm.data-path}")
private String savePath;
```
通过注解的方式去获取 ` pkm.data-path ` 变量的值。前面分析 ` application.properties ` 的时候说过，里面有一个变量就是 ` pkm.data-path=/data/pkm/ ` 。分析loadRandomKey函数其实就是去/data/pkm/路径下的random-key文件里面提取出randomKey。
使用randomKey作为输入密钥对AesHex做aes解密后拿到明文的AesHex，并且从新封装到profile这个对象里面。

这里的serviceProfileMap、aesKey怎么来的，都是在前面分析GodController的`/createService `的时候已经分析过了，这里就不用再做解释说明了。



回到save函数，这里使用明文的AesHex作为输入密钥对生成的私钥进行aes加密，然后创建了ServiceKey这个对象，封装了加密后的私钥，公钥，公钥哈希，servicid，最终调用`skDao.save(serviceKey);`进行保存，代码如下：

```
public boolean save(ServiceKey serviceKey) {
    serviceKeyMap.put(serviceKey.getPubKeyHash(), serviceKey);
    FileData data = new FileData();
    data.getServiceKeys().addAll(serviceKeyMap.values());
    boolean success;
    byte[] content = AESUtil.aesEncrypt(new Gson().toJson(data).getBytes(), Constant.ADMIN_KEY);

    success = FileUtil.save(Converter.byteArrayToHexString(content), savePath + FILE_NAME);
    if (success) {
        success = FileUtil.backup(savePath + FILE_NAME, savePath + BACKUP_FILE_NAME);
    }

    return success;
}
```

代码很简单，传进来的ServiceKey对象首先保存到一个叫serviceKeyMap的HashMap里面，这个serviceKeyMap的定义为：

```
Map<String, ServiceKey> serviceKeyMap
```
说明是用公钥哈希作为key，serviceKey作为value进行存储到这个HashMap里面去了的。


然后在FileData这个对象定义为：

```
private class FileData {
    private List<ServiceKey> serviceKeys = new ArrayList<>();
}
```
一个ServiceKey类型的List，不断将serviceKeyMap的所有数据都保存到这个List里面去。
然后将FileData经过Gson转换为json格式的数据，并且使用Constant.ADMIN_KEY作为输入密钥进行aes加密后持久性存入到/data/pkm/service-key文件里面。这里也用到了最前面分析的ADMIN_KEY是拿来干嘛了的。

>**注意：**
>分析到这里，涉及到aes加密的有三个  key ，` AesHex ` ，` randomKey ` ，` ADMIN_KEY ` 他们的作用分别是：
> - AesHex作为aes算法的输入密钥对私钥进行加密；
> - randomKey作为aes算法的输入密钥对AesHex进行加密；
> - ADMIN_KEY作为aes算法的输入密钥对整体数据进行加密。

同分析GodController的`/createServie`接口一样，在Dao层的ServiceKeyDao里面也有init函数并且被注解为@PostConstruct，说明也是在加载javabean的时候就需要初始化次函数。init函数的功能和分析上个接口的ServiceProfileDao的init函数功能类似：

>- 初始化serviceKeyMap这个HashMap。
>- 从service-key这个文件里面读取值来判断内容是否为空，如果不为空的话，那么就会使用Gson把数据还原成FileData类型。FileData里面有一个变量是ServiceKey类型的List ` private List<ServiceKey> serviceKeys = new ArrayList<>();`。那么不为空的情况下，把所有的ServiceKey对象取出来，从新封装到serviceKeyMap这个HashMap里面去。统一到我猜测这种情况也是在服务掉电重启这样的类似的情况下出现后，需要做的事情。因为分析后面的签名接口，数据都不会从service-key文件里面查找而是直接从这个HashMap里面去查找私钥的。


**说明**

这个接口生成的私钥是拿来给网关发起的交易做签名用，因为此接口生成的公钥需要配置到ofgp上面。
#### /sign

此接口的函数定义如下：

```
public CommonResult<ResGenerateSignature> generateSignature(@Validated @RequestBody ReqGenerateSignature body) {
    ECDSASignature signature = keyService.sign(body.getInputHex(), body.getPubHashHex(), body.getType());
    ResGenerateSignature res = new ResGenerateSignature(signature);

    return CommonResult.make(res);
}
```
通过body传输过来的数据有三个，待签名数据、公钥哈希、类型。代码比较简单调用 keyService的sign进行进行验证签名。sign函数在KeyServiceImpl里面具体实现：

```
public ECDSASignature sign(String inputHex, String pubHashHex, int type) {
    return ECDSAUtil.sign(Converter.hexStringToByteArray(inputHex), keystoreService.getPriv(pubHashHex), type);
}
```
从上面的代码可以看到首先需要调用keystoreService的getPriv函数使用公钥哈希获取到私钥。

```
public byte[] getPriv(String pubKeyHash) {
    Key key = get(pubKeyHash);
    if (key == null) {
        logger.info("can not find private key");
        return new byte[0];
    }

    ServiceProfile sp = spRepository.findByServiceId(key.getServiceId());

    return AESUtil.aesDecrypt(key.getPriv(), Converter.hexStringToByteArray(sp.getAesHex()));
}
```
首先通过公钥哈希从通过get函数从serviceKeyMap里面提出出来ServiceKey对象，然后ServiceKey对象提出出来加密后到私钥，公钥，serviceid封装到Key对象里面。
然后通过serviceid到ServiceProfile对象里面提出aesKey作为aes的加密输入密钥对加密后的私钥进行解密还原出来真正的私钥。

回到sign函数，此函数里面需要调用ECDSAUtil类里面的sign函数,函数定义如下：

```
public static ECDSASignature sign(byte[] hash, byte[] privateKey, int type) {
    if (type == Constant.SIGNATURE_TYPE_ETH) {
        return signEth(hash, privateKey);
    } else if (type == Constant.SIGNATURE_TYPE_EOS) {
        return signEos(hash, privateKey);
    } else {
        return generateSignature(hash, privateKey, 0);
    }
}
```
嗯，这里也是根据传入的type的不同调用不同的函数进行签名。最终生成签名的r和s值，具体签名算法实现就不再这里做具体分析了。



#### /verify

此接口的实现函数是：

```
public CommonResult<ResVerifySignature> verifySignature(@Validated @RequestBody ReqVerifySignature body) {
    boolean verifyResult;
    if (StringUtils.isEmpty(body.getSignatureDerHex())) {
        verifyResult = keyService.verify(body.getInputHex(), body.getPubKeyHash(), body.getR(), body.getS());
    } else {
        verifyResult = keyService.verify(Converter.hexStringToByteArray(body.getInputHex()),
                Converter.hexStringToByteArray(body.getPubKeyHash()),
                Converter.hexStringToByteArray(body.getSignatureDerHex()));
    }

    return CommonResult.make(new ResVerifySignature(verifyResult));
}
```
代码看起来比较简单。
从函数代码来看，就是提取body里面的各个字段然后经过一个判断后走不通的分支分别调用keyService里面的重载函数verify进行验证签名。这里的判断是一个SignatureDerHex的变量是否为空。对于请求的body的ReqVerifySignature的对象定义如下：

```
public class ReqVerifySignature extends BaseRequest {
    @ApiModelProperty(value = "der编码序列化的签名RS，此项不空时以此校验签名")
    private String signatureDerHex;
    @ApiModelProperty(value = "R值(16进制字符串), signatureDerHex不传或为空时有效")
    private String R;
    @ApiModelProperty(value = "S值(16进制字符串)，signatureDerHex不传或为空时有效")
    private String S;
    @ApiModelProperty(value = "被签名的数据流(16进制表示)")
    private String inputHex;
    @ApiModelProperty(value = "公钥hash")
    private String pubKeyHash;
}
```
这里代码对字符串signatureDerHex有详细的定义说明。

回到上面的verifySignature函数，如果此字符串为空，那么将会走到

```
verifyResult = keyService.verify(body.getInputHex(), body.getPubKeyHash(), body.getR(), body.getS());
```
分支里面。这里的verify函数的实现在KeyServiceImpl

```
public boolean verify(String dataHex, String pubHashHex, String rHex, String sHex) {
    BigInteger r = new BigInteger(rHex, 16);

    return verify(Converter.hexStringToByteArray(dataHex),
            keystoreService.getPub(pubHashHex),
            new BigInteger(rHex, 16),
            new BigInteger(sHex,16));
}
```
这里的`keystoreService.getPub(pubHashHex)`经过跟踪，最终调用是去serviceKeyMap这个HashMap里面根据公钥哈希查找出来公钥。

如果不为空，那么将会走到下面分支：

```
verifyResult = keyService.verify(Converter.hexStringToByteArray(body.getInputHex()),            Converter.hexStringToByteArray(body.getPubKeyHash()),
Converter.hexStringToByteArray(body.getSignatureDerHex()));
```

其实两个分支最后都是调用了KeyServiceImpl里面的`private boolean verify(byte[] data, byte[] pub, BigInteger r, BigInteger s)`函数进行验证签名，函数的具体实现如下：

```
private boolean verify(byte[] data, byte[] pub, BigInteger r, BigInteger s) {
    ECDSASigner signer = new ECDSASigner();
    signer.init(false, new ECPublicKeyParameters(Constant.CURVE.getCurve().decodePoint(pub),
            Constant.CURVE));
    return signer.verifySignature(data, r, s);
}
```
这里需要根据公约哈希通过函数getPub查找出公钥对签名数据进行解密验证签名，通过跟踪函数最终调用是去serviceKeyMap里面根据公钥哈希查找出来公钥，然后调用spongycastle库的verifySignature函数进行验证签名，然后返回一个bool变量验证验签是否成功。

OK，到这里。两个Controller里面的所有接口都流程都分析完了。

#### 访问控制权限

下面我们来分析一下接口的访问控制权限。

存在`com.rst.pkm.controller.interceptor`目录，如果熟悉java MVC的开发框架的话，应该清楚这里应该是定义了拦截器来做访问权限的控制。两个拦截器的生效配置在`com.rst.pkm.config.WebMvcConfigurer`类里面：

```
@Configuration
public class WebMvcConfigurer extends WebMvcConfigurerAdapter {
    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(new RequestCheckInterceptor()).addPathPatterns("/key/**");
        registry.addInterceptor(new GodCheckInterceptor()).addPathPatterns("/god/**");
    }
```
从上面的代码其实就可以看到两个拦截器分别生效的路径了。

RequestCheckInterceptor拦截器对应于的`/key/**`生效，也就是KeyController下面的所有接口。
GodCheckInterceptor拦截器对应于的`/god/**`生效，也就是GodController下面的所有接口。

在这个目录下定义了两个拦截器GodCheckInterceptor和RequestCheckInterceptor，两个拦截器都是实现了HandlerInterceptor这个接口。熟悉Spring MVC的开发，都知道在实现了HandlerInterceptor接口的preHandle方法。


RequestCheckInterceptor的preHandle方法代码：

```
public boolean preHandle(HttpServletRequest request,
                         HttpServletResponse response,
                         Object handler) throws Exception {

    if (handler != null && handler instanceof HandlerMethod) {
        HandlerMethod handlerMethod = (HandlerMethod) handler;
        Class beanType = handlerMethod.getBeanType();
        if (beanType.isAnnotationPresent(DisableRequestCheck.class)) {
            return true;
        }

        Method method = handlerMethod.getMethod();
        if (method.isAnnotationPresent(DisableRequestCheck.class)) {
            return true;
        }
    }

    //do some check
    requestControlService.check(request);

    return true;
}
```

前面的几段代码其实就是判断自定义的注解类，跟踪这两个自定义的注解类，里面什么都没有实现，直接跳过。

主要是`requestControlService.check(request);`，check函数的实现在RequestControlServiceImpl类里面，

```
public void check(HttpServletRequest request) {
    String serviceId = request.getHeader(Constant.SERVICE_ID);
    if (StringUtils.isEmpty(serviceId)) {
        CustomException.response(Error.SID_NOT_PRESENT);
    }

    ServiceProfile serviceProfile = spRepository.findByServiceId(serviceId);
    if (serviceProfile == null || serviceProfile.isLocked()) {
        CustomException.response(Error.SID_INVALID);
    }

    checkIp(request.getRemoteAddr(), serviceProfile.getAllowIps());
    checkIp(IpUtil.clientIpFrom(request), serviceProfile.getAllowIps());
}
```

从代码来看主要就是提取了http请求里面的自定义的serviceid的header，判断是否合法，以及提出报文的请求ip和允许通过的ip做比较。



GodCheckInterceptor的preHandle方法代码：

```
public boolean preHandle(HttpServletRequest request,
                         HttpServletResponse response,
                         Object handler) throws Exception {
    String fromIp = request.getRemoteAddr();

    //只允许以127.0.0.1方式直接访问
    if (!VALID_IP.equals(fromIp)) {
        CustomException.response(Error.IP_INVALID);
    }

    String password = request.getHeader(ADMIN_PASSWORD);
    if (StringUtils.isEmpty(password) ||
            !Arrays.equals(AESUtil.aesKeyFrom(password), Constant.ADMIN_KEY)) {
        CustomException.response(Error.ADMIN_PWD_INVALID);
    }

    return true;
}
```
从上面代码可以看到，god这个目录第所有接口应该是一个管理接口，所以只允许127.0.0.1地址访问，其实也就是只能在本机上面发起http请求访问这个目录下的接口。从访问控制权限上面就限制了禁止非本机发起http请求。

## 总结


### 接口访问控制权限

> - `/god`路径下的管理接口只能本机127.0.0.1访问。
> - 通过在管理配置配置允许访问ip的白名单，以及通过在http的报文头部添加serviceid这样的字段来限制/key路径下的业务接口的访问控制权限。

### 密钥管理

> - 继承SecurRandom类来重写随机数生成函数来生成随机的私钥。
> - 公钥基于椭圆曲线算法根据私钥导出。
> - 对于私钥加密存储管理逻辑是：
>   * `/god`路径下的管理接口生成AesHex作为aes算法的输入密钥对/key路径下的业务接口生成的私钥进行加密；
>   * `/god`路径下的管理接口生成randomKey作为aes算法的输入密钥对AesHex进行加密；
>   * 环境变量PASSWORD的值作为函数输入导出ADMIN_KEY作为aes算法的输入密钥对整体数据进行加密。


### 基础环境

>   - 基础环境的对外端口关闭，登录跳板机的方式访问pkm的服务器。
>   - 采用docker的方式部署pkm服务。







































