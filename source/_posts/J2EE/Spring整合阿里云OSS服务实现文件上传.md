---
title: Spring整合阿里云OSS服务实现文件上传
date: 2018-1-19
categories: J2EE
---

## 关配置

在阿里云控制台生成访问密钥(AccessKey)

![AccessKey](http://img-blog.csdn.net/20180119085829700?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

创建OSS bucket。

![OSS](http://img-blog.csdn.net/20180119085846587?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

> 阿里云提供了Endpoint(是阿里云自己的域名)。数据库中存储的文件地址如果包含阿里云的域名，将来如果不使用阿里云(使用其他的云服务，或者是自己搭建图片服务器)，那么数据库中的地址全部要修改(这里面包括富文本内容，修改富文本内容中的图片地址复杂程度可想而知)。
>
> 域名解析选择CNAME类型解析即可
>
> ![CNAME域名解析](http://img-blog.csdn.net/20180119085907748?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
>
> 如果域名是使用同一个账号管理的，直接在oss控制面板绑定域名
>
> ![绑定域名](http://img-blog.csdn.net/20180119085931617?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sbW9meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

## 现代码

添加OSS服务SDK依赖

```xml
<!-- 阿里云OSS服务API -->
<dependency>
  <groupId>com.aliyun.oss</groupId>
  <artifactId>aliyun-sdk-oss</artifactId>
  <version>2.8.3</version>
</dependency>
```

在properties文件中配置下面的相关信息

```properties
## 里云访问密钥
aliyun.access.key.id=XXX
aliyun.access.key.secret=XXXX

## 里云OSS服务相关配置
## SS的endpoint,这里是华南地区(也就是深圳)
aliyun.oss.endpoint=http://oss-cn-shenzhen.aliyuncs.com
## 是创建的bucket
aliyun.oss.bucket.name=XXX
## 里已经把自己的域名映射到bucket地址了。
aliyun.oss.img.domain=XXX.XXX.com
```

在Spring的xml中配置OSSClient

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:context="http://www.springframework.org/schema/context"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd http://www.springframework.org/schema/context http://www.springframework.org/schema/context/spring-context.xsd">

    <context:property-placeholder location="classpath:aliyun.properties"/>

    <context:component-scan base-package="com.cni.aliyun"/>

    <bean id="ossClient" class="com.aliyun.oss.OSSClient" destroy-method="shutdown">
        <constructor-arg name="endpoint" value="${aliyun.oss.endpoint}"/>
        <constructor-arg name="accessKeyId" value="${aliyun.access.key.id}"/>
        <constructor-arg name="secretAccessKey" value="${aliyun.access.key.secret}"/>
    </bean>

</beans>
```

上传文件的代码：

```java
import com.aliyun.oss.OSS;
import com.aliyun.oss.model.ObjectMetadata;
import org.apache.http.HttpHost;
import org.apache.http.client.utils.URIUtils;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Component;

import java.io.InputStream;
import java.net.URI;
import java.net.URISyntaxException;
import java.net.URL;
import java.util.Date;

/**
 * 阿里云OSS对象存储服务工具类
 * <p>
 * Author:Holmofy
 */
@Component
public class AliyunOss {

    @Autowired
    private OSS ossClient;

    @Value("${aliyun.oss.bucket.name}")
    private String bucketName;

    @Value("${aliyun.oss.img.domain}")
    private String mappingDomain;

    /**
     * 上传数据流
     *
     * @param is          数据输入流
     * @param path        文件在OSS上的存储路径(也就是OSS对象的key)
     * @param contentType 内容的mime-type
     * @return 返回已经映射的url字符串
     */
    public String upload(InputStream is, String path, String contentType) throws URISyntaxException {
        ObjectMetadata metadata = new ObjectMetadata();
        metadata.setContentType(contentType);
        ossClient.putObject(bucketName, path, is, metadata);
        URL url = generateUrl(path);
        URI uri = url.toURI();
        // 使用自己的域名替代阿里云提供的访问域名
        HttpHost httpHost = new HttpHost(mappingDomain);
        // 这个工具类是HttpClient里的，OSS-SDK本身就依赖这个jar包发送请求
        // 这里就可以拿这个工具类来用了
        URI mappingURI = URIUtils.rewriteURI(uri, httpHost);
        return mappingURI.toString();
    }

    /**
     * 生成URL
     *
     * @param key 上传资源的标识
     * @return 生成的URL
     */
    public URL generateUrl(String key) {
        // 100年不过期
        final long duration = 1000L * 60L * 60L * 24L * 365L * 100L;
        long time = System.currentTimeMillis() + duration;
        Date expiration = new Date(time);
        return ossClient.generatePresignedUrl(bucketName, key, expiration);
    }

}
```

## 试代码

```java
@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration("classpath:app-aliyun.xml")
public class AliyunTest {
  @Autowired
    AliyunOss aliyunOss;

    @Test
    public void testOss() throws FileNotFoundException, URISyntaxException {
        // 本地文件
        InputStream is = new FileInputStream("C:\\Users\\CNI\\Pictures\\5a55a640N8e90e084.jpg");
        // 上传到OSS的路径
        String key = "2018/1/12/img.jpg";
        // 调用上传方法
        String url = aliyunOss.upload(is, key, "image/jpeg");
        // 打印生成的访问路径
        System.out.println(url);
    }

}
```

