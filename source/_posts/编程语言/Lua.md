---
title: Lua
date: {{ date }}
categories:
- 编程语言
- Lua
---

# Lua

## Lua 程序设计

Lua 是由巴西里约热内卢天主教大学（Pontifical Catholic University of Rio de Janeiro）里的一个研究小组于1993年开发的一种轻量、小巧的脚本语言，用标准 C 语言编写，其设计目的是为了嵌入应用程序中，从而为应用程序提供灵活的扩展和定制功能。

官网：http://www.lua.org/

Redis 在 2.6 版本中推出了脚本功能，允许开发者将 Lua 语言编写的脚本传到 Redis 中执行。使用 Lua 脚本的优点有如下几点:

- 减少网络开销：本来需要多次请求的操作，可以一次请求完成，从而节约网络开销；

- 原子操作：Redis 会将整个脚本作为一个整体执行，中间不会执行其它命令；

- 复用：客户端发送的脚本会存储在 Redis 中，从而实现脚本的复用。

## Lua 基本语法

### 注释

```lua
-- 两个减号是行注释

--[[

 这是块注释

 --]]
```

### 变量

#### 数字类型

Lua的数字只有double型，64bits

你可以以如下的方式表示数字

 ```lua
num = 1024

num = 3.0

num = 3.1416

num = 314.16e-2

num = 0.31416E1

num = 0xff

num = 0x56
 ```

#### 字符串

可以用单引号，也可以用双引号

也可以使用转义字符‘\n’ （换行）， ‘\r’ （回车）， ‘\t’ （横向制表）， ‘\v’ （纵向制表）， ‘\\’ （反斜杠）， ‘\”‘ （双引号）， 以及 ‘\” （单引号)等等

下面的四种方式定义了完全相同的字符串（其中的两个中括号可以用于定义有换行的字符串）

```
a = 'alo\n123"'

a = "alo\n123\""

a = '\97lo\10\04923"'

a = [[alo

123"]]
```

#### 空值

C语言中的NULL在Lua中是nil，比如你访问一个没有声明过的变量，就是nil

#### 布尔类型

只有nil和false是 false

数字0，‘’空字符串（’\0’）都是true

#### 作用域

lua中的变量如果没有特殊说明，全是全局变量，那怕是语句块或是函数里。

变量前加local关键字的是局部变量。

### 控制语句

#### while循环

```lua
while 0 <= 10 do
    print(i)
i = i +1
end
```

#### if-else

```lua
local function main()
local age = 140
local sex = 'Male'
  if age == 40 and sex =="Male" then
    print(" 男人四十一枝花 ")
  elseif age > 60 and sex ~="Female" then
    print("old man without country!")
  elseif age < 20 then
    io.write("too young, too naive!\n")
  else
  print("Your age is "..age)
  end
end

-- 调用
main()
```

#### for循环

```lua
sum = 0
for i = 100, 1, -2 do
    sum = sum + i
end
print(sum)
```

#### 函数

普通函数

```lua
function myPower(x,y)
  return y+x
end
power2 = myPower(2,3)
print(power2)
```

匿名函数

```lua
function newCounter()
   local i = 0
   return function()     -- anonymous function
        i = i + 1
        return i
    end
end

c1 = newCounter()

print(c1())  --> 1
print(c1())  --> 2
print(c1())
```

#### 返回值

多返回值

```lua
name, age, bGay = "yiming", 37, false, "yimingl@hotmail.com"
print(name,age,bGay)
```

函数返回值

 ```lua
function isMyGirl(name)
  return name == 'xiao6' , name
end

local bol,name = isMyGirl('xiao6')

print(name,bol)
 ```

### 集合

#### Table

key，value的键值对 类似 map

```lua
lucy = {name='xiao6',age=18,height=165.5}
xiao6.age=35

print(xiao6.name,xiao6.age,xiao6.height)
print(xiao6)
```

#### 数组

```lua
arr = {"string", 100, "xiao6",function() print("memeda") return 1 end}

print(arr[4]())
```

#### 数组遍历

```lua
for k, v in pairs(arr) do
   print(k, v)
end
```

#### 面向对象

