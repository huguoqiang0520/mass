欢迎来到[「我是真的狗杂谈世界」](https://github.com/huguoqiang0520/mass/blob/main/README.md)，关注不迷路

# 背景

最近做的项目多次遇到了分享邀请的需求点，即需要在接受邀请时能识别到邀请者的信息，又需要考虑信息敏感性，没找到成熟的三方实现，于是自己思考实现了两套。

# 思路方案

不能直接将邀请信息用于传递，需要对信息（一般是字符串，不是字符串也可以转换为字符串）进行加密处理，或者说编码处理。但同时需要满足一下要求：

## 要求

- 可逆：加密（编码）后的密文应当能通过解密（解码）获得原文，否则就无法获得邀请者信息了；
- 长度：加密（编码）后的密文应该尽可能与原文长度相当，可以略多（如果能更少也好，不过那需要涉及压缩了，不是今天的重点）；
- 内容：加密（编码）后的密文应当是可预知的字符集合，如果可设置更好；
- 算法可公开（意味着存在额外依赖）：不单单依靠一个固定算法，即便算法公开仍旧能保证安全性，否则算法一旦被破解也就没什么了；
- 复杂度适当：太复杂的一般对计算要求很高，对开发（本人）成本也高，能满足目前的项目需求即可（我绝对不会说我懒）；

## 常见算法

常见的一些加密解密、编码解码算法：

- 单向：md系列、sha1；
- 对称：des、aes；
- 非对称：rsa、dsa；
- 简单编码：base64；

# 移位法

## 本质

- 维护全量字符可能的顺序，对给定字符串的每个字符按照其位置进行移位转换，得到结果；
- 位置到字符移位偏移量通过一套外部输入的规则来指定；
- 上述两步相当于将字符枚举、顺序、偏移量规则三套作为算法变量交由用户控制，在不能完全知道三个信息的情况下，即便知道算法，也能保障密文安全性；

## 防篡改

- 虽然用移位法可以保障用户无法通过密文还原原文，但用户可以用大量无序密文来攻击，造成大量垃圾数据或暴力猜测到一些信息，因此还需要支持能校验密文是否合法的功能，也就是防篡改。
- 通用的思路是额外加一点校验数据，由原文构造而成加入到密文中，解密（解码）时校验一遍作为验证即可，比如网络IP和TCP层的累加和就是这种思路。

## 实现

```php
    public function encrypt(string $value): string
    {
        if (0 == $this->sortCount) {
            throw new EncrypterException('未设置字符排布阵列');
        }
        if (0 == $this->shiftingCount) {
            throw new EncrypterException('未设置编码密钥');
        }
        $encrypted = '';
        for ($i = 0; $i < strlen($value); $i++) {
            if (!key_exists($value[$i], $this->sortList)) {
                throw new EncrypterException('发现不期待的字符');
            }
            $index = $this->sortList[$value[$i]];
            $index += $this->shiftingList[$i % $this->shiftingCount];
            $index %= $this->sortCount;
            $encrypted .= $this->sort[$index];
        }

        // 添加校验位
        if ($this->checkSum) {
            $sum = 0;
            for ($i = 0; $i < strlen($value); $i++) {
                $sum += ord($value[$i]);
            }
            $sum %= $this->sortCount;
            $start = $this->sort[$sum];
            $end = $this->sort[$this->sortCount - $sum - 1];
            $encrypted = $start . $encrypted . $end;
        }

        return $encrypted;
    }
```

```php
    public function decrypt(string $value): string
    {
        if (0 == $this->sortCount) {
            throw new EncrypterException('未设置字符排布阵列');
        }
        if (0 == $this->shiftingCount) {
            throw new EncrypterException('未设置编码密钥');
        }

        // 拿出校验位
        if ($this->checkSum) {
            if (strlen($value) < 2) {
                throw new EncrypterException('解密校验失败');
            }
            $start = $value[0];
            $end = $value[strlen($value) - 1];
            $value = substr($value, 1, -1);
        }

        $decrypted = '';
        for ($i = 0; $i < strlen($value); $i++) {
            if (!key_exists($value[$i], $this->sortList)) {
                throw new EncrypterException('发现不期待的字符');
            }
            $index = $this->sortList[$value[$i]];
            $index -= $this->shiftingList[$i % $this->shiftingCount];
            while ($index < 0) {
                $index += $this->sortCount;
            }
            $decrypted .= $this->sort[$index];
        }

        // 校验校验位
        if ($this->checkSum) {
            $sum = 0;
            for ($i = 0; $i < strlen($decrypted); $i++) {
                $sum += ord($decrypted[$i]);
            }
            $sum %= $this->sortCount;
            if ($start != $this->sort[$sum] || $end != $this->sort[$this->sortCount - $sum - 1]) {
                throw new EncrypterException('解密校验失败');
            }
        }

        return $decrypted;
    }
```

## 特点分析

- 需要知道原文全部字符构成可能；
- 原文和密文的字符构成是同一个集合；
- 密文与原文长度相等，即使加入校验位也仅多2位；
- 适用于前端、用户可见可感知的传递场景；

# 异或法

## 本质

- 利用a^b^b=a，也就是两次异或复位的特性（字符可转换成一个数值也就是一个多位的二进制，单位异或的特性在多位场景下同样成立）
- 同样与原文、密文每个字符进行异或操作的字符应该与其位置规则有关，同移位法相关规则

## 可读性

由于异或后的值可能超过输入原值，字符转换时可能转换为非常用字符影响阅读，因此可以选择嵌套一层base64加解密方便阅读。

## 实现

```php
    public function encrypt(string $value): string
    {
        $value = $this->handle($value);
        if ($this->base64) {
            $value = base64_encode($value);
        }
        return $value;
    }
```

```php
    public function decrypt(string $value): string
    {
        if ($this->base64) {
            $value = base64_decode($value);
        }
        return $this->handle($value);
    }
```

```php
    protected function handle(string $string): string
    {
        $result = '';
        for ($i = 0; $i < strlen($string); $i++) {
            $t = $this->secret[$i % $this->secretCount];
            $result .= chr(ord($string[$i]) ^ ord($t));
        }
        return $result;
    }
```

## 分析

- 即便不知道原文字符全部可能也可以使用这套算法；
- 但原文和密文的字符构成不是同一个集合；
- base64后密文一般较原文长，具体见base64编码算法规则；
- 适用于接口间、服务间数据传输场景；

[1]: https://github.com/huguoqiang0520/mass/blob/main/README.md