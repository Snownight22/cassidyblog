---
layout: post
title: Nginx源码分析-数据结构之字符串string 
date:   2022-07-07 15:31:00 +0800
categories: Nginx源码分析
tags: C nginx
topping: false
---

Nginx 中封装了自定义的字符串类型和一系列字符串操作函数，定义在 ngx_string.c/h 中。  

### 自定义字符串类型

ngx_str_t 是 nginx 的自定义字符串类型，包含一个字符串长度和不以 '\0' 结尾的字符串。  

```
typedef struct {
    size_t      len;    //字符串长度
    u_char     *data;    //字符串数据
} ngx_str_t;
```

自定义字符串的初始化和赋值是以宏的形式定义的，**需要注意的是**，两个宏中的参数 str 需要传一个常量字符串：  

```
#define ngx_string(str)     { sizeof(str) - 1, (u_char *) str }    //此宏与下面 ngx_str_set 宏中str 需要传一个常量字符串
#define ngx_null_string     { 0, NULL }
#define ngx_str_set(str, text)                                               \
    (str)->len = sizeof(text) - 1; (str)->data = (u_char *) text
#define ngx_str_null(str)   (str)->len = 0; (str)->data = NULL
```

### 字符串操作函数

nginx 中封装了很多字符串操作函数，下面按其用途分类介绍并详细分析其中的一些重点函数：  

#### 字符串大小写转换

```
#define ngx_tolower(c)      (u_char) ((c >= 'A' && c <= 'Z') ? (c | 0x20) : c)    //转为小写
#define ngx_toupper(c)      (u_char) ((c >= 'a' && c <= 'z') ? (c & ~0x20) : c)    //转为大写

//将字符串转为小写
void ngx_strlow(u_char *dst, u_char *src, size_t n);
```

#### 计算字符串长度

```
#define ngx_strlen(s)       strlen((const char *) s)

//在n范围内计算字符串长度
size_t ngx_strnlen(u_char *p, size_t n);
```

在 n 的范围内计算字符串长度其实就是遍历字符串，直到出现 '\0' 或到 n 的位置：  

```
//在n范围内计算字符串长度
size_t
ngx_strnlen(u_char *p, size_t n)
{
    size_t  i;

    for (i = 0; i < n; i++) {

        if (p[i] == '\0') {
            return i;
        }
    }

    return n;
}
```

#### 字符串包含

```
#define ngx_strstr(s1, s2)  strstr((const char *) s1, (const char *) s2)    //字符串包含字符串

#define ngx_strchr(s1, c)   strchr((const char *) s1, (int) c)    //字符串包含字符

//字符串某个范围内第一次出现字符c 的位置，没有返回NULL
static ngx_inline u_char * ngx_strlchr(u_char *p, u_char *last, u_char c)

u_char *ngx_strnstr(u_char *s1, char *s2, size_t n);

u_char *ngx_strstrn(u_char *s1, char *s2, size_t n);
//不区分大小写
u_char *ngx_strcasestrn(u_char *s1, char *s2, size_t n);
u_char *ngx_strlcasestrn(u_char *s1, u_char *last, u_char *s2, size_t n);
```

我们来看一下 ngx_strnstr 的实现，ngx_strstrn, ngx_strcasestrn, ngx_strlcasestrn 都与之类似。这个函数会先比较与 s2 字符串的第一个字符相等的位置，然后调用 strncmp 比较字符串是否相同。  

```
u_char *
ngx_strnstr(u_char *s1, char *s2, size_t len)
{
    u_char  c1, c2;
    size_t  n;

    c2 = *(u_char *) s2++;

    n = ngx_strlen(s2);

    do {
        //先查找到第一个字符相等地位置，然后通过strncmp比较函数来判断字符串是否相同
        do {
            if (len-- == 0) {
                return NULL;
            }

            c1 = *s1++;

            if (c1 == 0) {
                return NULL;
            }

        } while (c1 != c2);

        if (n > len) {
            return NULL;
        }

    } while (ngx_strncmp(s1, (u_char *) s2, n) != 0);

    return --s1;
}
```

#### 字符串拷贝与初始化

