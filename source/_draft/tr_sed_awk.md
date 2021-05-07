* 单字符处理：tr、sed
* 自定义分隔符处理：awk
* 字符截取：cut
* 统计聚合：sort、uniq、wc

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

## sed

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

### s替换函数

```sh
echo 'dcbaefg-dcbaefg' | sed 's/dcba/abcd/g'    # abcdefg-abcdefg
```

上面例子是：`s/regular expression/replacement/flags`，`s`表示**替换(substitute)**函数，`g`表示全局(global)替换的flag。

#### flag

flag除了最常用的`g`以外，还有`i`（ignore,忽略大小写），还可以指定替换具体第几个匹配的字符串。

```sh
echo 'DCBAefg-dcbaefg' | sed 's/dcba/abcd/gi'   # abcdefg-abcdefg
echo 'DCBAefg-dcbaefg' | sed 's/dcba/abcd/2i'   # DCBAefg-abcdefg
```

#### 正则替换

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

#### 范围替换

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

#### 多个匹配

可以在一个表达式里写多个替换规则，比如下面的`1,2s/dcba/abcd/g`指定1～2行把`dcba`替换成`abcd`，`3,$s/DCBA/abcd/g`指定第三行3之后的行`DCBA`替换成`abcd`。

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

### d函数

`d`函数是用来删除(delete)指定行的

```sh
echo '1abcdefg-dcbaefg\n2abcdefg-dcbaefg\n3abcdefg-dcbaefg\n4abcdefg-dcbaefg' > a.txt
1abcdefg-dcbaefg
2abcdefg-dcbaefg
3abcdefg-dcbaefg
4abcdefg-dcbaefg
cat a.txt | sed '3d'
1abcdefg-dcbaefg
2abcdefg-dcbaefg
4abcdefg-dcbaefg
cat a.txt | sed '2,3d'
1abcdefg-dcbaefg
4abcdefg-dcbaefg
cat a.txt | sed '/3a/d'
1DCBAefg-dcbaefg
2DCBAefg-dcbaefg
4DCBAefg-dcbaefg
cat a.txt | sed '/[24]a/d'
1abcdefg-dcbaefg
3abcdefg-dcbaefg
```

### a函数和i函数

`a`函数和`i`函数分别表示追加(append)和插入(insert)

```sh
echo '1abcdefg-dcbaefg\n2abcdefg-dcbaefg\n3abcdefg-dcbaefg\n4abcdefg-dcbaefg' > a.txt
1abcdefg-dcbaefg
2abcdefg-dcbaefg
3abcdefg-dcbaefg
4abcdefg-dcbaefg
cat a.txt | sed '3d'
1abcdefg-dcbaefg
2abcdefg-dcbaefg
4abcdefg-dcbaefg
cat a.txt | sed '2,3d'
1abcdefg-dcbaefg
4abcdefg-dcbaefg
cat a.txt | sed '/3a/d'
1DCBAefg-dcbaefg
2DCBAefg-dcbaefg
4DCBAefg-dcbaefg
cat a.txt | sed '/[24]a/d'
1abcdefg-dcbaefg
3abcdefg-dcbaefg
```



