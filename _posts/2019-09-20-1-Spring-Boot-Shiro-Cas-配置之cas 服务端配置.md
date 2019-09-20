---
layout:     post
title:      Spring Boot、Shiro、Cas配置之服务端配置
subtitle:	从零开始配置Cas服务端
date:       2019-09-30
author:     Gahon
catalog:    true
tag:
    - spring boot
    - cas
---

## 1. 本文目的

## 一、本文目的

本文主要讲解如何从零开始搭建 `CAS 5.3.12` 服务端版本搭建，并且开始使用http验证和自定义数据库连接等相关知识



## 二、相关链接

> CAS 官网： [https://www.apereo.org/projects/cas](https://www.apereo.org/projects/cas)
>
> CAS 下载： [https://github.com/apereo/cas-overlay-template/tree/5.3](https://github.com/apereo/cas-overlay-template/tree/5.3)



## 三、 搭建步骤

### 1. 下载cas

在这里我们采用cas官网推荐的overlay方式进行部署，由于电脑上使用的java版本是jdk1.8, 而目前cas6.0 需要的jdk版本为11，所以我们切换到5.3分支，然后下载对应的zip包=到本地解压



### 2. 导入项目

解压过后得到的是一个maven项目，在这里通过idea将该maven项目进行导入，导入过后等待更新完maven依赖。

maven更新好以后能够看到多了一个`overlay`  的目录，该目录的内容便是我们要进行替换的内容，据官网介绍，该目录下的所有文件都可以通过在src文件夹下建立对应路径下的同名文件进行替换，这一点之后我们会提到



### 3. 允许cas通过http访问

cas为了安全起见，默认只能通过https进行访问，而要想使用https，还得配置一些ssl证书，为了节省开发时间，先开始http支持。



新建文件夹 `src/java/resource`， 将`overlays\org.apereo...\WEB-INF\classes`  下的 `application.properties` 复制到`resource`下，`resource`目录下的文件都会覆盖掉`classes`里对应的文件。

我们在`application.properties`中添加如下几行数据：

```properties
# 是否开始https
cas.tgc.secure=false
# 从json中加载server数据
cas.serviceRegistry.initFromJson=true
# 将原有的ssl相关的注释掉
#server.ssl.key-store=file:/etc/cas/thekeystore
#server.ssl.key-store-password=changeit
#server.ssl.key-password=changeit
```

关闭https后我们还需要在services中添加https支持，同理我们建立文件夹`services`， 然后将overlays里边services文件夹下的`HTTPSandIMAPS-10000001.json`文件复制到该文件夹中，将该文件中的serviceId修改为如下格式便能够支持http了。

```properties
"serviceId" : "^(https|imaps|http)://.*"
```



### 4. 修改cas外部配置

修改项目中`etc/cas/config/cas.properties`文件中的cas.server.name和cas.server.prefix的值为实际值，例如修改为下：

```properties
# 服务名称我这里手动修改了hosts文件, 将cas.server.com指向了127.0.0.1
# 端口为 application.properties里边设置的端口
cas.server.name: http://cas.server.com:8087
cas.server.prefix: http://cas.server.com:8087/cas

```

修改完成以后，在终端执行命令  `build.cmd copy` ，便可以将配置文件复制到项目所在路径的根目录的 /etc/cas中，程序监听该文件，当文件改变时，自动重新启动。



### 5. 启动第一个demo

打开http以后就可以开始尝试运行一下啦，不过在运行之前需要先安装好maven，并且将maven路径配置到环境变量中，这样便可以直接运行了。

**注： 如果需要使用idea的tomcat进行部署，方法在后边会有讲**



打开terminal，在项目根目录执行 `buid.cmd run` ， 然后等待编译运行， 然后打开浏览器输入 `地址:port/cas/login` 便能够进入登录页面了，此时使用的是静态账号密码，位置在 application.properties的最后几行中，怎么修改为数据库验证在之后会讲。



登录成功以后就证明cas服务端已经能够正常运行啦



## 四、 修改为数据库方式验证

### 1. 添加pom依赖

要想通过数据库验证，需要先添加对应的依赖包才行，在pom.xml的dependence中添加一下依赖：

```xml
<dependencies>
    <dependency>
        <groupId>org.apereo.cas</groupId>
        <artifactId>cas-server-support-jdbc</artifactId>
        <version>${cas.version}</version>
    </dependency>
    <dependency>
        <groupId>org.apereo.cas</groupId>
        <artifactId>cas-server-support-jdbc-drivers</artifactId>
        <version>5.3.11</version>
    </dependency>
    <dependency>
        <groupId>mysql</groupId>
        <artifactId>mysql-connector-java</artifactId>
        <version>5.1.36</version>
    </dependency>
</dependencies>
```



### 2. 配置数据库信息

在 application.proprities中添加如下几行，然后做对应修改

```properties
# 配置数据库地址
# 该条为查询记录的sql
cas.authn.jdbc.query[0].sql=SELECT * FROM 用户所在表 WHERE 登录名称对应字段名 =?
cas.authn.jdbc.query[0].fieldPassword=数据库中密码所在字段
cas.authn.jdbc.query[0].url=jdbc:mysql://ip:port/数据库?useUnicode=true&characterEncoding=UTF-8&useSSL=false
cas.authn.jdbc.query[0].dialect=org.hibernate.dialect.MySQLDialect
cas.authn.jdbc.query[0].user=数据库用户名
cas.authn.jdbc.query[0].password=数据库登录密码
cas.authn.jdbc.query[0].driverClass=com.mysql.jdbc.Driver
```



### 3. 运行

按照上述配置好以后便可以运行，然后使用数据库中的账号密码进行登录啦

**在配置好数据库验证以后记得把配置文件中的那个静态账号密码给删除掉哟**



### 4. 修改密码验证

直接的将明文密码存储到数据库中是个不明智的选择，所以一般都会选择将密码字段进行加密，而cas也提供了方式进行加密校验，只需要在配置文件中添加以下几行即可（[详细介绍点这](https://apereo.github.io/cas/5.3.x/installation/Configuration-Properties-Common.html#password-encoding) ).

```properties
## 使用系统自带的MD5加密, MD5 也可更换为SHA1，所有加密算法可以查看官网相关信息
cas.authn.jdbc.query[0].passwordEncoder.type=DEFAULT
cas.authn.jdbc.query[0].passwordEncoder.characterEncoding=UTF-8
cas.authn.jdbc.query[0].passwordEncoder.encodingAlgorithm=MD5
```

##### 自定义加密算法

有时候在存储密码的时候可能使用到了自定义的加密算法，而cas并没有提供对应的算法的时候就需要我们自定义加密算法了，方法很简单, 在`src/main/java` 下新建一个加密类，实现`PasswordEncoder`接口即可，示例如下

```java
public class MyPasswordEncoder implements PasswordEncoder {

    private String md5(String pass) {
        return DigestUtils.md5DigestAsHex(pass.getBytes());
    }

    /**
     * 加密算法，自定义的算法在这里边实现
     * @param charSequence 前段输入的密码
     */
    @Override
    public String encode(CharSequence charSequence) {
        return md5(charSequence.toString());
    }

    /**
     * 密码校验类
     *
     * @param charSequence 前段输入的密码
     * @param s            数据库里边的密码
     * @return
     */
    @Override
    public boolean matches(CharSequence charSequence, String s) {
        return encode(charSequence).equals(s);
    }
}
```



编写好自定义加密类以后只需要在配置文件中指定使用该类加密即可，如下

```properties
#-------------------自定义密码加密类----------------------------
cas.authn.jdbc.query[0].passwordEncoder.type=com.***.MyPasswordEncoder
cas.authn.jdbc.query[0].passwordEncoder.characterEncoding=UTF-8
```



然后运行便可以使用自定义的密码加密类进行登录验证啦



### 5. 自定义属性

默认情况下，cas会将登录用户名返回给其他系统，有时候我们需要改用户对应的一些其他信息也给返回，这时候就需要我们自定义返回的字段了。

1.  在配置文件中添加如下几行

```properties
#-------------------单行数据返回--------------------------
cas.authn.attributeRepository.jdbc[0].singleRow=true
cas.authn.attributeRepository.jdbc[0].order=0
cas.authn.attributeRepository.jdbc[0].url=${cas.authn.jdbc.query[0].url}
cas.authn.attributeRepository.jdbc[0].user=${cas.authn.jdbc.query[0].user}
cas.authn.attributeRepository.jdbc[0].password=${cas.authn.jdbc.query[0].password}
cas.authn.attributeRepository.jdbc[0].username=用户名字段
cas.authn.attributeRepository.jdbc[0].sql=通过上边的用户名字段去查询的需要返回的sql语句

cas.authn.attributeRepository.jdbc[0].dialect=${cas.authn.jdbc.query[0].dialect}
cas.authn.attributeRepository.jdbc[0].ddlAuto=none
cas.authn.attributeRepository.jdbc[0].driverClass=${cas.authn.jdbc.query[0].driverClass}
cas.authn.attributeRepository.jdbc[0].leakThreshold=10
cas.authn.attributeRepository.jdbc[0].batchSize=1
```



2.   在services的json文件中添加如下一行数据(注意格式，每行行末需要加逗号):

```properties
"attributeReleasePolicy": {
	"@class": "org.apereo.cas.services.ReturnAllAttributeReleasePolicy"
}
```

完成以后重新运行登录后就能够查看到返回了哪些数据给前段啦。



### 五、 自定义登录界面、

不同的项目可能需要不同的登录界面，而cas也是可以根据不用域名设置不同的登录界面主题的。

由于时间有限，现在先不写，以后有时间再来补充吧



## FAQ

### 1. 执行 `build(.cmd) run`  出现 invalid entry CRC

可能是依赖包出现问题，去maven仓库里边将`cas-server-webapp-...`相关的文件夹删掉，然后重新更新maven依赖。

### 2. 使用静态账号密码登陆不上

这个原因可能是因为之前部署过一次了，而且更改了静态账号密码或者说更换了验证方式，而系统在build的时候可能使用到了之前已经编译好的文件，从而导致验证方式还是之前设置的方式， 解决方法建议在导入项目以后更改项目的