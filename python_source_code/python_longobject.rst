longobject
=============

python3中, int和long合并了, 都属于longobject

1. 无限长度数字是用数组来实现的. 基数x=2**30, 数组那么第一个元素就是m*(x ** 0) = m, 第二个元素之后表示有区别, 假设第二个元素的值是4, 也就是100

   那么实际上表示的是2**32, 所以如果n = 2**32 + 5, 那么数组就是[5, 4], 4的由来是1 << (32 - 30) = 4

2. longobject中的size为数组的长度, 然后如果size是负数, 则表示longobject是负数.

3. 从存储数组到实际以10为底的数字(比如打印数字的时候), 打印的时候输出的数字是以10**9为底的10进制数

4. 小整数[-5, 255)会被缓存, 所以全局都是同一个对象

5. python3.6不会缓存大整数对象, 而python2.x则在回收大整数对象的时候会缓存起来, 造成内存增大

6. 大数相乘是使用Karatsuba算法, 复杂度降低到3 * (n ** log3)(log3是2为底) , 而经典算法(就是小学教的画图进位的方法)时间复杂度O(n ** 2)
   
   
Karatsuba算法
==============

参考 [7]_

设, x = a * 10 ** (n/2) + b, y = c * 10 ** (n/2) + d

x * y = ac * 10** n + (ad + bc) * 10 ** (n/2) + bd, 由于 ad + bc = (a+b) * (c+d) - ac - bd, 所以

a*d和b*c则不需要分别计算, 直接用ac, bd, (a+b)*(c+d)结果用加法就可以算出来了, 乘法的个数从四次减少到了三次


longobject存储示例
========================

如果x = 2**32 + 1 = 4294967297, 那么:

1. size=2, 这是因为两次x>>30计算才为0
   
2. 存储的数组ob_digit: [1, 4], 这是因为1 << (32 - 30) = 4, 4的二进制是100, 所以数组第二个元素的值是4(100)也就是表示2**32

   如果x = -(2**32 + 1) = -4294967297, 那么存储的内容和上面的一致, 只是size = -2

如果x=2**61 + 1 = 2305843009213693953, 那么有:

1. size = 3

2. ob_digit: [1, 0, 2], 也就是2的二进制是10, 2所在的数组开始的是2**60, 2的二进制是10, 也就是2**60往左移动1位, 就是数字就是2**61, 然后就是2**61 + 0 + 1

如果x = 2**61+2**33+1 = 2305843017803628545, 那么:

1. size = 3

2. ob_digit: [1, 8, 2], 1 << (33 - 30) = 8, 所以8(1000)代表2**33, 所以, 1 << (61 - 60) = 2, 2(10)代表2**61

如果x = x=2**61 + 3 + 2**34 + 4 = 2305843026393563143, 那么:

1. size = 3

2. ob_digit: [7, 16, 2], 7=3+4 


longobject从存储到十进制表示
=================================

当调用longobject的__str__或者__repr__方法的时候, 会打印出以10为底的整数

由于c语言中的溢出限制, 所以把ob_digit数组中的数转换成10为底的数的时候, 不能直接计算n次方, 而是使用数组存储数字的形式, 

并且数组每个元素的值都是小于10**9(64位下), 之所以是10**9, 是因为4字节的整数最大不超过10**9, 所以用10**9作为基数

如: x = 2**61 + 3 + 2**34 + 4 = 2305843026393563143, 存储的话上一节的一样, ob_digit = [7, 16, 2],

然后打印longobject的话, 逐个打印数字, 所以打印的时候应该打印2305843026393563143. 打印的时候是根据一个输出数组pout来打印

最终pout将是[393563143, 305843026, 2], 打印的时候反过来输出, 也就是输出2, 305843026, 393563143, 连起来刚刚好是原来的数字!!!!!

同样, 如果x = 2**91+1+2**32+2+5 = 2,475880078,570760554,093215752

那么pout数组就是[93215752, 570760554, 475880078, 2], 打印的时候, 读取2, 475880078, 570760554, 93215752], 至于

**570760554和93215752之间有个0, 怎么显示, 目前不清楚**

猜测是如果数字小鱼10**9的话, 补0, 比如93215752小于10**9, 那么前面应该补0, 补的个数是93215752和10**9的位数差值

93215752和10**9差1位, 所以补一个0.


PyLongObject
==============


cpython/Include/longobject.h

