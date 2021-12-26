# CS144-All-in-one

CS144是斯坦福的一门计算机网络课程。全称：CS 144: Introduction to Computer Networking

## 常用网址
课程主页：https://cs144.github.io/ 

课程课件：https://github.com/khanhnamle1994/computer-networking/tree/master/Unit1-Internet-and-IP

## 实验笔记

### lab0 

> [实验讲义](https://cs144.github.io/assignments/lab0.pdf )


#### Writing webget

> 在正式敲代码之前务必仔细阅读实验指导中对C++的要求和说明以及帮助文档[Socket](https://cs144.github.io/doc/lab0/class_socket.html), [TCPSocket](https://cs144.github.io/doc/lab0/class_t_c_p_socket.html), [Address](https://cs144.github.io/doc/lab0/class_address.html)。

本实验的目的就是完成`webget.cc`中的`get_URL`函数，实现对网页数据的获取。（sponge/apps/webget.cc）
可以读下`doctests/socket_example_2.cc`中是如何创建`TCPSocket`以及建立连接，发送并读取数据。也可查看cs144的[Socket帮助文档](https://cs144.github.io/doc/lab0/class_socket.html)。
+ 创建一个`TCPSocket`并与服务器建立连接。
+ 向服务器发送请求，格式参照前面`fetch a web page`部分，注意在`HTTP`中每行的结尾应该为`\r\n`。
+ 发送完请求后，客户端应该关闭`TCPSocket`的写功能，对应前面的`Connection: close`，告诉服务器请求已经发送完毕，服务器只要回复完数据后就可以立刻断开连接。
+ 循环读取从服务器发送过来的信息，直到遇到`EOF (end of file)`。
+ 最后记得需要关闭前面创建的`TCPSocket`。 

最后代码实现如下：

```CPP
void get_URL(const string &host, const string &path) {
    // Your code here.

    // You will need to connect to the "http" service on
    // the computer whose name is in the "host" string,
    // then request the URL path given in the "path" string.

    // Then you'll need to print out everything the server sends back,
    // (not just one call to read() -- everything) until you reach
    // the "eof" (end of file).

    cerr << "Function called: get_URL(" << host << ", " << path << ").\n";
    cerr << "Warning: get_URL() has not been implemented yet.\n";

    // connect to the "http" service
    TCPSocket sock1;
    sock1.connect(Address(host, "http"));

    sock1.write("GET " + path + " HTTP/1.1\r\nHost: " + host + "\r\n\r\n");

    //// Tell the server that it's not going to send any more requests
    sock1.shutdown(SHUT_WR);

    while(!sock1.eof()) {
        cout << sock1.read();
    }

    sock1.close();
    return;
}
```
+ 连接用Socket.connect()
+ 注意请求的结尾要加两个`\r\n`  

实现函数之后，在`build`目录下执行`make`，运行`./apps/webget cs144.keithw.org /hello`。就可以初步检查自己的代码正确性了。

接着可以在`sponge/build`下运行`make check_webget`来验证代码正确性。（CS144的代码评测对蹭课同学还是很友好的，不需要enrollment）。

`4/4 Test #4: lab0_webget ...................... Passed 0.14 sec
100% tests passed, 0 tests failed out of 4 `

出现这个结果就是代码在这个case下正确了。
斯坦福会用不同的主机名和路径来运行我们的webget程序。我们没有斯坦福账号，无法交作业。做到这个程度，其实已经可以了。

#### An in-memory reliable byte stream
简单来说就是实现一个字节流缓冲区，从一端写，另一段读。需要有`pop`和`peek`操作。
函数还定义了一些特殊变量：缓存区的容量、输入端是否写入完毕、以及写入和读取的字节数。
缓冲区应能容量控制，流中的数据达到容量上限时，便无法再写入新的数据。
可以用C++`std::queue`实现。（会很麻烦，因为要peek 前n个字节）
改用`std::deque`实现。

代码实现如下： 

`hh`文件：
~~~CPP
private:
    size_t _capacity = 0;
    std::deque<char> _buf{}; // 这里要写大括号
    bool _end_input = false;
    size_t _bytes_written = 0;
    size_t _bytes_read = 0;

    bool _error{};  //!< Flag indicating that the stream suffered an error.
~~~
`cc`文件：
~~~CPP
ByteStream::ByteStream(const size_t capacity) : _capacity(capacity) {}

size_t ByteStream::write(const string &data) {
    if(remaining_capacity() == 0) return 0;
    size_t len_to_write = min(data.size(), remaining_capacity());

    for(size_t i = 0; i < len_to_write; i++) {
        _buf.push_back(data[i]);
        _bytes_written++;
    }
    return len_to_write;
}

//! \param[in] len bytes will be copied from the output side of the buffer
string ByteStream::peek_output(const size_t len) const {
    size_t len_to_peek = min(_buf.size(), len);
    // 对C++不熟悉，借鉴了大佬的代码
    return string().assign(_buf.begin(), _buf.begin() + len_to_peek);
    
}

//! \param[in] len bytes will be removed from the output side of the buffer
void ByteStream::pop_output(const size_t len) { 
    size_t len_to_pop = min(_buf.size(), len);
    for(size_t i= 0; i < len_to_pop; i++) {
        _buf.pop_front();
        _bytes_read++;
    }
 }

//! Read (i.e., copy and then pop) the next "len" bytes of the stream
//! \param[in] len bytes will be popped and returned
//! \returns a string
std::string ByteStream::read(const size_t len) {
    // 我的写法
    // string res;
    // size_t len_to_pop = min(_buf.size(), len);
    // for(size_t i= 0; i < len_to_pop; i++) {
    //     res += _buf.pop_front();
    //     _bytes_read++;
    // }
    // return res;

    // 大佬的写法
    string res = peek_output(len);
    read(len);
    return res;
}

void ByteStream::end_input() { _end_input = true; }

bool ByteStream::input_ended() const { return _end_input; }

size_t ByteStream::buffer_size() const { return _buf.size(); }

bool ByteStream::buffer_empty() const { return !_buf.size(); }

bool ByteStream::eof() const { return buffer_empty() && input_ended(); }

size_t ByteStream::bytes_written() const { return _bytes_written; }

size_t ByteStream::bytes_read() const { return _bytes_read; }

size_t ByteStream::remaining_capacity() const { return _capacity - buffer_size(); }

~~~
一些看了大佬代码后解决的bug：
+ `std::deque<char> _buf{}; // 这里要写大括号`
+ `ByteStream::ByteStream(const size_t capacity) : _capacity(capacity) {}` 大佬的初始化方式
+  `string().assign(_buf.begin(), _buf.begin() + len_to_peek);` peek deque前len个元素 

在`build`文件夹下执行`make`，最后用`make check_lab0`验证代码正确性。

最后的结果： 

`100% tests passed, 0 tests failed out of 9
Total Test time (real) =   1.58 sec` 

