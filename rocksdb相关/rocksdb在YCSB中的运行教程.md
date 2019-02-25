# 1 介绍
使用YCSB测试rocksdb（自己修改的代码）的性能，运行环境：
* 操作系统：ubuntu 14.04
* rocksdb的版本：5.18.3
* YCSB的版本：0.15.0
# 2 rocksdb的jni包生成
## 2.1 rocksdb的版本代码
通过github获取rocksdb版本5.18.3的代码作为基础：
```
> git clone https://github.com/facebook/rocksdb.git
> cd rocksdb 
> git tag  # 查看发布的版本号，对应分支的标签
> git checkout v5.18.3 # 切换分支，查看当前分支 git branch 查看所有分支 git branch -a
> git checkout -b doycsb v5.18.3 # 创建复制分支v5.18.3为新的分支doycsb，方便修改v5.18.3的代码提交，当然最好的方法是先fork
```
## 2.2 rocksdb的java编译
1. rocksdb的文件夹中java文件夹便是生成jni包，由于我是在linux64位系统中运行，所以无需完成跨平台编译，压缩方式也无需5种(太多了下载过程可能会出错)，只需要snappy压缩即可，修改rocksdb/Makefile:
```
ifneq ($(ROCKSDB_JAVA_NO_COMPRESSION), 1)
# JAVA_COMPRESSIONS = libz.a libbz2.a libsnappy.a liblz4.a libzstd.a
JAVA_COMPRESSIONS = libsnappy.a 
endif

#JAVA_STATIC_FLAGS = -DZLIB -DBZIP2 -DSNAPPY -DLZ4 -DZSTD
JAVA_STATIC_FLAGS = -DSNAPPY 
#JAVA_STATIC_INCLUDES = -I./zlib-$(ZLIB_VER) -I./bzip2-$(BZIP2_VER) -I./snappy-$(SNAPPY_VER) -I./lz4-$(LZ4_VER)/lib -I./zstd-$(ZSTD_VER)/lib/include
JAVA_STATIC_INCLUDES =  -I./snappy-$(SNAPPY_VER)
```
2. 使用make rocksdbjavastaticrelease 编译规则，找到该规则，取消vagrantd的跨平台，无需安装Vagrant和Virtualbox，即在rocksdbjavastaticrelease规则下，注释掉：
```
#cd java/crossbuild && vagrant destroy -f && vagrant up linux32 && vagrant halt linux32 && vagrant up linux64 && vagrant halt linux64
```
3. 在rocksdb文件夹下，执行`make rocksdbjavastaticrelease -j16`,会出现如下错误：
```
cd java/target;jar -uf rocksdbjni-5.18.3.jar librocksdbjni-*.so librocksdbjni-*.jnilib
librocksdbjni-*.jnilib : no such file or directory
make: *** [rocksdbjavastaticrelease] Error 1
```
这是因为我们取消了跨平台java包，而.jnilib是mac os平台下的，所以只需把librocksdbjni-*.jnilib去掉即可：
```
cd java/target;jar -uf $(ROCKSDB_JAR_ALL) librocksdbjni-*.so
```
继续编译`make rocksdbjavastaticrelease -j16`。

4. jni包会生成在`rocksdb/java/target/rocksdbjni-5.18.3.jar`，后面会用到。
# 3 YCSB的编译与运行
## 3.1 YCSB的版本代码获取
```
> git clone https://github.com/brianfrankcooper/YCSB.git
> git checkout 0.15.0
```
## 3.2 YCSB的编译

```
> mvn -pl com.yahoo.ycsb:rocksdb-binding -am clean package
```
这是通过mvn下载rocksdb-binding模块的依赖包，千万不要直接`mvn clean package`，这将会下载所有数据库模块的依赖，耗时非常长。

可以在YCSB/pom.xml文件中看到`<rocksdb.version>5.11.3</rocksdb.version>`,这将会通过mvn在<https://repo.maven.apache.org/maven2/org/rocksdb/rocksdbjni/>中下载对应的5.11.3版本的jni包。如果不想要用自己的rocksdbjni包，可以直接修改对应版本，来获取新的版本的jni包。

