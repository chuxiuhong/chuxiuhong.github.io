
## 实现目的

最近一直在用Rust做一些底层库，性能用起来也很顺手，但是碍于Rust目前极差的GUI生态，还是决定将Rust导出为DLL然后用C#来调用，用C#做个GUI的壳给用户使用，所有的内容还是用Rust实现。值得一看的是关于中文字符串参数的处理

## Rust部分

定义个函数，负责读入一个字符串然后写到文件里。

```rust
//lib.rs
use std::os::raw::c_char;
use std::ffi::CStr;
use std::fs::File;
use std::io::Write;
#[no_mangle]
pub fn get_str(s:*const c_char){
    let mut f = File::create("test.file").unwrap();
    let s = unsafe{CStr::from_ptr(s)}.to_bytes().to_vec();
    let s = String::from_utf8(s).unwrap();
    f.write(s.as_bytes());
}
```

这里no_mangle的意思是在编译期间保留这个函数名，让编译器不要改掉，这样C#才能顺利找到入口。后面的Cstr::from_ptr是从参数里读出字符串，读的类型按照的是C标准，详情请看Rust的FFI文档。这里转换了一下utf-8，如果单纯是为了写在文件里，倒是不必，直接写字节即可。我是为了其他功能必须读到中文。

```rust
//Cargo.toml
[package]
name = "hello"
version = "0.1.0"
authors = ["chuxiuhong <chuxiuhong@chuxiuhong.com>"]
edition = "2018"

[dependencies]

[lib]
name = "dexterlib"
crate-type=["cdylib"]
```

在Cargo.toml里定义出我们要编译的类型是C标准的动态链接库，这里注意你要把需要导出的函数像前文一样处理然后放到lib.rs里。

## C#部分

```c#
using System.Collections.Generic;
using System.Linq;
using System.Runtime.InteropServices;
using System.Text;
using System.Threading.Tasks;

namespace testRustDll
{
    class Program
    {
        [DllImport("dexterlib.dll")]
        static extern void get_str(byte[] s);
        static void Main(string[] args)
        {
            var s = "艾欧尼亚，昂扬不灭";
            var encoder = new UTF8Encoding();
            var s_utf_8 = encoder.GetBytes(s);
            get_str(s_utf_8);
        }
    }
}
```

这种C#调用DLL的代码到处都有介绍，我主要介绍的是解决中文乱码问题。C#的Winform从外界读回来的字符串貌似是UTF-16编码（对C#不是很精通），如果按照正常的思路来调用，那么定义应该是下面这种

```c#
        [DllImport("dexterlib.dll")]
        static extern void get_str(string s);
```

但是这种定义传到Rust里在转换UTF-8编码时十有八九会出问题，所以要单独再先做一个UTF-8的字节转换，将原来的字符串转换为一个UTF-8的字节数组传进去。

