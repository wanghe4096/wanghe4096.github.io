---
title: 手动编写json解析器(译)
---

原文地址： http://notes.eatonphil.com/writing-a-simple-json-parser.html


编写一个json解析器是一种最容易熟悉json解析原理方式。 它的格式非常简单。 因为递归式定义，所以与解析BrainFuck相比， 稍微有些挑战。 你可能已经在使用json了。 除了最后一点之外， 解析Scheme的S表达式也许是更简单的任务。 

如果你只是想看看这个库的代码，  请在github点击查看。
<!-- more -->

# 关于解析的定义

解析一般分为两个阶段，即： 词法分析和语法分析。 词法分析将源输入分解为一门语言最为简单的元素， 被称之为"tokens"。 语法分析（通常也称作语法解析)接收这些“token”列表，然后试图从中找出和被解析语言相匹配的模式。 

解析不能确定输入源的语义可行性。 一个输入源的语议可行性可能包括一个变量在定义前是否被定议。 一个被函数被调用时参数是否正确，或者一个变量某作用域下被二次声明。 

当然， 人们选择解析和应用语议规则的方式总是有所不同， 但我假设用一种”传统“的方法来解释核概念。 

# JSON库的接口
最终，应该有一个from_string 方法来接收json编码的字符串，并返回等价于python 字典。 

例如： 

assert_equal(from_string('{"foo": 1}'),
             {"foo": 1})

# 词法分析
词法分析将输入源分解为多个token.  注释和空格通常在词法分析的过程会被丢弃， 因此您只需要一个简单的输入即可在语法分析过程搜索匹配语法。 

假设有一个简单的词法分析器， 你也许可以在一个输入字符串(或流) 中迭代所有字符， 并将它们拆分为最基本的组成部分， 非递归的语言构造像整型、字符串或布尔类类型，特别像字符串必须是词法分析的一部分， 由于你在不知道空格是否是字符串的一部分的情况，不能丢弃空格。 


"在一个优秀的词分析器中， 持续追踪被忽略的空格、注释、当前行号和文件， 以便你能在源码分析过程所有产生的错误中引用这些信息。 javascript v8 引擎最近能够复现一个函数的确切源码。 这至少需要引入词法分析器的帮助才能成为可能。”

# 实现一个json词法分析器
json词法分析器要点是迭代输入源，并尝试查找字符串， 数字，布尔值， 空值或JSON语法中的模式，最终将这些元素作为列表返回。 

以下是词法分析器应该为示例输入所返回的内容： 
```python
assert_equal(lex('{"foo": [1, 2, {"bar": 2}]}'),
             ['{', 'foo', ':', '[', 1, ',', 2, '{', 'bar', ':', 2, '}', ']', '}'])
```

词法分析的逻辑可能开始看起来类似于如下代码逻辑：

```python
def lex(string):
    tokens = []

    while len(string):
        json_string, string = lex_string(string)
        if json_string is not None:
            tokens.append(json_string)
            continue

        # TODO: lex booleans, nulls, numbers

        if string[0] in JSON_WHITESPACE:
            string = string[1:]
        elif string[0] in JSON_SYNTAX:
            tokens.append(string[0])
            string = string[1:]
        else:
            raise Exception('Unexpected character: {}'.format(string[0]))

    return tokens
```

这段代码目的是匹配字符串，数字， 布尔值和空值并将它们添加到tokens 列表中。 如果这些都不匹配，检查字符串是否为空格，如果是， 则将其丢弃。 否则将其作为一个token 存储. 如果是json语法的一部分，也将其做为token存储。  如果字符/字符串没有对应的模式，那么最终抛出一个异常。 

让我们在此扩展这个核心的逻辑，让其支持所有的类型并添加函数存根。 

```python
def lex_string(string):
    return None, string

def lex_number(string):
    return None, string

def lex_bool(string):
    return None, string

def lex_null(string):
    return None, string

def lex(string):
    tokens = []

    while len(string):
        json_string, string = lex_string(string)
        if json_string is not None:
            tokens.append(json_string)
            continue

        json_number, string = lex_number(string)
        if json_number is not None:
            tokens.append(json_number)
            continue

        json_bool, string = lex_bool(string)
        if json_bool is not None:
            tokens.append(json_bool)
            continue

        json_null, string = lex_null(string)
        if json_null is not None:
            tokens.append(None)
            continue

        if string[0] in JSON_WHITESPACE:
            string = string[1:]
        elif string[0] in JSON_SYNTAX:
            tokens.append(string[0])
            string = string[1:]
        else:
            raise Exception('Unexpected character: {}'.format(string[0]))

    return tokens
```

## string 词法
对于lex_string函数， 要点是检查第一个字符是否引号。如果是，则迭代输入字符串，直到找到结束引号。 如果找不到初始引用， 则返回None和原始列表。 如果找到的初始引号和结束引号， 则返回引号内的字符串和未选中的输入字符串的其余部分。 

```python
def lex_string(string):
    json_string = ''

    if string[0] == JSON_QUOTE:
        string = string[1:]
    else:
        return None, string

    for c in string:
        if c == JSON_QUOTE:
            return json_string, string[len(json_string)+1:]
        else:
            json_string += c

    raise Exception('Expected end-of-string quote')
```

## Number词法
对于lex_number 函数， 要点是一直迭代输入， 直到找到不能成为数字一部分的字符。 （当然， 这是一个非常简单的定义，更精确的实现留给读者作为练习) 找到一个不能为数字一部分的字符后， 如果你累计的数字大于0，那返回一个浮点数或整数。 

