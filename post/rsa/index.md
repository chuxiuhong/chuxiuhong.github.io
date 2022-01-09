
RSA是由MIT的三位数学家R.L.Rivest，A.Shamir和L.Adleman[Rivest等1978, 1979]提出的一种用数论构造双钥的方法，被称为MIT体制，后来被广泛称之为RSA体制。其既可以作为加密，又可以用于数字签字。RSA算法的安全性基于数论中大整数分解的困难性。

**算法描述**

1.独立的选取两个大素数p和q
2.计算$n = p * q$，其欧拉函数值为$\phi(n) = (p-1) * (q-1)$
3.随机选一个整数$e$，$1\leq e\leq \phi(n) , gcd(\phi(n),e) = 1$ #gcd为最大公约数
4.在模$\phi(n)$下，计算e的逆元$d = e^{-1} mod \phi(n)$ #意思是$(e * d) mod \phi(n) =1$
5.以n,e为公钥，密钥为d

**加密**
将明文分组，每组的数要求小于n（字符串就想办法映射到整数）
计算$c = m^e mod n$，其中m为明文，c为可以传输的密文

**解密**
计算$m = c^d mod n$

这个过程中只有算法描述中的第三步可能不是直接想到求解方法，对应这个问题，可以用扩展的欧几里得算法来求逆元。

以下算法内容来源自华中科技大学的PPT，在此注明。

问题：求A关于模N的逆元B
迭代计算
 N ＝ A × a0 ＋ r0
 A ＝ r0 × a1 ＋ r1
 r0＝ r1 × a2 ＋ r2
 r1＝ r2 × a3 ＋ r3
……
rn-2＝ rn-1 × an  ＋ rn
rn-1＝ rn-2× an+1＋ 0 
对上面的商数逆向排列(不含余数为0的商)
![image1](/static/oujilide1.jpg)
其中$b_{-1} = 1$，$b_0 = a_n$，$b_i = a_{n-i} * b_{i-1} + b_{i-2}$
如果商的个数为偶数，则$b_n$就是所求的B
如果商的个数为奇数，则$N-b_n$就是所求的B

例1：求61关于模105的逆元
    		105＝61×1＋44
    		61 ＝44×1＋17
    		44 ＝17×2＋10
    		17 ＝10×1＋7
    		10 ＝7 ×1＋3
    		7  ＝3 ×2＋1
    		3  ＝3 ×1＋0

![image2](/static/oujilide2.jpg)
由于商的个数为偶数（不包括余数为0的商），所以31就是61关于105的逆元

例二：求31关于模105的逆元
    		105＝31×3＋12
    		31 ＝12×2＋7
    		12 ＝7 ×1＋5
    		 7 ＝5 ×1＋2
    		 5 ＝2 ×2＋1
  		    2 ＝1 ×2＋0
![image3](/static/oujilide3.jpg)
商的个数是奇数，所以105-44 = 61为31关于模105的逆元

代码实现如下：
```python
# coding=utf8
class RSA:
    def encrypt(self, string, n, e):
        '''
        use RSA algorithm to encrypt string
        :param string: the String need to be encrypt
        :param n: p * q
        :param e: encrypt code number
        :return: encrypt number
        这里是将字符先转换为ASCII值再加密
        '''
        s = []
        for i in string:
             s.append(str(ord(i)))
        for i in range(len(s)):
            s[i] = int(s[i]) ** e % n
        return s

    def decrypt(self, p, q, e, encoding):
        '''
        :param p: prime number p
        :param q: prime number q
        :param e: encrypt code number
        :param encoding: the string that need to be decrypted
        :return: the string that decrypt the encoding number
        这里相应的多了一步将ASCII转为字符后拼接的过程
        '''
        f_n = (p - 1) * (q - 1)
        n = p * q
        d = RSA.ext_euclid(e, f_n)
        s = []
        for i in range(len(encoding)):
            s.append(chr(encoding[i] ** d % n))
        return ''.join(s)

    @staticmethod
    def ext_euclid(e, f_n):
        '''
        :param e: encrypt code number
        :param f_n: p * q
        :return: the number that multiply e mod n equals 1
        '''
        nc = f_n
        if e > f_n or type(e) != int or type(f_n) != int:
            return -1
        quotient = []  #商的列表
        remainder = -1 #余数
        while remainder != 0:
            quotient.append(f_n / e)
            remainder = f_n - (f_n / e) * e
            f_n = e
            e = remainder
        quotient = quotient[:-1][::-1] #对应上面写的将商逆序写出来
        d_list = [1, quotient[0]] 
        for i in range(1, len(quotient)):
            d_list.append(d_list[-1] * quotient[i] + d_list[-2])
        return d_list[-1] if len(quotient) % 2 == 0 else nc - d_list[-1] #如果商的个数是偶数，直接返回bn，否则返回N - bn


if __name__ == '__main__':
    r = RSA()
    p = int(raw_input("p = "))
    q = int(raw_input("q = "))
    e = int(raw_input("e = "))
    string = raw_input("String: ")
    en = r.encrypt(string, p * q, e)
    print "encrypted code: ", ' '.join(map(str, en))
    print "decrypted code: ", r.decrypt(p, q, e, en)
```

运行如图：
![image4](/static/RSA1.png)
这里可以用小素数的原因是在代码中将明文简单的按字符分组了，但这样会收到频率分析的攻击，在此仅是实验用。