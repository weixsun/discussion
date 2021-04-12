通常我们递增的id作为请求资源的标识，但如果站点直接使用递增的id字段作为业务属性使用，那么对站点会造成如下影响（不限于如下列出的）：

- 对于用户资源，站点很容易被窥测出总注册用户量、时间段内注册用户量等。`A站`和`B站`的个人主页地址采用的就是递增的id，e.g.: 第一位用户：`~/1`；第二位用户：`~/2`，以此类推。

- 对于视频资源，很容通过爬虫得到站点所有视频资源。`B站`已经从原来的`"av" + 递增id`改为[`"BV" + base58(av号)`](https://www.bilibili.com/read/cv5247479)，但是本质只是对数据库表av号进行了[base58](https://www.zhihu.com/question/381784377/answer/1099438784)操作，在视频页，通过在浏览器F12 Consoles中输入aid即可获得BV号所对应的av号，并且视频页的HTML.meta标签中依然存在av号地址。

当某些资源对外提供时，我们希望能够有使用一种编码算法能够将具有递增性的标识码进行编码为无递增性的标识。

## Hashids

[Hashids](https://hashids.org)是一款非常小巧跨语言的开源库，可以将`数字`或者`16进制字符串`转为短且唯一不连续的字符串，`Hashids`是双向编码（支持`encode`和`decode`），比如，它可以将`347`之类的数字转换为`yr8`之类的字符串，也可以将`yr8`之类的字符串重新解码为`347`之类的数字。

`youtobe`的视频地址链接中`?v=`参数后使用的就是Hashids。`腾讯视频`和`爱奇艺`使用的应该是类似这种的Hash算法。

> 与youtobe不同，TikTok网页版的短视频编码是基于发号器的唯一id。

## Java版实现的Hashids

Hashids有着不同语言的实现，可根据不同语言去官方参考相应的工具库使用，我们挑选Java版本的[`org.hashids >> hashids >> 1.0.3`](https://github.com/jiecao-fm/hashids-java)实现库进行说明。

Hashids java版本的原理实现可参照源代码和借鉴[这篇文章](https://blog.csdn.net/x_iya/article/details/107065379)。

**`pom.xml`依赖**

```xml
<dependency>
	<groupId>org.hashids</groupId>
	<artifactId>hashids</artifactId>
	<version>1.0.3</version>
</dependency>
```

## 测试用例

Hashids可以自定义salt，但必须保证编码和解码过程中使用同一套salt，务必妥善保管salt。测试用例中使用的salt全部为`this is my salt`。

需要注意的点：

- 在生产环境中设置的salt要保持足够的复杂度，并且要妥善保存好salt，泄漏或者丢失salt要做的弥补工作会非常麻烦。

- Hashids可以自定义编码后结果的最小长度，测试用例中统一使用`11`位最小编码结果长度。11位最小编码长度，意味着不同的输入得到的结果长度是可能超过11位的。

- `org.hashids >> hashids >> 1.0.3` 在编码和解码过程中严格区别大小写。

### 编码`long类型`的可变参数为字符串

```java
@Test
void test_hashids_encode_method() {
final String SALT = "this is my salt";
final int MIN_HASH_LENGTH = 11;

Hashids hashids = new Hashids(SALT, MIN_HASH_LENGTH);
String encryptString = hashids.encode(347L);

System.out.println(encryptString); // Y5bAyr8dLO4
}
```

`encode(long...numbers)`中的参数是`long类型`的可变参数，可以将多个`long类型`的参数编码到一个字符串中，将可变参数编码为一个字符串的场景可自行思考灵活设计。

### 解码字符串为Long类型数组

```java
@Test
void test_hashids_decode_method() {
	final String SALT = "this is my salt";
	final int MIN_HASH_LENGTH = 11;

	Hashids hashids = new Hashids(SALT, MIN_HASH_LENGTH);
	long[] decrypedNumbers = hashids.decode("Y5bAyr8dLO4");

	Arrays.stream(decrypedNumbers).forEach(item -> System.out.println(item)); // 347

}
```

如果编码和解码过程中使用的salt不一致，则`long[]`为空数组。

## 编码16进制字符串为Hashids字符串

Hashids支持对16进制字符串进行编码，所以如果使用`mongodb`数据库时，那么就可以将系统自动生成的`_id`字符串使用Hashids进行编码。

```java
/**
 * {@link <a href="https://stackoverflow.com/a/27137224">node.js - get hash from strings, like hashids - Stack Overflow</a>}
 * <p>
 * {@link <a href="https://hashids.org/java/">hashids</a>}
 */
@Test
void test_hashids_encodeHex_method() {
	final String SALT = "this is my salt";
	final int MIN_HASH_LENGTH = 11;

	Hashids hashids = new Hashids(SALT, MIN_HASH_LENGTH);
	String encryptString = hashids.encodeHex(cn.hutool.core.util.HexUtil.encodeHexStr("this is a string"));
	System.out.println(encryptString); // 1prnZLrKPlS5EEe61reMCNzkJXP
}
```

### 解码Hashids字符串为16进制字符串

与编码16进制字符串相应的，可以根据字符串解码得到16进制字符串。

```java
/**
 * {@link <a href="https://stackoverflow.com/a/27137224">node.js - get hash from strings, like hashids - Stack Overflow</a>}
 * <p>
 * {@link <a href="https://hashids.org/java/">hashids</a>}
 */
@Test
void test_hashids_decodeHex_method() {
	final String SALT = "this is my salt";
	final int MIN_HASH_LENGTH = 11;

	Hashids hashids = new Hashids(SALT, MIN_HASH_LENGTH);
	String decrypedNumbers = hashids.decodeHex("1prnZLrKPlS5EEe61reMCNzkJXP");

	System.out.println(cn.hutool.core.util.HexUtil.decodeHexStr(decrypedNumbers)); // this is a string
}
```

如果编码和解码过程中使用的salt不一致，则`long[]`为空数组。

### 自定义字母表映射集

默认映射字母表为：`abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ1234567890`，可以根据其构造函数自定义映射字母表，测试用例中我们使用和`youtobe`相似的映射字母表。

> 自定义的字母表中的字符至少应含有16个字符。

```java
@Test
void test_hashids_costom_alphabet_by_Constructor() {
	final String SALT = "this is my salt";
	final int MIN_HASH_LENGTH = 11;
	final String ALPHABET = "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ1234567890_-";


	Hashids hashids = new Hashids(SALT, MIN_HASH_LENGTH, ALPHABET);
	String encryptString = hashids.encode(347L);

	System.out.println(encryptString); // kqBg-Kpg7_J

}
```

---

文章同步发布在各大主流知识共享平台，所以设`github`为统一的反馈区

疑问、讨论、问题反馈：https://github.com/weixsun/discussion

---

关注微信公众号 **obmq** 及时了解最新动态

![](https://gitee.com/weixsun/imgs/raw/master/obmq-mp-weixin.png)

