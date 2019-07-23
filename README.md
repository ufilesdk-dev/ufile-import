# UFile迁移工具使用说明
<!-- TOC --> 
 - [介绍](#介绍)    
      - [适用情况](#适用情况)        
      - [准备工作](#准备工作)              
      - [名词定义](#名词定义)    
      - [配置文件说明](#配置文件说明)    
 - [安装](#安装)        
      -  [通用步骤](#通用步骤) 
 - [使用方法](#使用方法)    
      - [从阿里云oss迁移到UFile对象存储](#从阿里云oss迁移到UFile对象存储)
      - [从七牛云Kodo迁移到UFile对象存储](#从七牛云Kodo迁移到UFile对象存储)
      - [UFile对象存储不同bucket之前的数据迁移](#UFile对象存储不同bucket之前的数据迁移)
      - [支持s3协议的对象存储迁移到UFile对象存储](#支持s3协议的对象存储迁移到UFile对象存储)
      - [分布式方式](#分布式方式)
       
 <!-- /TOC -->
## 介绍
ufile-import是对象存储UFile提供的一款将数据迁移至UFile(Bucket)的工具。您可以将ufile-import部署在本地服务或者云主机上，轻松将您其他云存储的数据迁移到UFile。

### 适用情况

 - 阿里云对象存储数据迁移到UFile对象存储
 - 七牛云对象存储数据迁移到UFile对象存储
 - UFile对象存储不同bucket之前的数据迁移
 - 支持s3协议的对象存储迁移到UFile对象存储

### 准备工作
1. 根据需要迁移的文件的总大小，选择硬盘合适的云主机。必须要保证`硬盘存储量大于文件迁移数据量`,否则可能会因为硬盘大小不够，而造成迁移数据不完整。  
   举例：假设您有总量为100G的文件需要迁移，设置的处理文件并发数是40(即ufile-import.json中的`concurrent`参数，可以往下阅读，了解该参数的使用。)，在全部的迁移文件中，单个文件最大大小为1G左右，则您至少需要大小为:40乘以1乘以2=80G,左右大小的硬盘，来缓存下载过程中的临时文件,来保证迁移过程中，有足够的硬盘容量，来缓存临时文件,否则可能会造成迁移文件的不完整

### 名词定义
这里我们将介绍一些名词，稍后再使用`ufile-import`工具的过程中，您会看到他们。工具中将只显示英文，为了方便您理解，我们特别将关键字都以中文对照以及解释的方式列在下面，方便参考。

|序号(Index)|英文(English)|中文(Chinese)|工具当中的选项(Corresponding Options in "ufile-import")|释义|
| --------   | -----:  | :----:  | :----:  | :----:  |
|0|endpoint|域名|"endpoint"|表示OSS对外服务的访问域名.例如：oss-cn-hangzhou.aliyuncs.com，具体请参考[如何获取阿里云访问域名](https://helpcdn.aliyun.com/document_detail/31837.html)|
|1|bucket|存储空间|“bucket”|对象存储开通的存储空间的名称|
|2|bucket_name|存储空间|"bucket_name"|对象存储开通的存储空间的名称|
|3|accessID|公钥|"accessID"|对应您在阿里云账户的公钥。具体请参考[如何获取阿里云accessID](https://help.aliyun.com/knowledge_detail/94557.html)|
|4|accesskey|私钥|"accesskey"|对应您在阿里云账户的密钥。具体请参考[如何获取阿里云accesskey](https://help.aliyun.com/knowledge_detail/94557.html)|
|5|public_key|公钥|"public_key"|对应您在UCloud账户的公钥。具体请参考[如何获取UCloud账户公钥](https://docs.ucloud.cn/ai/uai-train/basic/key?s[]=%E5%85%AC%E9%92%A5)|
|6|private_key|私钥|"private_key"|对应您在UCloud账户的私钥。具体请参考[如何获取UCloud账户私钥](https://docs.ucloud.cn/ai/uai-train/basic/key?s[]=%E5%85%AC%E9%92%A5)|
|7|file_host|host相关信息|"file_host"|地域域名，例如:UFile域名是www.test.cn-bj.ufileos.com，则file_host为：cn-bj.ufileos.com|
|8|bucket_host|无效信息，请忽略|无效信息，请忽略|无效信息，请忽略|
|9|	job|	迁移工作|	${job_test}|在开始迁移之前，您必须创建一个迁移工作(job)，迁移工作是一个单独的目录，应该包含三个文件，分别:*.oss.json, *.ufile.json, ufile-import.json|
|10|	*.oss.json|	OSS空间描述文件|	oss.json	|这个文件描述了一个oss的bucket信息，包括bucket的名称，接入点(endpoint)，AccessID和AccessKey。星号"*"可以代表任意字符串。后缀oss.json不可更改。对于后缀为oss.json的配置文件，ufile-import工具会认为其描述的空间是阿里云OSS的空间。|
|11|	*.kodo.json|	Kodo空间描述文件|	kodo.json	|这个文件描述了一个Kodo的bucket信息，包括bucket的名称，绑定的域名，AccessKey和SecretKey。星号"*"可以代表任意字符串。后缀kodo.json不可更改。对于后缀为kodo.json的配置文件，ufile-import工具会认为其描述的空间是七牛云Kodo的空间。|
|12|	*.ufile.json|	UFILE空间描述文件	|ufile.json|	这个文件描述了一个ufile的bucket信息，包括bucket的名称，接入域名(bucket_host)，公钥和私钥。星号"*"可以代替任意字符串。后缀ufile.json不可更改。对于后缀为ufile.json的配置文件，ufile-import工具会认为其描述的空间是UFILE的空间。|
|13|	*.s3.json|	支持s3协议的空间描述文件	|s3.json|	这个文件描述了一个ufile的bucket信息，包括bucket的名称，接入地域，接入域名(bucket_host)，公钥和私钥。星号"*"可以代替任意字符串。后缀s3.json不可更改。对于后缀为s3.json的配置文件，ufile-import工具会认为其描述的空间是支持s3协议的空间。|
|14|ufile-import.json|	ufile-import配置文件|	ufile-import.json	|这个文件描述了ufile-import工具运行的配置信息，每个迁移工作(job)一份，也就是说，每个迁移工作(job)可以使用不同的配置。可以在ufile-import.json中指定源bucket信息和目标bucket信息的配置文件。这个两个文件默认和ufile-import.json位于同一个文件夹下|


## 配置文件说明
 - #### 阿里云配置文件说明
   >
   >{    
  	"endpoint": "",       //阿里云OSS域名  
  	"bucket": "%BUCKET%", //阿里云OSS存储空间名称  
 	 "accessID": "",         //公钥信息  
 	 "accessKey": "",         //私钥信息  
	 "prefix":""           //文件前缀  
   >}    

 - #### 七牛云配置文件说明
   >
   >{    
  	"bucket": "%BUCKET%",                    //七牛云Kodo存储空间名称         
    "domain": "%DOMAIN%",               //kodo-test-bucket绑定的CDN域名     
    "accessKey": "",                    //公钥信息             
    "secretKey": ""                     //私钥信息     
   >}    

 - #### UFile配置文件
   >
   >{  
        "public_key":"",     //公钥  
        "private_key":"",    	 //私钥  
        "bucket_name":"%BUCKET%", //bucket名称  
        "file_host":"", //bucket的host信息，例如:cn-bj.ufileos.com  
        "bucket_host":"" //为空  
      } 

- #### s3配置文件
   >
   >{  
        "bucket":"", 
        "region":"", 
        "endPoint":"", 
        "accessKeyId":"",          //公钥  
        "secretAccessKey":""       //私钥
      }  
 
 - #### ufile-import配置文件说明
   >
   >{  
     "redis": "localhost:6379", //本地redis服务端口号  
     "concurrent": 40, //每秒处理的线程数  
     "temp": "/tmp/", //存放文件的临时文件夹目录   
     "retry_count": 4, //如果失败了，尝试重试的次数。  
     "source": "${filename}.oss.json",  //源站的配置文件名称  
     "destine": "${filename}.ufile.json"//目标空间的配置文件名称  
  }  
 

## 安装

### 通用步骤:

####  1. 下载安装包
 - Linux64位操作系统请下载
	下载地址: https://github.com/ufilesdk-dev/ufile-import/archive/master.zip

####  2. 安装程序。
 -  进入下载安装包目录，解压文件.   
   `unzip master.zip`   
   `tar zxvf  ufile-import.tgz`
#### 3. 启动redis服务。
 - 服务依赖于redis服务，安装包中已经还有redis服务的相关配置，直接启动即可。  
     `1 cd ufile-import`  
     `2 cd redis`  
     `3 ./start.sh`  
     可以通过执行 `./ps.sh`命令来查看redis服务状态，redis服务正常启动，状态如下:
     ```html  
     root      5318     1  0 14:50 pts/0    00:00:15 ./redis-server 127.0.0.1:6379
     ```
## 使用方法
   ### 从阿里云oss迁移到UFile对象存储
   假设我在阿里云oss有一个bucket，名字为`oss-test-bucket`,所在地域为华北2（北京）,对应的外网访问EndPoint为:`oss-cn-beijing.aliyuncs.com`,accessId为:`osstestaccessId`,accessKey为:`osstestaccessKeyDate`  
   我在UFile对象存储有一个bucket,名字为`ufile-test-bucket`,所在地域为上海，对应的外网访问域名host为:`ufile-test-bucket.cn-sh2.ufileos.com`,公钥为:`ufiletestpublickey`,私钥为:`ufileprivatekeydata`  

   - #### 首先，进入`ufile-import`目录，编写oss配置文件。  
     - `1. cd ufile-import`  进入文件目录  
     - `2. mkdir job_test` 创建存放配置文件的文件夹  
     - `3. cp template/oss.json.template ./job_test/src.oss.json` 复制配置文件模板到指定目录  
     - 编辑src.oss.json文件，填写内容如下:
        >
        >{      
  	     "endpoint": "oss-cn-beijing.aliyuncs.com",      //阿里云OSS域名            
  	     "bucket": "oss-test-bucket",                    //阿里云OSS存储空间名称         
 	       "accessID": "osstestaccessId",                     //公钥信息             
 	       "accessKey": "osstestaccessKeyDate",                //私钥信息  
	     "prefix":"test2/"                                  //文件前缀       
	       }  
    
   - #### 复制ufile配置文件，并且编辑填写相应内容:
     - 复制配置文件模板`cp template/ufile.json.template ./job_test/dst.oss.json`
     - 编辑`dst.oss.json`,填写内容如下
         >
         > {    
         "public_key":"ufiletestpublickey",        //公钥              
   	     "private_key":"ufileprivatekeydata",    	 //私钥  
         "bucket_name":"ufile-test-bucket", //bucket名称  
         "file_host":"cn-sh2.ufileos.com", //bucket的host信息，例如:cn-bj.ufileos.com  
         "bucket_host":"" //为空  
         } 
     
   - #### 复制ufile-import配置文件，并且编辑填写相应内容:
     - 复制配置文件模板 `cp template/ufile-import.json.template ./job_test/ufile-import.json`
     - 编写ufile-import.json配置文件，填写内容如下：
        >
	      >{  
         "redis": "localhost:6379", //本地redis服务端口号  
         "concurrent": 40, //每秒处理的线程数  
         "temp": "/tmp/", //存放文件的临时文件夹目录  
         "retry_count": 4, //如果失败了，尝试重试的次数。  
         "source": "src.oss.json",  //源站的配置文件名称  
         "destine": "dst.ufile.json"//目标空间的配置文件名称  
       }
     
     #### 启动ufile-import服务
     显示执行:`./ufile-import ./job_test`,启动服务。
     如果要后台执行服务,可以执行`nohup ./ufile-import ./job_test &`  启动服务。
 
 ### 从七牛云Kodo迁移到UFile对象存储
   假设我在七牛云Kodo有一个bucket，名字为`kodo-test-bucket`,所在地域为华北,绑定的CDN域名为:`kodo-test.clouddn.com`,accessKey为:`kodoAccessKey`,secretKey为:`kodoSecretKey`  
   我在UFile对象存储有一个bucket,名字为`ufile-test-bucket`,所在地域为上海，对应的外网访问域名host为:`ufile-test-bucket.cn-sh2.ufileos.com`,公钥为:`ufiletestpublickey`,私钥为:`ufileprivatekeydata`  

   - #### 首先，进入`ufile-import`目录，编写kodo配置文件。  
     - `1. cd ufile-import`  进入文件目录  
     - `2. mkdir job_test` 创建存放配置文件的文件夹  
     - `3. cp template/kodo.json.template ./job_test/src.kodo.json` 复制配置文件模板到指定目录  
     - 编辑src.kodo.json文件，填写内容如下:
        >
        >{             
  	     "bucket": "kodo-test-bucket",                    //七牛云Kodo存储空间名称         
  	     "domain": "http://kodo-test.clouddn.com",               //kodo-test-bucket绑定的CDN域名     
 	       "accessKey": "kodoAccessKey",                    //公钥信息             
 	       "secretKey": "kodoSecretKey"                     //私钥信息       
	       }  
    
   - #### 复制ufile配置文件，并且编辑填写相应内容:
     - 复制配置文件模板`cp template/ufile.json.template ./job_test/dst.ufile.json`
     - 编辑`dst.ufile.json`,填写内容如下
         >
         > {    
         "public_key":"ufiletestpublickey",        //公钥              
   	     "private_key":"ufileprivatekeydata",    	 //私钥  
         "bucket_name":"ufile-test-bucket", //bucket名称  
         "file_host":"cn-sh2.ufileos.com", //bucket的host信息，例如:cn-bj.ufileos.com  
         "bucket_host":"" //为空  
         } 
     
   - #### 复制ufile-import配置文件，并且编辑填写相应内容:
     - 复制配置文件模板 `cp template/ufile-import.json.template ./job_test/ufile-import.json`
     - 编写ufile-import.json配置文件，填写内容如下：
        >
	      >{  
         "redis": "localhost:6379", //本地redis服务端口号  
         "concurrent": 40, //每秒处理的线程数  
         "temp": "/tmp/", //存放文件的临时文件夹目录  
         "retry_count": 4, //如果失败了，尝试重试的次数。  
         "source": "src.kodo.json",  //源站的配置文件名称  
         "destine": "dst.ufile.json"//目标空间的配置文件名称  
       }
     
     #### 启动ufile-import服务
     显示执行:`./ufile-import ./job_test`,启动服务。
     如果要后台执行服务,可以执行`nohup ./ufile-import ./job_test &`  启动服务。

 ### UFile对象存储不同bucket之前的数据迁移
   假设我在UFile对象存储有一个bucket,名字为`ufile-bucket-A`,所在地域为上海，对应的外网访问域名host为:`ufile-bucket-A.cn-sh2.ufileos.com`,公钥为:`ufiletestpublickeyA`,私钥为:`ufileprivatekeydataA`  
   我想将数据迁到另外一个bucket，名字为`ufile-bucket-B`,所在地域为北京，对应的外网访问域名host为:`ufile-bucket-B.cn-bj.ufileos.com`,公钥为:`ufiletestpublickeyB`,私钥为:`ufileprivatekeydataB`.
   - #### 首先，进入`ufile-import`目录，编写UFile配置文件。  
     - `1. cd ufile-import`  进入文件目录  
     - `2. mkdir job_test` 创建存放配置文件的文件夹  
     - `3. cp template/ufile.json.template ./job_test/src.ufile.json` 复制配置文件模板到指定目录  
     - 编辑src.oss.json文件，填写内容如下:
       >
       >{      
       "public_key":"ufiletestpublickeyA",        //公钥             
   	   "private_key":"ufileprivatekeydataA",    	 //私钥  
       "bucket_name":"ufile-bucket-A", //bucket名称  
       "file_host":"cn-sh2.ufileos.com", //bucket的host信息，例如:cn-bj.ufileos.com  
       "bucket_host":"" //为空  
      }
     
   - #### 复制ufile配置文件，并且编辑填写相应内容:
     - 复制配置文件模板`cp template/ufile.json.template ./job_test/dst.oss.json`
     - 编辑`dst.oss.json`,填写内容如下
       >
       >{    
       "public_key":"ufiletestpublickeyB",        //公钥           
   	   "private_key":"ufileprivatekeydataB",    	 //私钥
       "bucket_name":"ufile-bucket-B", //bucket名称
       "file_host":"cn-bj.ufileos.com", //bucket的host信息，例如:cn-bj.ufileos.com  
       "bucket_host":"" //为空  
       >}   
     
   - #### 复制ufile-import配置文件，并且编辑填写相应内容:
     - 复制配置文件模板 `cp template/ufile-import.json.template ./job_test/ufile-import.json`
     - 编写ufile-import.json配置文件，填写内容如下：
       >
	     >{  
        "redis": "localhost:6379", //本地redis服务端口号  
         "concurrent": 40, //每秒处理的线程数  
        "temp": "/tmp/", //存放文件的临时文件夹目录  
        "retry_count": 4, //如果失败了，尝试重试的次数。   
        "source": "src.ufile.json",  //源站的配置文件名称  
        "destine": "dst.ufile.json"//目标空间的配置文件名称  
        >}   
     #### 启动ufile-import服务
      显示执行:`./ufile-import ./job_test`,启动服务。
      如果要后台执行服务,可以执行`nohup ./ufile-import ./job_test &`  启动服务。


 ### 支持s3协议的对象存储迁移到UFile对象存储
   假设我在某一支持s3协议的对象存储产品中有一个bucekt，名字为`s3-bucket`,所在地域为华北2（北京）,对应的外网访问EndPoint为:`s3-cn-beijing.xxx.com`,accessKeyId为:`s3accessKeyId`,secretAccessKey为:`s3secretAccessKey`  
   我在UFile对象存储有一个bucket,名字为`ufile-test-bucket`,所在地域为上海，对应的外网访问域名host为:`ufile-test-bucket.cn-sh2.ufileos.com`,公钥为:`ufilePublickey`,私钥为:`ufilePrivatekey`  

   - #### 首先，进入`ufile-import`目录，编写oss配置文件。  
     - `1. cd ufile-import`  进入文件目录  
     - `2. mkdir job_test` 创建存放配置文件的文件夹  
     - `3. cp template/s3.json.template ./job_test/src.s3.json` 复制s3配置文件模板到指定目录  
     - 编辑src.s3.json文件，填写内容如下:
        >
        >{                
  	     "bucket": "s3-bucket",                    //存储空间名称         
         "region": "cn-beijing",                  //接入地域
  	     "endpoint": "s3-cn-beijing.xxx.com",      //接入域名  
 	       "accessID": "s3accessKeyId",                     //公钥信息             
 	       "accessKey": "s3secretAccessKey",                //私钥信息  
	       }  
    
   - #### 复制ufile配置文件，并且编辑填写相应内容:
     - 复制配置文件模板`cp template/ufile.json.template ./job_test/dst.ufile.json`
     - 编辑`dst.ufile.json`,填写内容如下
         >
         > {    
         "public_key":"ufilePublickey",        //公钥              
   	     "private_key":"ufilePrivatekey",    	 //私钥  
         "bucket_name":"ufile-test-bucket", //bucket名称  
         "file_host":"cn-sh2.ufileos.com", //bucket的host信息，例如:cn-bj.ufileos.com  
         "bucket_host":"" //为空  
         } 
     
   - #### 复制ufile-import配置文件，并且编辑填写相应内容:
     - 复制配置文件模板 `cp template/ufile-import.json.template ./job_test/ufile-import.json`
     - 编写ufile-import.json配置文件，填写内容如下：
        >
	      >{  
         "redis": "localhost:6379", //本地redis服务端口号  
         "concurrent": 40, //每秒处理的线程数  
         "temp": "/tmp/", //存放文件的临时文件夹目录  
         "retry_count": 4, //如果失败了，尝试重试的次数。  
         "source": "src.s3.json",  //源站的配置文件名称  
         "destine": "dst.ufile.json"//目标空间的配置文件名称  
       }
     
     #### 启动ufile-import服务
     显示执行:`./ufile-import ./job_test`,启动服务。
     如果要后台执行服务,可以执行`nohup ./ufile-import ./job_test &`  启动服务。
 
 ### 分布式方式
  - 分布式  
    支持分布式模式启动，提供更快的速度迁移文件。  
    执行:  
    `./ufile-import master 9090 ./${job_test}  `  
    将在本地启动一个http服务，监听端口号是9090  
    `./ufile-import slave http://${localhost}:9090 `
    这个进程将向9090端口发送请求，执行迁移任务。这个请求执行的个数，代表着分布式的个数。意味着你可以开启多个服务向9090端口请求，来加快迁移的速度。  


