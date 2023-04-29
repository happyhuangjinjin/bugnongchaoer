### 简介
之前，在《》介绍过使用 openssl 生成 PCKS1 格式的 RSA 密钥，然后再转换成 PCKS8 格式的密码。但是转换后去除了秘钥的密码。那如果没有去除秘钥的密码，如何加载带有密码的密钥呢？Java 自带的 API 没有找到，如果需要实现加载带密码的 RSA 需要用到 bouncycastle 库。

在百度搜索，几乎搜索不到。唯一安装关键字 **用Java加载加密的PCKS8 PEM私钥** 能够搜出一下内容。但是需要发送**暗号**才能查看文章内容。其实内容是如下帖子的翻译：

```
https://stackoverflow.com/questions/66286457/load-an-encrypted-pcks8-pem-private-key-in-java
```
而且关键问题是没有输出有用的代码。

### pom.xml 引入
```
<dependency>
    <groupId>org.bouncycastle</groupId>
    <artifactId>bcprov-jdk15on</artifactId>
    <version>1.68</version>
</dependency>
<dependency>
    <groupId>org.bouncycastle</groupId>
    <artifactId>bcpkix-jdk15on</artifactId>
    <version>1.68</version>
</dependency>
```
`bcprov-jdk15on`和`bcpkix-jdk15on`是 bouncycastle 提供的两个加解密包

### 加载 PCKS8 密钥
代码中有三个条件判断分支
- PKCS8EncryptedPrivateKeyInfo ：PCKS8 格式加密密钥
- PEMEncryptedKeyPair ：PCKS1 格式加密密钥
- PEMKeyPair ：非加密密钥

