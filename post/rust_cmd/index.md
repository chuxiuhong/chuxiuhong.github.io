


背景：最近希望利用Tshark来做一些数据包过滤分析的功能，主体用Rust写，调用部分懒得找dll的资料，想直接用CMD命令来执行后截取标准输出，然后做parser。

在CMD中使用的代码大概如此：

```
tshark.exe -r xxx.pcap -T fields -e dns.a -e dns.qry.name -V 'udp.srcport==53'
```

在Rust里本想是件不难的事，可是一写起来发现有点蹊跷：

```rust
fn main(){
    use std::process::Command;
    let output = Command::new("cmd").arg("/c").arg(r"tshark.exe").arg(r"-r").arg( r"xxx.pcap")
        .arg(r"-T").arg(r"fields")
        .arg(r"-e").arg(r"dns.a")
        .arg(r"-e").arg(r"dns.qry.name")
        .arg(r"-V").arg("'udp.srcport==53'").output().unwrap();
    println!("{}",String::from_utf8_lossy(&output.stdout));
}

```

不panic也不输出。把.arg都合成一句也不行。然后测试echo 123，而这是可以获取输出的。

那么问题来了，到底区别在哪？查了查资料之后发现，是Windows CMD的引号捕获问题，在命令里原本的引号，既不要用`'`也不要用`\"`来表示，直接去掉原来的引号，才等价于原本的命令。

```rust
    let output = Command::new("cmd").arg("/c").arg(r"tshark.exe").arg(r"-r").arg( r"xxx.pcap")
        .arg(r"-T").arg(r"fields")
        .arg(r"-e").arg(r"dns.a")
        .arg(r"-e").arg(r"dns.qry.name")
        .arg(r"-V").arg("udp.srcport==53").output().unwrap();
```

