---
layout: post
title: "RSA加解密实践"
description: ""
category: 
tags: [RSA]
---
{% include JB/setup %}

日常开发中时常会有有敏感信息需要存储，简单处理的有MD5, Base64。如果需要强度更大的可以用AES/RSA等，本文并打算不讨论加密算法的原理，密码学是一门很高深的学问；
对接过支付宝快捷支付的朋友可能都知道，支付宝才支付订单环节采用的及时RSA加解密；下面讲一下如果在应用中方便的利用RSA存储信息；

##私钥准备
首先我们需要准备一套RSA的秘钥，利用openssl工具可以生成一个强度较大的秘钥；一般来说公钥和私钥都是以文件的形式存储的，公钥用于加密，私钥用于解密；
但是我不希望吧私钥文件暴露出去，因此得到秘钥后，还需要分别生成pkcs8格式的私钥串和公钥串，然后我们吧这两个字符串放到native的c/c++层；


### 生成RSA秘钥

	openssl genrsa -out private-key.pem 2048


```
aven-mac-pro:testRSA aven$ openssl genrsa -out private-key.pem 2048
Generating RSA private key, 2048 bit long modulus
.................................+++
.........................................................+++
e is 65537 (0x10001)
```


### 输出不加密的私钥

	openssl pkcs8 -topk8 -inform PEM -in private-key.pem -outform PEM -nocrypt

```
aven-mac-pro:testRSA aven$ openssl pkcs8 -topk8 -inform PEM -in private-key.pem -outform PEM -nocrypt
-----BEGIN PRIVATE KEY-----
MIIEugIBADANBgkqhkiG9w0BAQEFAASCBKQwggSgAgEAAoIBAQC5mjxU8eZ5BEJx
G2vKcD+dnHnu5sfU0puxz16am73V4Je2JmxY8x5B8AlYaM15I2vpv+NVxlJagy8T
WZLuNElqYNQu4SllM8GwfA44KxwNzZzR1cVm88whX0/Z4JZ+kj8ttsp6PpxbOhAF
RnxwgjMU2b0tbukQuJTExoM+8tOV8K/YlEkjDgLWA15bJVoNq25Fz0jOLrBlnwGA
s1EVOKUebsXZ/GyaV8OItKj5/9VYrp7umoojAeWmBvchx0WqEHlw2r4oPIXd9UN3
SN3PM0Bui0zjoIpRzlTE8g2cwkXJDA0exhFkyaRuGcM+91gFpT1nsnWFxeG38BxF
KSqRilizAgMBAAECggEALOHHRSNaAFmvV3qyDjomqA52zfawzB5B2DW1Qt32ggnV
pg6UlM31uyw4llCBn5GZPuVQLCXRNGIUuDEo/sFWH4taxBtez0I8zFizd5G1LwFR
ssxm+AZsjoVl4eIVgnYLIRray8ToOodH6H6rCOnzQE+HF72CTrDUCOGYS1idIdyw
JETRHSXgUaTXFW8YSfsrsb4WreSpqQiJPT5oE6suEn+RC/pACTIXN7Ic327ALx7K
TVF09+wCU/v+fEVmt2bIm15cj/Vbl0vy34dVmV4kmkfjKE2jqBSvvBtwkWSukPf3
wRNvO43sMhl/arLWa5A79ao4pYXZfQ5wNCX1TKXuQQKBgQDybYtorSl527MMqLpv
rDoUUG/sN0GvbVaZAqYlD3cua1q3W6t5suU9oSBfDUGSDABTketU2FWqlUZIgxf/
Pby9KKNdmWD0zWjrYRj5lErcmAo9aZs5jukVR1/i17xTfyRVI15OimCW/tXw8E5w
lm94CAGvanPs6rkC5YLXygT9TQKBgQDD/kYsSREo1xSarICpnsUg48TMb7dAlkr0
vYIJfDRyuyDsD8X22r+qUWUtFQDzV41ylLVfyQqo6Lgc+dtLIILGMqbm2kutzqZI
pwQAIcbz637nq+PIE0UCOUmYRs/uD+fWE0LMC28AQMe9C9AKvUkImyolWgUU7q+2
eQINNnmt/wKBgFAchRolRvR+9o8zXtCycErwPdwocmtfTWOo7XCHyNGtJkA7adIA
nSKdkU332nhBwQXczZCvILgLNjuWHqL5KtqziDDRE6oyCv7lilRHfemh0Jh0wpfl
sv6WJIiY1CIffMkps+tubPbY5agGMVWhUNqwgqYOHprnAhaD85YNq1JtAn9ATy63
WUJIJEqedfvBrFcCc7ofWojGqInvxD7m3dpXyw8CZiqO1TgOqqaIJFwrfI7tCd55
j33v7mx7FYDfJcvDPNuG5Bnw7d2h+StW375oSt1ZJw2WmLwL/sAnNxUDCDUKCUfh
q97ANoFThoy8+V79c+xgVSlVtPvy48HIlBdZAoGARvO9UJ8sP55jJ11QGQmF3W5s
+gq5X5xGtBLfBBgSpkpXz1Qko6eijip9nepPbyUhMbE98fwsDT/x5i/UjsDR2DPb
3cMKCPfjsjF7LoQVdmZDz0NlqKZJ2og6qH15LxrJlnnENaevb08EDSz4bYo3mOTZ
EaEJ2Pzcrbi9kgLp/JA=
-----END PRIVATE KEY-----
```

