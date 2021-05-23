---
title: Linux文本处理与分析工具
date: 2021-05-21 15:50
categories: Linux运维
---

* 文件读取：cat、head、tail
* 字符处理：tr、sed、awk
* 字符截取：cut
* 搜索过滤：grep、egrep、fgrep、zgrep、zegrep、zfgrep
* 统计聚合：sort、uniq、wc

> GNU项目的coreutils源码地址：https://github.com/coreutils/coreutils

<!-- more -->

# cat,head,tail

`cat`是最常用的读取文件的命令，有时候还可以用`-n`参数输出每一行数据的行号：

```sh
cat -n file
```

`head`用来读取文件的前几行，`-n`参数可以指定行数(默认10行)：

```sh
head -n 100 file
```

`tail`用来读取文件末尾的几行，和`head`一样可以`-n`参数指定行数(默认10行)：

```sh
tail -n 100 file
```

`tail`命令通过`-f`参数可以读取文件最新的输入，按`Ctrl+C`可以中止：

```tail
tail -f file
```

# tr

[tr命令](https://www.gnu.org/software/coreutils/manual/html_node/tr-invocation.html)是`translate characters`的缩写，用来单个字符或字符集。

最简单的用法是直接传两个参数，将第一个参数字符替换成第二个参数字符。

```sh
echo 'abcdefg' | tr 'a' 'b'                  # bbcdefg
```

或者将一个字符集替换成另一个字符集：

```sh
echo 'abcdefg' | tr 'abcd' 'jkmn'            # jkmnefg
echo 'abcdefg' | tr "[:lower:]" "[:upper:]"  # ABCDEFG
```

如果带上`-d`参数的话，可以删除(delete)参数里指定的字符：

```sh
echo 'abcdefg' | tr -d 'abc'                 # defg
```

如果带上`-c`参数的话，表示取反：

```sh
# 只保留aceg，其他替换为b
echo 'abcdefg' | tr -c 'aceg' 'b'            # abcbebgb
# 只保留aceg，其他全部删除
echo 'abcdefg-abcdefg' | tr -cd 'aceg'       # acegaceg
```

# sed

[sed命令](https://www.gnu.org/software/sed/)是`stream editor`的缩写，和tr有点类似，但是功能更强大。

sed命令，用法如下

```sh
sed command [file ...]
```

其中command表示*editing commands*，命令语法如下：

```sh
[address[,address]]function[arguments]
```

`address`是可选的，是命令执行的范围

`function`是sed的功能函数

`arguments`是函数的参数。

下面看最常用的`s`——替换(substitute)函数。

## s替换函数

```sh
echo 'dcbaefg-dcbaefg' | sed 's/dcba/abcd/g'    # abcdefg-abcdefg
```

上面例子是：`s/regular expression/replacement/flags`，`s`表示**替换(substitute)**函数，`g`表示全局(global)替换的flag。

### flag

flag除了最常用的`g`以外，还有`i`（ignore,忽略大小写），还可以指定替换具体第几个匹配的字符串。

```sh
echo 'DCBAefg-dcbaefg' | sed 's/dcba/abcd/gi'   # abcdefg-abcdefg
echo 'DCBAefg-dcbaefg' | sed 's/dcba/abcd/2i'   # DCBAefg-abcdefg
```

### 正则替换

注意`s`是支持regular expression(正则表达式)的，替换(replacement)的字符支持占位符。

```sh
echo 'DCBAefg-dcbaefg' | sed 's/^\([A-Z]*\).*-/\1/g'
# DCBA
```

上面`^\([A-Z]*\).*`正则匹配大写字母开头的字符，`\1`把整个匹配的字符串第一个匹配组的替换整个匹配组。

```sh
echo 'DCBAefg-dcbaefg' | sed 's/^\([A-Z]*\).*-\([a-d]*\).*/\1-\2/g'
# DCBA-dcba
```

把正则写的再复杂一点，后面的`-\([a-d]*\).*`匹配`-`后面的a到d的字符，然后使用`\1-\2`替换整个匹配组。

正则的规则可以参考[regex cheat sheet](https://cheatography.com/hff/cheat-sheets/regex/)，这个命令使用的多好，就看正则表达式掌握的多好了。

### 范围替换

`s`命令前面还可以指定替换第几行的替换范围。

```sh
echo 'dcbaefg-dcbaefg\ndcbaefg-dcbaefg\ndcbaefg-dcbaefg\ndcbaefg-dcbaefg' > a.txt
dcbaefg-dcbaefg
dcbaefg-dcbaefg
dcbaefg-dcbaefg
dcbaefg-dcbaefg
cat a.txt | sed "2,3s/dcba/ABCD/g"
dcbaefg-dcbaefg
ABCDefg-ABCDefg
ABCDefg-ABCDefg
dcbaefg-dcbaefg
```

这个范围也就是`address`，数字表示的是行号，`$`表示最后一行

```sh
cat a.txt | sed '3,$s/dcba/abcd/g'
dcbaefg-dcbaefg
dcbaefg-dcbaefg
ABCDefg-ABCDefg
ABCDefg-ABCDefg
```

`address`还可以是正则表达式

```sh
echo 'DCBAefg-dcbaefg\ndcbaefg-dcbaefg\ndcbaefg-dcbaefg\nDCBAefg-dcbaefg' > a.txt
DCBAefg-dcbaefg
dcbaefg-dcbaefg
dcbaefg-dcbaefg
DCBAefg-dcbaefg
cat a.txt | sed '/DCBA/s/dcba/ABCD/g'
DCBAefg-ABCDefg
dcbaefg-dcbaefg
dcbaefg-dcbaefg
DCBAefg-ABCDefg
```

### 多个匹配

可以在一个表达式里写多个替换规则，用分号`;`隔开，比如下面的`1,2s/dcba/abcd/g`指定1～2行把`dcba`替换成`abcd`，`3,$s/DCBA/abcd/g`指定第三行3之后的行`DCBA`替换成`abcd`。

```sh
echo 'DCBAefg-dcbaefg\nDCBAefg-dcbaefg\nDCBAefg-dcbaefg\nDCBAefg-dcbaefg' > a.txt
DCBAefg-dcbaefg
DCBAefg-dcbaefg
DCBAefg-dcbaefg
DCBAefg-dcbaefg
cat a.txt | sed '1,2s/dcba/abcd/g; 3,$s/DCBA/abcd/g'
DCBAefg-abcdefg
DCBAefg-abcdefg
abcdefg-dcbaefg
abcdefg-dcbaefg
```

上面以`;`分割开的多个替换规则，与下面多个`-e`参数的方式等价：

```sh
sed -e '1,2s/dcba/abcd/g' -e '3,$s/DCBA/abcd/g'
```

## d函数

`d`函数是用来删除(delete)指定行的

```sh
cat a.txt
1abcdefg
2abcdefg
3abcdefg
4abcdefg
cat a.txt | sed '3d'
1abcdefg
2abcdefg
4abcdefg
cat a.txt | sed '2,3d'
1abcdefg
4abcdefg
cat a.txt | sed '/3a/d'
1DCBAefg
2DCBAefg
4DCBAefg
cat a.txt | sed '/[24]a/d'
1abcdefg
3abcdefg
```

## a函数和i函数

`a`函数和`i`函数分别表示追加(append)和插入(insert)

```sh
cat a.txt
1abcdefg
2abcdefg
3abcdefg
4abcdefg
cat a.txt | sed '3i---'
1abcdefg
2abcdefg
---
3abcdefg
4abcdefg
cat a.txt | sed '2,3i---'
1abcdefg
---
2abcdefg
---
3abcdefg
4abcdefg
cat a.txt | sed '3a---'
1abcdefg
2abcdefg
3abcdefg
---
4abcdefg
cat a.txt | sed '2,3a---'
1abcdefg
2abcdefg
---
3abcdefg
---
4abcdefg
```

更多的函数使用可以参考[man手册](https://linux.die.net/man/1/sed)

# awk

[awk](https://en.wikipedia.org/wiki/AWK)是一个文本处理语言，没错它是个编程语言，像其他语言一样支持if else，while循环等各种流程控制语句，也支持函数。让我们一步步地来慢慢丰满一个翔实的awk脚本。

我们就拿`netstat`输出结果来做示例：

```sh
netstat -nap tcp
tcp4       0      0  192.168.10.101.62566   199.232.68.133.443     SYN_SENT   
tcp4       0      0  192.168.10.101.62564   112.60.8.158.443       ESTABLISHED
tcp4       0      0  192.168.10.101.62563   112.60.14.170.443      ESTABLISHED
tcp4       0      0  192.168.10.101.62560   17.57.145.20.5223      ESTABLISHED
tcp4       0      0  192.168.10.101.62558   120.241.186.223.8080   ESTABLISHED
tcp4       0      0  192.168.10.101.62554   104.224.178.228.10000  ESTABLISHED
tcp4       0      0  127.0.0.1.1087         127.0.0.1.62553        ESTABLISHED
tcp4       0      0  127.0.0.1.62553        127.0.0.1.1087         ESTABLISHED
tcp4       0      0  127.0.0.1.62547        127.0.0.1.1087         CLOSE_WAIT 
tcp4       0      0  127.0.0.1.1087         127.0.0.1.62542        FIN_WAIT_2 
tcp4     129      0  127.0.0.1.62542        127.0.0.1.1087         CLOSE_WAIT 
tcp4       0      0  192.168.10.101.62529   104.224.178.228.10000  ESTABLISHED
...
```

如果我们只想要远端的socket地址(也就是第5列数据)，那我们可以用：

```sh
netstat -nap tcp | awk '{print $5}'
```

看起来很简单，其实这里面隐含了几个awk默认的设置：

`RS`(record separator)，默认是换行符，也就是说awk**默认会把一行当成一条记录**进行处理

`FS`(field separator)，默认是空格符，也就是说awk**默认会把以空格符隔开的字符当成一个字段**处理

`OFS`(output field separator)，默认是空格，也就是说输出的时候两个字段之间用空格来分开

`ORS`(output record separator)，默认是换行，也就是说输出的时候两条记录之间用换行分开

`NR`(number of record)，扫描过程中当前行的行号，类似的还有`NF`(number of field)，`FNR`(current file number of record)

另外，`$1`~`$n`表示第几个字段，`$0`表示整条记录。

> awk更多的内置变量可以看[awk的man手册](https://man7.org/linux/man-pages/man1/awk.1p.html)

如果我只想要TCP状态为`LISTEN`行数据，那我们可以用：

```sh
netstat -nap tcp | awk '$6=="LISTEN"'
tcp46      0      0  *.58299                *.*                    LISTEN     
tcp4       0      0  127.0.0.1.47833        *.*                    LISTEN     
tcp4       0      0  127.0.0.1.1087         *.*                    LISTEN     
tcp4       0      0  127.0.0.1.1080         *.*                    LISTEN     
tcp46      0      0  *.11085                *.*                    LISTEN     
tcp6       0      0  *.57977                *.*                    LISTEN     
tcp4       0      0  *.57977                *.*                    LISTEN     
tcp4       0      0  192.168.101.143.5786   *.*                    LISTEN     
tcp46      0      0  *.5432                 *.*                    LISTEN     
tcp4       0      0  *.59390                *.*                    LISTEN     
tcp4       0      0  127.0.0.1.59376        *.*                    LISTEN     
tcp4       0      0  127.0.0.1.63342        *.*                    LISTEN     
tcp4       0      0  127.0.0.1.6942         *.*                    LISTEN     
tcp46      0      0  *.13231                *.*                    LISTEN     
tcp4       0      0  127.0.0.1.8080         *.*                    LISTEN     
tcp46      0      0  *.8080                 *.*                    LISTEN     
tcp6       0      0  *.16800                *.*                    LISTEN     
tcp4       0      0  *.16800                *.*                    LISTEN 
```

如果我们想把状态为`ESTABLISHED`的第4列数据和第5列数据都拿出来：

```sh
netstat -nap tcp | awk '$6=="ESTABLISHED" {print $4,$5}'
192.168.101.143.58585 54.182.0.111.443
192.168.101.143.58171 59.36.121.194.8080
192.168.101.143.58148 108.177.97.188.5228
192.168.101.143.58144 59.37.96.203.8080
192.168.101.143.58143 59.37.96.203.8080
192.168.101.143.58141 203.119.169.109.443
192.168.101.143.58124 59.82.40.137.443
192.168.101.143.58029 104.22.52.136.443
192.168.101.143.58027 113.96.202.106.8080
fe80::32b5:4c0b:.1025 fe80::4e79:e738:.9763
fe80::32b5:4c0b:.1024 fe80::4e79:e738:.1024
192.168.101.143.53477 17.57.145.132.5223
192.168.101.143.55123 199.232.68.133.443
```

有些地址是ipv6的，我只想要ipv4的状态为`ESTABLISHED`的连接，那可以用：

```sh
netstat -nap tcp | awk '$1=="tcp4" && $6=="ESTABLISHED"'
tcp4       0      0  192.168.101.143.58654  114.55.105.214.443     ESTABLISHED
tcp4       0      0  192.168.101.143.58653  222.73.192.244.443     ESTABLISHED
tcp4       0      0  192.168.101.143.58652  203.208.40.34.443      ESTABLISHED
tcp4       0      0  127.0.0.1.59390        127.0.0.1.58651        ESTABLISHED
tcp4       0      0  127.0.0.1.58651        127.0.0.1.59390        ESTABLISHED
tcp4       0      0  192.168.101.143.58585  54.182.0.111.443       ESTABLISHED
tcp4       0      0  192.168.101.143.58171  59.36.121.194.8080     ESTABLISHED
tcp4       0      0  192.168.101.143.58148  108.177.97.188.5228    ESTABLISHED
tcp4       0      0  192.168.101.143.58144  59.37.96.203.8080      ESTABLISHED
tcp4       0      0  192.168.101.143.58143  59.37.96.203.8080      ESTABLISHED
tcp4       0      0  192.168.101.143.58141  203.119.169.109.443    ESTABLISHED
tcp4       0      0  192.168.101.143.58124  59.82.40.137.443       ESTABLISHED
tcp4       0      0  192.168.101.143.58029  104.22.52.136.443      ESTABLISHED
tcp4       0      0  192.168.101.143.58027  113.96.202.106.8080    ESTABLISHED
tcp4       0      0  192.168.101.143.53477  17.57.145.132.5223     ESTABLISHED
tcp4       0      0  192.168.101.143.55123  199.232.68.133.443     ESTABLISHED
```

当然也可以用正则：

```sh
netstat -nap tcp | awk '/^tcp4 / && $6=="ESTABLISHED"'
tcp4       0      0  192.168.101.143.58654  114.55.105.214.443     ESTABLISHED
tcp4       0      0  192.168.101.143.58653  222.73.192.244.443     ESTABLISHED
tcp4       0      0  192.168.101.143.58652  203.208.40.34.443      ESTABLISHED
tcp4       0      0  127.0.0.1.59390        127.0.0.1.58651        ESTABLISHED
tcp4       0      0  127.0.0.1.58651        127.0.0.1.59390        ESTABLISHED
tcp4       0      0  192.168.101.143.58585  54.182.0.111.443       ESTABLISHED
tcp4       0      0  192.168.101.143.58171  59.36.121.194.8080     ESTABLISHED
tcp4       0      0  192.168.101.143.58148  108.177.97.188.5228    ESTABLISHED
tcp4       0      0  192.168.101.143.58144  59.37.96.203.8080      ESTABLISHED
tcp4       0      0  192.168.101.143.58143  59.37.96.203.8080      ESTABLISHED
tcp4       0      0  192.168.101.143.58141  203.119.169.109.443    ESTABLISHED
tcp4       0      0  192.168.101.143.58124  59.82.40.137.443       ESTABLISHED
tcp4       0      0  192.168.101.143.58029  104.22.52.136.443      ESTABLISHED
tcp4       0      0  192.168.101.143.58027  113.96.202.106.8080    ESTABLISHED
tcp4       0      0  192.168.101.143.53477  17.57.145.132.5223     ESTABLISHED
tcp4       0      0  192.168.101.143.55123  199.232.68.133.443     ESTABLISHED
```

也可以对某一列进行正则匹配：

```sh
netstat -nap tcp | awk ' $6=="ESTABLISHED" && $1~/tcp4$/'
tcp4       0      0  127.0.0.1.59390        127.0.0.1.59071        ESTABLISHED
tcp4       0      0  127.0.0.1.59071        127.0.0.1.59390        ESTABLISHED
tcp4       0      0  192.168.101.143.59061  13.67.9.5.443          ESTABLISHED
tcp4       0      0  192.168.101.143.59058  54.182.0.111.443       ESTABLISHED
tcp4       0      0  192.168.101.143.59040  59.82.33.252.443       ESTABLISHED
tcp4       0      0  192.168.101.143.58171  59.36.121.194.8080     ESTABLISHED
tcp4       0      0  192.168.101.143.58148  108.177.97.188.5228    ESTABLISHED
tcp4       0      0  192.168.101.143.58144  59.37.96.203.8080      ESTABLISHED
tcp4       0      0  192.168.101.143.58143  59.37.96.203.8080      ESTABLISHED
tcp4       0      0  192.168.101.143.58141  203.119.169.109.443    ESTABLISHED
tcp4       0      0  192.168.101.143.58029  104.22.52.136.443      ESTABLISHED
tcp4       0      0  192.168.101.143.58027  113.96.202.106.8080    ESTABLISHED
tcp4       0      0  192.168.101.143.53477  17.57.145.132.5223     ESTABLISHED
tcp4       0      0  192.168.101.143.55123  199.232.68.133.443     ESTABLISHED
```

如果想要把表头也带上，就可以用`NR`变量：

```sh
netstat -nap tcp | awk '$1=="tcp4" && $6=="ESTABLISHED" || NR==2'
Proto Recv-Q Send-Q  Local Address          Foreign Address        (state)    
tcp4       0      0  127.0.0.1.59390        127.0.0.1.58672        ESTABLISHED
tcp4       0      0  127.0.0.1.58672        127.0.0.1.59390        ESTABLISHED
tcp4       0      0  192.168.101.143.58670  13.67.9.5.443          ESTABLISHED
tcp4       0      0  192.168.101.143.58669  13.67.9.5.443          ESTABLISHED
tcp4       0      0  192.168.101.143.58665  117.18.232.200.443     ESTABLISHED
tcp4       0      0  192.168.101.143.58652  203.208.40.34.443      ESTABLISHED
tcp4       0      0  192.168.101.143.58585  54.182.0.111.443       ESTABLISHED
tcp4       0      0  192.168.101.143.58171  59.36.121.194.8080     ESTABLISHED
tcp4       0      0  192.168.101.143.58148  108.177.97.188.5228    ESTABLISHED
tcp4       0      0  192.168.101.143.58144  59.37.96.203.8080      ESTABLISHED
tcp4       0      0  192.168.101.143.58143  59.37.96.203.8080      ESTABLISHED
tcp4       0      0  192.168.101.143.58141  203.119.169.109.443    ESTABLISHED
tcp4       0      0  192.168.101.143.58124  59.82.40.137.443       ESTABLISHED
tcp4       0      0  192.168.101.143.58029  104.22.52.136.443      ESTABLISHED
tcp4       0      0  192.168.101.143.58027  113.96.202.106.8080    ESTABLISHED
tcp4       0      0  192.168.101.143.53477  17.57.145.132.5223     ESTABLISHED
tcp4       0      0  192.168.101.143.55123  199.232.68.133.443     ESTABLISHED
```

只想看两个socket地址：

```sh
netstat -nap tcp | awk '$1=="tcp4" && $6=="ESTABLISHED" {print $4,$5}'
127.0.0.1.59390 127.0.0.1.58699
127.0.0.1.58699 127.0.0.1.59390
192.168.101.143.58680 59.36.89.161.443
192.168.101.143.58676 203.208.41.33.443
192.168.101.143.58171 59.36.121.194.8080
192.168.101.143.58148 108.177.97.188.5228
192.168.101.143.58144 59.37.96.203.8080
192.168.101.143.58143 59.37.96.203.8080
192.168.101.143.58141 203.119.169.109.443
192.168.101.143.58124 59.82.40.137.443
192.168.101.143.58029 104.22.52.136.443
192.168.101.143.58027 113.96.202.106.8080
192.168.101.143.53477 17.57.145.132.5223
192.168.101.143.55123 199.232.68.133.443
```

为这两列加上表头：

```sh
netstat -nap tcp | awk '$1=="tcp4" && $6=="ESTABLISHED" || NR==2 {print $4,$5}'
Local Address
127.0.0.1.59390 127.0.0.1.58707
127.0.0.1.58707 127.0.0.1.59390
192.168.101.143.58705 180.163.151.34.443
192.168.101.143.58703 54.182.0.111.443
192.168.101.143.58702 13.67.9.5.443
192.168.101.143.58701 13.67.9.5.443
192.168.101.143.58171 59.36.121.194.8080
192.168.101.143.58148 108.177.97.188.5228
192.168.101.143.58144 59.37.96.203.8080
192.168.101.143.58143 59.37.96.203.8080
192.168.101.143.58141 203.119.169.109.443
192.168.101.143.58124 59.82.40.137.443
192.168.101.143.58029 104.22.52.136.443
192.168.101.143.58027 113.96.202.106.8080
192.168.101.143.53477 17.57.145.132.5223
192.168.101.143.55123 199.232.68.133.443
```

但是发现表头有问题，因为表头行的`Local Address`被当成了两列，`$4`变成了`Local`，`$5`变成了`Address`。那我们手动加上表头：

```sh
netstat -nap tcp | awk '$1=="tcp4" && $6=="ESTABLISHED" {print $4,$5} BEGIN {print "本机地址","远程地址"}'
本机地址 远程地址
192.168.101.143.58743 113.96.233.155.443
192.168.101.143.58732 199.193.127.100.17898
127.0.0.1.1080 127.0.0.1.58731
127.0.0.1.58731 127.0.0.1.1080
192.168.101.143.58715 122.228.66.242.443
192.168.101.143.58705 180.163.151.34.443
192.168.101.143.58703 54.182.0.111.443
192.168.101.143.58171 59.36.121.194.8080
192.168.101.143.58148 108.177.97.188.5228
192.168.101.143.58144 59.37.96.203.8080
192.168.101.143.58143 59.37.96.203.8080
192.168.101.143.58141 203.119.169.109.443
192.168.101.143.58124 59.82.40.137.443
192.168.101.143.58029 104.22.52.136.443
192.168.101.143.58027 113.96.202.106.8080
192.168.101.143.53477 17.57.145.132.5223
192.168.101.143.55123 199.232.68.133.443
```

输出结果没有对齐，把输出用`\t`隔开：

```sh
netstat -nap tcp | awk '$1=="tcp4" && $6=="ESTABLISHED" {print $4,$5} BEGIN {print "本机地址","远程地址"}' OFS="\t"
本机地址	远程地址
192.168.101.143.58751	13.67.9.5.443
127.0.0.1.59390	127.0.0.1.58750
127.0.0.1.58750	127.0.0.1.59390
192.168.101.143.58743	113.96.233.155.443
192.168.101.143.58732	199.193.127.100.17898
127.0.0.1.1080	127.0.0.1.58731
127.0.0.1.58731	127.0.0.1.1080
192.168.101.143.58703	54.182.0.111.443
192.168.101.143.58171	59.36.121.194.8080
192.168.101.143.58148	108.177.97.188.5228
192.168.101.143.58144	59.37.96.203.8080
192.168.101.143.58143	59.37.96.203.8080
192.168.101.143.58141	203.119.169.109.443
192.168.101.143.58124	59.82.40.137.443
192.168.101.143.58029	104.22.52.136.443
192.168.101.143.58027	113.96.202.106.8080
192.168.101.143.53477	17.57.145.132.5223
192.168.101.143.55123	199.232.68.133.443
```

用`\t`和`print`不好对齐，那换成`printf`命令吧：

```sh
netstat -nap tcp | awk '$1=="tcp4" && $6=="ESTABLISHED" {printf "%-40s %s\n",$4,$5} BEGIN {printf "%-40s %s\n","本机地址","远程地址"}'
本机地址                             远程地址
192.168.101.143.59457                    13.67.9.5.443
192.168.101.143.59261                    104.22.52.136.443
192.168.101.143.59260                    104.22.52.136.443
192.168.101.143.59208                    183.3.235.67.8080
192.168.101.143.59200                    14.215.158.119.443
192.168.101.143.59182                    183.3.235.67.8080
192.168.101.143.59181                    183.3.235.67.8080
192.168.101.143.59176                    203.119.169.22.443
192.168.101.143.53477                    17.57.145.132.5223
192.168.101.143.55123                    199.232.68.133.443
```

最后输出一下总共有多少行：

```sh
netstat -nap tcp | awk '$1=="tcp4" && $6=="ESTABLISHED" {printf "%-40s %s\n",$4,$5; s+=1} BEGIN {printf "%-40s %s\n","本机地址","远程地址"} END {print "总连接数" s}'
本机地址                             远程地址
192.168.101.143.59555                    13.224.161.27.443
192.168.101.143.59554                    108.177.125.188.443
192.168.101.143.59550                    59.82.40.137.443
192.168.101.143.59548                    222.73.192.116.443
192.168.101.143.59547                    222.73.192.116.443
192.168.101.143.59208                    183.3.235.67.8080
192.168.101.143.59200                    14.215.158.119.443
192.168.101.143.59182                    183.3.235.67.8080
192.168.101.143.59181                    183.3.235.67.8080
192.168.101.143.59176                    203.119.169.22.443
192.168.101.143.53477                    17.57.145.132.5223
192.168.101.143.55123                    199.232.68.133.443
总连接数12
```

最后让我们看看这个慢慢被我们丰满的awk脚本：

```awk
$1=="tcp4" && $6=="ESTABLISHED" {
	printf "%-40s %s\n",$4,$5; 
	s+=1
}
BEGIN {
	printf "%-40s %s\n","本机地址","远程地址"
}
END {
	print "总连接数" s
}
```

`BEGIN`和`END`是awk执行开始前与执行完成后的标记，`$1=="tcp4" && $6=="ESTABLISHED"`是一个记录过滤规则的标记，`{}`内的是标记里的应该执行的命令。

```awk
BEGIN {
    print "netstat 统计每个连接状态的连接个数"
}

NR != 1 && NR != 2 {
    map[$6]+=1;
}

END {
    for (key in map)
        print key ":" map[key]
}
```

把这个脚本存到`awk.awk`文件中，在用`-f`选项使用这个文件：

```sh
netstat -nap tcp | awk -f awk.awk
netstat 统计每个连接状态的连接个数
LISTEN:16
FIN_WAIT_1:1
FIN_WAIT_2:1
CLOSE_WAIT:2
TIME_WAIT:22
ESTABLISHED:47
```

> Refs: Linux三剑客老大awk：https://zhuanlan.zhihu.com/p/68188159
>
> 更详细的awk语言的教程可以参考[gnu的awk用户指南](https://www.gnu.org/software/gawk/manual/gawk.html)

# cut

`cut`命令和上面几个相比简单很多

比如我们想对下面的结果进行截取

```sh
cat /dev/random | od -x | head > hex.dump
0000000      4b94    cf26    b92e    2750    1dba    efed    3d03    bf9c
0000020      adcb    9faf    d760    2474    71ae    773c    4a96    44aa
0000040      ec58    8cc8    8035    3bf8    8f33    7290    c97c    3e96
0000060      9c01    58d3    4d35    2330    918b    b745    ed5f    368b
0000100      11d1    2129    c554    ab0d    354a    9c75    cc35    7fbb
0000120      7943    4404    bb45    fd5f    299f    11c3    426d    ba39
0000140      a8d6    4c9c    22ec    ab00    1ea9    a5b7    e50e    7453
0000160      ff52    c73b    ad9b    2a2d    8fb8    8eff    0049    17d2
0000200      4401    8894    d714    7279    d44b    137d    bd42    ad76
0000220      5af5    7747    d8b2    9842    f3c1    8f20    f156    92cd
```

截取每一行的`1-25`个字符：

```sh
cat hex.dump | cut -c1-25
0000000      4b94    cf26
0000020      adcb    9faf
0000040      ec58    8cc8
0000060      9c01    58d3
0000100      11d1    2129
0000120      7943    4404
0000140      a8d6    4c9c
0000160      ff52    c73b
0000200      4401    8894
0000220      5af5    7747
```

`-c`代表以字符的方式进行截取`1-25`代表截取的范围，这个范围可以有多个，然后用`,`分隔开：

```sh
cat hex.dump | cut -c-10,25-35,40-
0000000   6    b92e  50    1dba    efed    3d03    bf9c
0000020   f    d760  74    71ae    773c    4a96    44aa
0000040   8    8035  f8    8f33    7290    c97c    3e96
0000060   3    4d35  30    918b    b745    ed5f    368b
0000100   9    c554  0d    354a    9c75    cc35    7fbb
0000120   4    bb45  5f    299f    11c3    426d    ba39
0000140   c    22ec  00    1ea9    a5b7    e50e    7453
0000160   b    ad9b  2d    8fb8    8eff    0049    17d2
0000200   4    d714  79    d44b    137d    bd42    ad76
0000220   7    d8b2  42    f3c1    8f20    f156    92cd
```

没有开头的范围代表从`1`开始，没有结尾代表截到最末尾。

`cut`还支持以字节为单位截取每一行：

```sh
cat hex.dump | cut -b -10,40-
0000000   50    1dba    efed    3d03    bf9c
0000020   74    71ae    773c    4a96    44aa
0000040   f8    8f33    7290    c97c    3e96
0000060   30    918b    b745    ed5f    368b
0000100   0d    354a    9c75    cc35    7fbb
0000120   5f    299f    11c3    426d    ba39
0000140   00    1ea9    a5b7    e50e    7453
0000160   2d    8fb8    8eff    0049    17d2
0000200   79    d44b    137d    bd42    ad76
0000220   42    f3c1    8f20    f156    92cd
```

`cut`也可以和awk一样实现简单的字段截取，但是`cut`只支持单字符的分隔符，所以先把上面的文件转成`,`分割

```sh
cat hex.dump | sed -E 's/ +/,/g' | cut -d',' -f1,4,5
```

`-d`(delimiter)指定分隔字符，`-f`(fields)指定字段的范围。

# grep,egrep,fgrep,zgrep,zegrep,zfgrep

`grep`是使用正则搜索过滤的常用工具。与之相关的还有egrep、fgrep、zgrep、zegrep、zfgrep，他们的含义如下：

* `grep`: Global Regular Expressions Print
* `egrep`: Extended Global Regular Expressions Print，与`grep -E`等价。`egrep`是对`grep`的扩展，原来的grep不支持`+`,`?`,`{m,n}`,`(a|b)`等正则，egrep提供了这些支持。
* `fgrep`: Fixed-string Global Regular Expressions Print，与`grep -F`等价。`fgrep`就是简单的字符匹配，不使用正则。
* `zgrep`: Zip Global Regular Expressions Print，与`grep -Z`等价，可以对压缩文件进行匹配。
* `zegrep`: zgrep 和 egrep的结合
* `zfgrep`: zgrep 和 fgrep的结合

像线程堆栈这种，有时候需要查看匹配项的前几行或后几行，-A(after),-B(before),-C(context)返回匹配项上下文的几行

```sh
cat numbers.txt
1
2
3
4
5
6
7
8
9
# 后两行
cat numbers.txt | grep -A2 '5'
5
6
7
# 前两行
cat numbers.txt | grep -B2 '5'
3
4
5
# 前后各两行
cat numbers.txt | grep -C2 '5'
3
4
5
6
7
```
使用`-v`(invert)反向匹配，也就是现实不匹配规则的行
```sh
cat numbers.txt | egrep -v '(0|2|4|6|8)'
1
3
5
7
9
```
注意上面使用了`egrep`，因为`grep`不支持`(|)`的写法
使用`-c`参数可以现实匹配的行数
```sh
cat numbers.txt | egrep -c '(0|2|4|6|8)'
5
```
使用`-m`可以选择显示几个匹配行，`-n`可以现实匹配的行号
```sh
cat numbers.txt | egrep -nm2 '(0|2|4|6|8)'
2:2
4:4
```

# sort

`sort`用来进行排序的，`sort`是支持大文件排序的，用到外排序的算法，所以[`sort`的源码](https://github.com/coreutils/coreutils/blob/v8.32/src/sort.c)也是学习外排序算法的最佳工程实践。

```sh
cat file | sort
```

`sort`命令默认是按照字母序排序的，所以`55`会排在`6`的前面，如果使用数字序可以用`-n`(**numeric**)参数：

```sh
cat file | sort -n
```

对于包含`1.6E-35`这类[科学记数法](https://en.wikipedia.org/wiki/Scientific_notation)的数字排序，可以使用`-g`(**general-numeric**)参数

```sh
cat file | sort -g
```

逆序`-r`(**reverse**)：

```sh
cat file | sort -r
```

随机序(或者叫随机打乱)`-R`(**random**):

```sh
cat file | sort -R
```

对字段(**key**)进行排序，比如下面对`/etc/passwd`文件，按照`:`字符进行字段分割，然后对第3个字段进行排序：

```sh
sort --field-separator=: -k=3n /etc/passwd
```

# uniq

`uniq`通常和`sort`配合使用，因为重复的行如果不相邻，`uniq`命令无法检测重复，所以排重之前必须先排序。

文件内容去重，每个唯一的行只显示一次：

```sh
sort file | uniq
```

只显示不重复的唯一行：

```sh
sort file | uniq -u
```

只显示重复的行：

```sh
sort file | uniq -d
```

去重并显示每个唯一行出现的次数：

```sh
sort file | uniq -c
```

按照每个唯一行出现的次数倒序排序：

```sh
 sort file | uniq -c | sort -nr
```

# wc

`wc`(word count)不仅可以用来统计单词个数，还可以统计数据行数，字符个数。

行数：

```sh
cat file | wc -l
```

单词个数：

```sh
cat file | wc -w
```

字符个数：

```sh
# 单字节字符
cat file | wc -c
# 多字节字符
cat file | wc -m
```

# 实践试试看

统计nginx访问次数最多的10个ip地址：

```sh
cat access.log | awk '{print $1}' | sort | uniq -c | sort -nr | head -n 10
```

统计ssh登陆失败的用户与来访ip地址：

```sh
cat /var/log/secure | grep 'Failed password' | sed 's/.*Failed password for \(.*\) from \(.*\) port.*/\1 \2/g' | sed 's/invalid user //g' | sort
```

统计ssh登陆失败的最多的10个ip地址：

```sh
cat /var/log/secure | grep 'Failed password' | sed 's/.*Failed password for \(.*\) from \(.*\) port.*/\1 \2/g' | sed 's/invalid user //g' | awk '{print $2}' | sort | uniq -c | sort -nr | head
   8801 115.84.121.234
   3147 115.159.25.205
   2232 175.190.126.170
   1051 95.111.236.177
    989 221.234.36.43
    383 45.248.189.3
    367 61.222.164.71
    315 205.185.120.48
    315 159.75.91.118
    279 222.186.150.98
```

统计ssh暴力攻击最喜欢用那些用户名：

```sh
cat /var/log/secure | grep 'Failed password' | sed 's/.*Failed password for \(.*\) from \(.*\) port.*/\1 \2/g' | sed 's/invalid user //g' | awk '{print $1}' | sort | uniq -c | sort -nr | head -20
  34167 root
   1585 admin
    960 user
    791 oracle
    757 test
    462 ftpuser
    421 ubuntu
    375 postgres
    352 test1
    323 test2
    313 git
    292 usuario
    197 mysql
    170 guest
    158 hadoop
    157 nagios
    150 deploy
    141 pi
    133 jenkins
    119 support
```

