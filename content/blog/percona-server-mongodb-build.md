+++
title = "Percona Server for MongoDB 源码编译"
description = ""
date = 2022-01-19
+++

适用版本：

* Percona Server for MongoDB v5.0.3
* Percona Server for MongoDB v3.6.23

编译环境：
```
❯ cat /etc/os-release 
NAME="CentOS Linux"
VERSION="7 (Core)"
ID="centos"
ID_LIKE="rhel fedora"
VERSION_ID="7"
PRETTY_NAME="CentOS Linux 7 (Core)"
ANSI_COLOR="0;31"
CPE_NAME="cpe:/o:centos:centos:7"
HOME_URL="https://www.centos.org/"
BUG_REPORT_URL="https://bugs.centos.org/"

CENTOS_MANTISBT_PROJECT="CentOS-7"
CENTOS_MANTISBT_PROJECT_VERSION="7"
REDHAT_SUPPORT_PRODUCT="centos"
REDHAT_SUPPORT_PRODUCT_VERSION="7"
```

## 环境准备


### Clone 代码

```
❯ git clone https://github.com/percona/percona-server-mongodb.git
...
❯ cd percona-server-mongodb && git checkout release-5.0.3-2
```

### 安装依赖
```
❯ sudo yum -y install centos-release-scl epel-release 
...
❯ sudo yum -y install python3 python3-devel scons gcc gcc-c++ cmake3 openssl-devel \
cyrus-sasl-devel snappy-devel zlib-devel bzip2-devel libcurl-devel lz4-devel \
openldap-devel krb5-devel xz-devel e2fsprogs-devel expat-devel devtoolset-8-gcc \
devtoolset-8-gcc-c++
...
```


`v5.x.x` 和 `v3.x.x` 依赖不同版本的Python来编译；

对于`v5.0.3`，官方指南指出需要`Python 3.7+`，我的环境中只有`Python 3.6.8`，也可以编译成功。

```
❯ python3 -V
Python 3.6.8

❯ python3 -m pip -V
pip 9.0.3 from /usr/lib/python3.6/site-packages (python 3.6)

```

安装好对应Python后，接着安装编译、测试需要的三方库

```
❯ python3 -m pip install -r etc/pip/compile-requirements.txt
```


对于`v3.6.23`，需要`Python 2.7+`，但是我在CentOS上的pip总在安装依赖库时报错，上网查了说是版本兼容性问题，建议直接安装`Miniconda`。

```
❯ wget https://repo.anaconda.com/miniconda/Miniconda2-py27_4.8.3-Linux-x86_64.sh
...
❯ bash Miniconda2-py27_4.8.3-Linux-x86_64.sh
...


❯ python2 -V
Python 2.7.18 :: Anaconda, Inc.

❯ python2 -m pip -V
pip 19.3.1 from /home/jojo/miniconda2/lib/python2.7/site-packages/pip (python 2.7)
```

安装好对应Python后，接着安装编译、测试需要的三方库

```
❯ python2.7 -m pip install -r buildscripts/requirements.txt
```

### 编译 curl-7.66.0
```
❯ wget https://curl.se/download/curl-7.66.0.tar.gz
❯ tar -xvzf curl-7.66.0.tar.gz && cd curl-7.66.0
❯ ./configure
...
❯ sudo make install
...
```


### 编译 aws-sdk-cpp

```
❯ mkdir -p /tmp/lib/aws
❯ export AWS_LIBS=/tmp/lib/aws
❯ git clone https://github.com/aws/aws-sdk-cpp.git
...
❯ cd aws-sdk-cpp && git checkout 1.8.56
❯ mkdir build && cd build
❯ cmake .. -DCMAKE_BUILD_TYPE=Release -DBUILD_ONLY="s3;transfer" -DBUILD_SHARED_LIBS=OFF \
-DMINIMIZE_SIZE=ON -DCMAKE_INSTALL_PREFIX="${AWS_LIBS}"
...
❯ make install
...

```

## 编译 Percona Server for MongoDB


`v5.0.3`

```
❯ python3 buildscripts/scons.py CC=/opt/rh/devtoolset-8/root/usr/bin/gcc \
CXX=/opt/rh/devtoolset-8/root/usr/bin/g++ -j16 --jlink=2  --disable-warnings-as-errors \
--ssl --opt=on --use-sasl-client --wiredtiger --audit --inmemory --hotbackup \
CPPPATH="${AWS_LIBS}/include" LIBPATH="${AWS_LIBS}/lib64" MONGO_VERSION=5.0.3 install-all
...
```
二进制文件输出在 `build/install/bin/`