### 输出公钥

	openssl rsa -in private-key.pem -pubout

```
aven-mac-pro:testRSA aven$ openssl rsa -in private-key.pem -pubout
writing RSA key
-----BEGIN PUBLIC KEY-----
MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEAuZo8VPHmeQRCcRtrynA/
nZx57ubH1NKbsc9empu91eCXtiZsWPMeQfAJWGjNeSNr6b/jVcZSWoMvE1mS7jRJ
amDULuEpZTPBsHwOOCscDc2c0dXFZvPMIV9P2eCWfpI/LbbKej6cWzoQBUZ8cIIz
FNm9LW7pELiUxMaDPvLTlfCv2JRJIw4C1gNeWyVaDatuRc9Izi6wZZ8BgLNRFTil
Hm7F2fxsmlfDiLSo+f/VWK6e7pqKIwHlpgb3IcdFqhB5cNq+KDyF3fVDd0jdzzNA
botM46CKUc5UxPINnMJFyQwNHsYRZMmkbhnDPvdYBaU9Z7J1hcXht/AcRSkqkYpY
swIDAQAB
-----END PUBLIC KEY-----
```

##java调用rsa加解密

	//加密
    public static byte[] encrypt(byte[] bytes) {
        try {
            KeyFactory keyFactory = KeyFactory.getInstance("RSA");
            X509EncodedKeySpec pubSpec = new X509EncodedKeySpec(Base64.decode(PUBLIC_KEY, Base64
                .DEFAULT));
            Key encryptionKey = keyFactory.generatePublic(pubSpec);

            Cipher rsa = Cipher.getInstance("RSA");
            rsa.init(Cipher.ENCRYPT_MODE, encryptionKey);
            return rsa.doFinal(bytes);
        } catch (Exception e) {
            e.printStackTrace();
        }
        return null;
    }
	//解密
    public static byte[] decrypt(byte[] bytes) {
        try {
            KeyFactory keyFactory = KeyFactory.getInstance("RSA");
            PKCS8EncodedKeySpec keySpec = new PKCS8EncodedKeySpec(Base64.decode(PRIVATE_KEY, Base64
                .DEFAULT));
            Key decryptionKey = keyFactory.generatePrivate(keySpec);

            Cipher rsa = Cipher.getInstance("RSA");
            rsa.init(Cipher.DECRYPT_MODE, decryptionKey);
            return rsa.doFinal(bytes);
        } catch (Exception e) {
            e.printStackTrace();
        }
        return null;
    }

[http://www.grandville.net/?n=OpenSSL.Rsa-java-openssl](http://www.grandville.net/?n=OpenSSL.Rsa-java-openssl)