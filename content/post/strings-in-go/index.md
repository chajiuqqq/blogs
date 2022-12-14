---
title: "GoLang中的Strings, bytes, runes和characters"
date: 2022-10-07T12:16:11+08:00
draft: false
description: go blog 翻译之一
categories:
    - 技术
    - golang
    - 翻译
---

> 原文标题：Strings, bytes, runes and characters in Go
> 
> 原文地址：https://go.dev/blog/strings
> 
> 译者：z
> 

# 介绍

[前文](https://blog.golang.org/slices)介绍了slice如何工作，用一些例子展示了背后实现原理。基于那样的背景下，这篇文章将讨论Go中的strings——字符串。一开始你可能觉得对于一个blog的主题来说字符串太过于简单，但是想要很好地使用它们不仅需要理解它们的工作原理，也要明白byte、character、rune间的区别，unicode和UTF-的区别，字符串和字符串字面值的区别，可能还有其他更细微之处的区别。

开始这个主题的一个方法是思考这么一个经常被提及的问题：“当我在Go中获取字符串在n位置的东西时，为什么我没有获取到第n个字符？”，正如你所见，这个问题可以让我们思考关于字符串如何工作的更多细节问题。

一个很棒的，不局限于Go语言的回答是Joel Spolsky的著名blog： [The Absolute Minimum Every Software Developer Absolutely, Positively Must Know About Unicode and Character Sets (No Excuses!)](http://www.joelonsoftware.com/articles/Unicode.html)。这篇文章会重述他的一些观点。

# 什么是字符串

让我们从基础开始吧。

Go中，一个字符串实际上只是只读的byte slice而已。如果你完全不理解什么是byte的slice或者其工作原理，请读之前这篇[blog](https://blog.golang.org/slices)。这里我们默认你已经读过了。

首先要声明的很重要的一点是，字符串可以包括任意个数的字节。不一定要包含unicode格式文本、UTF-8格式文本、或其他预定义格式的文本。如果只考虑字符串的内容，它其实就是等同于byte slice。

这是一个字符串字面值（后面会有更多），它使用`\xNN`符号定义一个字符串常量，它会包含各种奇怪的byte（当然，byte范围是从16进制的00到FF，不包括FF）

    const sample = "\xbd\xb2\x3d\xbc\x20\xe2\x8c\x98"

# 打印字符串

由于举例的字符串中的一些字节是非ASCII字符，甚至不是UTF-8，直接打印字符串会产生不好看的输出，打印语句如下：
    
    fmt.Println(sample)

会产生这样一堆东西（环境不同显示的东西也不同）
    
    ��=� ⌘

为了找出字符串到底包含什么，我们需要把它分开，并单独检验每一块。有一些方法可以做到，最明显的就是遍历他的内容，并把每个字节单独输出，看这个for循环：
   
    for i := 0; i < len(sample); i++ {
        fmt.Printf("%x ", sample[i])
    }

如上面展示的，通过索引可以获取到字符串的每个字节，而不是字符。下面我们再细致的看看循环输出的字节：

    bd b2 3d bc 20 e2 8c 98

注意一下每个字节是如何和与字符串中定义的16进制值对应的。

一个更短的产生可观输出的方法是使用fmt的`%X`（16进制）格式。它会以16进制输出字符串的字节序列，每个字节是两位：
    
    fmt.Printf("%x\n", sample)

它的输出是：
    
    bdb23dbc20e28c98

一个小技巧是在%和x中间添加空格，比较一下区别：
    
    fmt.Printf("% x\n", sample)

注意下字节间如何打印空格的：

    bd b2 3d bc 20 e2 8c 98

使用`%q`可以转义输出无法显示的字节，这样输出就会很清晰：
    
    fmt.Printf("%q\n", sample)

当大多数字符串都很正常而有一些包含了特殊符号需要剔除时，这个方法很方便，它会产生：
    
    "\xbd\xb2=\xbc ⌘"

如果我们仔细看它，会发现在乱码中有一个ASCII的等号，还有一个正常的空格，最后有个很著名的“Place of Interest” 符号。这个符号的unicode编码是U+2318，在空格（16进制值是20）后面用UTF-8编码：e2 8c 98.

如果我们不熟悉或被字符串中的奇怪东西绕晕了，我们可以使用+号结合`%q`。这个组合会输出不可打印的字符的转义格式，以及非ASCII字符的转义格式，这些格式都是UTF-8的. 非ASCII数据会以合适的UTF-8编码打印出来：
    
    fmt.Printf("%+q\n", sample)

这种格式下，unicode格式的“Place of Interest” 符号会被展示为带`\u`转义序列：
    
    "\xbd\xb2=\xbc \u2318"

这种打印的方法非常适合字符串内容的debug，下面讨论的时候也会很方便。值得一提的是，所有这些方法应用在字节slice或字符串中时表现一致。

这是我们提及的所有打印方法，你可以在浏览器中试试运行：
    
    package main

    import "fmt"

    func main() {
        const sample = "\xbd\xb2\x3d\xbc\x20\xe2\x8c\x98"

        fmt.Println("Println:")
        fmt.Println(sample)

        fmt.Println("Byte loop:")
        for i := 0; i < len(sample); i++ {
            fmt.Printf("%x ", sample[i])
        }
        fmt.Printf("\n")

        fmt.Println("Printf with %x:")
        fmt.Printf("%x\n", sample)

        fmt.Println("Printf with % x:")
        fmt.Printf("% x\n", sample)

        fmt.Println("Printf with %q:")
        fmt.Printf("%q\n", sample)

        fmt.Println("Printf with %+q:")
        fmt.Printf("%+q\n", sample)
    }

# UTF-8和字符串字面值

如我们所见，索引一个字符串会获得它的字节，而不是字符，一个字符串只是一串的字节而已。这意味着当我们在字符串中存储一个字符时，我们同时存了它的字节表示。让我们看一个更细致的例子：

这是一个小程序，它按照三种不同的方式打印了只有一个字符的字符串，一种按照字符串打印，一种只打印ASCII字符，一种按照16进制打印每一个字节。为了避免混淆，我们创建了一个原始字符串，由反引号围起来，这样它就只能包含字面值（如果是普通的字符串，由于被双引号包围，可能会包含上面提到的转义序列）

    func main() {
        const placeOfInterest = `⌘`

        fmt.Printf("plain string: ")
        fmt.Printf("%s", placeOfInterest)
        fmt.Printf("\n")

        fmt.Printf("quoted string: ")
        fmt.Printf("%+q", placeOfInterest)
        fmt.Printf("\n")

        fmt.Printf("hex bytes: ")
        for i := 0; i < len(placeOfInterest); i++ {
            fmt.Printf("%x ", placeOfInterest[i])
        }
        fmt.Printf("\n")
    }

输出是：
    
    plain string: ⌘
    quoted string: "\u2318"
    hex bytes: e2 8c 98

这提醒了我们U+2318是“Place of Interest” 符号 ⌘的unicode编号，并且由字节表示为e2 8c 98，这些字节是16进制数2318按照UTF-8编码后的字节表示。

依赖于你对UTF-8的熟悉程度，你可能很清楚这些或者有点模糊，但是花点时间解释一下UTF-8是如何表示我们创建的字符串是很有必要的。最简单的事实是：当源码写入时，它就创建出来了。

源码在Go中被定义为UTF-8文本，并且不允许其他格式的表示。这意味着在源码中，当我们输入：
    `⌘`
时，文本编辑器会把`⌘`符号的UTF-8编码存入源码中，当我们输出16进制字节时，我们只是复制输出了文件中编辑器放入的数据。

更精简点来说，Go语言源码是UTF-8，所以字符串字面量的源码就是UTF-8文本。如果字符串字面值不包含转义序列（反引号构建的原始字符串不能包含转义字符），构造出来的字符串引号中间的文本就是源码文本。因此原始字符串总是会包含合法的UTF-8文本。类似的，普通字符串经常由合法的UTF-8文本组成，除非它包含上面提到的不在UTF-8编码中的转义序列。

> 译注：
> 
> 反引号 `` 组成的字符串不会进行转义处理，所以字符串内容=存入的源码
> 
> 普通字符串 "" 会进行转义处理，比如`\x2b`会识别成一个16进制数据存入，而不是存入\、x、2、b。如果这个16进制数不在UTF-8编码中，那就是非法的UTF-8编码，无法被打印出来。

一些人认为Go字符串总是UTF-8编码，其实不是：只有字符串字面值是UTF-8编码.正如我们上面提到的，字符串值可以包含任意的字节；字符串字面值只要没有字节等级的转义就总是UTF-8文本。

总结来说，字符串可以包含任意字节，但是从字面值构建时，这些字节通常都是UTF-8编码的。

# Code points、字符、runes

目前为止我们都很小心地使用单词“字节”和“字符”。一部分原因是因为字符串包含字节，另一部分是因为字符这个概念有点难以定义。Unicode标准使用“Code point”定义单一值表示的东西，比如Code points U+2318，16进制值2318，代表了⌘符号（更多Code point参考[Unicode page](http://unicode.org/cldr/utility/character.jsp?a=2318)）

一个更普通地例子是Unicode Code point U+0061是小写字母：a

但是小写重音字母à如何表示？这是一个字符，也是Code point U+00E0，但是它有其他表示方法。比如我们可以结合重音Code point，U+0300，附加到小写字母a，U+0061上，构建出同样的字符à。通常来说一个字符可以被多种不同的Code point组合表示，因此有不同的UTF-8字节序列。

因此计算机的字符概念是模糊的，或者容易混淆，所以我们要小心使用。为了让他更可靠，有一些标准化技术可以保证一个字符只有一种Code point的表示方式，但是这个主题不在我们讨论的范围内，之后会有一个新的blog介绍Go库是如何处理标准化的。

“Code point”有点绕口，为此在Go中有个更短的术语：rune。这个术语会在库和源码中出现，和Code point意思相同，但有一个新的特点。

Go语言中rune是类型int32的别名，所以程序可以理解一个int类型的数值表示为Code point的状况。除此以外，一个字符常量在Go中其实被称为一个rune常量。
    
    '⌘'

这个符号的类型是rune，值是整型的0x2318.

总的来说，这是几个重点：
- Go源码总是UTF-8的
- 一个字符串可以包含任意字节
- 一个字符串字面值如果没有字节层面的转义，那就是合法的UTF-8序列。
- 这些序列代表了unicode的Code point，也就是rune。
- 无法保证Go中字符串的字符是标准化过的。

# `for range`循环

除了不言自明的一个细节——go源码是UTF-8编码的，只有在一种情况下，go会特殊处理UTF-8编码，那就是当你对字符串使用`for range`循环的时候。

我们知道普通的`for`循环会发生什么，对于`for range`循环，每次遍历时都会解码一个UTF-8的rune。每次循环时，下标是当前rune的开始位置，测量单位是字节，code point就是他的值。这个例子使用了另一个打印格式`%#U`，可以打印unicode编码和代表的字符：
    
    const nihongo = "日本語"
    for index, runeValue := range nihongo {
        fmt.Printf("%#U starts at byte position %d\n", runeValue, index)
    }

输出会显示每个code point占据了多个字节：

    U+65E5 '日' starts at byte position 0
    U+672C '本' starts at byte position 3
    U+8A9E '語' starts at byte position 6

# 标准库

Go的标准库对解释UTF-8文本提供了强大支持。如果`for range`循环对你来说不够用，那么可以试试库里的包。

最重要的包是`unicode/utf8`，提供了UTF-8字符串的验证、拆分、组装方法。这个程序等同于上面的`for range`循环，但是使用了`DecodeRuneInString`方法实现。返回值是rune和他的UTF-8编码字节个数。
    
    const nihongo = "日本語"
    for i, w := 0, 0; i < len(nihongo); i += w {
        runeValue, width := utf8.DecodeRuneInString(nihongo[i:])
        fmt.Printf("%#U starts at byte position %d\n", runeValue, i)
        w = width
    }

运行可以看到结果相同。`for range`循环和`DecodeRuneInString `方法会产生相同的迭代结果。

这个包的其他方法请查看[unicode/utf8文档](https://go.dev/pkg/unicode/utf8/)

# 结论

回答一下文章开始提出的问题：字符串由字节组成，因此索引字符串会获得字节，而不是字符。一个字符串甚至可能不包含字符。实际上“字符”的定义就是模糊的，如果你把字符串定义为由字符组成，你可能会遇到一些模糊性问题。

还有很多关于unicode，UTF-8和多语言文本处理的东西可以讲，但是请等下一个blog吧，现在我希望你更理解Go的字符串如何呈现的了，即使它们可能包含任意的字节，它的核心设计的一部分仍旧是UTF-8编码.