# 参考
https://docs.oracle.com/javase/7/docs/api/java/util/regex/Pattern.html
https://en.wikipedia.org/wiki/Regular_expression

# what 正则表达式能做什么？


## case1 验证
public class Demo {

    private static final String EMAIL = "^[a-zA-Z0-9_-]+@[a-zA-Z0-9_-]+(\\.[a-zA-Z0-9_-]+)+$";
    
    public static boolean isLegalEmail(String email) {
        Pattern pattern = Pattern.compile(EMAIL);
        return pattern.matcher(email).matches();
    }
}

## case2 


# how 正则表达式怎么实现的？复杂度分析，背后的理论是什么？


## 基本要素 & 如何使用？
Pattern + Matcher + 

## 数据结构和算法


# when 什么时候适合用正则，什么时候不适合（有什么坑）？