```
#define ngx_memzero(buf, n)       (void) memset(buf, 0, n)    //内存置0
#define ngx_memset(buf, c, n)     (void) memset(buf, c, n)    //内存置值c

//为防止编译器死区消除优化策略(dead store elimination optimization)，不要优化掉memset
void ngx_explicit_memzero(void *buf, size_t n);    

//这两个区别在于返回值，ngx_memcpy 返回拷贝完的首地址，ngx_cpymem 返回拷贝完的结尾位置
#define ngx_memcpy(dst, src, n)   (void) memcpy(dst, src, n)
#define ngx_cpymem(dst, src, n)   (((u_char *) memcpy(dst, src, n)) + (n))

static ngx_inline u_char * ngx_copy(u_char *dst, u_char *src, size_t len)

//这两个区别与 ngx_memcpy 和 ngx_cpymem 相同，ngx_memmove 返回 dst 首地址，ngx_movemem 返回 dst 尾地址
#define ngx_memmove(dst, src, n)   (void) memmove(dst, src, n)
#define ngx_movemem(dst, src, n)   (((u_char *) memmove(dst, src, n)) + (n))

u_char *ngx_cpystrn(u_char *dst, u_char *src, size_t n);
//从内存池中申请内存并拷贝字符串，不包含结尾'\0' 字符
u_char *ngx_pstrdup(ngx_pool_t *pool, ngx_str_t *src);
```

### 字符串比较

```
#define ngx_strncmp(s1, s2, n)  strncmp((const char *) s1, (const char *) s2, n)    //字符串比较
#define ngx_strcmp(s1, s2)  strcmp((const char *) s1, (const char *) s2)    //字符串比较

ngx_int_t ngx_strcasecmp(u_char *s1, u_char *s2);
ngx_int_t ngx_strncasecmp(u_char *s1, u_char *s2, size_t n);

//字符串比较，从右侧字符开始
//这个函数与下一个函数ngx_rstrncasecmp都是从右侧开始比较，
//相等返回0，不相等返回从右侧开始第一个不相等字符的差值，
//ngx_rstrncasecmp是不区分大小写的比较
ngx_int_t ngx_rstrncmp(u_char *s1, u_char *s2, size_t n);
ngx_int_t ngx_rstrncasecmp(u_char *s1, u_char *s2, size_t n);

//给定两块内存及长度比较大小
ngx_int_t ngx_memn2cmp(u_char *s1, u_char *s2, size_t n1, size_t n2);

//给不同应用定制的字符串比较函数
ngx_int_t ngx_dns_strcmp(u_char *s1, u_char *s2);
ngx_int_t ngx_filename_cmp(u_char *s1, u_char *s2, size_t n);

```

### 格式化输出

```

u_char * ngx_cdecl ngx_sprintf(u_char *buf, const char *fmt, ...);
u_char * ngx_cdecl ngx_snprintf(u_char *buf, size_t max, const char *fmt, ...);
u_char * ngx_cdecl ngx_slprintf(u_char *buf, u_char *last, const char *fmt,
    ...);
u_char *ngx_vslprintf(u_char *buf, u_char *last, const char *fmt, va_list args);
#define ngx_vsnprintf(buf, max, fmt, args)                                   \
    ngx_vslprintf(buf, buf + (max), fmt, args)
```

格式化输出支持的输出格式在注释中有说明：  

```
/*
 * supported formats:
 *    %[0][width][x][X]O        off_t
 *    %[0][width]T              time_t
 *    %[0][width][u][x|X]z      ssize_t/size_t
 *    %[0][width][u][x|X]d      int/u_int
 *    %[0][width][u][x|X]l      long
 *    %[0][width|m][u][x|X]i    ngx_int_t/ngx_uint_t
 *    %[0][width][u][x|X]D      int32_t/uint32_t
 *    %[0][width][u][x|X]L      int64_t/uint64_t
 *    %[0][width|m][u][x|X]A    ngx_atomic_int_t/ngx_atomic_uint_t
 *    %[0][width][.width]f      double, max valid number fits to %18.15f
 *    %P                        ngx_pid_t
 *    %M                        ngx_msec_t
 *    %r                        rlim_t
 *    %p                        void *
 *    %[x|X]V                   ngx_str_t *
 *    %[x|X]v                   ngx_variable_value_t *
 *    %[x|X]s                   null-terminated string
 *    %*[x|X]s                  length and string
 *    %Z                        '\0'
 *    %N                        '\n'
 *    %c                        char
 *    %%                        %
 *
 *  reserved:
 *    %t                        ptrdiff_t
 *    %S                        null-terminated wchar string
 *    %C                        wchar
 */
```

