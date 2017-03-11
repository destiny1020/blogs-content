---
title: '[PHP]利用openssl_random_pseudo_bytes和base64_encode函数来生成随机字符串'
date: 2015-04-09 23:22:00
categories: ['工具, 库与框架', LAMP]
tags: [Ubuntu, LAMP, PHP, 字符串]
---

[openssl_random_pseudo_bytes](http://php.net/manual/en/function.openssl-random-pseudo-bytes.php)函数本身是用来生成指定个数的随机字节，因此在使用它来生成随机字符串时，还需要配合使用函数[base64_encode](http://php.net/manual/en/function.base64-encode.php)。如下所示：

<!-- More -->

```php
public static function getRandomString($length = 42)
    {
        /*
         * Use OpenSSL (if available)
         */
        if (function_exists('openssl_random_pseudo_bytes')) {
            $bytes = openssl_random_pseudo_bytes($length * 2);

            if ($bytes === false)
                throw new RuntimeException('Unable to generate a random string');

            return substr(str_replace(['/', '+', '='], '', base64_encode($bytes)), 0, $length);
        }

        $pool = '0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ';

        return substr(str_shuffle(str_repeat($pool, 5)), 0, $length);
    }
```

在调用base64_encode函数之后，还对结果进行了一次替换操作，目的是要去除随机生成的字符串中不需要的字符。

当然，在使用openssl_random_pseudo_bytes函数之前，最好使用function_exists来确保该函数在运行时是可用的。如果不可用，则使用Plan B：

```php
substr(str_shuffle(str_repeat($pool, 5)), 0, $length);
```

这个函数的通用性很强，可以根据业务的需要进行适当修改然后当作静态方法进行调用。