```
import org.bouncycastle.asn1.pkcs.PrivateKeyInfo;
import org.bouncycastle.jce.provider.BouncyCastleProvider;
import org.bouncycastle.openssl.PEMEncryptedKeyPair;
import org.bouncycastle.openssl.PEMKeyPair;
import org.bouncycastle.openssl.PEMParser;
import org.bouncycastle.openssl.bc.BcPEMDecryptorProvider;
import org.bouncycastle.openssl.jcajce.JcaPEMKeyConverter;
import org.bouncycastle.operator.InputDecryptorProvider;
import org.bouncycastle.pkcs.PKCS8EncryptedPrivateKeyInfo;
import org.bouncycastle.pkcs.PKCSException;
import org.bouncycastle.pkcs.jcajce.JcePKCSPBEInputDecryptorProviderBuilder;

import java.io.IOException;
import java.io.StringReader;
import java.security.PrivateKey;
import java.security.Security;
import java.security.cert.CertificateException;

public class PKCS8Main {
    public static void main(String[] args) throws IOException, PKCSException, CertificateException {
        // https://stackoverflow.com/questions/66286457/load-an-encrypted-pcks8-pem-private-key-in-java
        System.out.println("Load an Encrypted PCKS8 PEM Private Key In Java");
        // you need 2 bouncy castle libraries, I'm using the actual ones version 1.68:
        // https://mvnrepository.com/artifact/org.bouncycastle/bcprov-jdk15on
        // bcprov-jdk15on-1.68.jar
        // https://mvnrepository.com/artifact/org.bouncycastle/bcpkix-jdk15on
        // bcpkix-jdk15on-1.68.jar

        Security.addProvider(new BouncyCastleProvider());
        char[] password = "aWCTJPET9fL7UBTp97hX99gdofeWKUf5tuxSuJeST2sEkyvkyinrfrj6EiSUTErF".toCharArray();
        String privKeyStrBase64Encoded =
                "-----BEGIN ENCRYPTED PRIVATE KEY-----\n" +
                        "MIIJrTBXBgkqhkiG9w0BBQ0wSjApBgkqhkiG9w0BBQwwHAQIRZpyCAQQR8oCAggA\n" +
                        "MAwGCCqGSIb3DQIJBQAwHQYJYIZIAWUDBAEqBBC8h4heO8aR5NaJq5o89myBBIIJ\n" +
                        "UDo2+MIuXH+y+hB9fx1kVe7yIogh0DEeYTnTLAraeg2yudSoNLq4xF2Ftid7Ax9Q\n" +
                        "6BwdrY6YB8/Pk9vn7gWBoTdrfUpJZZu6DtLu7URXPTN5mgRNmJzC4uAv9y+T8CRv\n" +
                        "sLXjwYOSbF4Vsbm8JlgJPuKPKkH1lTi3t1jxik+nNrDyAXRJOYfowkW9xnGqNSxq\n" +
                        "GEFXN1S16msrEq+chTblSpn8liY6o2Yk405qJ2G+dz/Lv50vOjoc1GTh8vNw23ER\n" +
                        "HckRiRS8TY/sjyVtI6tZp+30azbMcXgLEHFoMDIm/bu366/aYyRI2fI76LWJUuHA\n" +
                        "ZmU8GYU8j0+IewLPgtAXkjkTMuvrucHgCH3/ZRDDqFI+YTj3q19r4xmPGRWoxp/c\n" +
                        "2Oex3HfUCutLonof+89I5OGfvlnaESaQcSzLJSzPV6qXGMMTFVZ6G2ZW1gfaJGC1\n" +
                        "VwyL2k7Ejg8ncnOPKsf27oCFOflDLvfVQc/iW/XXZfFkGpO3xgoBE1CYq9y1w4VZ\n" +
                        "gzbi88AwIV2oJTqUQkNFkDMYwPnCj/D4BAT1ipysMA3Q9gs4+EnfD9nMevtef9dR\n" +
                        "wYaPNcnC/dy3Soacqm6VPOQ9RP77BIMNAQKlE95lsB++DXhPs3uDyxavar0pnJn2\n" +
                        "nBfPxRGbJeVc9hxyxzQtosgkrVIIt/0nEUigTukYCESpvJAd1vsupbi6v/FGXvzg\n" +
                        "4aU2aGTYUWcPbsi4kZUQH8ysmaPAC4QmHBPumzmeA+k4LxUaGJaWwYxCqD+WBCye\n" +
                        "D1w1utaFTjG+HSwIqGvVTxMXZswm3Tw8+bAu5UpPeIpXejWa06duCxDkvncpOW04\n" +
                        "En3hJuPTBEMxzCVIlYjk3Z7bNtjOayJRqxrHlY0OkHoy8BXRSN6f9O84go8S2qie\n" +
                        "MmchFTarLfG7dir9SyDJdH2bTephRdGV3nmPLX/xza3ES4JmyHEOmYxbgqnSXsSG\n" +
                        "8RZQ7Cg5tQVDHf1ydgyfqJ/lcG3SaehKHVuUR/Ulv8u5WpAYoxr1aGWafUtFYSoC\n" +
                        "g/RF6E9gEbrcl/KnnPEcG5jI+86BeEsfVkpjqsh10lHG008oyeI92nXzYvZv76+d\n" +
                        "9bshKT6qERGlA+2HYvZkNMtuwh0eUlq0sHCKQW3D7PTerfok3EP5ohiHwIZ0cME6\n" +
                        "Jq8Dz4ORGFFAdYi86NhCkW7s4nXtP1utppaZBeZLF7nxk0XYIb3NP+a/Ll9eQQSe\n" +
                        "WNDdv/387J++PzpygviGmZfF3rBl253cbPX/nhhNAOiPajdN2q3qDTpAnZpE3G+v\n" +
                        "t9OnNFtgmYZ0SEOMxo8T3kMVnmUhP3OsVoWcwcqIOjwPXmwCL8c4Dju8sxfZyCQh\n" +
                        "rRGKsbJyON+UTXA07rpH1JzqmWSQDiir75JBsLlX56yKIzKe86CWDlohbyykVkPs\n" +
                        "vT0qTgpSvkU8N3jErjZUJsIk29uMFVcVjvR1GmMiCOIG7jZ+KefjXx43JFl6Av4w\n" +
                        "fjdfaMb2J2/jd9suBhGDpakftZl57Qhz/FN8yDONmetJMcum5+dwxnbB9hCP7fpJ\n" +
                        "T5tnE/w0KNFJEtXeylOHlU225czr4+gdsj+ncKiTEwIm0htulrDDgKsN74EXPcor\n" +
                        "ywEpc3oZZMFF8084g6j0rjAnLO/fgtf0nXfMGDGUfIGo9AJFqoYKKMu5u5vQ+XY0\n" +
                        "cc93QAB1lJ1a8yyRwfUoLHNbq2AJ9NMw1sNvkRV26dpk7ecZ8LgoUdizbztz5vb2\n" +
                        "6VfArHvT1pZEdjgPwsQnegu4i4/ELXZWTZu2hfIM/aTgP9avAQmQDiKEnOnOQZ/3\n" +
                        "QyFEFo5qFfH4wUKbLoQXVWslIyz7bRd+F26GoJTLPOHkPAZZ3UCCzVUHwCzc78eO\n" +
                        "o9V4wgfVNFNkdyXi81X97v0bKKbkxfanz0+kBSDmeOUOKTDmyFOmhbC5SKLBF3k/\n" +
                        "gNHR6BzCET7ReGZ+qIVF10Oy6SzP7M/Fmt1TU2y58CoiM3pKPDUzDYX47NCoUU6c\n" +
                        "S8iberxh1O4mMxDNfwQmFzXe8cst8JGBxWX2O+Oqvl9EpGojWGXY9ydut//nDRPv\n" +
                        "sQyMngcK9KLAG8/JL4hp8plKee+JtNzelCHzbjObE18waF3cveKns8WaumqpgzyT\n" +
                        "2nX0vct0UxBeynIefaMEwT4WbsFXu/iMrRQpsrQmxlq3LLiet5lj3UdE9EiBvQox\n" +
                        "FftQdOR8t36zNzCDnHmPzmrxggiKEjDw6BFN4sV+jm6SZWzAlypplzBHGfexYS8t\n" +
                        "4taG0Z6lXev9xYgAcTFZrNJRVQ18c8/8W2V+LMB86LNTG7IKa8Cbo4FMPFGQhlAZ\n" +
                        "n3URL2jaV7Cf2tTuHq0IT4Sv4/dx/Cttx8qdFjGbTt3ILCpUsh9KjxTEjteA6Ydu\n" +
                        "e8lYsU8C5E3mdldojkie8iZHFSjrwRuk4EyUGXRoMe900CDHXmNQ3g6Cb2cE3AgM\n" +
                        "RQvCLTQgDpQe6WJ4/HFMRXtCE3dX6P46E1968MYlgn4RAmYel1HPIh/8oWaLmwxX\n" +
                        "IPEO/kxjybWkrvRDw8tQxVbR8D3sdurmYuMid2EpzI1OFLPp08JcpHn+9LyvBEw9\n" +
                        "9w8ngP260HSt0rckOCyRm+JWmFRmql6LRwdWl0ht1yTASDn1+/BkQm9JfOyjMxlE\n" +
                        "mXFdCHL8EK0+xcYn1IMypP0oG7TA0o1BK+vsmDoEO1pT3Qh4pTA2lFAoshWR9YBP\n" +
                        "ZMW2Pyscia5+wkRuL06yAujyJP5OOmHnLp9uni8tpo4OtSqt4DRCYLM9hsB2zL6Y\n" +
                        "3WvgavGznflve604cZ1jkJFkzg74WgwdXXn9XrI9OoEJm84avdQIdK2WlOiA5md5\n" +
                        "lMfyMVZtCZLh/6KpC8jB4t0FlOGdoQxubslWDXcJwhPFO0KvdNv+6TeWjt8ZBECI\n" +
                        "zRMq+jAR6yLN0gld3y4YI37cll6kr6wNYBd0NoL7Bzl/WtPn8MJTrwcRpohNqQkJ\n" +
                        "8MOrqL14yRDvPtQ2Rijzztnd3Vb9EL0Zgkrwh9uD1ZTVDyHWHnmwW7LeBOi0/vD3\n" +
                        "k/bi3qVGEqAc5YkCZbydMfzw0W506WsEMlysfbAhTEx+w/IV31mfD4VwpZ+ueSyE\n" +
                        "crFUOhrrE+9a34z2mSDke3Hte1pvjhIIH6B30mjFyQh3sOoouwl9PkvbuUo8Q7v2\n" +
                        "ojIAhmdgVMd19lfA+ihPnmalOtR7hCwdrjoHR3N5A+Ng\n" +
                        "-----END ENCRYPTED PRIVATE KEY-----";
        PrivateKey rsaPrivateKey = stringToPrivateKey(privKeyStrBase64Encoded, password);
        System.out.println("privateKey: " + rsaPrivateKey);
    }

    static public PrivateKey stringToPrivateKey(String s, char[] password)
            throws IOException, PKCSException {
        PrivateKeyInfo pki;
        try (PEMParser pemParser = new PEMParser(new StringReader(s))) {
            Object o = pemParser.readObject();
            if (o instanceof PKCS8EncryptedPrivateKeyInfo) { // encrypted private key in pkcs8-format
                System.out.println("key in pkcs8 encoding");
                PKCS8EncryptedPrivateKeyInfo epki = (PKCS8EncryptedPrivateKeyInfo) o;
                System.out.println("encryption algorithm: " + epki.getEncryptionAlgorithm().getAlgorithm());
                JcePKCSPBEInputDecryptorProviderBuilder builder =
                        new JcePKCSPBEInputDecryptorProviderBuilder().setProvider("BC");
                InputDecryptorProvider idp = builder.build(password);
                pki = epki.decryptPrivateKeyInfo(idp);
            } else if (o instanceof PEMEncryptedKeyPair) { // encrypted private key in pkcs1-format
                System.out.println("key in pkcs1 encoding");
                PEMEncryptedKeyPair epki = (PEMEncryptedKeyPair) o;
                PEMKeyPair pkp = epki.decryptKeyPair(new BcPEMDecryptorProvider(password));
                pki = pkp.getPrivateKeyInfo();
            } else if (o instanceof PEMKeyPair) { // unencrypted private key
                System.out.println("key unencrypted");
                PEMKeyPair pkp = (PEMKeyPair) o;
                pki = pkp.getPrivateKeyInfo();
            } else {
                throw new PKCSException("Invalid encrypted private key class: " + o.getClass().getName());
            }
            JcaPEMKeyConverter converter = new JcaPEMKeyConverter().setProvider("BC");
            return converter.getPrivateKey(pki);
        }
    }
}
```
