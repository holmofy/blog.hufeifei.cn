---
title: 一枚Javer对Ruby的吐槽
date: 2019-01-13
categories: Ruby
mathjax: true
---
> 公司收购了个项目，技术都很老，Web用的Ruby on Rails，前端还在用jQuery，后台定时任务用Java(用的技术也都很老)。自己以前没有接触过Ruby，只是听几个朋友大学里玩过这玩意儿，所以对我来说Ruby就是一门全新的语言。我个人是非常抵触学一门新语言的，Java都还没吃透就并行学Ruby，只会分散深入学习Java的精力。但没办法谁让公司抽到我，还是好好学吧！

##1、安装Ruby

Windows上傻瓜式安装Ruby就不赘述了(有[RubyInstaller](https://rubyinstaller.org/)会点下一步就行)，其他操作系统都可以用系统相应的包依赖管理器安装Ruby(Linux上的yum、apt-get，Mac上的homebrew)。但是这些包管理器通常只能安装yum源中已有的二进制包，如果你想要的Ruby版本没有在对应操作系统已编译的二进制包，那就只能下载源码自行编译了。虽然通常只需三个命令就能完成编译，但是多个Ruby之间版本的切换也是个问题，这也是小型脚本语言的痛。像Java大版本内不会有语法上改变，小版本也只是修复些bug，切换版本重新编译费时费劲，但Node.js和Ruby这样小巧而且版本间差异相对较大的语言，就需要运行环境的版本管理工具了。

大多数Rubyist都使用Ruby管理工具管理多个Ruby，[官方列出来的Ruby管理工具](https://www.ruby-lang.org/en/documentation/installation/#managers)有四个，最常用也是最好用的就是[RVM](https://github.com/rvm/rvm)(和Node.js对应有[NVM](https://github.com/creationix/nvm))。安装RVM只需要从`https://get.rvm.io`下载一个[rvm-installer的Bash脚本](https://raw.githubusercontent.com/rvm/rvm/master/binscripts/rvm-installer)即可。`ruby-installer`支持默认支持两个版本`master`和`stable`，需要稳定版直接`curl -sSL https://get.rvm.io | bash -s stable`一个管道命令解决。

> 和`nvm`的`$HOME/.nvm`一样`rvm`也会在用户目录下创建`.rvm`目录保存各个版本的Ruby

安装完RVM后，使用以下命令就可以安装各种版本的Ruby了。

`rvm install INTERPRETER[-VERSION] OPTIONS`

其中`INTERPRETER`指的是不同语言实现的Ruby解释器，没有指定默认就是[CRuby](https://en.wikipedia.org/wiki/Ruby_MRI)，也可以指定Java实现的jruby或.Net实现的ironruby等，使用`rvm list known`可以查看可选的Ruby版本。

切换Ruby版本只需用`rvm use INTERPRETER[-VERSION]`命令。

##2、Ruby周边

**gem**

[RubyGems](https://en.wikipedia.org/wiki/RubyGems)是Ruby的包管理工具(和Java的Maven，JS的npm类似)，从Ruby1.8开始就称为了Ruby默认的包管理工具。使用上和npm大体一致，详情可以参看[官方文档](https://guides.rubygems.org/)

[**bundler**](https://github.com/bundler/bundler)

如果说gem是npm的阉割版，bundler就是为了补全被阉割的那部分。[gem的`.gemspec`](https://guides.rubygems.org/specification-reference/)虽然定义了包本身的信息，但是不能像`npm install`一样安装并管理这个包依赖的其他包。Bundler需要定义一个`Gemfile`的文件，这个文件中可以定义应用依赖的gems的兼容版本(和npm的package.json类似)。

所以[Bundler官网](https://bundler.io/)标题就说Bundler是管理Ruby的gems最好的方式。

> Gemfile与.gemspec的区别详请参考[这篇文章](https://medium.com/@divya.n/gemfile-vs-gemspec-ee72512da246)

**Ruby on Rails**

[Ruby on Rails](https://rubyonrails.org/)是一个基于Ruby的MVC框架，和[express.js](http://expressjs.com/)类似，也正是因为这个框架使得默默无闻的Ruby一夜崛起。也是这个框架最早提出[Convention Over Configuration](https://en.wikipedia.org/wiki/Convention_over_configuration)的概念，后来其他语言的框架也纷纷效仿，如今口口相传的SpringBoot也深受它的影响。

##3、 初学Ruby的20条经验

**1、** Ruby和JS、Python一样是解释性脚本语言，使用源文件即可执行，按照惯例源文件以`.rb`为后缀。和Python的强制缩进不通，Ruby的缩进并不重要，但是为了代码可读性必须得缩进。

**2、** 不像JS中`0`、`undefined`、`null`、`""`等值都能被转成false，在Ruby中除了`false`和`nil`这两个保留字，其他全为true。

**3、** Ruby方法调用时，括号是可选的。方法中如果没有明确的`return`，则返回最后一条语句的结果。

```ruby
foobar
foobar()
foobar(a,b,c)
foobar a,b,c

def my_method(a,b)
    puts a+b
    a*b     # 这条语句的结果将会作为返回值
end
```

> 详情参考[官方文档](https://docs.ruby-lang.org/en/2.6.0/syntax/methods_rdoc.html)

**4、** Ruby除了支持`+`、`-`、`*`、`/`、`%`等算数运算符外，还支持`**`幂运算符，但是不支持`++`、`--`运算符。这点和Python一致。在C和Java中`%`表示取余操作，但Python和Ruby中`%`表示求模运算。

$$
取余：rem(a, b) = a-int(a/b)*a
$$
$$
求模： mod(a, b) = a-floor(a/b)*a
$$

```
// Java 结果的正负与被除数一致
System.out.println(5 % 3);     //  2
System.out.println(-5 % 3);    // -2
System.out.println(5 % -3);    //  2
System.out.println(-5 % -3);   // -2

// Ruby 结果的正负与除数一致
puts (5 % 3)     # prints  2  
puts (-5 % 3)    # prints  1  
puts (5 % -3)    # prints -1  
puts (-5 % -3)   # prints -2

// Python
print (5 % 3)     # prints  2  
print (-5 % 3)    # prints  1  
print (5 % -3)    # prints -1  
print (-5 % -3)   # prints -2  
```

**5、** Ruby中一切皆对象，不像Java还有几个基本类型。Ruby中数值类型也是对象。Ruby中Float是双精度浮点型(也就是Java里的Double)。并且Ruby原生就支持BigNum(也就是Java中的BigInteger)，所以再也不用担心算数溢出的问题了。并且Ruby 2.4之后[Fixnum和Bignum统一为Integer](https://github.com/rails/rails/pull/25056)了。

```ruby
##Before Ruby 2.4
1.class         #=> Fixnum
(2 ** 62).class #=> Bignum

##Ruby 2.4
1.class         #=> Integer
(2 ** 62).class #=> Integer
```

> 有关Fixnum和Bignum更多细节可参考[这篇文章](https://blog.bigbinary.com/2016/11/18/ruby-2-4-unifies-fixnum-and-bignum-into-integer.html)，除了Integer和Float这两种常用的数值类型，[Numeric](https://docs.ruby-lang.org/en/2.0.0/Numeric.html)还有[Rational](https://docs.ruby-lang.org/en/2.0.0/Rational.html)(有理数)和[Complex](https://docs.ruby-lang.org/en/2.0.0/Complex.html)(复数)两个子类，用处不大这里不过多讨论。

**6、** Ruby的变量命名会影响变量的作用域：

* 局部变量的命名规则和C系列语言命名一样，字母下划线开头数字字母下划线的组合；

  > 使用`local_variables`方法可以查看当前作用域内的局部变量

* 在局部变量的开头加上`@`，表示这个变量是实例对象的属性变量(@sign，@_，@Counter)；

  > 使用`instance_variables`方法可以查看对象的实例变量

* 在局部变量的开头加上`@@`，表示类变量；

  > 使用`class_variables`方法可以查看类变量

* 在局部变量的开头加上`$`，表示全局变量；

  > 使用`global_variables`方法可以查看全局变量

* 按照Ruby的约定，全大写的变量是常量，Ruby允许修改常量，但是会报警告。

  ```ruby
  PI = 3.141592653  # 第一次定义常量
  PI = 3.14         # 修改常量
  warning: already initialized constant PI
  warning: previous definition of PI was here
  puts PI           # 3.14
  ```

**7、** Ruby中字符串可以使用单引号也可以使用双引号，但是单引号中只会对`\'`和`\\`进行转义，双引号中除了会对各种转移字符转义外，还支持变量插值。

```ruby
puts '\\hello \'\n\' world\\'  
> \hello '\n' world\

puts "\\hello \'\n\' world\\"
> \hello '
  ' world\
    
name = 'holmofy'
puts "hello #{name}"
> hello holmofy
```

**8、** Ruby中每声明一个字符串都会创建一个新的字符串对象，而且在Ruby中要判断两个对象是否是同一个对象不能简单的用`==`，对象的`.eql?`方法和`==`都只会判断对象的内容是否相同，`.equal?`方法才能判断是否为同一个对象。

```ruby
s1 = "hello world"
s2 = "hello world"
puts s1.object_id    # 70254227595080
puts s2.object_id    # 70254227587240
puts s1 == s1        # true
puts s1.eql? s2      # true
puts s1.equal? s2    # false
```

**9、** Ruby中声明一个字符串数组，可以使用`%w`快捷方式：

```ruby
names1 = [ 'ann', 'richard', 'william', 'susan', 'pat' ]  
puts names1[0] # ann  
puts names1[3] # susan  

##使用%w快捷方式声明数组，其中w就是word的缩写
##其实等价于'ann richard william susan pat'.split(' ')
names2 = %w{  ann richard william susan pat }  
puts names2[0] # ann  
puts names2[3] # susan  
```

更多字符串快捷方式可以参考[这篇文章](https://simpleror.wordpress.com/2009/03/15/q-q-w-w-x-r-s/)

**10、** Java中不支持多行文本字符串，但是JS(反点)和Python(三个连续引号)这些脚本语言中都支持，Ruby中可以使用[Here Document](https://docs.ruby-lang.org/en/2.0.0/syntax/literals_rdoc.html#label-Here+Documents)。Ruby中追加字符串可以使用`<<`操作符。

```ruby
expected_result = <<HEREDOC
This would contain specially formatted text.
That might span many lines
HEREDOC

a = "hello"
a << "world"
puts a       # hello world
```

**11、** 对象的Mutable和Immutable一直是令人争议的话题，可变意味着不安全难以维护，不可变意味着效率低下。比如在Java、JS和Python中字符串对象都是不可变的，在JS和Python中字符串拼接通常效率极低，在Java中有StringBuilder解决这个问题，但是Ruby为了解决这些问题，在String中提供了两组方法，其中方法名的末尾加上`!`表示该方法会修改对象内容。`!`符号作为约定被用在了标准库和第三方库中。

```ruby
a = "hello"
puts a.object_id   # 70254210667380
b = a.reverse
puts a             # hello
puts b             # olleh
puts a.object_id   # 70254210667380
puts b.object_id   # 70254210630260

a = "hello"
puts a.object_id   # 70254227510440
b = a.reverse!
puts a   # olleh
puts b   # olleh
puts a.object_id   # 70254227510440
puts b.object_id   # 70254227510440
```

**12、** 方法名除了有`!`后缀表示会修改对象或入参的内容，Ruby还有`?`后缀表示这个方法返回`true`或`false`，`=`后缀表示这个方法是对象属性的setter方法，可以使用赋值运算符修改对象属性。

```ruby
puts "hello".start_with? "hell"               # true
```

**13、** Ruby内建的正则表达式对象和JS一样，使用`/.../`双斜杠创建正则字面量，也可以使用`%r{...}`或[Regexp](http://ruby-doc.org/core-2.6/Regexp.html)的构造函数。

```ruby
##和JS不同的是，//斜杠中间可以使用变量插值
place = "東京都"
m = /#{place}/.match("Go to 東京都")
    #=> #<MatchData "東京都">
puts m[0]  # 東京都
##和其他语言一样m[0]是整个正则匹配组，如果正则中有其他捕获组索引从1开始
```

**14、** 由于Ruby中每声明一个字符串都会创建一个字符串对象，并且字符串对象是可变的，所以Ruby还提供了一个与字符串类似，但不可变且全局唯一的Symbol类对象。

```ruby
puts "a".object_id     # 70107393092020
puts "a".object_id     # 70107393041480
puts :a.object_id      # 372328
puts :a.object_id      # 372328
puts (:"a").object_id  # 372328
```

Symbol包括变量名、方法名、类名，每声明一个变量，方法，类都会在符号表中记录下相应的名字，使用`Symbol.all_symbols`方法可以查看当前符号表里所有的符号。

```ruby
##不要尝试使用Symbol.all_symbols.include? :new_symbol的方式检测符号是否存在
##因为当这条语句解析时:new_symbol已经添加到符号表中了，执行的结果肯定是true
##所以这里要转成字符串
curr_symbols = Symbol.all_symbols.map {|symbol| symbol.to_s}
puts curr_symbols.include? '_any_variable_name_'  # false
_any_variable_name_ = 'this is new variable'      # 这里可以是新变量、方法、类的声明
curr_symbols = Symbol.all_symbols.map {|symbol| symbol.to_s}
puts curr_symbols.include? '_any_variable_name_'  # true
```

**15、** Ruby内建了[Array](https://github.com/ruby/ruby/blob/trunk/array.c)、[Hash](https://github.com/ruby/ruby/blob/trunk/hash.c)、[Range](https://github.com/ruby/ruby/blob/trunk/range.c)三种数据结构。可以简单地与Java中的ArrayList、HashMap、[Guava中的Range](https://github.com/google/guava/wiki/RangesExplained)等同。

```ruby
arr = [0, 1, 2]
puts arr[0]             # 访问数组下标从0开始
arr = [1, "two", 3.0]   # 可以存放不通类型的对象
puts arr[-1]            # 索引值可以为负数，最后一个元素索引为-1


grades = { "Jane Doe" => 10, "Jim Doe" => 6 }
puts grades["Jane Doe"]     # 10
##使用Symbol作为Key
options = { :font_size => 10, :font_family => "Arial" }
options[:font_size]         # 10
##使用Symbol作为Key可以简写成下面这种形式
options = { font_size: 10, font_family: "Arial" }


##(start..end)语法表示Range对象，两个点包括end，三个点不包括end
(-1..-5).to_a      #=> []
(-5..-1).to_a      #=> [-5, -4, -3, -2, -1]
('a'..'e').to_a    #=> ["a", "b", "c", "d", "e"]
('a'...'e').to_a   #=> ["a", "b", "c", "d"]
```

> [Ruby2.6开始支持Endless Range,即end值可以不指定，默认到无穷](https://medium.com/square-corner-blog/rubys-new-infinite-range-syntax-0-97777cf06270)
>
> 更多内容参考[Array](http://ruby-doc.org/core-2.6/Array.html)、[Hash](http://ruby-doc.org/core-2.6/Hash.html)、[Range](http://ruby-doc.org/core-2.6/Range.html)的官方文档

**16、** 调用一个方法就相当于发一个消息，Ruby的传参还是比较复杂的

```ruby
my_method(arg1, arg2)
my_method arg1, arg2
##但是同时调用两个方法发生歧义，Ruby会报语法错误
##比如下面的arg3不知道应该穿给method_one还是method_two
method_one arg1，method_two arg2，arg3

##可以传hash参数
def my_method(arg)
    puts arg
end
##调用时不用{}
my_method('key'=>'value')  # {"key"=>"value"}
my_method(key:'value')     # {:key=>"value"}

##可以使用*号，代表可变参数
def rest_method(arg1, *args)
    puts "first argument is #{arg1}"
    puts "other arguments is #{args}"
end
rest_method(1,2,3,4)
##first argument is 1
##other arguments is [2, 3, 4]
rest_method(1, key1:2, key2:3)  # hash参数必须放在最后
rest_method(1, key1:2, key2:3 ,4) # 否则会报语法错误

##方法支持默认参数
def default_arg_method(a, b, c = 3, d = 4)
  p [a, b, c, d]
end
default_arg_method(1, 2)        # [1,2,3,4]
default_arg_method(1, 2, 5)     # [1,2,5,4]
```

**17、** 代码块作为参数是Ruby的一大亮点，也是初学者最看不懂的(说的是自己:anguished:)。在向方法发送消息时，代码块参数始终是最后一个。代码块可以使用do … end或{ ... }

```ruby
my_method do
  # ...
end

my_method {
  # ...
}
##do...end 的优先级比 {...} 低
##通常{ ... } 用在单行

##传入的代码块也支持参数
my_method do |arg1, arg2|
  # ...
end

def my_method
  r = yield(1,2)  # 用yield关键字可以调用代码块参数
  puts r
end

my_method {|a,b| a+b}  # print 3

##yield传参也可以不带括号
def my_method
  yield self
end

place = "world"

##分号后声明代码块内的局部变量
my_method do |obj; place|
  place = "block" # 这里的局部变量不会修改外面的变量
  puts "hello #{obj} this is #{place}"
end

puts "place is: #{place}"  # 输出 place is: world
```

> 更多Ruby函数调用的内容可以参考[官方文档](https://docs.ruby-lang.org/en/2.6.0/syntax/calling_methods_rdoc.html)

**18、** Ruby的控制流语句和C系列语言不同，代码块没有花括号，看到这我觉得Python的语法还是可以的，还有Ruby给它垫底:mask:

```ruby
def condition(a)
    if a == 0
      puts "a is zero"
    elsif a == 1
      puts "a is one"
    elsif a >= 2
      puts "a is greater than two"
      puts "or a equal to two"
    else
      puts "a is some other value"
    end
end

##除了if ... elsif ... else 
##Ruby还有unless这种奇葩语法
unless true
  puts "the value is a false-value"
end
##等价于
if not true
  puts "the value is a false-value"
end
##unless也可以与else结合使用
unless true
  puts "the value is false"
else
  puts "the value is true"
end

#if 条件可以写在末尾
a = 0
a += 1 if a.zero?
p a # print 1
##unless也一样
a = 0
a += 1 unless a.zero?
p a # print 1


##循环语句
a = 0
while a < 10 do
  p a      # 0，1，2，3，4，5，6，7，8，9
  a += 1
end
p a        # 10

##
a = 0
until a > 10 do
  p a     # 0，1，2，3，4，5，6，7，8，9，10
  a += 1
end
p a       # 10


##for循环
for value in [1, 2, 3] do
  puts value
end
for value in 1..3 do
  puts value
end
for (key,value) in {k1:1, k2:2, k3:3} do
  puts "#{key}=>#{value}"
end

##Java和JS使用forEach遍历时无法打断循环
##但Ruby可以
[1, 2, 3].each do |value|
  break if value.even?
  puts value
end
##print 1

##Ruby中没有continue但有next
result = [1, 2, 3].map do |value|
  next if value.even?
  value * 2
end
p result # prints [2, nil, 6]

##next接受一个返回值
result = [1, 2, 3].map do |value|
  next value if value.even?
  value * 2
end
p result # prints [2, 2, 6]

```

**19、** 类的定义，与类的属性

```ruby
class Greeter
    def initialize(name = "World")
        @name = name
    end
    def say_hi
        puts "Hi #{@name}!"
    end
    def say_bye
        puts "Bye #{@name}, come back soon."
    end
end

greeter = Greeter.new("Pat")
greeter.say_hi   # Hi Pat!
greeter.say_bye  # Bye Pat, come back soon.

greeter.name   # 报语法错误，不能直接访问实例变量

##类允许定义多次，前面的定义会和后面的定义merge
class Greeter
    attr_accessor :name  # 指明name属性可以访问，相当于给name属性加了Getter/Setter
end
greeter.name
greeter.name = 'Andy'
greeter.say_hi   # Hi Andy

##通常我们要限制属性的写操作
class Person
    def initialize(name="holmofy",age=0)
        @name = name
        @age = age
    end
    # getter
    def age
        @age
    end
    # setter
    def age=(age)
        if age<@age
            puts "Age can only grow"
        else
            puts "#{@name} grow up to #{age} years old"
            @age=age
        end
    end
end
```

**20、** 类的继承和大多数语言一样

```ruby
class A
  Z = 1
  def z
    Z
  end
end

class B < A
end
##和大多数语言一样，子类允许调用父类的方法
p B.new.z #=> 1

class A
  def m
    1
  end
end

class B < A
  # 子类可以重写父类的方法
  def m
    2
  end
end

class B < A
  def m
    2 + super  # 如果想要调用父类的同名方法可以使用super关键字
  end
end

p B.new.m #=> 3
```



> 最后吐槽一下：排名低就是有排名低的理由，和排名前十的语言相比感觉还是差了点。