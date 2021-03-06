# 《Redis部分源码分析-C字符串改造》

  Redis没有直接使用C语言传统的字符串表示（以空字符结尾的字符数组，以下简称C字符串），而是自己构建了一种名为简单动态字符串（simple dynamic string，SDS）的抽象类型，并将SDS用作Redis的默认字符串表示。  
**本节内容**  
* [Redis字符串定义](#1)
* [SDS与字符串得区别](#2)
	* [对求字符串长度(strlen)的改造](#3)
	* [杜绝缓冲溢出](#4)
	* [减少修改字符串时带来得内存重分配次数](#5)
	* [二进制安全](#6)
	* [总结](#7)
* [SDS其他函数注释](#8)

<h2 id="1">Redis字符串定义</h2>  

```C  
struct sdshdr{  
	int len;  //buf中已使用的字节数量  
	int free;  //未使用的字节数量  
	char buf[];  //字节数组，用于保存字符串  
}；  
```  

下图中展示了一个SDS示例  
![image](https://github.com/xiethon/Redis-3.0/blob/master/doc/photos/SDS字符串.png)  

* free属性的值为0，表示这个SDS没有分配任何未使用空间。  
* len属性的值为5，表示这个SDS保存了一个五字节长的字符串。  
* buf属性是一个char类型的数组，数组的前五个字节分别保存了'R'、'e'、'd'、 'i'、's'五个字符，而最后一个字节则保存了空字符'\0'。  

SDS遵循C字符串以空字符结尾的惯例，保存空字符的1字节空间不计算在SDS的len属性里面，并且为空字符分配额外的1字节空间，以及添加空字符到字符串末尾等操作，都是由SDS函数自动完成的，所以这个空字符对于SDS的使用者来说是完全透明的。遵循空字符结尾这一惯例的好处是，SDS可以直接重用一部分C字符串函数库里面的函数  

<h2 id="2">2. SDS与C字符串的区别</h2> 
<h3 id="3">2.1 对求字符串长度(strlen)的改造</h3>  

下图中展示了标准C库计算一个C字符串长度的过程。  
![](https://github.com/xiethon/Redis-3.0/blob/master/doc/photos/C字符串长度.png)  

和C字符串不同，因为SDS在len属性中记录了SDS本身的长度，所以获取一个SDS长度的复杂度仅为O(1)。 
sdslen实现相关源码如下(sds/sds.h)  

```C
/* 计算sds的长度，返回的size_t类型的数值 */
/* size_t,它是一个与机器相关的unsigned类型，其大小足以保证存储内存中对象的大小。 */
static inline size_t sdslen(const sds s) 
{
    struct sdshdr *sh = (void*)(s-(sizeof(struct sdshdr)));
    return sh->len;
}
```

### 
<h3 id="4">2.2 杜绝缓冲溢出</h3>  

因为C字符串不记录自身的长度，所以strcat假定用户在执行这个函数时，已经为dest分配了足够多的内存，可以容纳src字符串中的所有内容，而一旦这个假定不成立时，就会产生缓冲区溢出。  

与C字符串不同，SDS的空间分配策略完全杜绝了发生缓冲区溢出的可能性：当SDS API需要对SDS进行修改时，API会先检查SDS的空间是否满足修改所需的要求，如果不满足的话，API会自动将SDS的空间扩展至执行修改所需的大小，然后才执行实际的修改操作，所以使用SDS既不需要手动修改SDS的空间大小，也不会出现前面所说的缓冲区溢出问题。  

SDS的sdscat在执行拼接操作之前检查s的长度是否足够，在发现s目前的空间不足以拼接字符串之后，sdscat就会先扩展s的空间，然后才执行拼接的操作  

sdscat实现相关源码如下(sds/sds.c)：  
```C
sds sdscat(sds s, const char *t) 
{
    return sdscatlen(s, t, strlen(t));
}
sds sdscatlen(sds s, const void *t, size_t len)  
{  
	struct sdshdr *sh;  
	size_t curlen = sdslen(s);  
	s = sdsMakeRoomFor(s,len);  //为原字符串扩展len长度空间
	if (s == NULL) return NULL;
	sh = (void*) (s-(sizeof(struct sdshdr)));
	memcpy(s+curlen, t, len); 	//多余的数据以t作初始化
	sh->len = curlen+len;//更改相应的len,free值
	sh->free = sh->free-len;
	s[curlen+len] = '\0';
	return s;
}  
```
###  
<h3 id="5">2.3 减少修改字符串时带来得内存重分配次数</h3>  

因为内存重分配涉及复杂的算法，并且可能需要执行系统调用，所以它通常是一个比较耗时的操作。  
为了避免C字符串的这种缺陷，SDS通过未使用空间解除了字符串长度和底层数组长度之间的关联：在SDS中，buf数组的长度不一定就是字符数量加一，数组里面可以包含未使用的字节，而这些字节的数量就由SDS的free属性记录。  
通过空间预分配策略，Redis可以减少连续执行字符串增长操作所需的内存重分配次数。  

###   
<h3 id="6">2.4 二进制安全</h4>  

C字符串中的字符必须符合某种编码（比如ASCII），并且除了字符串的末尾之外，字符串里面不能包含空字符，否则最先被程序读入的空字符将被误认为是字符串结尾，这些限制使得C字符串只能保存文本数据，而不能保存像图片、音频、视频、压缩文件这样的二进制数据。如果有一种使用空字符来分割多个单词的特殊数据格式。

##  
<h3 id="7">2.5 总结</h3>  

下图中展示了SDS与C字符串的区别总结  
![](https://github.com/xiethon/Redis-3.0/blob/master/doc/photos//C字符串与SDS字符串的区别.png)  
图4中展示了SDS主要API  
![](https://github.com/xiethon/Redis-3.0/blob/master/doc/photos/SDS主要接口函数.png)  


<h2 id="8">3. SDS其他函数注释</h2>  

* [sdsMakeRoomFor：字符串扩容](#9)
* [sdsnew:创建一个字符串](#10)
* [sdsIncrLen:改变字符串长度](#11)
* [更多注释请参阅源码注释](https://github.com/xiethon/Redis-3.0/tree/master/src_note/sds)

<h4 id="9">sdsMakeRoomFor：字符串扩容</h6>  

```C
/* Enlarge the free space at the end of the sds string so that the caller
 * is sure that after calling this function can overwrite up to addlen
 * bytes after the end of the string, plus one more byte for nul term.
 *
 * Note: this does not change the *length* of the sds string as returned
 * by sdslen(), but only the free buffer space we have. */
/* 在原有字符串中取得更大的空间，并返回扩展空间后的字符串 */
sds sdsMakeRoomFor(sds s, size_t addlen) {
    struct sdshdr *sh, *newsh;
    //获取当前字符串的可用长度
    size_t free = sdsavail(s);
    size_t len, newlen;

	//如果当前可用空间已经大于需要值，直接返回原字符串
    if (free >= addlen) return s;
    len = sdslen(s);
    sh = (void*) (s-(sizeof(struct sdshdr)));
    //计算要获取新字符串所要的长度大小=原长度+addlen
    newlen = (len+addlen);
    if (newlen < SDS_MAX_PREALLOC)
        newlen *= 2;
    else
        newlen += SDS_MAX_PREALLOC;
    newsh = zrealloc(sh, sizeof(struct sdshdr)+newlen+1);
    if (newsh == NULL) return NULL;

	//新字符串可用空间等于新长度减去原使用的长度
    newsh->free = newlen - len;
    //返回洗字符串中的buf字符串数组
    return newsh->buf;
}
```

<h4 id="10">sdsnew：创建一个sds字符串</h6>  

```C
/* Create a new sds string with the content specified by the 'init' pointer
 * and 'initlen'.
 * If NULL is used for 'init' the string is initialized with zero bytes.
 *
 * The string is always null-termined (all the sds strings are, always) so
 * even if you create an sds string with:
 *
 * mystring = sdsnewlen("abc",3");
 *
 * You can print the string with printf() as there is an implicit \0 at the
 * end of the string. However the string is binary safe and can contain
 * \0 characters in the middle, as the length is stored in the sds header. */
/* 创建新字符串方法，传入目标长度，初始化方法 */
sds sdsnewlen(const void *init, size_t initlen) 
{
    struct sdshdr *sh;

    if (init) {
        sh = zmalloc(sizeof(struct sdshdr)+initlen+1);
    } else {
    	//当init函数为NULL时候，又来了zcalloc的方法
        sh = zcalloc(sizeof(struct sdshdr)+initlen+1);
    }
    if (sh == NULL) return NULL;
    sh->len = initlen;
    sh->free = 0;
    if (initlen && init)
        memcpy(sh->buf, init, initlen);
   //最末端同样要加‘\0’结束符
    sh->buf[initlen] = '\0';
    //最后是通过返回字符串结构体中的buf代表新的字符串
    return (char*)sh->buf;
}

/* Create an empty (zero length) sds string. Even in this case the string
 * always has an implicit null term. */
/* 其实就是创建一个长度为0的空字符串 */
sds sdsempty(void) {
    return sdsnewlen("",0);
}

/* Create a new sds string starting from a null termined C string. */
/* 根据init函数指针创建字符串 */
sds sdsnew(const char *init) {
    size_t initlen = (init == NULL) ? 0 : strlen(init);
    return sdsnewlen(init, initlen);
}
}
```

<h4 id="11">sdsIncrLen：改变字符串常量</h6>  

```C
/* 改变字符串中的长度以使用量的使用情况数值 */
void sdsIncrLen(sds s, int incr) {
    struct sdshdr *sh = (void*) (s-(sizeof(struct sdshdr)));

    if (incr >= 0)
        assert(sh->free >= (unsigned int)incr);
    else
        assert(sh->len >= (unsigned int)(-incr));
    sh->len += incr;
    sh->free -= incr;
    s[sh->len] = '\0';
}

/* Grow the sds to have the specified length. Bytes that were not part of
 * the original length of the sds will be set to zero.
 *
 * if the specified length is smaller than the current length, no operation
 * is performed. */
/* 扩展字符串到指定的长度 */
sds sdsgrowzero(sds s, size_t len) {
    struct sdshdr *sh = (void*)(s-(sizeof(struct sdshdr)));
    size_t totlen, curlen = sh->len;

	//如果当前长度已经大于要求长度，直接返回
    if (len <= curlen) return s;
    //如果小于，则重新为此字符串分配新空间，得到新字符串
    s = sdsMakeRoomFor(s,len-curlen);
    if (s == NULL) return NULL;

    /* Make sure added region doesn't contain garbage */
    //确保多余的字符串不包含垃圾数据，置空处理
    sh = (void*)(s-(sizeof(struct sdshdr)));
    memset(s+curlen,0,(len-curlen+1)); /* also set trailing \0 byte */
    totlen = sh->len+sh->free;
    sh->len = len;
    sh->free = totlen-sh->len;
    return s;
}

/* Append the specified binary-safe string pointed by 't' of 'len' bytes to the
 * end of the specified sds string 's'.
 *
 * After the call, the passed sds string is no longer valid and all the
 * references must be substituted with the new pointer returned by the call. */
/* 以t作为新添加的len长度buf的数据，实现追加操作 */
sds sdscatlen(sds s, const void *t, size_t len) {
    struct sdshdr *sh;
    size_t curlen = sdslen(s);
	
	//为原字符串扩展len长度空间
    s = sdsMakeRoomFor(s,len);
    if (s == NULL) return NULL;
    sh = (void*) (s-(sizeof(struct sdshdr)));
    //多余的数据以t作初始化
    memcpy(s+curlen, t, len);
    //更改相应的len,free值
    sh->len = curlen+len;
    sh->free = sh->free-len;
    s[curlen+len] = '\0';
    return s;
}
```







