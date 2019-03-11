# 1 环境选择的注意事项
在linux下，最好选Fedora或者centos系统，可以编译内核后直接下载ndctl和pmdk，yum search能搜索到相关包。

由于本地服务器是ubuntu系统，踏上的寻坑之路……  
ubuntu 版本14.04

## 2 内核编译
内核编译教程<https://www.jianshu.com/p/5e37d91bfbbe>  
编译内核的配置，官方教程<http://pmem.io/2016/02/22/pm-emulation.html>  
内核版本的选择是**非常重要的! 非常重要的! 非常重要的!**  
内核版本阿里云镜像网址：<http://mirrors.aliyun.com/linux-kernel/>  
一定要新的内核版本，有两个要求：  
1. 内核配置make nconfig时`[*]   PFN: Map persistent (device) memory `选项不一定需要，但是`[*] Device memory (pmem, etc...) hotplug support`选项一定必须。
2. 内核里面一定要有`/linux-4.xx/inclue/linux/ndctl.h`文件，在`/linux-4.xx/inclue/linux/uapi/ndctl.h`中的文件没用，说明内核版本过低，这关系到后面编译pmdk。  
注：我尝试过内核版本4.14.22还不够新，不满足条件2。

注：关于配置内存`memmap=nn[KMG]!ss[KMG]`，例如`memmap=8G!4G`表示地址从4G开始，大小为8G的内存用来模拟，起始地址要大于或等于4G，因为前面4G有可能系统占用。

## 3 ndctl安装
由于在ubuntu下`apt-cache search ndctl`没有找到现成的安装包，所以只能采用git clone方式安装，
```
> git clone git@github.com:pmem/ndctl.git
> git checkout v63
```
切换到v63版本，v62、v63、v64版本编译过程都会有错误，v63错误简单一点，安装教程在代码里面有。

除了依赖包缺少错误外，v63 版本会报错误`asciidoctor -I.` 参数错误，这是因为asciidoctor命令格式错误，在`ndctl\Documentation\ndctl\Makefile.am`文件中将`$(ASCIIDOC) ... -I.`改为`$(ASCIIDOC) ... -I .`。

然后`ndctl --version`查看版本。
## 4 pmdk安装
pmdk也采用git clone方式安装，同样切换到稳定版本。

make过程如果出现`<linux/ndctl.h>`文件找不到，说明内核里面还没有配备该头文件，可以去找新的某个内核版本`linux-xx/include/linux/ndctl.h`复制到`usr/include/linux`下，一样可以，但是`linux-xx/include/linux/uapi/ndctl.h`文件不行，这个文件不适用。