.. code-block:: c

   typedef struct _longobject PyLongObject;

_longobject是在cpython/Include/longintrepr.h

.. code-block:: c

    struct _longobject {
        // 其中head里面就包含了数组长度了.
    	PyObject_VAR_HEAD
        // 又看到了初始化长度为1但是一定会越界的数组
    	digit ob_digit[1];
    };


所以整数就是存储在ob_digit这个数组中的, 每个数组是digit类型, digit是根据PYLONG_BITS_IN_DIGIT宏决定的:

.. code-block:: c

    #if PYLONG_BITS_IN_DIGIT == 30
    typedef uint32_t digit;
    typedef int32_t sdigit; /* signed variant of digit */
    typedef uint64_t twodigits;
    typedef int64_t stwodigits; /* signed variant of twodigits */
    #define PyLong_SHIFT	30
    #define _PyLong_DECIMAL_SHIFT	9 /* max(e such that 10**e fits in a digit) */
    #define _PyLong_DECIMAL_BASE	((digit)1000000000) /* 10 ** DECIMAL_SHIFT */

一般如果没有定义PYLONG_BITS_IN_DIGIT的话, 默认会把PYLONG_BITS_IN_DIGIT设置为30:

.. code-block:: c

    /* If PYLONG_BITS_IN_DIGIT is not defined then we'll use 30-bit digits if all
       the necessary integer types are available, and we're on a 64-bit platform
       (as determined by SIZEOF_VOID_P); otherwise we use 15-bit digits. */
    
    #ifndef PYLONG_BITS_IN_DIGIT
    #if SIZEOF_VOID_P >= 8
    #define PYLONG_BITS_IN_DIGIT 30
    #else
    #define PYLONG_BITS_IN_DIGIT 15
    #endif
    #endif


64位平台下:

1. digit类型则是32位, 而twodigits则是64位

2. _PyLong_DECIMAL_SHIFT这个是用来转换成10进制的时候的底数, 是10**9

PyLong_FromLong
====================


.. code-block:: c

    PyObject *
    PyLong_FromLong(long ival)
    {
        PyLongObject *v;
        unsigned long abs_ival;
        unsigned long t;  /* unsigned so >> doesn't propagate sign bit */
        int ndigits = 0;
        int sign;
    
        // 小整数就从缓存拿
        // 这个宏里面有个return, 所以如果是小整数, 直接return
        CHECK_SMALL_INT(ival);
    
        // 下面是判断符号位的
        if (ival < 0) {
            /* negate: can't write this as abs_ival = -ival since that
               invokes undefined behaviour when ival is LONG_MIN */
            abs_ival = 0U-(unsigned long)ival;
            sign = -1;
        }
        else {
            abs_ival = (unsigned long)ival;
            sign = ival == 0 ? 0 : 1;
        }
    
        /* Fast path for single-digit ints */
        // 构造PyLongObject
        // 这里向右移1位为空表示该整数的ob_digit长度为1, 也就是位数了
        // 也就是大小小于2**30
        // 大于2**30的在下面继续求位数
        if (!(abs_ival >> PyLong_SHIFT)) {
            v = _PyLong_New(1);
            if (v) {
                Py_SIZE(v) = sign;
                v->ob_digit[0] = Py_SAFE_DOWNCAST(
                    abs_ival, unsigned long, digit);
            }
            return (PyObject*)v;
        }
    
    #if PyLong_SHIFT==15
    // 这一部分代码省略了
    #endif
    
        /* Larger numbers: loop to determine number of digits */
        // ob_digit长度超过1的整数继续求位数
        // 向右移动30位不为空, 那么ob_digit长度加1, 也就是位数加1
        t = abs_ival;
        while (t) {
            ++ndigits;
            t >>= PyLong_SHIFT;
        }
        v = _PyLong_New(ndigits);
        if (v != NULL) {
            digit *p = v->ob_digit;
            Py_SIZE(v) = ndigits*sign;
            t = abs_ival;
            while (t) {
                // 每一个ob_digit的元素赋值
                *p++ = Py_SAFE_DOWNCAST(
                    t & PyLong_MASK, unsigned long, digit);
                t >>= PyLong_SHIFT;
            }
        }
        return (PyObject *)v;
    }

转成十进制
=============

