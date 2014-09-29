# Muduo网络库中Buffer类的设计实现

本文主要参考了[陈硕的博客](http://blog.csdn.net/solstice/article/details/6329080)以及[GitHub上的代码](https://github.com/chenshuo/muduo/blob/master/muduo/net/Buffer.h)。

## Buffer的数据结构

Buffer类的成员变量有vector of char（连续内存容器），readerIndex_，writerIndex_（指向vector中的元素，应对迭代器失效），vector的内存分配如下：

	+-------------------+------------------+------------------+
	| prependable bytes |  readable bytes  |  writable bytes  |
	|                   |     (CONTENT)    |                  |
	+-------------------+------------------+------------------+
	|                   |                  |                  |
	0      <=      readerIndex   <=   writerIndex    <=     size

### prependable、readable、writeable大小的计算

prependable、readable、writeable bytes大小计算公式如下：

prependable bytes = readerIndex

readable bytes    = writerIndex - readerIndex

writeable bytes   = size() - writerIndex

其中在代码中的实现如下：


``` c

size\_t readableBytes() const
{ return writerIndex_ - readerIndex_; }

size\_t writableBytes() const
{ return buffer_.size() - writerIndex_; }

size\_t prependableBytes() const
{ return readerIndex_; }

```

### Buffer的初始化

在Buffer的构造函数中实现了Buffer数据结构的初始化，其定义了两个常量kCheapPrepend, kInitialSize,分别表示prependable、writeable的初始化大小，初始化的数据结构如下所示：

	+-------------------+-------------------------------------+
	| kCheapPrepend(8)  |           kInitialSize(1024)        |
	+-------------------+-------------------------------------+
	|                   |                                     |
	0              readerIndex                              size
				   writerIndex

代码实现如下：

``` c

explicit Buffer(size_t initialSize = kInitialSize)
    : buffer_(kCheapPrepend + initialSize),
      readerIndex_(kCheapPrepend),
      writerIndex_(kCheapPrepend)
  {
    assert(readableBytes() == 0);
    assert(writableBytes() == initialSize);
    assert(prependableBytes() == kCheapPrepend);
  }


```

### read write的操作

初始化后有人向Buffer写入了200字节，那么Buffer的布局如下(writerIndex向后移动了200字节，readerIndex保持不变)：

	+-------------------+------------------+------------------+
	| prependable bytes |  readable bytes  |  writable bytes  |
	|                   |        (200)     |                  |
	+-------------------+------------------+------------------+
	|                   |                  |                  |
	0              readerIndex(8)     writerIndex(208)       size

代码实现如下：

``` c
  
  void append(const StringPiece& str)
  {
    append(str.data(), str.size());
  }

  void append(const char* /*restrict*/ data, size_t len)
  {
    ensureWritableBytes(len);
    std::copy(data, data+len, beginWrite());
    hasWritten(len);
  }

  void append(const void* /*restrict*/ data, size_t len)
  {
    append(static_cast<const char*>(data), len);
  }

  void ensureWritableBytes(size_t len)
  {
    if (writableBytes() < len)
    {
	  // 内部腾挪
      makeSpace(len);
    }
    assert(writableBytes() >= len);
  }

  char* beginWrite()
  { return begin() + writerIndex_; }

  const char* beginWrite() const
  { return begin() + writerIndex_; }

  void hasWritten(size_t len)
  {
    assert(len <= writableBytes());
    writerIndex_ += len;
  }

```

之后有人从Buffer读取了50字节，则Buffer的布局如下（writerIndex不变，readerIndex向后移动50字节）：

	+-------------------+------------------+------------------+
	| prependable bytes |  readable bytes  |  writable bytes  |
	|                   |        (150)     |                  |
	+-------------------+------------------+------------------+
	|                   |                  |                  |
	0             readerIndex(58)     writerIndex(208)       size

代码实现如下：

``` c
  
  void retrieve(size_t len)
  {
    assert(len <= readableBytes());
    if (len < readableBytes())
    {
      readerIndex_ += len;
    }
    else
    {
      retrieveAll();
    }
  }

  void retrieveUntil(const char* end)
  {
    assert(peek() <= end);
    assert(end <= beginWrite());
    retrieve(end - peek());
  }

  void retrieveAll()
  {
    readerIndex_ = kCheapPrepend;
    writerIndex_ = kCheapPrepend;
  }
	
```

### Buffer的内部腾挪

有时经过多次的读写readerIndex就会移到了比较靠后的位置prependable的空间就比较大，例如下面这种情况：

	+-------------------+------------------+------------------+
	| prependable bytes |  readable bytes  |  writable bytes  |
	|      (500)        |        (300)     |      （232)		  |
	+-------------------+------------------+------------------+
	|                   |                  |                  |
	0            readerIndex(500)     writerIndex(800)       size