ngx_sprintf, ngx_snprintf, ngx_slprintf 通过调用 ngx_vslprintf 来实现字符串格式化，下面我的来看一下 ngx_vslprintf 的实现：  

```
//格式化字符串输出，返回写入后的位置
u_char *
ngx_vslprintf(u_char *buf, u_char *last, const char *fmt, va_list args)
{
    u_char                *p, zero;
    int                    d;
    double                 f;
    size_t                 slen;
    int64_t                i64;
    uint64_t               ui64, frac;
    ngx_msec_t             ms;
    ngx_uint_t             width, sign, hex, max_width, frac_width, scale, n;
    ngx_str_t             *v;
    ngx_variable_value_t  *vv;

    while (*fmt && buf < last) {

        /*
         * "buf < last" means that we could copy at least one character:
         * the plain character, "%%", "%c", and minus without the checking
         */

        //当出现% 字符时进行格式判断，否则直接拷贝
        if (*fmt == '%') {

            i64 = 0;
            ui64 = 0;

            //各标志位初始化
            zero = (u_char) ((*++fmt == '0') ? '0' : ' ');    //占位字符为0还是空格，第一个格式控制信息
            width = 0;    //宽度
            sign = 1;    //是否有符号
            hex = 0;    //是否是十六进制
            max_width = 0;    //最大宽度
            frac_width = 0;    //小数点后宽度
            slen = (size_t) -1;    //字符串长度

            //第二个格式控制信息，计算宽度
            while (*fmt >= '0' && *fmt <= '9') {
                width = width * 10 + (*fmt++ - '0');
            }


            //其它格式控制信息，是否无符号，最大宽度，十六进制，小数点位数，字符串长度，字符串长度由后面可变参数中获取
            for ( ;; ) {
                switch (*fmt) {

                case 'u':
                    sign = 0;
                    fmt++;
                    continue;

                case 'm':
                    max_width = 1;
                    fmt++;
                    continue;

                case 'X':
                    hex = 2;
                    sign = 0;
                    fmt++;
                    continue;

                case 'x':
                    hex = 1;
                    sign = 0;
                    fmt++;
                    continue;

                case '.':
                    fmt++;

                    while (*fmt >= '0' && *fmt <= '9') {
                        frac_width = frac_width * 10 + (*fmt++ - '0');
                    }

                    break;

                case '*':
                    slen = va_arg(args, size_t);
                    fmt++;
                    continue;

                default:
                    break;
                }

                break;
            }


            switch (*fmt) {

            case 'V':
                v = va_arg(args, ngx_str_t *);

                buf = ngx_sprintf_str(buf, last, v->data, v->len, hex);
                fmt++;

                continue;

            case 'v':
                vv = va_arg(args, ngx_variable_value_t *);

                buf = ngx_sprintf_str(buf, last, vv->data, vv->len, hex);
                fmt++;

                continue;

            case 's':
                p = va_arg(args, u_char *);

                buf = ngx_sprintf_str(buf, last, p, slen, hex);
                fmt++;

                continue;

            case 'O':
                i64 = (int64_t) va_arg(args, off_t);
                sign = 1;
                break;

            case 'P':
                i64 = (int64_t) va_arg(args, ngx_pid_t);
                sign = 1;
                break;

            case 'T':
                i64 = (int64_t) va_arg(args, time_t);
                sign = 1;
                break;

            case 'M':
                ms = (ngx_msec_t) va_arg(args, ngx_msec_t);
                if ((ngx_msec_int_t) ms == -1) {
                    sign = 1;
                    i64 = -1;
                } else {
                    sign = 0;
                    ui64 = (uint64_t) ms;
                }
                break;

            case 'z':
                if (sign) {
                    i64 = (int64_t) va_arg(args, ssize_t);
                } else {
                    ui64 = (uint64_t) va_arg(args, size_t);
                }
                break;

            case 'i':
                if (sign) {
                    i64 = (int64_t) va_arg(args, ngx_int_t);
                } else {
                    ui64 = (uint64_t) va_arg(args, ngx_uint_t);
                }

                if (max_width) {
                    width = NGX_INT_T_LEN;
                }

                break;

            case 'd':
                if (sign) {
                    i64 = (int64_t) va_arg(args, int);
                } else {
                    ui64 = (uint64_t) va_arg(args, u_int);
                }
                break;

            case 'l':
                if (sign) {
                    i64 = (int64_t) va_arg(args, long);
                } else {
                    ui64 = (uint64_t) va_arg(args, u_long);
                }
                break;

            case 'D':
                if (sign) {
                    i64 = (int64_t) va_arg(args, int32_t);
                } else {
                    ui64 = (uint64_t) va_arg(args, uint32_t);
                }
                break;

            case 'L':
                if (sign) {
                    i64 = va_arg(args, int64_t);
                } else {
                    ui64 = va_arg(args, uint64_t);
                }
                break;

            case 'A':
                if (sign) {
                    i64 = (int64_t) va_arg(args, ngx_atomic_int_t);
                } else {
                    ui64 = (uint64_t) va_arg(args, ngx_atomic_uint_t);
                }

                if (max_width) {
                    width = NGX_ATOMIC_T_LEN;
                }

                break;

            case 'f':
                //浮点数单独做处理
                f = va_arg(args, double);

                if (f < 0) {
                    *buf++ = '-';
                    f = -f;
                }

                ui64 = (int64_t) f;    //取浮点数整数部分
                frac = 0;

                if (frac_width) {

                    scale = 1;
                    for (n = frac_width; n; n--) {
                        scale *= 10;
                    }

                    //取小数点后的位置，+0.5 用于做四舍五入
                    frac = (uint64_t) ((f - (double) ui64) * scale + 0.5);
                    //如果四舍五入后小数点后为0,　则整数部分+1
                    if (frac == scale) {
                        ui64++;
                        frac = 0;
                    }
                }

                //写入整数部分
                buf = ngx_sprintf_num(buf, last, ui64, zero, 0, width);

                //写入小数部分
                if (frac_width) {
                    if (buf < last) {
                        *buf++ = '.';
                    }

                    buf = ngx_sprintf_num(buf, last, frac, '0', 0, frac_width);
                }

                fmt++;

                continue;

#if !(NGX_WIN32)
            case 'r':
                i64 = (int64_t) va_arg(args, rlim_t);
                sign = 1;
                break;
#endif

            case 'p':
                //十六进制显示指针
                ui64 = (uintptr_t) va_arg(args, void *);
                hex = 2;
                sign = 0;
                zero = '0';
                width = 2 * sizeof(void *);
                break;

            case 'c':
                //写入一个字符
                d = va_arg(args, int);
                *buf++ = (u_char) (d & 0xff);
                fmt++;

                continue;

            case 'Z':
                //写入\0
                *buf++ = '\0';
                fmt++;

                continue;

            case 'N':
                //写入换行，windows下为\r\n，其它为\n
#if (NGX_WIN32)
                *buf++ = CR;
                if (buf < last) {
                    *buf++ = LF;
                }
#else
                *buf++ = LF;
#endif
                fmt++;

                continue;

            case '%':
                //写入%
                *buf++ = '%';
                fmt++;

                continue;

            default:
                *buf++ = *fmt++;

                continue;
            }

            if (sign) {
                if (i64 < 0) {
                    *buf++ = '-';
                    ui64 = (uint64_t) -i64;

                } else {
                    ui64 = (uint64_t) i64;
                }
            }

            //按指定格式写入数字
            buf = ngx_sprintf_num(buf, last, ui64, zero, hex, width);

            fmt++;

        } else {
            *buf++ = *fmt++;
        }
    }

    return buf;
}

static u_char *
ngx_sprintf_num(u_char *buf, u_char *last, uint64_t ui64, u_char zero,
    ngx_uint_t hexadecimal, ngx_uint_t width)
{
    u_char         *p, temp[NGX_INT64_LEN + 1];
                       /*
                        * we need temp[NGX_INT64_LEN] only,
                        * but icc issues the warning
                        */
    size_t          len;
    uint32_t        ui32;
    static u_char   hex[] = "0123456789abcdef";
    static u_char   HEX[] = "0123456789ABCDEF";

    //先将格式化后的数据放到temp中
    p = temp + NGX_INT64_LEN;

    if (hexadecimal == 0) {

        //分开成32位还是64位因为两者除法算法不同，区分开可以更快
        if (ui64 <= (uint64_t) NGX_MAX_UINT32_VALUE) {

            /*
             * To divide 64-bit numbers and to find remainders
             * on the x86 platform gcc and icc call the libc functions
             * [u]divdi3() and [u]moddi3(), they call another function
             * in its turn.  On FreeBSD it is the qdivrem() function,
             * its source code is about 170 lines of the code.
             * The glibc counterpart is about 150 lines of the code.
             *
             * For 32-bit numbers and some divisors gcc and icc use
             * a inlined multiplication and shifts.  For example,
             * unsigned "i32 / 10" is compiled to
             *
             *     (i32 * 0xCCCCCCCD) >> 35
             */

            ui32 = (uint32_t) ui64;

            do {
                *--p = (u_char) (ui32 % 10 + '0');
            } while (ui32 /= 10);

        } else {
            do {
                *--p = (u_char) (ui64 % 10 + '0');
            } while (ui64 /= 10);
        }

    } else if (hexadecimal == 1) {

        do {

            /* the "(uint32_t)" cast disables the BCC's warning */
            *--p = hex[(uint32_t) (ui64 & 0xf)];

        } while (ui64 >>= 4);

    } else { /* hexadecimal == 2 */

        do {

            /* the "(uint32_t)" cast disables the BCC's warning */
            *--p = HEX[(uint32_t) (ui64 & 0xf)];

        } while (ui64 >>= 4);
    }

    /* zero or space padding */

    len = (temp + NGX_INT64_LEN) - p;

    //不够宽度在前面补0 或空格
    while (len++ < width && buf < last) {
        *buf++ = zero;
    }

    /* number safe copy */
    //计算数字占用的长度，拷贝到buf 并返回拷贝后的位置
    len = (temp + NGX_INT64_LEN) - p;

    if (buf + len > last) {
        len = last - buf;
    }

    return ngx_cpymem(buf, p, len);
}

//格式化输出，hexadecimal: 0-默认输出，1-十六进制小写输出，2-十六进制大写输出
//返回写入buf 后的位置
static u_char *
ngx_sprintf_str(u_char *buf, u_char *last, u_char *src, size_t len,
    ngx_uint_t hexadecimal)
{
    static u_char   hex[] = "0123456789abcdef";
    static u_char   HEX[] = "0123456789ABCDEF";

    if (hexadecimal == 0) {

        if (len == (size_t) -1) {
            while (*src && buf < last) {
                *buf++ = *src++;
            }

        } else {
            len = ngx_min((size_t) (last - buf), len);
            buf = ngx_cpymem(buf, src, len);
        }

    } else if (hexadecimal == 1) {

        if (len == (size_t) -1) {

            while (*src && buf < last - 1) {
                *buf++ = hex[*src >> 4];
                *buf++ = hex[*src++ & 0xf];
            }

        } else {

            while (len-- && buf < last - 1) {
                *buf++ = hex[*src >> 4];
                *buf++ = hex[*src++ & 0xf];
            }
        }

    } else { /* hexadecimal == 2 */

        if (len == (size_t) -1) {

            while (*src && buf < last - 1) {
                *buf++ = HEX[*src >> 4];
                *buf++ = HEX[*src++ & 0xf];
            }

        } else {

            while (len-- && buf < last - 1) {
                *buf++ = HEX[*src >> 4];
                *buf++ = HEX[*src++ & 0xf];
            }
        }
    }

    return buf;
}
```