.. code-block:: c

    static int
    long_to_decimal_string_internal(PyObject *aa,
                                    PyObject **p_output,
                                    _PyUnicodeWriter *writer,
                                    _PyBytesWriter *bytes_writer,
                                    char **bytes_str)
    {
    
        // 拿到PyLongObject
        a = (PyLongObject *)aa;
    
        // 传入的PyLongObject的数组
        pin = a->ob_digit;
        // pout也是一个digit类型的数组
        pout = scratch->ob_digit;
        size = 0;
        // 下面的循环就是转成以10**9为底的10进制的过程
        // 没怎么看懂
        for (i = size_a; --i >= 0; ) {
            digit hi = pin[i];
            for (j = 0; j < size; j++) {
                twodigits z = (twodigits)pout[j] << PyLong_SHIFT | hi;
                hi = (digit)(z / _PyLong_DECIMAL_BASE);
                pout[j] = (digit)(z - (twodigits)hi *
                                  _PyLong_DECIMAL_BASE);
            }
            while (hi) {
                pout[size++] = hi % _PyLong_DECIMAL_BASE;
                hi /= _PyLong_DECIMAL_BASE;
            }
            /* check for keyboard interrupt */
            SIGCHECK({
                    Py_DECREF(scratch);
                    return -1;
                });
        }
    
    }


小整数池
==========

python中会全局缓存小整数, 缓存的小整数的范围是[-5, 257):


.. code-block:: c

    #ifndef NSMALLPOSINTS
    #define NSMALLPOSINTS           257
    #endif
    #ifndef NSMALLNEGINTS
    #define NSMALLNEGINTS           5
    #endif

    /* Small integers are preallocated in this array so that they
       can be shared.
       The integers that are preallocated are those in the range
       -NSMALLNEGINTS (inclusive) to NSMALLPOSINTS (not inclusive).
    */
    static PyLongObject small_ints[NSMALLNEGINTS + NSMALLPOSINTS];


CHECK_SMALL_INT
----------------

如果是小整数, 则返回, 注意的是, 这里是带有return的

.. code-block:: c

    #define CHECK_SMALL_INT(ival) \
        // 判断大小
        do if (-NSMALLNEGINTS <= ival && ival < NSMALLPOSINTS) { \
            return get_small_int((sdigit)ival); \
        } while(0)


get_small_int
------------------

.. code-block:: c

    static PyObject *
    get_small_int(sdigit ival)
    {
        PyObject *v;
        assert(-NSMALLNEGINTS <= ival && ival < NSMALLPOSINTS);
        // 从小整数数组拿出对应数值的对象返回
        v = (PyObject *)&small_ints[ival + NSMALLNEGINTS];
        Py_INCREF(v);
    #ifdef COUNT_ALLOCS
        if (ival >= 0)
            quick_int_allocs++;
        else
            quick_neg_int_allocs++;
    #endif
        return v;
    }


py3去掉PyIntBlock
====================

参考1: http://www.wklken.me/posts/2014/08/06/python-source-int.html

参考2: http://www.wklken.me/posts/2014/08/06/python-source-int.html

py2中, dealloc一个整数之后会判断是否是整数, 如果是整数那么回到缓存的free_list, 不是整数则释放内存:

.. code-block:: c

    static void
    int_dealloc(PyIntObject *v)
    {
        if (PyInt_CheckExact(v)) {
            // 这里只要是整数就回到free_list
            Py_TYPE(v) = (struct _typeobject *)free_list;
            free_list = v;
        }
        else
            // 不是整数就释放内存
            Py_TYPE(v)->tp_free((PyObject *)v);
    }


所以py2也是会缓存大整数的, 而py3是直接释放到全局的内存池:

.. code-block:: c

    PyTypeObject PyLong_Type = {
        long_dealloc,                               /* tp_dealloc */
        PyObject_Del,                               /* tp_free */
    };

.. code-block:: c

    static void
    long_dealloc(PyObject *v)
    {
        // 直接调用tp_free, 也就是PyObject_Del
        Py_TYPE(v)->tp_free(v);
    }

PyObject_Del定义为PyObject_Free, 根据python内存中的机制去决定是否去真正释放内存.

具体流程参考: python_memory_management.rst

例子, 分别在python2和python3中, 执行 *x=list(range(5000000))*, 可以看到, 内存都会增长(我的机器下是大概200M)

然后执行 *del x*, 可以看到, python2中内存没有下降, 而python3中内存下降了