```python
def lex_number(string):
    json_number = ''

    number_characters = [str(d) for d in range(0, 10)] + ['-', 'e', '.']

    for c in string:
        if c in number_characters:
            json_number += c
        else:
            break

    rest = string[len(json_number):]

    if not len(json_number):
        return None, string

    if '.' in json_number:
        return float(json_number), rest

    return int(json_number), rest
```

## 布尔和null词法
查到布尔和null值是一个非常简单的字符串匹配。 

```python
def lex_bool(string):
    string_len = len(string)

    if string_len >= TRUE_LEN and \
       string[:TRUE_LEN] == 'true':
        return True, string[TRUE_LEN:]
    elif string_len >= FALSE_LEN and \
         string[:FALSE_LEN] == 'false':
        return False, string[FALSE_LEN:]

    return None, string


def lex_null(string):
    string_len = len(string)

    if string_len >= NULL_LEN and \
       string[:NULL_LEN] == 'null':
        return True, string[NULL_LEN]

    return None, string
```
现在，词法解析器就完成了! 完整代码可以点击这里[pj/lexer.py](https://github.com/eatonphil/pj/blob/master/pj/lexer.py)


# 语法分析
语法分析器的（基本)工作是迭代一维分词列表。 并根据语言的定义将分词组匹配的语言的各个部分。 如果在语法分析的过程中， 解析器无法将当前的分词集合与语言的有效的语法匹配， 那么解析器的解析将会失败， 并且可能会为您提供有用的信息， 千知您所提供的内容， 位置以及预期内容。  

## 实现一个JSON 解析器

JSON解析器的要点是迭代调用lex后收到的token，并尝试将token与对象，列表或普通值进行匹配。


以下是解析器应返回的示例输入：
```python
tokens = lex('{"foo": [1, 2, {"bar": 2}]}')
assert_equal(tokens,
             ['{', 'foo', ':', '[', 1, ',', 2, '{', 'bar', ':', 2, '}', ']', '}'])
assert_equal(parse(tokens),
             {'foo': [1, 2, {'bar': 2}]})
```
最初的逻辑可能是下面的样子： 
```py
def parse_array(tokens):
    return [], tokens

def parse_object(tokens):
    return {}, tokens

def parse(tokens):
    t = tokens[0]

    if t == JSON_LEFTBRACKET:
        return parse_array(tokens[1:])
    elif t == JSON_LEFTBRACE:
        return parse_object(tokens[1:])
    else:
        return t, tokens[1:]
```

这个词法分析器和解析器之间的关键结构差异是词法分析器返回一维token数组。解析器通常以递归方式定义，并返回递归的类似于树的对象。由于JSON是一种数据序列化格式而不是语言，因此解析器应该在Python中生成对象而不是语法树，您可以在其上执行更多分析（或者在编译器的的代码生成过程中生成对象）

而且， 使词法分析独立于解析器的优点是两段代码都更简单，只需要关注特定元素。 

## 解析数组
解析数组是一个解析数组内成员并期望它们之间的逗号标记或指示数组末尾的右括号的问题。 

```py
def parse_array(tokens):
    json_array = []

    t = tokens[0]
    if t == JSON_RIGHTBRACKET:
        return json_array, tokens[1:]

    while True:
        json, tokens = parse(tokens)
        json_array.append(json)

        t = tokens[0]
        if t == JSON_RIGHTBRACKET:
            return json_array, tokens[1:]
        elif t != JSON_COMMA:
            raise Exception('Expected comma after object in array')
        else:
            tokens = tokens[1:]

    raise Exception('Expected end-of-array bracket')
```

## 解析对象
  解析对象是解析内部由冒号分隔的键值对，并用逗逗分隔，直到到达对象的末尾。 

  ```python
  def parse_object(tokens):
    json_object = {}

    t = tokens[0]
    if t == JSON_RIGHTBRACE:
        return json_object, tokens[1:]

    while True:
        json_key = tokens[0]
        if type(json_key) is str:
            tokens = tokens[1:]
        else:
            raise Exception('Expected string key, got: {}'.format(json_key))

        if tokens[0] != JSON_COLON:
            raise Exception('Expected colon after key in object, got: {}'.format(t))

        json_value, tokens = parse(tokens[1:])

        json_object[json_key] = json_value

        t = tokens[0]
        if t == JSON_RIGHTBRACE:
            return json_object, tokens[1:]
        elif t != JSON_COMMA:
            raise Exception('Expected comma after pair in object, got: {}'.format(t))

        tokens = tokens[1:]

    raise Exception('Expected end-of-object brace')
  ```
  现在，解析器的代码已经写完了！ 点击这里查看完整的代码. [pj/parser.py](https://github.com/eatonphil/pj/blob/master/pj/parser.py)

# 统一的库
提供一个理想的接口， 创建一个from_string函数来封装lex 和 parse 函数。

```py
def from_string(string):
    tokens = lex(string)
    return parse(tokens)[0]
```

这个库也完成了。 github地址在[project on Github](https://github.com/eatonphil/pj) 

# 附录 A： 单步解析
在某些解析器中， 选择在一个阶段中实现词法和句法分析。 对于某些语言， 这可以完全简化解析步骤。 或者， 在更强大的语言(如Common Lisp) 中， 它允许您使用读取器宏来动态库展词法分析器和语法解析器。


我用python 编写了这个库， 使更多的人可以访问它。 但是， 所使和的许多技术更适合具有ssaj式切尔西和支持单值运算法的语言。 像ML . 如果你对标准ML感兴趣，请查看[Ponyo中的JSON代码](https://github.com/eatonphil/ponyo/blob/master/src/Encoding/Json.sml). 
