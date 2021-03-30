
regex这个Crate是Rust中常用的正则表达式处理库。除了常见的字符串以外，它还提供了直接对字节进行处理的接口，这大大有助于进行某些底层数据的匹配和抽取，不过有几处需要注意的地方。

## 引入路径

字节匹配的接口全部放在了`regex::bytes`这个路径下，引入的时候应该类似下面这样：

```rust
extern crate regex;
use regex::bytes::Regex;
```

而不是直接从regex引入。

## 不符合Unicode码点

虽然提供了对字节进行匹配的接口，但是默认情况下还是遵守了Unicode码点范围，超出的话就不会匹配。

需要你在表达式外层增加一个标记，例如

```rust
    let r = Regex::new(r"(?-u:\x01\x02)");
```

使用`?-u:()`来包裹要匹配的内容

## 默认不换行匹配

匹配数据时通常和匹配字符串不同，很可能根本不考虑换行符，但是regex匹配时仍然默认不进行换行匹配，也就是说当你数据中恰好有和换行符相同的字节时，就会少匹配到应有的数据。

这里需要你用到`RegexBuilder`来自定义`Regex`。代码类似下面所示：

```rust
    let mut rb = RegexBuilder::new(s.as_str());
    rb.dot_matches_new_line(true);
    let r = rb.build().unwrap();
```



## 最后总结

把上面几点综合起来，可以封装一个创建面向字节流的正则对象生成器。

```rust
fn compile(reg:&str) -> Regex{
    let s = "(?-u:".to_string()+reg+")";
    let mut rb = RegexBuilder::new(s.as_str());
    rb.dot_matches_new_line(true);
    rb.build().unwrap()
}
```

这里直接`unwrap()`是比较合理的，因为正则表达式通常是预先写好的，应该是经过大量测试而不是临时生成，如果连正则表达式都写错了，程序就基本没有跑下去的必要。