### 字符串与其它类型转换

```
ngx_int_t ngx_atoi(u_char *line, size_t n);    //字符串转int
ngx_int_t ngx_atofp(u_char *line, size_t n, size_t point);    //float字符串转int，n-取字符串长度，point-小数点后取的位数
ssize_t ngx_atosz(u_char *line, size_t n);    //字符串转size类型，与ngx_atoi基本相同，只对于最大值范围不同
off_t ngx_atoof(u_char *line, size_t n);    //字符串转off_t
time_t ngx_atotm(u_char *line, size_t n);    //字符串转time_t
ngx_int_t ngx_hextoi(u_char *line, size_t n);    //十六进制字符串转int

//将src的值转为十六进制放到dst中
u_char *ngx_hex_dump(u_char *dst, u_char *src, size_t len);
```

下面是 ngx_atoi 实现代码，其它函数与之类似。  

```
ngx_int_t
ngx_atoi(u_char *line, size_t n)
{
    ngx_int_t  value, cutoff, cutlim;

    if (n == 0) {
        return NGX_ERROR;
    }

    //NGX_MAX_INT_T_VALUE 最大的int 值
    cutoff = NGX_MAX_INT_T_VALUE / 10;
    cutlim = NGX_MAX_INT_T_VALUE % 10;

    for (value = 0; n--; line++) {
        if (*line < '0' || *line > '9') {
            return NGX_ERROR;
        }

        //这里是关于int 最大值的判断
        if (value >= cutoff && (value > cutoff || *line - '0' > cutlim)) {
            return NGX_ERROR;
        }

        value = value * 10 + (*line - '0');
    }

    return value;
}
```