## 3.2 YCSB的运行
1. YCSB的执行程序有三个:
* bin/ycsb       # python脚本，对应跨平台
* bin/ycsb.sh # linux脚本，适合linux系统
* bin/ycsb.bat # windows脚本，适合windows系统

我选择bin/ycsb.sh脚本来运行，执行：
```
> bin/ycsb.sh load rocksdb -s -P workloads/workloada -p rocksdb.dir=/home/lzw/ceshi
```
其中rocksdb.dir是数据库路径，运行load 即加载数据，配置是workloads/workloada文件。
执行load命令的前段输出结果：
```
/usr/lib/jvm/jdk1.8.0_181/bin/java  -classpath /home/lzw/ce_ycsb/YCSB/conf:/home/lzw/ce_ycsb/YCSB/core/target/core-0.15.0.jar:/home/lzw/ce_ycsb/YCSB/rocksdb/target/rocksdb-binding-0.15.0.jar:/home/lzw/ce_ycsb/YCSB/rocksdb/target/dependency/jcip-annotations-1.0.jar:/home/lzw/ce_ycsb/YCSB/rocksdb/target/dependency/rocksdbjni-5.11.3.jar:/home/lzw/ce_ycsb/YCSB/rocksdb/target/dependency/slf4j-api-1.7.25.jar:/home/lzw/ce_ycsb/YCSB/rocksdb/target/dependency/slf4j-simple-1.7.25.jar com.yahoo.ycsb.Client -load -db com.yahoo.ycsb.db.rocksdb.RocksDBClient -s -P workloads/workloada -p rocksdb.dir=/home/lzw/ceshi

```
这段运行命令很重要，这是classpath，即jni包，也可查看bin/ycsb.sh脚本的内容，这些jni包决定了运行的结果。

上面load运行结果会出现如下问题：
```
Loading workload...
Exception in thread "main" java.lang.NoClassDefFoundError: org/apache/htrace/core/Tracer$Builder
	at com.yahoo.ycsb.Client.getTracer(Client.java:903)
	at com.yahoo.ycsb.Client.main(Client.java:752)
Caused by: java.lang.ClassNotFoundException: org.apache.htrace.core.Tracer$Builder
	at java.net.URLClassLoader.findClass(URLClassLoader.java:381)
	at java.lang.ClassLoader.loadClass(ClassLoader.java:424)
	at sun.misc.Launcher$AppClassLoader.loadClass(Launcher.java:349)
	at java.lang.ClassLoader.loadClass(ClassLoader.java:357)
	... 2 more
```
这是由于YCSB的core模块的依赖缺少htrace-core4，可以查看`YCSB/core/pom.xml`文件，发现core模块的依赖都没有下载,将`YCSB/core/pom.xml`的内容：
```
<dependency>
      <groupId>org.apache.htrace</groupId>
      <artifactId>htrace-core4</artifactId>
      <version>4.1.0-incubating</version>
</dependency>
```
添加到`YCSB/rocksdb/pom.xml`文件的`<dependencies>`标签下，然后运行：
```
> mvn -pl com.yahoo.ycsb:rocksdb-binding -am clean package
```
可以发现在`YCSB/rocksdb/target/dependency/`目录下新增了htrace-core4-4.1.0-incubating.jar。

继续运行load命令，会出现如下问题：
```
Exception in thread "Thread-3" java.lang.NoClassDefFoundError: org/HdrHistogram/EncodableHistogram
	at com.yahoo.ycsb.measurements.Measurements.constructOneMeasurement(Measurements.java:129)
	at com.yahoo.ycsb.measurements.Measurements.getOpMeasurement(Measurements.java:220)
	at com.yahoo.ycsb.measurements.Measurements.measure(Measurements.java:188)
	at com.yahoo.ycsb.DBWrapper.measure(DBWrapper.java:178)
	at com.yahoo.ycsb.DBWrapper.insert(DBWrapper.java:223)
	at com.yahoo.ycsb.workloads.CoreWorkload.doInsert(CoreWorkload.java:588)
	at com.yahoo.ycsb.ClientThread.run(Client.java:468)
	at java.lang.Thread.run(Thread.java:748)
Caused by: java.lang.ClassNotFoundException: org.HdrHistogram.EncodableHistogram
	at java.net.URLClassLoader.findClass(URLClassLoader.java:381)
	at java.lang.ClassLoader.loadClass(ClassLoader.java:424)
	at sun.misc.Launcher$AppClassLoader.loadClass(Launcher.java:349)
	at java.lang.ClassLoader.loadClass(ClassLoader.java:357)
	... 8 more

```
这个问题和上面的问题一样，缺少core模块的依赖HdrHistogram,将`YCSB/core/pom.xml`的内容：
```
<dependency>
      <groupId>org.hdrhistogram</groupId>
      <artifactId>HdrHistogram</artifactId>
      <version>2.1.4</version>
</dependency>
```
添加到`YCSB/rocksdb/pom.xml`文件的`<dependencies>`标签下，然后运行：
```
> mvn -pl com.yahoo.ycsb:rocksdb-binding -am clean package
```
可以发现在`YCSB/rocksdb/target/dependency/`目录下新增了HdrHistogram-2.1.4.jar。