```lua
person = {name='xiao6',age = 18}
  function  person.eat(food)
    print(person.name .." eating "..food)
  end
person.eat("xxoo")
```

## Lua 整合 redis

### 在redis中执行简单脚本

登录到客户端后执行

#### Hello world

```properties
#命令    脚本        	参数个数
eval	"return 1+1"    0
```

#### 参数

```lua
EVAL "local msg='hello world' return msg..KEYS[1]" 1 AAA BBB
```

表是基于1的，也就是说索引以数值1开始。所以在表中的第一个元素就是mytable[1]，第二个就是mytable[2]等等。 表中不能有nil值。如果一个操作表中有[1, nil, 3, 4]，那么结果将会是[1]——表将会在第一个nil截断。

###  独立脚本

#### 获取key的value

```lua
local key=KEYS[1]

local list=redis.call("get",key);  

return list;
```

#### 读取redis集合中的数据

```lua
local key=KEYS[1]

local list=redis.call("lrange",key,0,-1);

return list;
```

#### 统计点击次数

```lua
local msg='count:'
local count = redis.call("get","count")
if not count then
        redis.call("set","count",1)
end

redis.call("incr","count")

return msg..count+1
```

#### 执行lua脚本

##### 本地执行

```
redis-cli --eval test.lua aaa,bbb
```

##### 远程执行

```lua
redis-cli -h 192.168.2.161 -a密码 --eval /usr/local/luascript/test.lua name age , xiao6
```

### Lua 与 Redis 交互

#### Lua 脚本获取 EVAL & EVALSHA 命令的参数

通过 Lua 脚本的全局变量 KEYS 和 ARGV，能够访问 EVAL 和 EVALSHA 命令的 key [key ...] 参数和 arg [arg ...] 参数。

作为 Lua Table，能够将 KEYS 和 ARGV 作为一维数组使用，其下标从 1 开始。

#### Lua 脚本内部执行 Redis 命令

Lua 脚本内部允许通过内置函数执行 Redis 命令：

redis.call()

redis.pcall()

两者非常相似，区别在于：

若 Redis 命令执行错误，redis.call() 将错误抛出（即 EVAL & EVALSHA 执行出错）；

redis.pcall() 将错误内容返回。

local msg='count:'  local count = redis.call("get","count")  if not count then          redis.call("set","count",1)  end  redis.call("incr","count")  return msg..count+1

### redis WATCH/MULTI/EXEC 与Lua

redis 原生支持 监听、事务、批处理，那么还需要lua吗？

- 两者不存在竞争关系，而是增强关系，lua可以完成redis自身没有的功能

- 在lua中可以使用上一步的结果，也就是可以开发**后面操作依赖前面操作的执行结果的应用**，MULT中的命令都是独立操作

- redis可以编写模块增强功能，但是c语言写模块，太难了，lua简单的多
- 计算向数据移动
- 原子操作

lua脚本尽量短小并且尽量保证同一事物写在一段脚本内，因为redis是单线程的，过长的执行会造成阻塞，影响服务器性能。

### Redis Lua 脚本管理

1.script load  此命令用于将Lua脚本加载到Redis内存中  

2.script exists  scripts exists sha1 [sha1 …]  此命令用于判断sha1是否已经加载到Redis内存中  

3.script flush  此命令用于清除Redis内存已经加载的所有Lua脚本,在执行script flush后,sha1不复存在  

4.script kill  此命令用于杀掉正在执行的Lua脚本

### 死锁

下面代码会进入死循环，导致redis无法接受其他命令。

```lua
eval "while true do end" 0 
```

```lua
127.0.0.1:6379> keys *
(error) BUSY Redis is busy running a script. You can only call SCRIPT KILL or SHUTDOWN NOSAVE.
```

但是可以接受 SCRIPT KILL or SHUTDOWN NOSAVE. 两个命令

SHUTDOWN NOSAVE 不会进行持久化的操作

SCRIPT KILL 可以杀死正在执行的进程

### 生产环境下部署

#### 加载到redis

```lua
redis-cli script load "$(cat test.lua)"
```

得到sha1值

执行

```lua
redis-cli evalsha "7a2054836e94e19da22c13f160bd987fbc9ef146" 0
```
