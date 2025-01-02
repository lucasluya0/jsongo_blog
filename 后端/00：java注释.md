---
date: 2024-03-15
tags: 
summary: 详细介绍了Java中三种注释类型(单行注释、多行注释和Javadoc注释)的使用方法和最佳实践，包含了Javadoc注释中常用的HTML标签和文档标签的详细说明与示例代码。
coverImage: https://mmbiz.qpic.cn/sz_mmbiz_png/aAia1M9Vb9bicypPFaFMqhicicUAtWvRw62nKcgRjgQGicibK3Cq7cnTsqNdFvWDFm0YrdAjbS4O0Gibr0ILPFtzf6ssQ/640?wx_fmt=png&amp;from=appmsg
author: 柳树下的程序员
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
![title.png](https://mmbiz.qpic.cn/sz_mmbiz_png/aAia1M9Vb9bicypPFaFMqhicicUAtWvRw62nKcgRjgQGicibK3Cq7cnTsqNdFvWDFm0YrdAjbS4O0Gibr0ILPFtzf6ssQ/640?wx_fmt=png&amp;from=appmsg)
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




企业开发中，针对类、接口、方法编写需要增加必要的注释，

- 当代码出现 bug 时，同事可以通过注释迅速定位问题的所在，节省了大量时间，提高了工作效率。
- 代码经过一段时间后，开发者自己可能也会忘记当初的实现细节。通过详细的注释，可以快速理解自己曾经的逻辑，减少重新熟悉代码的时间。
- 提升代码可读性：注释可以解释复杂的逻辑和算法，使得代码更易于理解，尤其对于新加入的团队成员来说尤为重要。



**1. 单行注释不要写在代码的尾部**
```java
// 打印结果
System.out.println(result);
```

**2. 在写Map 结构注释时：可以这样进行**
```java
//key->用户对象的id，value->用户对象拥有的金额
Map<String, BigDecimal> testMap=new HashMap<>();
```

**3. 在写if...else...注释时，建议写上代码块注释**

```java

if (number > 0) {

    //region 该数字是正数

    return "该数字是正数。";

    //endregion

} else if (number < 0) {

    //该数字是负数。

    return "该数字是负数。";

} else {

    //该数字是零。

    return "该数字是零。";

}

```

> **代码示例:**

**entity**

```java

@Data

public class LoginVo {
    /**

     * 用户名

     * @see User#getUsername()

     */

    private String username;
    
    /**

     * 密码

     * @see User#getPassword()

     */

    private String password;

}



```

**Controller**

```java

/**

 * 处理用户登录请求。

 *

 * <p>示例：

 * <pre>{@code

 * POST请求：/user/login

 * 请求体：

 * {

 *   "username": "john_doe",

 *   "password": "password123"

 * }

 * }

 * </pre>

 *

 * @param loginVo 包含用户登录名和密码的 {@link LoginVo} 对象。

 * @return 如果用户认证成功返回true，否则返回false。

 */

@PostMapping("/login")

public boolean login(@RequestBody LoginVo loginVo) {

    return userService.authenticate(loginVo);

}

```

**Service**

```java

/**

 * 根据提供的登录信息进行用户认证。

 * @param loginVo 包含用户凭据的 {@link LoginVo} 对象。

 * @return 如果认证成功返回true，否则返回false。

 *

 */

boolean authenticate(LoginVo loginVo);

```



**ServiceImpl**

```java

/**

 * <pre>

 * 根据提供的登录信息进行用户认证。

 *     第一步：

 *     第二步：

 * </pre>

 * @param loginVo 包含用户凭据的 {@link LoginVo} 对象。

 * @return 如果用户凭据正确，则返回true，否则返回false。

 */

@Override

public boolean authenticate(LoginVo loginVo) {

    return false;

}



/**

 * 根据提供的登录信息进行用户认证。

 *     <li>第一步：</li>

 *     <li>第二步：</li>

 * @param loginVo 包含用户凭据的 {@link LoginVo} 对象。

 * @return 如果用户凭据正确，则返回true，否则返回false。

 */

@Override

public boolean authenticate(LoginVo loginVo) {

    // 根据用户名获取用户对象

    User user = getUserByUsername(loginVo.getUsername());

    if (user == null) {

        //说明用户名不存在，认证失败

        return false;

    }

    // 比较用户对象的密码与输入的密码是否相同，进行认证

    return user.getPassword().equals(loginVo.getPassword());

}



/**

 * 根据用户名获取用户对象。

 *

 * @param username 要查找的用户的用户名

 * @return 返回对应用户名的用户对象 {@link User} ，如果找不到则返回null。

 */

private User getUserByUsername(String username) {

    //模拟一个存在的用户

    return new User();

}

```







