---
date: 2024-03-15
tags: 旅行,日本,京都
summary: 记录了在京都的樱花季旅行见闻
coverImage: images/title.png
author: xu
---

推荐:编写 JavaDoc 注释时，可以使用多种 HTML 标签来增强注释的格式和可读性。以下是一些常用的 HTML 标签及其用途：
- `<p>`：段落标签，用于创建段落。
- `<ol>` 和 `<li>`：有序列表和列表项标签，用于创建编号列表。
- `<i>` 和 `<em>`：斜体标签，用于使文本倾斜。
- `<a>`：链接标签，用于创建超链接。
- `<pre>`：预格式化文本标签，用于包含预格式化的文本或代码块。
- `<code>`：代码块标签，用于包含代码块。

`Java` 提供了三种类型的注释：
- 单行注释
- 多行注释
- Javadoc 注释

**单行注释**

单行注释使用 // 开头，通常用于注释一行代码或说明某个代码块的功能。

~~~java
public class DocOne {
    public static void main(String[] args) {
        //输出到控制台
        System.out.println("Hello World!");
        int number=10;
        //检查一个数字是正数、负数还是零。
        if (number > 0) {
            //正数
            System.out.println("该数字是正数。");
        } else if (number < 0) {
            //负数
            System.out.println("该数字是负数。");
        } else {
            //零
            System.out.println("该数字是零。");
        }
    }
}
~~~
**多行注释**

多行注释使用 /* ... */ 包围，可以跨越多行。多行注释通常用于详细解释复杂的代码逻辑或禁用一段代码。

~~~java
public class DocTwo {
    public static void main(String[] args) {
        /*
         * 这里我们调用了 System.out.println 方法
         * 它用于输出文本到控制台
         */
        System.out.println("Hello, World!");
    }
}

~~~
***Javadoc 注释***

Javadoc 注释用于生成文档，是一种特殊的多行注释，使用 /** ... */ 包围，通常放在类、方法或字段的前面。它们可以包含 HTML 标签和 Javadoc 标签。
~~~java

/**
 * 该类演示了 Java 中 if..else 语句的使用。
 * @author xiaoliu
 * @version 1.0
 */
public class DocThree {

    /**
     * 检查一个数字是正数、负数还是零。
     * <p>
     * 该方法接受一个整数参数，并返回一个字符串，指示该整数是正数、负数还是零。
     * </p>
     *
     * @param number 要检查的数字
     * @return 表示该数字是正数、负数还是零的字符串
     * @throws IllegalArgumentException 如果输入的数字超出了允许的范围
     * @see java.lang.Integer
     */
    public static String checkNumber(int number) {
        if (number > 0) {
            return "该数字是正数。";
        } else if (number < 0) {
            return "该数字是负数。";
        } else {
            return "该数字是零。";
        }
    }

    /**
     * 主方法，用于运行 if..else 示例。
     * <p>
     * 该方法是程序的入口，演示了如何使用 {@link #checkNumber(int)} 方法。
     * </p>
     *
     * @param args 命令行参数
     * @see #checkNumber(int)
     */
    public static void main(String[] args) {
        int number = 5; // 示例数字
        String result = checkNumber(number); // 调用方法检查数字
        System.out.println(result); // 打印结果
    }
}

~~~
一些常用的 Javadoc 标签包括：
- `@param`：描述方法参数
- `@return`：描述方法返回值
- `@throws` 或 `@exception`：描述方法可能抛出的异常
- `@see`：参考其他相关类或方法
- `@since`：标明自哪个版本开始有这个元素
- `@deprecated`：标明该元素已过时
- `@version`：标明版本号
- `@author`：标明作者
- `@link`：创建到其他类或方法的链接






