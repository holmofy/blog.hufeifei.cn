---
title: Tesseract与tess4j验证码识别
date: 2018-06-30
categories: JAVA
---

验证码，英文名[CAPTCHA](https://en.wikipedia.org/wiki/CAPTCHA)，全称叫做：全自动区分计算机和人类的图灵测试。验证码主要为了防一些不怀好意的人(程序猿)，避免批量注册账户，暴力尝试多次登录失败等一些恶意行为。

最经典的就是文字型的验证码：

![搜狐验证码](https://v4.passport.sohu.com/i/captcha/picture?pagetoken=1)![腾讯验证码](https://hr.tencent.com/get_rand.php)![阿里云验证码](https://pin.aliyun.com/get_img?sessionid=0)![搜狐快站验证码](http://changyan.kuaizhan.com/verifyCode)![新浪微博验证码](https://weibo.com/signup/v5/pincode/pincode.php?sinaId=0)![网易邮箱验证码](http://reg.email.163.com/unireg/call.do?cmd=register.verifyCode)![新浪微博](https://login.sina.com.cn/cgi/pin.php)![新浪](https://mail.sina.com.cn/cgi-bin/imgcode.php)![华为验证码](https://uniportal.huawei.com/accounts/randomcode.jpg)![中国移动](https://login.10086.cn/captchazh.htm?type=01)![中国移动](https://login.10086.cn/captchazh.htm?type=02)![中国移动](https://login.10086.cn/captchazh.htm?type=03)![中国移动](https://login.10086.cn/captchazh.htm?type=05)![中国移动](https://login.10086.cn/captchazh.htm?type=06)![中国移动](https://login.10086.cn/captchazh.htm?type=08)![支付宝验证码](https://omeo.alipay.com/service/checkcode?sessionID=0)![360验证码](https://passport.360.cn/captcha.php?m=create&app=i360&scene=strictreg&level=default&sign=5a63a7)![金山](https://account.wps.cn/p/captcha)![网易邮箱验证码](https://ssl.mail.163.com/mobilemail/verifyCodeImg.jsp)![直聘](https://signup.zhipin.com/captcha/?randomKey=)![拉钩](https://passport.lagou.com/vcode/create)![世纪天成](https://captcha.tiancity.com/getimage.ashx?fid=400)

简单的文字型验证码容易被OCR识别，所以程序猿们让文字随机旋转、扭曲、黏连，在验证码上加干扰线、加噪点以降低自动化程序的识别率，但是很多扭曲变形的文字连人都识别不出来，比如中国移动这个验证码：

![中国移动验证码](https://login.10086.cn/captchazh.htm?type=06)

这些诡异的验证码反而降低了用户体验，所以现在比较流行的是行为式验证码

![用户点击](http://tva1.sinaimg.cn/large/bda5cd74gy1frwvdew2okj20d707kjuz.jpg)

![用户滑动](http://tva1.sinaimg.cn/large/bda5cd74gy1frwvgc9uapj20bi023743.jpg)

![用户滑动](http://tva1.sinaimg.cn/large/bda5cd74gy1frww3y1hk4j20a805ut8w.jpg)

这篇文章主要讨论入门级的传统文字型验证码的识别。

因为我自己也没有深入研究过图形学，所以这里拿一个最简单验证码举例：

 ![验证码](http://219.229.243.77/CheckCode.aspx)

## Tesseract

这里用[开源的Tesseract-OCR引擎](https://github.com/tesseract-ocr/tesseract)来识别字符，Tesseract目前最新版是4.0，[Wiki](https://github.com/tesseract-ocr/tesseract/wiki)里给出了各平台的安装方式。

> [tesseract-ocr](https://en.wikipedia.org/wiki/Tesseract_%28software%29)是谷歌赞助开发的一款开源的光学字符识别([**Optical character recognition**](https://en.wikipedia.org/wiki/Optical_character_recognition))引擎，通常拿来进行文档扫描，图片字符识别。最常见的例子就是车牌号的识别与pdf扫描

Windows下载地址：https://github.com/tesseract-ocr/tesseract/wiki/Downloads

除了软件安装包，还有官方提供的训练数据：https://github.com/tesseract-ocr/tesseract/wiki/Data-Files

我们需要设置`TESSDATA_PREFIX`变量为训练数据所在的目录。

![验证码](http://tva1.sinaimg.cn/large/bda5cd74gy1fssflvy51vj201g00p741.jpg)

将Tesseract安装目录配置到环境变量后，我们可以直接用`tesseract`命令来对上面的验证码进行识别：

![tesseract命令识别验证码](http://tva1.sinaimg.cn/large/bda5cd74gy1fssg03d4buj209n01p0sj.jpg)

> `--psm`选项用于是`page segmentation mode`，用于设置分页模式，tesseract有13中分页模式，因为验证码中字符基本在一行显示，所以选用`7`单行文本模式。
>
> `stdout`表示我们将识别结果输出到标准输出流(也就是控制台)，如果你想把识别结果输出到一个文件中可以直接指定文件名。比如`tesseract CheckCode.jpg --psm 7 output`会把识别结果输出到`output.txt`文件中。
>
> 关于tesseract命令的更多选项可以参看`tesseract --help`；

从控制台的输出结果可以看出直接拿原始图片进行识别，准确率非常的低。所以我们要对图片进行预处理。

## 图片预处理

预处理要根据验证码的背景、干扰线、旋转、扭曲、黏连的程度进行处理。不同的验证码需要做的处理也不一样。

常见的预处理手段有：灰度化与二值化(针对颜色各异的噪点与干扰线)、去除噪点、去除干扰线、字符分割(针对字符黏连)、字符归一化(针对字符旋转扭曲)。

> 这里有两个专栏专门针对[图像验证码识别](https://blog.csdn.net/ysc6688/article/category/2913009)和[OpenCV图像识别](https://blog.csdn.net/column/details/opencv3.html)进行了研究。

针对上面那个简单的验证码，我们用java提供的BufferedImage API就能进行一些简单的预处理。

```java
/**
 * 裁剪图片：去掉黑边
 */
public static BufferedImage clipImage(BufferedImage srcImage) {
    return srcImage.getSubimage(1, 1, srcImage.getWidth() - 2, srcImage.getHeight() - 2);
}

/**
 * 灰度化
 */
public static BufferedImage grayImage(BufferedImage srcImage) {
    return copyImage(srcImage, new BufferedImage(srcImage.getWidth(), srcImage.getHeight(), BufferedImage.TYPE_BYTE_GRAY));
}

/**
 * 二值化
 */
public static BufferedImage binaryImage(BufferedImage srcImage) {
    return copyImage(srcImage, new BufferedImage(srcImage.getWidth(), srcImage.getHeight(), BufferedImage.TYPE_BYTE_BINARY));
}

public static BufferedImage copyImage(BufferedImage srcImage, BufferedImage destImage) {
    for (int y = 0; y < srcImage.getHeight(); y++) {
        for (int x = 0; x < srcImage.getWidth(); x++) {
            destImage.setRGB(x, y, srcImage.getRGB(x, y));
        }
    }
    return destImage;
}
```

> 由于验证码相对简单，也不用我们设置二值化的阈值

```java
@Test
public void testImageProcess() throws IOException {
    BufferedImage originImage = ImageIO.read(new File("/path/to/CheckCode.jpg"));
    BufferedImage processedImage = ImageUtils.binaryImage(ImageUtils.grayImage(ImageUtils.clipImage(originImage)));
    ImageIO.write(processedImage, "JPEG", new File("/path/to/output.jpg"));
}
```

经过处理后的图片变成这个样子：

![处理后的图片效果](http://tva1.sinaimg.cn/large/bda5cd74gy1fssgvwtrysj201e00n0sh.jpg)

再去用`tesseract`命令进行识别：

![处理后识别](http://tva1.sinaimg.cn/large/bda5cd74gy1fssgzl33wxj20cb02hdfo.jpg)

识别率仍然很低(这么清晰的字还是别不出来，艹)。

识别率低主要原因是tesseract默认的训练数据主要用于正常文字(如扫描版pdf)的识别，不怎么适用于这种验证码，我们要专门针对验证码训练出自己的一套数据。

## tesseract训练

首先我们针对**验证码中出现的字体**生成图像，以这些字体图像为基础进行训练。

1、首先准备一个`chars.txt`文件，文件中包含验证码中可能出现的字符。

![所有可能出现的字符](http://tva1.sinaimg.cn/large/bda5cd74gy1fsshv48cr5j20i202zmx2.jpg)

2、使用[jTessBoxEditor](https://sourceforge.net/projects/vietocr/files/jTessBoxEditor/)可视化工具或者使用tesseract自带的`text2image`命令生成训练图片(tif格式)与字符定位文件(box后缀)

![jTessBoxEditor生成字体训练图片](http://tva1.sinaimg.cn/large/bda5cd74gy1fssi0dzyzij210u05vwen.jpg)

> 同样地，`text2image --text=chars.txt --outputbase=myeng.consolas.exp0 --font="Consolas" --fonts_dir=C:\Windows\Fonts`命令可以生成tif与box文件。

这里用`BellMT`、`Consolas`、`仿宋`、`华文楷体`、`微软雅黑`字体以及它们加粗的样式生成了一下几个文件。

![生成的字体文件和字符定位文件](http://tva1.sinaimg.cn/large/bda5cd74gy1fssic3kzoaj207s0dl0ss.jpg)

使用jTessBoxEditor工具可以更直观地查看tif文件与关联的box文件中的字符定位信息：

![box定位](http://tva1.sinaimg.cn/large/bda5cd74gy1fssixirx32j211x0k5ab3.jpg)

同时生成了一个`font_properties`后缀的文件：

![字体属性文件](http://tva1.sinaimg.cn/large/bda5cd74gy1fssigcq9rhj2066056745.jpg)

```
<font name> <italic?> <bold?> <fixed?> <serif?> <fraktur?>
```

每一列分别表示：字体名、[斜体](https://en.wikipedia.org/wiki/Italic_type)、加粗、[无衬线体](https://en.wikipedia.org/wiki/Sans-serif)、[衬线体](https://en.wikipedia.org/wiki/Serif)、[哥特手书体](https://en.wikipedia.org/wiki/Fraktur)

3、训练字体识别数据

可以直接使用jTessBoxEditor工具进行训练。

![使用jTessBoxEditor训练](http://tva1.sinaimg.cn/large/bda5cd74gy1fssj7idl4ij211y0jymy7.jpg)

这里面主要执行了一下几个步骤：

(1)、生成`tr`后缀的训练文件，这里面包含了训练数据。这一步本质上就是执行了下面的命令

> `tesseract myeng.consolas.exp0.tif myeng.consolas.exp0 nobatch box.train`

(2)、计算字符集：收集所有box文件中出现的字符并生成unicharset文件。

> `unicharset_extractor  myeng.consolas.exp0.box`
>
> 命令后面可以输入多个box文件

(3)、收集`tr`训练数据中的形状信息，生成形状表(Shape Table)，会生成一个`shapetable`文件

> `shapeclustering -F myeng.font_properties -U unicharset myeng.consolas.exp0.tr`

(4)、使用mftraining进行功能训练(feature training)，这一步会生成`inttemp`文件(形状原型)和`pffmtable`文件(字符的预期特征数)

> `mftraining -F myeng.font_properties -U unicharset -O myeng.unicharset myeng.consolas.exp0.tr`

(5)、使用cntraining进行，这一步生成`normproto`文件(normalization prototypes，字符标准化灵敏度原型)

>`cntraining myeng.consolas.exp0.tr`

(6)、生成字典数据。**这一步是可选的**，主要是为了帮助Tesseract决定字符组合的可能性，比如一个单词中两个辅音字母出现在一起的概率是非常小的，如果提供一个单词表，对识别肯定有所帮助。这一步会生成几个dawg文件（Directed Acyclic Word Graph，定向非循环词图）。

> `wordlist2dawg myeng.frequent_words_list myeng.freq-dawg myeng.unicharset`

(7)、含糊词配置，这步jTessBoxEditor工具没有支持，因为含糊词需要我们自己配置。含糊词指的是识别过程中可能导致误判的字符，比如识别过程中会把一个双引号`"`识别成了两个单引号`'`。模糊文件命名为`unicharambigs`，配置如下：

```
v1
2       ' '     1       "       1
1       m       2       r n     0
3       i i i   1       m       0
```

第一行是版本标识，其余行是以制表符`\t`分隔的字段：

`<原匹配字符数> <原匹配字符> <目标匹配字符数> <目标匹配字符> <类型>`，其中类型为表示强制替换，类型为0表示非强制替换。

`2 '' 1 " 1`表示强制将两个连续的单引号替换成一个双引号。

(8)、合并所有文件(包括shapetable、normproto、inttemp、pffmtable、unicharset、unicharambigs)，将它们重命名为相同的前缀，比如`myeng`，然后用`combine_tessdata myeng`命令即可合并。

合并后生成一个myeng.traineddata文件，这个就是我们最终的训练数据文件了。



训练完成后拿我们训练字体得到的数据进行识别：

```shell
tesseract output.jpg --tessdata-dir /training/tessdata -l myeng --psm 7 stdout
```

![训练后的数据进行识别](http://tva1.sinaimg.cn/large/bda5cd74gy1fssmbiqst5j20ib01zdfq.jpg)

你会发现识别率高了很多，不过仍然有错。

## 采样训练

我们采集一些验证码的样本，特别是那些容易识别错的验证码。

![采集验证码](http://tva1.sinaimg.cn/large/bda5cd74gy1fssmfkvr5bj20ib0a8dgd.jpg)

使用jTessBoxEditor生成样本的box文件，注意语言包选用之前生成的训练数据

![使用jTessBoxEditor生成box文件](http://tva1.sinaimg.cn/large/bda5cd74gy1fssmi82e0zj211f0ildh6.jpg)

用jTessBoxEditor对生成的box进行矫正。

矫正后和之前使用字体生成的box文件一起进行一轮训练，最后生成训练数据。

## 在Java中使用tesseract-ocr

在Java中我们可以使用[tess4j](https://github.com/nguyenq/tess4j)来调用tesseract-ocr的API进行识别

[tess4j](https://github.com/nguyenq/tess4j)是使用[JNA(Java Native Access)](https://github.com/java-native-access/jna )对tesseract-ocr进行了一层包装，JNA是社区开发的，与java官方的JNI有所不同。用起来比JNI简单，不需要用javah生成样板代码再用c调用原生API。

```java

public class ImageUtils {


    private static ITesseract TESSERACT = new Tesseract();

    static {
        // 配置自己的训练数据
        TESSERACT.setDatapath("/path/to/tessdata");
        TESSERACT.setLanguage("myeng");
    }

    /**
     * 识别验证码
     *
     * @param srcImage 图片
     * @return 图片中的验证码
     */
    public static String recognition(BufferedImage srcImage) throws TesseractException, IOException {
        BufferedImage reducedImage = binaryImage(grayImage(clipImage(srcImage)));
        return TESSERACT.doOCR(reducedImage).trim();
    }
    ...
}
```



参考链接：

https://github.com/tesseract-ocr/tesseract/wiki/TrainingTesseract-4.00

https://github.com/tesseract-ocr/tesseract/wiki/Training-Tesseract-3.03–3.05