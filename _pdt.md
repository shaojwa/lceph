* 要注意区别 软件版本号和路标版本号
* 公司用VRBD 命名法
* 需求->产品->路标
* 公司规格只关心到开发版本

### 产品路标(VR)

NAS V100R002 (简写V1R2)

### 开发版本=路标+B版本

    开发版本B01 2018年
    开发版本B02 2018年
    开发版本B03 2019年


### 软件版本 = 开发版本+D版本

但这里有一个映射：

    开发版本B01+B02 对应到软件版本 B01，现在的鉴定分支都还是B01，后续可能改成B03，比如 NAS V100R002B01D0029
    开发版本B03     对应到软件版本 B02，现在的主线分支都还是B02，后续可能改成B03，比如 NAS V100R002B02D005

### 基线版本

### 子版本

一般对应某个软件版本，即到D版本为止的版本。

### 对应平台基线版本号	

V100R002B30D001

### 内部代码库版本/分支版本

分支是代码库上的名字，比如B20_product，按照平台的命名来。

现在的主线： UniStorOS_V100R001B01

现在的分支： UniStorOS_V100R001B20_Product

都是平台对代码的命名，因为代码上是用的一套命名