运行load命令：
```
> bin/ycsb.sh load rocksdb -s -P workloads/workloada -p rocksdb.dir=/home/lzw/ceshi
```
终于可以正常运行了，在数据库路径下的LOG文件开头可以看到`RocksDB version: 5.11.3`，代表运行的是rocksdb v5.11.3。

2. 将上一步生成的 `rocksdb/java/target/rocksdbjni-5.18.3.jar`复制到`YCSB/rocksdb/target/dependency/`目录下，然后把`rocksdbjni-5.11.3.jar `删除。

然后删除数据库路径下的文件，即删除原有的数据库。

运行load命令：
```
> bin/ycsb.sh load rocksdb -s -P workloads/workloada -p rocksdb.dir=/home/lzw/ceshi
```
可以在数据库路径下的LOG文件开头可以看到`RocksDB version: 5.18.3`，代表运行的是rocksdb v5.18.3。

3. 运行run命令：
```
> bin/ycsb.sh run rocksdb -s -P workloads/workloada -p rocksdb.dir=/home/lzw/ceshi
```
# 4 YCSB下的rocksdb参数设置
1. 最简单的配置是`workloads/workloada`类似文件的配置，运行load的命令是简单地将`recordcount=1000`的条数插入数据库，而插入数据库的数据大小则是:
```
fieldcount=10 (默认)
fieldlength=100 (默认)
```
fieldcount表示一条记录有多少段，fieldlength表示一段的字节数，插入的一条数据value约等于fieldcount*fieldlength。

2. YCSB中连接rocksdb客户端的文件：

`YCSB\rocksdb\src\main\java\com\yahoo\ycsb\db\rocksdb\RocksDBClient.java`

YCSB中测试rocksdb的文件：

`YCSB\rocksdb\src\test\java\com\yahoo\ycsb\db\rocksdb\RocksDBClientTest.java`

在`RocksDBClient.java`文件中：
```
位于initRocksDB()函数中：
final Options options = new Options()
    .optimizeLevelStyleCompaction()
    .setCreateIfMissing(true)
    .setCreateMissingColumnFamilies(true)
    .setIncreaseParallelism(rocksThreads)
    .setMaxBackgroundCompactions(rocksThreads)
    .setInfoLogLevel(InfoLogLevel.INFO_LEVEL);

final DBOptions options = new DBOptions()
    .setCreateIfMissing(true)
    .setCreateMissingColumnFamilies(true)
    .setIncreaseParallelism(rocksThreads)
    .setMaxBackgroundCompactions(rocksThreads)
    .setInfoLogLevel(InfoLogLevel.INFO_LEVEL);

位于createColumnFamily()函数中：
final ColumnFamilyOptions cfOptions = new ColumnFamilyOptions().optimizeLevelStyleCompaction();
```
这两处都会对rocksdb的option产生影响，特别是rocksdb中的optimizeLevelStyleCompaction()函数，优化了多个参数。rocksdb中的参数结果会保存在数据库路径下的OPTIONS文件下。

如果修改了`RocksDBClient.java`文件的内容，可以再次编译YCSB：
```
> mvn -pl com.yahoo.ycsb:rocksdb-binding -am package
```
去掉clean，则不会将原来的`rocksdb/java/target/rocksdbjni-5.18.3.jar`删除。只会重新下载package，将新下载的`rocksdb/java/target/rocksdbjni-5.11.3.jar`删除即可。