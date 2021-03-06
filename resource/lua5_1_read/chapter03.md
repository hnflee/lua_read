# Lua5.1.4代码分析(三)表类型

来看Lua的表的定义:
```C
    typedef struct Table {
      CommonHeader;
      lu_byte flags;
      lu_byte lsizenode;  /* log2 of size of `node' array */
      struct Table *metatable;
      TValue *array;  /* array part */
      Node *node;
      Node *lastfree;  /* any free position is before this position */
      GCObject *gclist;
      int sizearray;  /* size of `array' array */
    } Table;
```

这里将Table分为了两个部分:数组部分,array指针指向数组部分的首地址,sizearray是数组的尺寸,绝大部分(注意:不是全部)正整数为key的数据都存放在数组部分;node指针指向一个hash桶,对于不能存放在数组部分的数据,都存放在hash中.如下图所示:

![](https://wyyhzc.gitbooks.io/lua5-1_read_info/content/table-300x276.png)

hash部分需要特别注意的一点是:在物理上,所有hash部分的数据,其实是存放一块连续的内存中的,即node指针指向的数组;但是从逻辑上来看,如果几块数据在同一个hash桶上,那么又是通过next指针串联起来的.
以图中的示例来分析,node数组的第一个和第三个元素,在物理上是第一和第三个元素,但是在逻辑上,它们是通过next指针串联起来的.

有了以上的了解,从Table中查找一个数据的伪代码就很显而易见了:

```
如果输入的Key是一个正整数,并且它的值 > 0 && <= 数组大小
				     尝试在数组部分查找
否则尝试在Hash部分进行查找:
	计算出该Key的Hash值(ltable.c中的mainposition函数),根据此Hash值访问node数组得到Hash桶所在位置
	遍历该Hash桶下的所有链表元素,直到找到该Key为止
	
```

以上已经明白了Table的大致结构,来看看Table中如果新加入新的数据会怎么处理.这里有一些内容,要留到后面讲解到Lua虚拟机的时候才触及,这里先讲解一下,当新插入数据时,Table内的数组和Hash部分,做了哪些变化.

这部分中,核心的算法在ltable.c的rehash函数中,这个函数是计算当新添加数据时,数组和hash重新分配之后各自的尺寸是多少,伪代码如下:

```
    首先分配一个位图nums,将其中的所有位置0,这个位图的意义在于:nums数组中第i个元素存放的是key在2^(i-1), 2^i之间的元素数量
    
    遍历lua Table中的数组部分,计算在数组部分中的元素数量,更新对应的nums数组元素数量.(numusearray函数)
    
    遍历lua Table中的Hash部分,因为其中也可能存放了正整数,也根据这里的正整数数量更新对应的nums数组元素数量.(numusehash函数)
    
    此时nums数组已经有了当前这个Table中所有正整数的分配统计,逐个遍历nums数组,如果当前已经有的根据新的数组大小和Hash大小重新
```

这里要特别讲解的是computesizes函数,在前面的两个函数调用numusearray函数和numusehash函数之后,此时在nums位图中,已经存放了所有有关整数key的信息,即在[2^(i-1), 2^i]范围内,有多少数据.前面曾经提到过,并不是所有的正整数,都会存放在数组部分的,即使它曾经在,也有可能在之后被分配到hash部分,那么判断的依据是什么?到底怎样的数据,在重新分配之后会从数组部分挪到hash部分?
来看computesizes函数的实现:

```C
    static int computesizes (int nums[], int *narray) {
      int i;
      int twotoi;  /* 2^i */
      int a = 0;  /* number of elements smaller than 2^i */
      int na = 0;  /* number of elements to go to array part */
      int n = 0;  /* optimal size for array part */
      for (i = 0, twotoi = 1; twotoi/2 < *narray; i++, twotoi *= 2) {
        if (nums[i] > 0) {
          a += nums[i];
          if (a > twotoi/2) {  /* more than half elements present? */
            n = twotoi;  /* optimal size (till now) */
            na = a;  /* all elements smaller than n will go to array part */
          }
        }
        if (a == *narray) break;  /* all elements already counted */
      }
      *narray = n;
      lua_assert(*narray/2 <= na && na <= *narray);
      return na;
    }
```

注意到这样的细节:这个函数在遍历nums位图数组的时候,会将当前数据数量存放在变量a中,如果a > twotoi/2,也就是当前有一半以上的空间被利用上了,那么这部分数据会继续留在数组部分,否则就会在之后挪到hash部分了.

为了证实这里的判断,简单的写一段lua代码做为实验:


```LUA

    function print_ipairs(t)
      print("in print_ipairs")
      for k, v in ipairs(t) do
        print(k)
      end
    end 
    
    function print_pairs(t)
      print("in print_pairs")
      for k, v in pairs(t) do
        print(k)
      end
    end 
    
    a = {}
    a={1,2,3,4,5,6,7,8,9,10}
    print_ipairs(a)
    a[2] = nil
    a[3] = nil
    a[4] = nil
    a[6] = nil
    a["k"] = "e"
    print_ipairs(a)
    print_pairs(a)
```

输出为:

```
in print_ipairs
1
2
3
4
5
6
7
8
9
10
in print_ipairs
1
in print_pairs
1
7
8
10
k
5
9
```

在这里,首先对表a赋值,有1-10共十个元素,通过调用函数print_ipairs可知,这些元素都是存放在数组部分的,这是因为ipairs取的是Table的数组部分元素.
在这之后,人为的将其中2,3,4,6元素删除,造成原来数组不满一半元素被利用上的现象,然后再插入一个新key "k",以让这个Table重新分配空间.再此之后,再次调用ipairs遍历a的数组部分,可以看到只有1被打印出来了,也就是说,在重新分配空间之后,除去已经被删除的2,3,4,6之外,只有1还在数组里面,剩下的5,7,8,9,10已经不在数组部分了.紧接着调用pairs遍历这个表,可以看出这些已经不在数组部分的值又被打印出来了,并且它们的顺序已经被打乱,不再按照数字大小顺序来排列了,它们在这次重新分配中被挪动到了hash部分.

这个实验既验证了我们前面的分析,同时也告诉我们,Table的重新分配,实际上代价是很大的,因此不建议在实际程序中,一个Table即有数组部分,也有Hash部分,纯粹一些,性能上会有提升.