### 字符串编解码

Base64 编解码相关，URL 的base64与普通base64算法基本类似，只是URL里有三个字符 '+', '/', '='有特殊含义，因此在 URL 的 base64 算法中这三个字符会被替换掉，'+' 替换为 '-'，'/' 替换为 '_'，'=' 省略，详见[RFC4684-Base16, Base32, 和 Base64 数据编码-中文翻译](https://snownight22.github.io/TTworksBlog/2022/07/04/rfc4684-baseEncode-cn/)  

```
//计算base64编解码后的长度
#define ngx_base64_encoded_length(len)  (((len + 2) / 3) * 4)
#define ngx_base64_decoded_length(len)  (((len + 3) / 4) * 3)

void ngx_encode_base64(ngx_str_t *dst, ngx_str_t *src);
void ngx_encode_base64url(ngx_str_t *dst, ngx_str_t *src);
ngx_int_t ngx_decode_base64(ngx_str_t *dst, ngx_str_t *src);
ngx_int_t ngx_decode_base64url(ngx_str_t *dst, ngx_str_t *src);
```

utf8 编解码：  

```
uint32_t ngx_utf8_decode(u_char **p, size_t n);
size_t ngx_utf8_length(u_char *p, size_t n);
u_char *ngx_utf8_cpystrn(u_char *dst, u_char *src, size_t n, size_t len);
```

uri 转码相关：  

```
//这几个函数都使用位来标识字符是否需要转码
uintptr_t ngx_escape_uri(u_char *dst, u_char *src, size_t size,
    ngx_uint_t type);
void ngx_unescape_uri(u_char **dst, u_char **src, size_t size, ngx_uint_t type);
uintptr_t ngx_escape_html(u_char *dst, u_char *src, size_t size);
uintptr_t ngx_escape_json(u_char *dst, u_char *src, size_t size);
```

### 其它函数

```
//字符串红黑树的插入与查找，红黑树节点应为ngx_str_node_t结构
void ngx_str_rbtree_insert_value(ngx_rbtree_node_t *temp,
    ngx_rbtree_node_t *node, ngx_rbtree_node_t *sentinel);    //插入
ngx_str_node_t *ngx_str_rbtree_lookup(ngx_rbtree_t *rbtree, ngx_str_t *name,
    uint32_t hash);    //查找

//排序，base- 列表基地址，n- 列表排序个数，size- 列表元素大小，cmp- 排序比较函数
//这个排序用的是插入排序，稳定排序
void ngx_sort(void *base, size_t n, size_t size,
    ngx_int_t (*cmp)(const void *, const void *));
#define ngx_qsort             qsort
```

ngx_sort 使用插入排序，详见[排序算法之-插入排序](https://snownight22.github.io/TTworksBlog/2022/04/01/insertSort/)，下面是其实现代码：  

```
void
ngx_sort(void *base, size_t n, size_t size,
    ngx_int_t (*cmp)(const void *, const void *))
{
    u_char  *p1, *p2, *p;

    //申请一个临时变量
    p = ngx_alloc(size, ngx_cycle->log);
    if (p == NULL) {
        return;
    }

    //使用插入排序
    //这里从第 2 个元素开始遍历列表元素
    for (p1 = (u_char *) base + size;
         p1 < (u_char *) base + n * size;
         p1 += size)
    {
        ngx_memcpy(p, p1, size);

        //与已排序好的元素进行比较，找到合适的位置
        for (p2 = p1;
             p2 > (u_char *) base && cmp(p2 - size, p) > 0;
             p2 -= size)
        {
            ngx_memcpy(p2, p2 - size, size);
        }

        //插入元素到已排序列表
        ngx_memcpy(p2, p, size);
    }

    ngx_free(p);
}
```

---
nginx源码基于nginx-1.22.0版本  
**参考**：  
[nginx平台初探](http://tengine.taobao.org/book/chapter_02.html#ngx-str-t-100)   
[nginx开发_字符串操作函数](http://t.zoukankan.com/atskyline-p-8124247.html)  
[Nginx源码分析 - 基础数据结构篇 - 字符串结构 ngx_string.c（08）](https://blog.csdn.net/initphp/article/details/50682163)  
