# 项目概述
### 项目文件结构
``` DIR
  
  /build  //cmake生成中间文件 编译项目目录

  /LibSoure //底层库源码

  /lib   //底层支持库 编译第三库的文件应该放到此处

  /src   //项目源文件+头文件目录
  -  /app           //应用层
  -  /common        //公共方法文件(通用方法)
  -  /log           //日志模块
  -  /thirdparty    //第三方库文件头文件

  CMakeLists.txt  

  LICENSE

  README.md
```
## 第三方库交叉编译
#### 介绍
+ 底层串口采用[CSerialPort](https://github.com/itas109/CSerialPort)串口收发处理 
CSerialPort是一个基于C/C++的轻量级开源跨平台串口类库，可以轻松实现跨平台多操作系统的串口读写，同时还支持C#, Java, Python, Node.js等。
接口全部使用回调函数
+ 底层网络采用[HP-Socket](https://github.com/ldcsaa/HP-Socket)网络处理
HPSocket 是一个小型高性能网络处理框架底层采用epoll作为异步模型可以自动拆包解包处理
#### 交叉编译第三方库
+ 默认工作路径 /home/forlinx/GKZD 位置不同自行修改
***安装依赖***
```
sudo apt-get install g++-aarch64-linux-gnu
sudo apt-get install gcc-aarch64-linux-gnu
sudo apt-get install libssl-dev
sudo apt-get install doxygen graphviz
```
## ***HP-Socket交叉编译***
```console
$ cd HP-Socket/Linux/script
$ sudo chmod 777 *.sh
//注意 要注释掉/include/hpsocket/GlobalDef.h   70行的typedef LLONG  __time64_t;
$ ./compile.sh -c aarch64-linux-g++ -p arm64

//编译生成的文件在 HP-Socket/Linux/lib/hpsocket/arm64/ 
将libhpsocket.so   libhpsocket.so.5  libhpsocket.so.5.9.1 复制到
/GKZD/lib 文件夹下
```
## ***CSerialPort交叉编译***
```console
$ cd CSerialPort
$ mkdir arm_build && cd arm_build
$ cmake .. -DCMAKE_TOOLCHAIN_FILE=./cmake/toolchain_aarch64.cmake
$ cmake --build .
//编译生成的文件在 CSerialPort/arm_build/lib/
将libcserialport.so 放到/GKZD/lib 文件夹下
```

## ***MQTT交叉编译***
```console
openssl 交叉编译
$ tar -xvf openssl-1.1.1u.tar.gz
$ mkdir openssl_aarch64
$ ./config no-asm shared --prefix=/ssl/openssl-1.1.1u/openssl_aarch64 --cross-compile-prefix=aarch64-linux-gnu-
修改 Makefile 文件，将 -m64 移除，否则会出现编译报错
$ make
$ make  install
//编译生成的文件在 /openssl_aarch64
```

```console
mqttc交叉编译
$ 修改创建 toolchain.linux-aarch64.cmake 模仿同文件夹下arm11 就可以
$ mkdir arm_build && cd arm_build
$ cmake  ..  -DPAHO_WITH_SSL=TRUE -DPAHO_BUILD_SAMPLES=TRUE \
    -DPAHO_BUILD_DOCUMENTATION=TRUE \
    -DCMAKE_INSTALL_PREFIX=./install  \
    -DOPENSSL_ROOT_DIR="/home/forlinx/GKZD/LibSoure/openssl-1.1.1u/openssl_aarch64/" \
    -DCMAKE_TOOLCHAIN_FILE=/home/forlinx/GKZD/LibSoure/paho.mqtt.c/cmake/toolchain.linux-aarch64.cmake
$ make && make install  生成的文件在 arm_build/install 下面
将 arm_build/install/lib 下libpaho-mqtt3as.so.1 libpaho-mqtt3as.so libpaho-mqtt3as.so.1.3.12复制到 /GKZD/lib 文件夹下
```

```console
mqttcpp交叉编译
$ mkdir arm_build && cd arm_build
$  cmake .. \
  -DCMAKE_CXX_COMPILER=aarch64-linux-gnu-g++ \
  -DCMAKE_INSTALL_PREFIX=./install \
  -DPAHO_MQTT_C_LIBRARIES=/home/forlinx/GKZD/LibSoure/paho.mqtt.c/arm_build/install/lib/libpaho-mqtt3a.so \
  -DPAHO_MQTT_C_INCLUDE_DIRS=/home/forlinx/GKZD/LibSoure/paho.mqtt.c/arm_build/install/include/ \
  -DOPENSSL_SSL_LIBRARY=/home/forlinx/GKZD/LibSoure/openssl-1.1.1u/openssl_aarch64/lib/libssl.so \
  -DOPENSSL_INCLUDE_DIR=/home/forlinx/GKZD/LibSoure/openssl-1.1.1u/openssl_aarch64/include \
  -DOPENSSL_CRYPTO_LIBRARY=/home/forlinx/GKZD/LibSoure/openssl-1.1.1u/openssl_aarch64/lib/libcrypto.so
  make && make install 生成的文件在 arm_build/install 下面
  将arm_build/install/lib 下libpaho-mqttpp3.so  libpaho-mqttpp3.so.1  libpaho-mqttpp3.so.1.2.0 复制到 /GKZD/lib 文件夹下
```
### 项目构建

使用**CMake**构建本项目.
当您在项目的根路径时, 可以执行以下指令:

```console
// BUILD
$ 将生成的第三方动态库文件放在 lib文件夹内
$ mkdir build
$ cd build
$ cmake .. // 默认为有日志
$ make
最后将在 bin 目录生成默认应用程序
```
### 项目部署
+ 将工作路径下 lib所有文件复制到OK3568 /usr/lib/ 下

## 项目架构
### 线程池模块 
线程池可以通过 提交submit 提交任务并执行
``` cpp
template<typename F, typename...Args>
auto submit(F&& f, Args&&... args) -> std::future<decltype(f(args...))> {
  std::function<decltype(f(args...))()> func = std::bind(std::forward<F>(f), std::forward<Args>(args)...);
  auto task_ptr = std::make_shared<std::packaged_task<decltype(f(args...))()>>(func);

  std::function<void()> wrapper_func = [task_ptr]() {
    (*task_ptr)(); 
  };

  m_queue.enqueue(wrapper_func);

  m_conditional_lock.notify_one();

  return task_ptr->get_future();
}
```
### 日志模块 
日志模块采用单例模式加锁的方式并提供三个宏定义并有三个级别的日志记录:
+ `LOG_INFO`
+ `LOG_WARNING`
+ `LOG_ERROR`
### 网络管理模块(NetWorkManager)
提供公用的网络收发处理
```CPP
//内部通过监听器接口处理数据
class NetWorkManager
{
private:
	/* data */
public:
	NetWorkManager(/* args */);
	~NetWorkManager();
private:
//内部创建 监听接口
	CListenerImpl *listener;
    IUdpClient    *client;
};
//通过监听器接口处理数据
EnHandleResult CListenerImpl::OnReceive(IUdpClient* pSender, CONNID dwConnID, const BYTE* pData, int iLength)
{  
    //这里对 pData做数据处理或者转发
    return HR_OK;
}
```
### 串口管理模块(SerialManager)
维护一个串口监听器 可以通过
```CPP
class SerialManager
{
private:
    /* data */
public:
    SerialManager(/* args */);
    ~SerialManager();
private:
    SerialListener * serialListener;
};

```










