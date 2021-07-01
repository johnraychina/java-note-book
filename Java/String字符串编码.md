

缘起：https://coolshell.cn/articles/11629.html

```Java
        String str = "test";
        byte[] bytes = str.getBytes(StandardCharsets.UTF_8); //编码为 byte[]
        String strB = new String(bytes);                     //编码为 byte[]
        str.toCharArray();                                   //解码为 char[]
```

看String源码，编码和解码都需要分配数组，并重新做copy。

java 内部用UTF-16编码，或者压缩的LATIN1编码

## 编码：将 byte[] 编码为指定字符集的 byte[]
```Java
byte[] bytes = str.getBytes(StandardCharsets.UTF_8);
```
String.getBytes(charset)
StringCoding.encode 这里面逻辑很多，todo
 


1. encodeUTF8  关键部分
byte c = val[sp];
c < 0 占2个新byte，c>0占1个新byte

2. 
## 解码：将byte[] 解码为 char[]

java.lang.String#toCharArray
java.lang.StringLatin1#toChars
java.lang.StringLatin1#inflate 



1. 如果原来是Latin1编码，解码只需要&ff将高位补1.
java.lang.StringLatin1#toChars

2. 否则走utf-16编码，2个byte合成1个char
java.lang.StringUTF16#toChars