`v3.6.23`
```
❯ python2 buildscripts/scons.py CC=/opt/rh/devtoolset-8/root/usr/bin/gcc \
CXX=/opt/rh/devtoolset-8/root/usr/bin/g++ -j16 --disable-warnings-as-errors --ssl \
--opt=on --use-sasl-client --wiredtiger --audit --inmemory --hotbackup \
CPPPATH="${AWS_LIBS}/include" LIBPATH="${AWS_LIBS}/lib64" MONGO_VERSION=3.6.23 all
...
```

二进制文件输出在 `build/opt/mongo/`


## 注意事项

### 动态库兼容问题

在编译准备步骤中我们安装了 `devtoolset-8-gcc/devtoolset-8-gcc-c++`，在它的安装目录下有一个`libstdc++.so`。

``` 
❯ pwd
/opt/rh/devtoolset-8/root/usr/lib/gcc/x86_64-redhat-linux/8

❯ ll
total 13708
...
-rw-r--r-- 1 root root 4887654 Mar 27  2020 libstdc++.a
...
-rw-r--r-- 1 root root 1928418 Mar 27  2020 libstdc++_nonshared.a
-rw-r--r-- 1 root root     210 Mar 27  2020 libstdc++.so
...

❯ cat libstdc++.so
/* GNU ld script
   Use the shared library, but some functions are only in
   the static library, so try that secondarily.  */
OUTPUT_FORMAT(elf64-x86-64)
INPUT ( /usr/lib64/libstdc++.so.6 -lstdc++_nonshared )
```

看文件大小就知道不可能是标准库，它是一个`链接脚本`，结合注释（使用共享库，但有些函数仅在静态库中，因此可以从第二个位置尝试。）与脚本内容可以猜出，
在链接`libstdc++.so`的时候，链接器会优先使用`/usr/lib64/libstdc++.so.6`进行链接，遇到了该库中不存在的函数（比如标准库新增的函数），就尝试使用静态库`libstdc++_nonshared.a`进行链接。所以我们编译出来的程序会依赖共享库`/usr/lib64/libstdc++.so.6`，需要确保这个共享库的版本与部署环境兼容。

```
❯ ldd mongod
        linux-vdso.so.1 =>  (0x00007ffc6cfbe000)
        libcurl.so.4 => /lib64/libcurl.so.4 (0x00007f98bd95e000)
        libgssapi_krb5.so.2 => /lib64/libgssapi_krb5.so.2 (0x00007f98bd711000)
        libsasl2.so.3 => /lib64/libsasl2.so.3 (0x00007f98bd4f4000)
        libldap-2.4.so.2 => /lib64/libldap-2.4.so.2 (0x00007f98bd29f000)
        liblber-2.4.so.2 => /lib64/liblber-2.4.so.2 (0x00007f98bd090000)
        libresolv.so.2 => /lib64/libresolv.so.2 (0x00007f98bce77000)
        libcrypto.so.10 => /lib64/libcrypto.so.10 (0x00007f98bca14000)
        libssl.so.10 => /lib64/libssl.so.10 (0x00007f98bc7a2000)
        libdl.so.2 => /lib64/libdl.so.2 (0x00007f98bc59e000)
        librt.so.1 => /lib64/librt.so.1 (0x00007f98bc396000)
        libstdc++.so.6 => /lib64/libstdc++.so.6 (0x00007f98bc08f000) # ！这里
        libm.so.6 => /lib64/libm.so.6 (0x00007f98bbd8d000)
        libgcc_s.so.1 => /lib64/libgcc_s.so.1 (0x00007f98bbb77000)
        libpthread.so.0 => /lib64/libpthread.so.0 (0x00007f98bb95b000)
        libc.so.6 => /lib64/libc.so.6 (0x00007f98bb58e000)
        /lib64/ld-linux-x86-64.so.2 (0x00007f98c10a7000)
        libidn.so.11 => /lib64/libidn.so.11 (0x00007f98bb35b000)
        libssh2.so.1 => /lib64/libssh2.so.1 (0x00007f98bb12e000)
        libssl3.so => /lib64/libssl3.so (0x00007f98baedc000)
        libsmime3.so => /lib64/libsmime3.so (0x00007f98bacb5000)
        libnss3.so => /lib64/libnss3.so (0x00007f98ba988000)
        libnssutil3.so => /lib64/libnssutil3.so (0x00007f98ba759000)
        libplds4.so => /lib64/libplds4.so (0x00007f98ba555000)
        libplc4.so => /lib64/libplc4.so (0x00007f98ba350000)
        libnspr4.so => /lib64/libnspr4.so (0x00007f98ba112000)
        libkrb5.so.3 => /lib64/libkrb5.so.3 (0x00007f98b9e29000)
        libk5crypto.so.3 => /lib64/libk5crypto.so.3 (0x00007f98b9bf6000)
        libcom_err.so.2 => /lib64/libcom_err.so.2 (0x00007f98b99f2000)
        libz.so.1 => /lib64/libz.so.1 (0x00007f98b97dc000)
        libkrb5support.so.0 => /lib64/libkrb5support.so.0 (0x00007f98b95cc000)
        libkeyutils.so.1 => /lib64/libkeyutils.so.1 (0x00007f98b93c8000)
        libcrypt.so.1 => /lib64/libcrypt.so.1 (0x00007f98b9191000)
        libselinux.so.1 => /lib64/libselinux.so.1 (0x00007f98b8f6a000)
        libfreebl3.so => /lib64/libfreebl3.so (0x00007f98b8d67000)
        libpcre.so.1 => /lib64/libpcre.so.1 (0x00007f98b8b05000)
```