若要往Buffer中写入300字节的内容，那该怎么办？Muduo在这种情况下不做内存的重新分配，而是将数据移动前面去，出现了如下的结构：

	+-------------------+------------------+------------------+
	| prependable bytes |  readable bytes  |  writable bytes  |
	|      (8)          |        (300)     |      （724)		  |
	+-------------------+------------------+------------------+
	|                   |                  |                  |
	0            readerIndex(8)     writerIndex(308)       size

这样就可以往writeable空间里写入数据了，其代码实现如下：

``` c
 void makeSpace(size_t len)
  {
    if (writableBytes() + prependableBytes() < len + kCheapPrepend)
    {
      // FIXME: move readable data
      buffer_.resize(writerIndex_+len);
    }
    else
    {
      // move readable data to the front, make space inside buffer
      assert(kCheapPrepend < readerIndex_);
      size_t readable = readableBytes();
      std::copy(begin()+readerIndex_,
                begin()+writerIndex_,
                begin()+kCheapPrepend);
      readerIndex_ = kCheapPrepend;
      writerIndex_ = readerIndex_ + readable;
      assert(readable == readableBytes());
    }
  }	
```

### Prepend操作

[Muduo 网络编程示例之二：Boost.Asio 的聊天服务器](http://blog.csdn.net/Solstice/article/details/6172391)在示范消息打包的过程中有如下的代码： buf.prepend(&len, sizeof len);该段代码用于在数据前添加几个字节数据，例如上面是将消息的长度加入报头，其实现的代码如下：

``` c
  void prepend(const void* /*restrict*/ data, size_t len)
  {
    assert(len <= prependableBytes());
    readerIndex_ -= len;
    const char* d = static_cast<const char*>(data);
    std::copy(d, d+len, begin()+readerIndex_);
  } 
```

## readFd()函数

陈硕大神说：在非阻塞网络编程中使用缓冲区时，一方面希望准备一个大的缓冲区(一次读到的数据越多越好)，当然如果有很多的连接，每个连接都使用较大的缓冲区时，缓冲区的利用效率会非常低，Muduo中采用了一个较为巧妙的做法：

iov数组中有两个元素，一个是栈上的stackbuf，一个是Buffer类中的writeable字节，如果读入的数据不多，则读到Buffer的writeable字节中，如果读入的字节超过Buffer的writeable字节数，则会存到stackbuf去，之后再将stackbuf里的数据append到Buffer中，其实现的代码如下所示：

``` c
ssize_t Buffer::readFd(int fd, int* savedErrno)
{
  // saved an ioctl()/FIONREAD call to tell how much to read
  char extrabuf[65536];
  struct iovec vec[2];
  const size_t writable = writableBytes();
  vec[0].iov_base = begin()+writerIndex_;
  vec[0].iov_len = writable;
  vec[1].iov_base = extrabuf;
  vec[1].iov_len = sizeof extrabuf;
  // when there is enough space in this buffer, don't read into extrabuf.
  // when extrabuf is used, we read 128k-1 bytes at most.
  const int iovcnt = (writable < sizeof extrabuf) ? 2 : 1;
  const ssize_t n = sockets::readv(fd, vec, iovcnt);
  if (n < 0)
  {
    *savedErrno = errno;
  }
  else if (implicit_cast<size_t>(n) <= writable)
  {
    writerIndex_ += n;
  }
  else
  {
    writerIndex_ = buffer_.size();
    append(extrabuf, n - writable);
  }
  // if (n == writable + sizeof extrabuf)
  // {
  //   goto line_30;
  // }
  return n;
}

```

## Buffer自动增长

利用vector特性可以实现Buffer长度的自动增长，然后有时候并不需要这么大的缓冲区长度，在应用代码中可以调用shrink()函数实现缓冲区的减小。其实现代码如下：


``` c
  void shrink(size_t reserve)
  {
    // FIXME: use vector::shrink_to_fit() in C++ 11 if possible.
    Buffer other;
    other.ensureWritableBytes(readableBytes()+reserve);
    other.append(toStringPiece());
    swap(other);
  }

  void swap(Buffer& rhs)
  {
    buffer_.swap(rhs.buffer_);
    std::swap(readerIndex_, rhs.readerIndex_);
    std::swap(writerIndex_, rhs.writerIndex_);
  }

```