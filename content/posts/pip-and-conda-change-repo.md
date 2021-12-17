---
title: "Pip and Conda Change Repo"
date: 2021-12-17T17:07:31+08:00
draft: true
---
# Pip Change Repository
## Config
Use `pip config` to configure PyPi repository on user or global level:
```
pip config [--user] set global.index https://my-company/nexus/repository/pypi-group/pypi
pip config [--user] set global.index-url https://my-company/nexus/repository/pypi-group/simple
pip config [--user] set global.trusted-host my-company
```
where `index-url` is used by `pip install`, and `index` is used by `pip search`.

Use `pip config -v list` to find the path of the configuration file.



## Famous Mirror Repository
```
aliyun's mirror: https://mirrors.aliyun.com/pypi/simple/
```



## Windows
use `pip config` above or create a `../pip/pip.ini` as following:
```
[global]
index-url = http://mirrors.aliyun.com/pypi/simple/
proxy=http://xx.xx:port
[install]
trusted-host=mirrors.aliyun.com
no-cache-dir=true
```
**Note**: you can also replace `http` protocol with `https`.



## Linux
use `pip config` above or create a `~/.pip/pip.conf` just like `pip.ini` in windows.



# Conda Change Repository
use `conda config` to configure channel:
```
conda config --set show_channel_urls yes
conda config --add channels [new_channel]
```



you can also create a `~/.condarc` and alter it as follows:
```
channels:
  - defaults
show_channel_urls: true
default_channels:
  - http://mirrors.aliyun.com/anaconda/pkgs/main
  - http://mirrors.aliyun.com/anaconda/pkgs/r
  - http://mirrors.aliyun.com/anaconda/pkgs/msys2
custom_channels:
  conda-forge: http://mirrors.aliyun.com/anaconda/cloud
  msys2: http://mirrors.aliyun.com/anaconda/cloud
  bioconda: http://mirrors.aliyun.com/anaconda/cloud
  menpo: http://mirrors.aliyun.com/anaconda/cloud
  pytorch: http://mirrors.aliyun.com/anaconda/cloud
  simpleitk: http://mirrors.aliyun.com/anaconda/cloud
``` 
which is from [aliyun's anaconda mirror](https://developer.aliyun.com/mirror/anaconda).