或者，通过链接静态库消除动态库依赖的影响：编译前把上面提到的链接脚本`libstdc++.so`移除（或者改个名字），链接器会使用该路径下的静态库`libstdc++.a`。

```
❯ ldd mongod
        linux-vdso.so.1 =>  (0x00007ffd9a8bd000)
        libcurl.so.4 => /lib64/libcurl.so.4 (0x00007f5ace674000)
        libgssapi_krb5.so.2 => /lib64/libgssapi_krb5.so.2 (0x00007f5ace427000)
        libsasl2.so.3 => /lib64/libsasl2.so.3 (0x00007f5ace20a000)
        libldap-2.4.so.2 => /lib64/libldap-2.4.so.2 (0x00007f5acdfb5000)
        liblber-2.4.so.2 => /lib64/liblber-2.4.so.2 (0x00007f5acdda6000)
        libresolv.so.2 => /lib64/libresolv.so.2 (0x00007f5acdb8d000)
        libcrypto.so.10 => /lib64/libcrypto.so.10 (0x00007f5acd72a000)
        libssl.so.10 => /lib64/libssl.so.10 (0x00007f5acd4b8000)
        libdl.so.2 => /lib64/libdl.so.2 (0x00007f5acd2b4000)
        librt.so.1 => /lib64/librt.so.1 (0x00007f5acd0ac000)
        libm.so.6 => /lib64/libm.so.6 (0x00007f5accdaa000)
        libgcc_s.so.1 => /lib64/libgcc_s.so.1 (0x00007f5accb94000)
        libpthread.so.0 => /lib64/libpthread.so.0 (0x00007f5acc978000)
        libc.so.6 => /lib64/libc.so.6 (0x00007f5acc5ab000)
        /lib64/ld-linux-x86-64.so.2 (0x00007f5ad1ef8000)
        libidn.so.11 => /lib64/libidn.so.11 (0x00007f5acc378000)
        libssh2.so.1 => /lib64/libssh2.so.1 (0x00007f5acc14b000)
        libssl3.so => /lib64/libssl3.so (0x00007f5acbef9000)
        libsmime3.so => /lib64/libsmime3.so (0x00007f5acbcd2000)
        libnss3.so => /lib64/libnss3.so (0x00007f5acb9a5000)
        libnssutil3.so => /lib64/libnssutil3.so (0x00007f5acb776000)
        libplds4.so => /lib64/libplds4.so (0x00007f5acb572000)
        libplc4.so => /lib64/libplc4.so (0x00007f5acb36d000)
        libnspr4.so => /lib64/libnspr4.so (0x00007f5acb12f000)
        libkrb5.so.3 => /lib64/libkrb5.so.3 (0x00007f5acae46000)
        libk5crypto.so.3 => /lib64/libk5crypto.so.3 (0x00007f5acac13000)
        libcom_err.so.2 => /lib64/libcom_err.so.2 (0x00007f5acaa0f000)
        libz.so.1 => /lib64/libz.so.1 (0x00007f5aca7f9000)
        libkrb5support.so.0 => /lib64/libkrb5support.so.0 (0x00007f5aca5e9000)
        libkeyutils.so.1 => /lib64/libkeyutils.so.1 (0x00007f5aca3e5000)
        libcrypt.so.1 => /lib64/libcrypt.so.1 (0x00007f5aca1ae000)
        libselinux.so.1 => /lib64/libselinux.so.1 (0x00007f5ac9f87000)
        libfreebl3.so => /lib64/libfreebl3.so (0x00007f5ac9d84000)
        libpcre.so.1 => /lib64/libpcre.so.1 (0x00007f5ac9b22000)
```

## 参考
[Percona Server for MongoDB 贡献指南](https://github.com/percona/percona-server-mongodb/blob/master/CONTRIBUTING.rst)

