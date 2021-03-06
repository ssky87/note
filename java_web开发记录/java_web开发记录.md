# 深圳开发 #

**hellmonky**

## 1 环境搭建 ##

当前深圳的所有项目开发采用javaEE组件方式进行，代码在svn进行版本管理，并且使用了Spring和myBatis进行数据库封装。
当前已经根据omni项目完成了一个程序框架，这个也是本次出差的主要学习任务和目的，所以需要在最短的时间内掌握当前开发环境的搭建和部署，并且熟悉当前已经搭建好的框架进行后续开发。

### （1） 相关的环境下载和设置 ###
当前还是基于javaEE开发web项目，并且统一管理数据库，相关的软件如下：

>对于已经使用过的软件和框架没有给出具体下载地址

 （1）JDK：1.7以上版本，基本开发环境；

 （2）tomcat：web服务容器；

 （3）[Spring Framework](http://repo.spring.io/libs-release-local/org/springframework/spring/)：综合框架，负责业务模型的建立；

 （4）[MyBatis](http://mybatis.github.io/)：ORM框架，轻量级数据库持久化框架（本项目中没有使用hibernate框架）；

 （5）MySQL：数据库支持，使用开源的数据库完成基本数据存储；

 （6）[STS](http://spring.io/tools/sts)：eclipse的Spring框架封装，方便快速开发；

 （7）svn：当前的开发环境中使用svn管理不同版本的代码，所以需要安装svn管理工具。

 （8）IDEA（可选）：另一个java IDE和eclipse的使用习惯不同，但是开发中的补全支持是很突出的。


当前使用上述的软件并且安装就可以搭建好几本的开发环境了。但是在开发中还需要一些其他第三方库的支持，会在后续的开发中逐步记录。

### （2） 基本环境设置 ###
上述环境安装的时候需要注意事项：

（1）tomcat的运行需要在环境变量中查找JAVA_HOME是否存在，该环境变量存储当前jdk的根目录，一般为：

```shell
C:\Program Files\Java\jdk1.8.0_45
```
完成这个环境变量的设置之后就可以启动tomcat的web服务支持了。

（2）IDEA中使用svn命令行工具：
因为IDEA只使用svn的命令行工具，所以在安装svn的时候一定要勾选svn的命令行支持。

（3）当前主要使用的框架为Spring和mybatis：
后台数据库的持久化和bean的转换使用mybatis来完成，后台业务层和前台交互使用Spring来完成封装，整体框架已经搭建起来了，有很多的配置是非常需要注意学习的。

（4）MySQL数据库的使用：
还是使用5.6版本的非安装MySQL数据库，然后使用navicat进行可视化管理，这儿需要注意的有：

#### <1> MySQL数据库的远程备份和恢复 ####
使用MySQL来进行数据库操作是非常方便的，并且navicat等管理工具都是建立在基本功能的封装上的，所以首先要学会基本工具的使用。例如为了数据安全需要将远程数据库的内容备份到本地进行测试，所有需要对使用命令行工具mysqldump来完成这个功能：

```shell
mysqldump -P [端口号] -h [IP地址] -u [用户名] -p [表名]>[导出数据库脚本的绝对路径]
```
这样操作的时候需要在命令行中输入密码，如果不考虑安全性因素，使用如下命令：

```shell
mysqldump -h[hosname] -u[user_name] -p[password] --default-character-set=n[char_set_name] [db_name] > [save_path]
```
使用如下的链接信息进行测试：
>IPAD:172.16.1.242

>PORT:3306

>USER:root

>PASS:P@55w0rd

>TABL:sys

实际测试代码为：

```shell
mysqldump.exe -P 3306 -h 172.16.1.242 -u root -p sys > d:\sys.sql

mysqldump.exe -h172.16.1.242 -uroot -pP@55w0rd --default-character-set=utf8 sys --skip-lock-tables> d:\sys.sql
```

而且MySQL从本地导入到远程数据库的命令和上面的唯一区别就在于>符号变为了<符号，表示不同的数据流方向。

#### <2> SQL的记录 ####
因为底层查询使用的内容都是和业务相关的，所以需要根据业务逻辑将查询内容记录下来，这样在后续文档的生成和自己重构代码的时候才方便自己回顾和优化查询代码，这一点是非常重要的，所以后续内容一定要按照开发流程和逻辑整理SQL代码。

### （3） 基本框架了解 ###
因为当前是属于接手开发，所以现在的架构都是已经定型了的，需要做的就是快速的熟悉整体架构，并且理解当前架构处于什么样的业务模型的情况下来选择的，这样做的优势和劣势是什么，多一点整体形势的思考和分析。
总体上来说就是使用javaEE进行java web开发，更具体一点就是开发一个在线的业务系统。当前的开发中，整体开发框架已经完成，通过不同的业务来填充代码从而成为不同的业务系统，所以对当前这个框架的理解是非常重要的。下面就进行更为详细的分析。

## 2 整体架构细节分析 ##

整体框架搭建还是建立在MVC的基础上的，并且对后台部分按照层次来定义调用接口，这样完成协作和同步开发，并且将测试与实现隔离，方便代码检查。因为是从已经存在的系统分析其架构，所以更像是反向思维的演练（盲人摸象），通过各种细节的构成分析，最终拼接为一个整体框架，来理解当前框架的特点和搭建过程。

根据介绍，总体上整个框架的构成为：
>**分层设计->定义业务接口->分module进行设计和实现->细化实现**

通过约定接口调用来约束不同层级之间的分离关系。从而分散整体工作，并行开发。
通过规定整个框架的基本代码组织关系：
优点：可以自动生成并且维护代码，代码层次结构固定统一，方便其他人合作开发；
缺点：一旦架构确定下来之后可变性受限，并且增加了学习成本，不容易让新的开发人员熟悉整个流程。

**现在通过从eclipse来作为基础搭建整个环境，熟悉和使用当前使用的各种框架，并且了解整体架构的特点所在。**

### （1）后台数据的持久化 ###

使用mybatis做为ORM框架来对数据库表和java对象之间去耦，并且通过配置xml文件来实现java对象的自动生成功能。
根据[官方文档](http://mybatis.github.io/mybatis-3/)给出的介绍
>MyBatis 是支持定制化 SQL、存储过程以及高级映射的优秀的持久层框架。

主要使用方式是通过配置xml文件来将数据库中的表映射为JOBO(Plain Old Java Objects)来完成去耦，对上层提供统一的操作方法。

做为一个ORM框架，基本上他和其他框架相同，都有共同的特点：
1. 从配置文件(通常是XML配置文件中)得到 sessionfactory.
2. 由sessionfactory  产生 session
3. 在session 中完成对数据的增删改查和事务提交等.
4. 在用完之后关闭session 。
5. 在java 对象和 数据库之间有做mapping 的配置文件，也通常是xml 文件。

#### <1> mybatis手动生成过程 ####

将下载到的库中的mybatis-x.x.x.jar 文件置于 classpath 中即可进行基本配置了。

>这个classpath指的是当前项目工程文件中.classpath文件中已经加入管理的，也就是在项目java build path中添加的库

##### (a) 根据数据库表生成相关的文件 #####
mybatis通过管理数据库表的java对象和在这些对象上的数据库操作来解耦对数据库的直接操作，所以需要根据数据库中的表来手动编写这些java文件和数据库表操作的配置xml文件。
需要的生产步骤如下：

> （1）创建数据库表对应的java bean结构，包含了数据库表中的字段和属性说明；

> （2）创建上述java bean对应的数据库基本操作mapper，包含对这个数据库表的增删查改等功能接口；

通过上述步骤就完成了数据库表到java对象之间的映射关系。

##### (b) 编写SQL映射mapper同名xml文件 #####
上述步骤中java bean表示了数据库表的抽象，mapper对象则是抽象了在这个表上的基本操作接口，现在还需要有一个mapper对象同名xml文件来实现mapper中定义的基本操作。
这个xml文件表示一种映射关系，描述了结果集ResultMap和SQL映射关系等内容，其中SQL映射关系包含有以下顶级元素：

```xml
cache –     给定命名空间的缓存配置。
cache-ref – 其他命名空间缓存配置的引用。
resultMap – 是最复杂也是最强大的元素，用来描述如何从数据库结果集中来加载对象。
sql –       可被其他语句引用的可重用语句块。
insert –    映射插入语句
update –    映射更新语句
delete –    映射删除语句
select –    映射查询语句
```

通过这些顶级标签元素就可以在xml中对标准SQL进行描写，从而完成后续动态SQL生成等高级功能，使得查询的调整在接口调用规定的基础上不再影响上层业务实现，从而去除耦合关联。
>更多的映射细节查看[官方文档](http://mybatis.github.io/mybatis-3/zh/sqlmap-xml.html#)

#### <2> mybatis的自动生成 ####
根据上面的流程，对于数据库表的java对象表示而言，每一个表都要手动写数据库表的java bean描述和对应数据库操作的mapper描述（包含mapper类和对应xml文件），这个是非常麻烦的事情，所以mybatis提供了自动生成工具：[MyBatis Generator](http://mybatis.github.io/generator/)，简称为MBG。通过建立起这个工具需要的配置文件，我们就能自动根据数据库中的表来生成bean结构以及mapper配置文件。

##### (a) mybatis 自动生成的配置 #####
MBG需要一个xml格式的配置文件来描述，这个配置文件包含获取数据库连接实例的数据源（DataSource）和决定事务范围和控制方式的事务管理器（TransactionManager），通过他就可以自动根据数据库中的表来生成对应的bean和mapper文件。
在网上有一篇非常好的博文：[MyBatis Generator 详解](http://blog.csdn.net/isea533/article/details/42102297) 可以参看。
在本工程中，这个xml文件为generatorConfig.xml，配置示例如下：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE generatorConfiguration
  PUBLIC "-//mybatis.org//DTD MyBatis Generator Configuration 1.0//EN"
  "http://mybatis.org/dtd/mybatis-generator-config_1_0.dtd">

<generatorConfiguration>
    <classPathEntry location="mysql-connector-java-5.1.6-bin.jar" />

    <context id="DB2Tables" targetRuntime="MyBatis3">

        <commentGenerator>
            <property name="suppressDate" value="true" />
        </commentGenerator>

        <jdbcConnection driverClass="com.mysql.jdbc.Driver"
            connectionURL="jdbc:mysql://localhost/test" userId="qgd" password="123456">
        </jdbcConnection>

        <javaTypeResolver>
            <property name="forceBigDecimals" value="false" />
        </javaTypeResolver>

        <javaModelGenerator targetPackage="test.model"
            targetProject="../src/main/java">
            <property name="enableSubPackages" value="true" />
            <property name="trimStrings" value="true" />
        </javaModelGenerator>

        <sqlMapGenerator targetPackage="test.dao"
            targetProject="../src/main/java">
            <property name="enableSubPackages" value="true" />
        </sqlMapGenerator>

        <javaClientGenerator type="XMLMAPPER"
            targetPackage="test.dao" targetProject="../src/main/java">
            <property name="enableSubPackages" value="true" />
        </javaClientGenerator>

        <table tableName="pet" domainObjectName="Pet">
        </table>

    </context>
</generatorConfiguration>
```
这个文件整体上分为两个个部分：xml文件头（用来验证xml文件的正确性）和整体配置信息（用来说明具体的配置细节）。
其中整体配置信息为configuration体，又包含多个具体的配置信息，包括：
> environment 元素体中包含了事务管理和连接池的配置。

> mappers 元素则是包含一组 mapper 映射器（这些 mapper 的 XML 文件包含了 SQL 代码和映射定义信息）。

>>更为详细的说明参看[官方文档配置说明部分](http://mybatis.github.io/mybatis-3/zh/configuration.html)。

这个配置文件提供了mybatis-generator所需要的参数信息：
  * 其中classPathEntry 是引用的jdbc的类路径，这里将jdbc jar和generator的jar包放在一起了；
  * commentGenerator 是用来除去时间信息的，这在配合类似subversion的代码管理工具时使用很有效，因为可以减少没有必要的注释迁入；
  * jdbcConnection是指定的jdbc的连接信息；
  * javaTypeResolver式类型转换的信息，这里并没有用到；
  * javaModelGenerator是模型的生成信息，这里将指定这些Java model类的生成路径；
  * sqlMapGenerator是mybatis 的sqlMapper XML文件的生成信息，包括生成路径等；
  * javaClientGenerator是应用接口的生成信息；
  * table是用户指定的被生成相关信息的表，它必须在指定的jdbc连接中已经被建立。


下面结合上面的例子和工程中实际用到的内容进行分析：

###### 1 配置头文件 ######
就是xml文件头，描述xml的格式情况。具体内容如下：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE generatorConfiguration
        PUBLIC "-//mybatis.org//DTD MyBatis Generator Configuration 1.0//EN"
        "http://mybatis.org/dtd/mybatis-generator-config_1_0.dtd">
```

###### 2 generatorConfiguration根节点 ######
generatorConfiguration节点没有任何属性，直接写节点即可，如下：

```xml
<generatorConfiguration>
    <!-- 具体配置内容 -->
</generatorConfiguration>
```
mybatis自动生成的配置选项都作为这个元素的子元素存在。

###### 3 根节点下的配置元素 ######
从这段开始，就是配置的主要内容，这些配置都是generatorConfiguration元素的子元素。包含以下子元素（有严格的顺序）：

```xml
<properties> (0个或1个)
<classPathEntry> (0个或多个)
<context> (1个或多个)
```

####### 3.1 properties元素 #######
这个元素用来指定外部的属性元素，不是必须的元素。
这个元素用于指定一个需要在配置中解析使用的外部属性文件，引入属性文件后，可以在配置中使用 ${property}这种形式的引用，通过这种方式引用属性文件中的属性值。对于后面需要配置的**jdbc**信息和targetProject属性会很有用。
这个属性可以通过resource或者url来指定属性文件的位置，这两个属性只能使用其中一个来指定，同时出现会报错。
>resource：指定**classpath**下的属性文件，使用类似com/myproject/generatorConfig.properties这样的属性值。

>url：可以指定文件系统上的特定位置，例如file:///C:/myfolder/generatorConfig.properties

####### 3.2 classPathEntry元素 #######
这个元素的作用是将MBG运行时需要用到的jar包(或zip格式)添加到**classpath**下。
最常见的用法是，当**classpath**下面**没有**JDBC驱动的时候，我们通常通过这个属性指定驱动的路径，例如：
```xml
<classPathEntry location="E:\mysql\mysql-connector-java-5.1.29.jar"/>
```
如果需要用到其他的jar包，也可以这么配置，例如如果你开发了一个MBG的插件，你就可以通过这种方式加入到**classpath**
>这里注意上面重点强调的**没有**，一般在项目中使用的时候，如果在项目中添加了驱动库，**classpath**下面都有JDBC驱动，因此从项目中启动的时候不需要配置该项。
>>建议:由于该参数使用了绝对路径，因此不利用在不同电脑上通用，因此建议最好把需要的jar包放到项目的**classpath**下，避免每个人都得单独配置路径。

####### 3.3 context元素 #######
在MBG的配置中，至少需要有一个<context>元素。<context>元素用于指定生成一组对象的环境。例如指定要连接的数据库，要生成对象的类型和要处理的数据库中的表。运行MBG的时候还可以指定要运行的<context>。
该元素只有一个**必选属性**id，用来唯一确定一个<context>元素，该id属性可以在运行MBG的使用。
在这个元素下面还有一些非常必要的子元素来进行配置，这些子元素（有严格的配置顺序）包括：

```xml
<property> (0个或多个)
<plugin> (0个或多个)
<commentGenerator> (0个或1个)
<jdbcConnection> (1个)
<javaTypeResolver> (0个或1个)
<javaModelGenerator> (1个)
<sqlMapGenerator> (0个或1个)
<javaClientGenerator> (0个或1个)
<table> (1个或多个)
```

这些子元素在自动生成中是非常关键的，下面就针对工程中常用的元素进行说明。

##### (b) mybatis generator的模板补充 #####
上述自动配置文件中，只是根据mybatis默认的内容进行了生成说明，有时候我们需要对生成内容进行自定义，就需要在配置文件中引入自定义的模板来指导生成过程。
在本工程中，就是用了org.mybatis.generator.plugins.AddLimitOffsetPlugin.java文件对生成内容进行说明，然后在generatoronfig.xml中的context元素下添加如下引入命令：

```xml
<plugin type="org.mybatis.generator.plugins.AddLimitOffsetPlugin" />
```
这个java代码的内容如下：

```java
public class AddLimitOffsetPlugin extends PluginAdapter {

    @Override
    public boolean modelExampleClassGenerated(TopLevelClass topLevelClass, IntrospectedTable introspectedTable) {
        PrimitiveTypeWrapper integerWrapper = FullyQualifiedJavaType.getIntInstance().getPrimitiveTypeWrapper();

        Field limit = new Field();
        limit.setName("limit");
        limit.setVisibility(JavaVisibility.PRIVATE);
        limit.setType(integerWrapper);
        topLevelClass.addField(limit);

        Method limitSet = new Method();
        limitSet.setVisibility(JavaVisibility.PUBLIC);
        limitSet.setName("setLimit");
        limitSet.addParameter(new Parameter(integerWrapper, "limit"));
        limitSet.addBodyLine("this.limit = limit;");
        topLevelClass.addMethod(limitSet);

        Method limitGet = new Method();
        limitGet.setVisibility(JavaVisibility.PUBLIC);
        limitGet.setReturnType(integerWrapper);
        limitGet.setName("getLimit");
        limitGet.addBodyLine("return limit;");
        topLevelClass.addMethod(limitGet);

        Field offset = new Field();
        offset.setName("offset");
        offset.setVisibility(JavaVisibility.PRIVATE);
        offset.setType(integerWrapper);
        topLevelClass.addField(offset);

        Method offsetSet = new Method();
        offsetSet.setVisibility(JavaVisibility.PUBLIC);
        offsetSet.setName("setOffset");
        offsetSet.addParameter(new Parameter(integerWrapper, "offset"));
        offsetSet.addBodyLine("this.offset = offset;");
        topLevelClass.addMethod(offsetSet);

        Method offsetGet = new Method();
        offsetGet.setVisibility(JavaVisibility.PUBLIC);
        offsetGet.setReturnType(integerWrapper);
        offsetGet.setName("getOffset");
        offsetGet.addBodyLine("return offset;");
        topLevelClass.addMethod(offsetGet);

        return true;
    }

    @Override
    public boolean sqlMapSelectByExampleWithoutBLOBsElementGenerated(XmlElement element, IntrospectedTable introspectedTable) {
        FullyQualifiedTable table = introspectedTable.getFullyQualifiedTable();

        //XmlElement lastElement = (XmlElement)element.getElements().get(element.getElements().size());

        XmlElement isNotNullElement = new XmlElement("if");
        isNotNullElement.addAttribute(new Attribute("test", "limit != null"));
        isNotNullElement.addElement(new TextElement("limit ${limit}"));
        element.getElements().add(isNotNullElement);


        isNotNullElement = new XmlElement("if");
        isNotNullElement.addAttribute(new Attribute("test", "offset != null"));
        isNotNullElement.addElement(new TextElement("offset ${offset}"));
        element.getElements().add(isNotNullElement);
        return true;
    }

    public boolean validate(List<String> warnings) {
        return true;
    }
}
```
下面就上面的配置说明。
通过查找，发现出处如下：
> [limit-offset for selectByExample plugin](http://ibatis.10938.n7.nabble.com/limit-offset-for-selectByExample-plugin-td15193.html)

> [AW: limit-offset for selectByExample plugin](http://ibatis.10938.n7.nabble.com/AW-limit-offset-for-selectByExample-plugin-td15227.html)

这个文件以附件的形式添加到回复中，但是没有其他任何的说明。代码内容和上面使用的还是有差别，附上代码如下：

```java
package org.apache.ibatis.ibator.plugins;

import java.util.List;
import org.apache.ibatis.ibator.api.FullyQualifiedTable;
import org.apache.ibatis.ibator.api.IbatorPluginAdapter;
import org.apache.ibatis.ibator.api.IntrospectedTable;
import org.apache.ibatis.ibator.api.dom.java.Field;
import org.apache.ibatis.ibator.api.dom.java.FullyQualifiedJavaType;
import org.apache.ibatis.ibator.api.dom.java.JavaVisibility;
import org.apache.ibatis.ibator.api.dom.java.Method;
import org.apache.ibatis.ibator.api.dom.java.Parameter;
import org.apache.ibatis.ibator.api.dom.java.PrimitiveTypeWrapper;
import org.apache.ibatis.ibator.api.dom.java.TopLevelClass;
import org.apache.ibatis.ibator.api.dom.xml.Attribute;
import org.apache.ibatis.ibator.api.dom.xml.TextElement;
import org.apache.ibatis.ibator.api.dom.xml.XmlElement;
import org.apache.ibatis.ibator.generator.ibatis2.XmlConstants;

/**
 * This plugin adds limit and offset clause to the example class and 
 * to the sqlMap selectByExample map.
 *
 * @author Marco Musu
 *
 */
public class AddLimitOffsetPlugin extends IbatorPluginAdapter {

    public boolean validate(List<String> warnings) {
        return true;
    }

    @Override
    public boolean modelExampleClassGenerated(TopLevelClass topLevelClass, IntrospectedTable introspectedTable) {
        PrimitiveTypeWrapper integerWrapper = FullyQualifiedJavaType.getIntInstance().getPrimitiveTypeWrapper();

        Field limit = new Field();
        limit.setName("limit");
        limit.setVisibility(JavaVisibility.PRIVATE);
        limit.setType(integerWrapper);
        topLevelClass.addField(limit);

        Method limitSet = new Method();
        limitSet.setVisibility(JavaVisibility.PUBLIC);
        limitSet.setName("setLimit"); 
        limitSet.addParameter(new Parameter(integerWrapper, "limit")); 
        limitSet.addBodyLine("this.limit = limit;"); 
        topLevelClass.addMethod(limitSet);

        Method limitGet = new Method();
        limitGet.setVisibility(JavaVisibility.PUBLIC);
        limitGet.setReturnType(integerWrapper);
        limitGet.setName("getLimit"); 
        limitGet.addBodyLine("return limit;"); 
        topLevelClass.addMethod(limitGet);

        Field offset = new Field();
        offset.setName("offset");
        offset.setVisibility(JavaVisibility.PRIVATE);
        offset.setType(integerWrapper);
        topLevelClass.addField(offset);

        Method offsetSet = new Method();
        offsetSet.setVisibility(JavaVisibility.PUBLIC);
        offsetSet.setName("setOffset"); 
        offsetSet.addParameter(new Parameter(integerWrapper, "offset")); 
        offsetSet.addBodyLine("this.offset = offset;"); 
        topLevelClass.addMethod(offsetSet);

        Method offsetGet = new Method();
        offsetGet.setVisibility(JavaVisibility.PUBLIC);
        offsetGet.setReturnType(integerWrapper);
        offsetGet.setName("getOffset"); 
        offsetGet.addBodyLine("return offset;"); 
        topLevelClass.addMethod(offsetGet);

        return true;
    }

    @Override
    public boolean sqlMapSelectByExampleWithoutBLOBsElementGenerated(XmlElement element, IntrospectedTable introspectedTable) {
        FullyQualifiedTable table = introspectedTable.getFullyQualifiedTable();
        element.getElements().remove(element.getElements().size() - 1);
        XmlElement isParameterPresenteElement =
                new XmlElement("isParameterPresent"); 
        element.addElement(isParameterPresenteElement);

        XmlElement includeElement = new XmlElement("include"); 
        includeElement.addAttribute(new Attribute("refid", 
                table.getSqlMapNamespace() + "." + XmlConstants.EXAMPLE_WHERE_CLAUSE_ID)); 
        isParameterPresenteElement.addElement(includeElement);

        XmlElement isNotNullElement = new XmlElement("isNotNull"); 
        isNotNullElement.addAttribute(new Attribute("property", "orderByClause"));  
        isNotNullElement.addElement(new TextElement("order by $orderByClause$")); 
        isParameterPresenteElement.addElement(isNotNullElement);

        isNotNullElement = new XmlElement("isNotNull"); 
        isNotNullElement.addAttribute(new Attribute("property", "limit"));  
        isNotNullElement.addElement(new TextElement("limit $limit$")); 
        isParameterPresenteElement.addElement(isNotNullElement);


        isNotNullElement = new XmlElement("isNotNull"); 
        isNotNullElement.addAttribute(new Attribute("property", "offset"));  
        isNotNullElement.addElement(new TextElement("offset $offset$")); 
        isParameterPresenteElement.addElement(isNotNullElement);
        return true;
    }
}
```

##### (c) mybatis generator的自动生成方式 #####
完成上述基本配置之后，mybatis需要根据这个配置文件生成相应的代码，mybatis generator有三种方式可以完成自动生成：命令行，eclipse插件和maven插件。
>[官方文档](http://mybatis.github.io/generator/running/running.html)有详细的说明
下面就针对这三种方式进行讲解。

###### 1 命令行下的自动生成 ######
mybatis generator最简单的就是命令行的方式，只需要指定相应的配置文件的路径即可：

```shell
java -jar /path/to/mybatis-generator-core-xxx.jar -configfile /path/to/config.xml -overwrite
```
将上述命令中的mybatis-generator-core和config.xml的路径替换为实际路径就可以生成了。

###### 2 eclipse集成插件完成自动生成 ######
eclipse提供了一个插件，安装之后可以对xml文件直接生成代码。现在简述安装过程如下：
打开eclipse的menu
```menu
help->install new software
```
然后输入如下地址：
```address
http://mybatis.googlecode.com/svn/sub-projects/generator/trunk/eclipse/UpdateSite/
```
然后点击“Add”添加，安装插件就可以了。
重启之后右键点击xml配置文件就可以根据配置生成代码了。

>**注意：**eclipse中手动新建的目录和工程中添加的目录是不同的，因为工程中添加的目录会修改当前工程的.classpath文件，在其中进行文件夹的管理。
>自己手动新建的文件夹在mybatis generator自动生成的时候回找不到路径而报错。因为没有在工程中进行管理。

###### 3 maven插件安装 ######
maven作为包java的管理器是非常好的，但是自己在实际工程中使用不多，现在简述一下插件安装过程感受一下。
参考[利用mybatis-generator自动生成代码](http://www.cnblogs.com/yjmyzz/p/mybatis-generator-tutorial.html)这一篇博文来学习。


#### <3> mybatis的java调用 ####
不论我们是手动还是通过mybatis-generator帮助我们生成了数据库表对应的bean、bean对应的mapper类和mapper对应的xml文件之后，还需要进一步的配置才能初始化mybatis，然后java才可以通过mybatis听过的session来对后台数据库操作完整封装使用。
> **工具只是手段，问题的解决才是核心**

##### (a) 创建MyBatis的mapper配置文件 #####
首先需要将给出当前mapper有多少个的具体配置，这样mybatis才能够管理这些mapper。这个过程也称为对mapper对象的配置。

根据官方文档 [运行 MyBatis Generator 后的任务](http://generator.sturgeon.mopaas.com/afterRunning.html) 的说明，mybatis需要MapperConfig.xml这个配置文件用于mybatis统一管理事务。
MapperConfig.xml需要添加所有的映射文件，所以我们按照如下格式手写配置文件：

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE configuration
    PUBLIC "-//mybatis.org//DTD Config 3.0//EN"
    "http://mybatis.org/dtd/mybatis-3-config.dtd">
<configuration>
    <environments default="development">
        <environment id="development">
            <transactionManager type="JDBC" />
            <dataSource type="POOLED">
                <property name="driver" value="com.mysql.jdbc.Driver" />
                <property name="url" value="jdbc:mysql://localhost/test" />
                <property name="username" value="qgd" />
                <property name="password" value="123456" />
            </dataSource>
        </environment>
    </environments>
    <mappers>
        <mapper resource="test/dao/PetMapper.xml" />
    </mappers>
</configuration>
```
在这个配置文件中mappers元素中添加了所有的SQL Mapper，也就是xml文件。

> 网上的文章 [Mybatis初始化配置](http://hao0.me/mybatis/2015/02/20/mybatis-init.html)对mybatis的初始化做出了完整的分析

##### (b) java调用mybatis提供的SQLSession来管理事务 #####
mybatis通过上述配置就可以完成java对象和数据库表的管理，接下来就需要通过java调用mybatis提供的SQLSession完成数据库操作了。
但是因为本项目中使用Spring来进行session管理，所以现在这一部分内容暂时没有考虑，不过官方网站给出了很详细的例子，可以参考学习。
总体上来说还是通过sqlsession来对后台数据操作进行管理，具体代码示例如下：

```java
package com.forum.test;

import java.io.IOException;

import org.apache.ibatis.io.Resources;
import org.apache.ibatis.session.SqlSession;
import org.apache.ibatis.session.SqlSessionFactory;
import org.apache.ibatis.session.SqlSessionFactoryBuilder;

import com.forum.dao.UserMapper;
import com.forum.po.User;

/**
 * myBatis数据库连接测试
 * 
 * @author db2admin
 * 
 */
public class MyBatisTest {
	/**
	 * 获得MyBatis SqlSessionFactory  
	 * SqlSessionFactory负责创建SqlSession，一旦创建成功，就可以用SqlSession实例来执行映射语句，commit，rollback，close等方法。
	 * @return
	 */
	private static SqlSessionFactory getSessionFactory() {
		SqlSessionFactory sessionFactory = null;
		String resource = "configuration.xml";
		try {
			sessionFactory = new SqlSessionFactoryBuilder().build(Resources
					.getResourceAsReader(resource));
		} catch (IOException e) {
			// TODO Auto-generated catch block
			e.printStackTrace();
		}
		return sessionFactory;
	}

	public static void main(String[] args) {
		SqlSession sqlSession = getSessionFactory().openSession();
		UserMapper userMapper = sqlSession.getMapper(UserMapper.class);
		User user = userMapper.findById("1");
		System.out.println(user.getName());

	}

}
```

#### <4> 总体配置使用流程总结 ####

上述<1>到<3>描述了java调用mybatis完成ORM框架的流程，总体上最为关键的就是mappers元素中说明的bean结构映射关系，这个文件中说明了基本的SQL语句的映射关系，将传统SQL转换为函数调用；然后通过一个mapper对象的管理来处理添加进入的所有mapper供java调用。整体思路如下：

> （1）创建数据库表：根据业务需要完成数据库设计之后，在数据库中create table，完成这一步骤；

> （2）创建java实体类：在工程中根据基本的table结构完成java bean的编写（类似C语言中的结构体，但是java中的一切都是对象，也就是class，所以这儿就是java bean了），并且添加基本的set和get函数；

> （3）创建dao接口：在完成java实体类之后，因为这个类表示了数据库表的抽象，所以还需要添加基本的增删查改操作，也就是一般意义的dao层，也就是mybatis中所谓的Mapper，表示了对抽象数据库表的基本操作；

> （4）创建dao的实现：和hibernate不同，mybatis使用一个xml文件来描述所谓的[映射语句](http://mybatis.github.io/mybatis-3/zh/sqlmap-xml.html)，通过这个xml文件中不同标签的设置，实现对数据库表的基本操作；

> （5）装载映射文件：完成上述步骤之后，需要在mybatis的总体配置文件的mapper标签下添加（4）中实现的所有xml文件，只有这样才能将（1）和（2）分别代表的数据库表项实体和java程序中的基本对象这两个内容链接在一起，完成ORM框架中的去耦作用。（这部分在后面更为详细的说明）

通过上述5个步骤，mybatis就完成了从数据库表实体到java对象的关联配置。可以在java程序中使用（2）中定义的实体类来根据在（4）中定义的同名xml中实现的基本操作来对据库表进行处理了。
手动生成和自动生成的唯一差别就在于（2）、（3）和（4）的生成过程上了，其他都没有任何区别。


### （2）java web工程 ###

>java web开发需要一些基本概念，这是自己现在没有的，需要进行学习和了解，本节内容就针对当前使用的java web开发基础知识进行学习。

#### <1> java本工程的基本概念 ####
java工程和java web工程有什么区别？为什么要分为javaee和javase这两种？而且很多工程也根据是否是javase和javaee来建立不同的工程。
其实这两种工程都是Java语言的应用，只是应用场合不同罢了，他们的本质区别在于：编译后路径。
虚拟机执行的是class文件而不是java文件，那么我们不管是何种项目都是写的java文件，怎么就不一样了呢？分成java和web两种了呢？

##### (a) .classpath文件的作用 #####
以eclipse的工程文件为例，从.classpath文件入手来看，这个文件在每个项目目录下都是存在的。这是一个XML文件，使用文本编辑器打开即可，示例如下：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<classpath>
	<classpathentry excluding="com/sys/filter/" kind="src" path="src"/>
	<classpathentry kind="src" path="web"/>
	<classpathentry kind="src" path="database"/>
	<classpathentry kind="src" path="plugin"/>
	<classpathentry kind="src" path="filter"/>
	<classpathentry kind="con" path="org.eclipse.jst.server.core.container/org.eclipse.jst.server.tomcat.runtimeTarget/Apache Tomcat v7.0">
		<attributes>
			<attribute name="owner.project.facets" value="jst.web"/>
		</attributes>
	</classpathentry>
	<classpathentry kind="con" path="org.eclipse.jst.j2ee.internal.web.container"/>
	<classpathentry kind="con" path="org.eclipse.jst.j2ee.internal.module.container"/>
	<classpathentry kind="lib" path="WebContent/WEB-INF/lib/aopalliance-1.0.jar"/>
	<classpathentry kind="lib" path="WebContent/WEB-INF/lib/chardet.jar"/>
	<classpathentry kind="output" path="build/classes"/>
</classpath>
```
这个XML文档包含一个根元素classpath，它由子元素classpathentry的kind参数来设置不同的属性和类型信息：
>src：源码相对于当前工程文件的具体路径

>con：JRE容器的信息，表示当前工程需要的基本java执行环境

>lib：项目依赖的第三方类库的相对路径

>output：src设置的所有源代码编译后的输出位置，按照源代码的结构进行组织

总体上说.classpath文件就是eclipse配置整个工程的运行环境。
是否是web工程的要点就在于最后output的位置不同，按照javaee规范放置在web目录下的就是web工程，没有的就是一般的java工程。

##### (b) .project文件的作用 #####
当前工程文件的根目录下还有一个.project文件，他也是一个xml格式的文件，打开可以看见如下结果：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<projectDescription>
	<name>sys</name>
	<comment></comment>
	<projects>
	</projects>
	
	<buildSpec>
		<buildCommand>
			<name>org.eclipse.jdt.core.javabuilder</name>
			<arguments>
			</arguments>
		</buildCommand>
	</buildSpec>
	
	<natures>
		<nature>org.eclipse.jdt.core.javanature</nature>
	</natures>
</projectDescription>
```
可以看出到这些配置主要是描述工程的基本信息：

```xml
<name></name>           工程名
<comment></comment>     工程注释描述
<buildSpec></buildSpec> 具体加载方式信息
<natures></natures>     运行时需要的额外Eclipse插件
```
也就是说.project是项目文件，项目的结构都在其中定义

##### (c) 其他配置文件相关 #####

**通过不同的配置文件设置class文件的输出位置就可以完成java不同项目的设置**

以Eclipse IDE for Java Developers这个基础IDE为例说明不同的java项目只在于组织结构不同：

>这个eclipse是最基本的用语开发java程序的IDE，对于开发web项目而言缺少一些自动生成工具来简化开发，所以如果要开发和管理web项目 需要手动安装WTP组件（Web Tools Platform）来进行管理。

在当前eclipse中按照如下网址安装WTP组件：
```url
http://download.eclipse.org/webtools/updates
```
安装完毕之后重启，然后新建一个web dynamic project就会在工程属性设置中找到“Deployment assembly”选项，然后在其中选择要部署的源代码位置，然后设置部署的路径保存。
这个选项将会在当前工程文件夹的根路径下生成”.setting“文件夹，并且在其中生成”org.eclipse.wst.common.component“配置文件。
这个文件是xml格式的配置文件，用语描述当前工程在部署的时候class文件的部署路径，用于生成war包或者tomcat调试的时候在临时文件夹下生成web工程目录结构。

```xml
<?xml version="1.0" encoding="UTF-8"?><project-modules id="moduleCoreId" project-version="1.5.0">
    <wb-module deploy-name="sys">
        <wb-resource deploy-path="/" source-path="/WebContent" tag="defaultRootSource"/>
        <wb-resource deploy-path="/WEB-INF/classes" source-path="/src"/>
        <wb-resource deploy-path="/WEB-INF/classes" source-path="/web"/>
        <wb-resource deploy-path="/WEB-INF/classes" source-path="/database"/>
        <wb-resource deploy-path="/WEB-INF/classes" source-path="/plugin"/>
        <wb-resource deploy-path="/WEB-INF/classes" source-path="/filter"/>
        <property name="context-root" value="sys"/>
        <property name="java-output-path" value="/sys/build/classes"/>
    </wb-module>
</project-modules>
```
其中的：
 deploy-path：表示部署路径
 source-path：表示源代码路径

这个内容和在IDE中设置的内容一一对应，所以插件帮助我么维护并且自动处理我们设置好的内容，简化我们开发。

综合起来.setting文件夹下存放的是eclipse的各种插件的配置文件，用于自动化处理。

##### (d) java web项目的基本结构 #####
javaee web项目的结构目录遵守javeee规范，要求必须的结构有：

```shell

```
在实际工程中除了这些必须的目录结构，还需要一些其他辅助的目录结构来帮助组织工程代码，举例如下：
```shell
web_project
------src (必须)
------src_resource(自建)
------WebRoot (必须)
-----------js (自建)
-----------css (自建)
-----------WEB-INF
----------------classes(必须)
----------------jsp(自建)
----------------lib(自建)
----------------web.xml(必须)
-----------META-INF
----------------MANIFEST.MF
-----------index.jsp
```

一般来说java的web开发工程都是按照这种目录结构组织的，然后通过war包等方式进行代码组织，最后部署到服务器容器中从而可以被访问。所以现在的代码结构也是按照这个形式进行处理的。

##### 综上所述 #####

eclipse工程通过.classpath和.project这两个文件进行基本描述，stackoverflow上有一个详细的说明[What's in an Eclipse .classpath/.project file?](http://stackoverflow.com/questions/7186676/whats-in-an-eclipse-classpath-project-file)：

```text
Eclipse is a runtime environment for plugins. Virtually everything you see in Eclipse is the result of plugins installed on Eclipse, rather than Eclipse itself.

The .project file is maintained by the core Eclipse platform, and its goal is to describe the project from a generic, plugin-independent Eclipse view. What's the project's name? what other projects in the workspace does it refer to? What are the builders that are used in order to build the project? (remember, the concept of "build" doesn't pertain specifically to Java projects, but also to other types of projects)

The .classpath file is maintained by Eclipse's JDT feature (feature = set of plugins). JDT holds multiple such "meta" files in the project (see the .settings directory inside the project); the .classpath file is just one of them. Specifically, the .classpath file contains information that the JDT feature needs in order to properly compile the project: the project's source folders (that is, what to compile); the output folders (where to compile to); and classpath entries (such as other projects in the workspace, arbitrary JAR files on the file system, and so forth).

Blindly copying such files from one machine to another may be risky. For example, if arbitrary JAR files are placed on the classpath (that is, JAR files that are located outside the workspace and are referred-to by absolute path naming), the .classpath file is rendered non-portable and must be modified in order to be portable. There are certain best practices that can be followed to guarantee .classpath file portability.
```

**从.java源代码到.class编译的VM执行程序这个过程不同的java工程没有任何区别，但是根据不同的协议来组织.class的结构形式产生了不同java项目。**

在学习java的时候一定要明白这个道理，才能掌握各种框架和工具的实现原理，把握住java开发中的设计思想和不同协议对项目架构带来的影响，才能抛开技术本身而关注与业务的逻辑和实现。

>**只有把握了技术的本质才能不受技术对思路的限制，否则拿着锤子看什么都是钉子。**

在目前工程开发的javaee项目的时候常用的还是已经配置了各种自动化工具的Eclipse IDE for Java EE Developers，只需要根据IDE提供的可视化插件来管理项目配置然后自动生成需要的文件就可以了，不用自己手动去组织代码的结构形式，方便和简化自己的开发。**但是一定不能因为有了自动化工具就忽视基本的组织原理，问题总是会出现，依赖工具并不能避免一切问题，一定要掌握原理。**

#### <2>java web相关的概念 ####
web开发是目前java的热点应用，这种现象的存是和java语言本身也有密切的关系的：

（1）Java是一种动态加载和运行的语言。也就是说当应用程序持有一个类的地址（CLASSPATH）和名称（包名和类名）的情况下，可以在程序运行期间任何时候加载这个类，并创建和使用该类的对象。

 （2）Java Servlet要求必须运行在Web服务器当中，与Web服务器之间属于分工和互补关系。确切的说，在实际运行的时候Java Servlet与Web服务器会融为一体，如同一个程序一样运行在同一个Java虚拟机（JVM）当中。
 
 （3）Servlet对每个请求都是单独启动一个线程，而不是进程。这种处理方式大幅度地降低了系统里的进程数量，提高了系统的并发处理能力。另外因为Java Servlet是运行在虚拟机之上的，也就解决了跨平台问题。
 
 （4）当Web容器接收到来自客户端的请求信息之后，会根据URL中的Web元件地址信息到Servlet队列中查找对应的Servlet对象，如果找到则直接使用，如果没有找到则加载对应的类，并创建对象。也就是说，Servlet对象是在第一次被使用的时候才创建的，并且一旦创建就会被反复使用，不再创建新的对象。所有创建出的Servlet对象会在Web服务器停止运行的时候统一进行垃圾回收。
 
 （5）为了解决客户端请求地址与Java Servlet之间对应关系问题，Web容器需要一个用来描述这种对应关系的文件，一般是web.xml文件。如果一个Web应用程序中存在很多个Servlet，那么web.xml会变得非常庞大。在Servlet 3.0规范推出之后，允许在Servlet代码中使用声明式语法来代替web.xml中的描述信息，这才让web.xml瘦身下来。下图是这个过程的一个示意图。

##### (a) web服务器 #####
Web服务器是可以向发出请求的浏览器提供文档的程序。web服务器作为互联网应用的基础平台，它实质上是一个网关，即介于多种协议之间的程序。
web服务器由两个部分组成：

>（1）HTTP守候程序（HTTPd），完成web服务器的基本功能，包括：和客户建立连接；接受客户提交的HTTP请求消息，将HTTP响应消息返回给客户，关闭连接等。狭义的web服务器就是指这个部分；
>（2）各种服务器端应用程序组成：这些应用程序作为HTTPd和其它外部系统之间的中介，完成服务器的扩展功能，平时所说的网关应用程序就是指这个部分。

也就是说：web服务器是一个只处理http协议，只给浏览器发送静态页面的应用程序；然后通过其他程序完成动态内容的处理，这些其他程序就是应用服务器。

在[Web 服务器与应用服务器的区别是什么？](http://www.zhihu.com/question/20096067)有一个清晰的说明：
>严格意义上Web服务器只负责处理HTTP协议，只能发送静态页面的内容。而JSP，ASP，PHP等动态内容需要通过CGI、FastCGI、ISAPI等接口交给其他程序去处理。这个其他程序就是应用服务器。
>比如Web服务器包括Nginx，Apache，IIS等。而应用服务器包括WebLogic，JBoss等。应用服务器一般也支持HTTP协议，因此界限没这么清晰。但是应用服务器的HTTP协议部分仅仅是支持，一般不会做特别优化，所以很少有见Tomcat直接暴露给外面，而是和Nginx、Apache等配合，只让Tomcat处理JSP和Servlet部分。

随着web技术的发展，web服务器和应用服务器之间的界限越来越小，这篇文章： [Web服务器的工作原理](http://www.importnew.com/15020.html) 就给出了很清晰的发展过程。

**注意：下文中所有的web应用服务器包含了基本web服务和应用服务，不再细分。**

##### (b) web容器 #####
web容器是web应用服务器中位于组件和平台之间的接口集合。
容器一般位于应用服务器之内，由应用服务器负责加载和维护。一个容器只能存在于一个应用服务器之内，一个应用服务器可以建立和维护多个容器。

>web容器为满足某种服务调用规范的框架，介于web应用服务器和web应用之间，他不一定是应用程序，也可以为一个进程。web容器为web应用提供了统一的访问接口，为web应用服务器提供了统一的管理接口，从而将web应用服务器和web程序解耦。web应用服务器通过创建不同的进程来实例化不同的web容器，一个web容器载入一个web应用，同一个web容器中使用线程来响应当前web程序的不同请求。

web应用服务器管理当前计算机中被指定的端口，所有按照一定协议访问这些端口的请求都会被web应用服务器接受处理，并且分发给不同的web应用，然后由不同的web应用程序返回请求结果，然后再有web应用服务器转换为对应的协议返回给访问者（一般而言是浏览器，当然也可以是应用程序，例如wget等）。

##### (c) java web容器 #####
在Java方面，web容器一般是指Servlet容器。Servlet容器是与Java Servlet交互的web容器的组件。
web容器负责管理Servlet的生命周期、把URL映射到特定的Servlet、确保URL请求拥有正确的访问权限和更多类似的服务。综合来看，Servlet容器就是用来运行你的Servlet和维护它的生命周期的运行环境。


##### (d) java servlet #####
明白了java开发中的web容器就等同于servlet容器，那么就需要理解java web开发中的核心概念java servlet。

Java Servlet就是你能够编写的根据请求动态生成内容的服务端组件。其主要功能在于交互式地浏览和修改数据，生成动态 Web 内容。
Java Servlet是一套规范，官方文档： [Java Servlet Technology](http://www.oracle.com/technetwork/java/index-jsp-135475.html) 给出了各个版本的规范要求（最新版本为3.0）。

狭义的 Servlet 是指Java语言根据servlet规范在javax.servlet包里定义的接口；
广义的 Servlet 是指任何根据servlet规范定义的接口的类；


java原生实现Servlet接口有3个实现类，FacesServlet、GenericServlet、HttpServlet：
 *FacesServlet类一般用于JSF的Servlet，很少使用。
 *GenericServlet是一个抽象类，有除了service()方法外的所有抽象方法的默认实现。
 *HttpServlet最常用，包含在javax.servlet.http.HttpServlet类中。

一般情况下，人们将 Servlet 理解为后者。比如Spring就使用了自己的DispatcherServlet来替换java原生的servlet从而完成更为复杂的管理。

java Servlet 运行于支持 Java 的应用服务器中，也就是说只要有JVM环境，java Servlet 就可以被支持。从实现上讲，Servlet 可以响应任何类型的请求，但绝大多数情况下 Servlet 只用来扩展基于 HTTP 协议的 Web 服务器。

java Servlet 是web容器最基本的组成单元，http请求是向web服务器请求一种信息资源，而servlet就充当了这种资源的最小单位，servlet可以无限扩展，使用java所有的类库资源，为用户返回文本、图片、音频的各类信息资源。

Java Servlet接口为Servlet的生命周期声明了三个基本方法——init()、service()和destroy()。
每个Servlet都要实现这些方法，并在它们的生命周期的特定时间由web容器来控制它的创建、初始化、提供服务、销毁等。它的各种行为方式通过web.xml文件中来配置。

由于Servlet是一个java接口，所以需要加载。这个加载动作有servlet容器完成，下面就结合上述的servlet容器和servlet来描述一下java web应用的整体行为。

##### (e) servlet容器的总体行为 #####
根据上述介绍可知，sevlet受控于另一个java应用程序，它就是web容器；sevlet没有main方法，说明它要被别的类web容器调用；servlet到web容器需要用xml文件注册，而xml的解析由web容器封装的方法完成。

整个java web应用启动过程为：
>**当Servlet容器启动时，它会部署并加载所有的web应用。当web应用被加载时，Servlet容器会一次性为每个应用创建Servlet上下文（ServletContext）并把它保存在内存里。Servlet容器会处理web应用的web.xml文件，并且一次性创建在web.xml里定义的Servlet、Filter和Listener，同样也会把它们保存在内存里。当Servlet容器关闭时，它会卸载所有的web应用和ServletContext，所有的Servlet、Filter和Listner实例都会被销毁。**

所以servlet容器管理整个Servlet的生命周期。整体上生命周期有4个阶段：加载、初始化、提供服务和销毁。

加载阶段是将请求的servlet类加载到java虚拟机中，这里需要通过公开的无参的构造方法来实例化，无没有则加载失败，也可以通过<load-on-startup>设置servlet在web容器启动时加载。当前web应用关闭的时候web容器会自动销毁当前所有的servlet。 上述这些过程都由web容器来控制，开发者关注最多的是初始化和提供服务两个阶段：
 *init()方法中，开发者可以获取配置在web.xml中的初始化参数；
 *service()方法会在Servlet请求时调用，处理业务逻辑。
通过这两个方法就完成了web应用的用户交互。

##### (f) java servlet容器配置 #####
在Servlet规范中定义了web.xml文件，它是Web应用的配置文件，Web.xml文件是和Web容器无关的。通过Web.xml文件可以配置Servlet类和url的映射、欢迎列表、过滤器以及安全约束条件等。
>xml文件的其他特点：对大小写敏感。

然后根据文档[web.xml Reference Guide for Tomcat](http://wiki.metawerx.net/wiki/Web.xml)说明：
```text
The web.xml Deployment Descriptor file describes how to deploy a web application in a servlet container such as Tomcat.
```
可以知道Web容器通过web.xml文件来配置web应用（但是这个web.xml文件并不是必须的）。web.xml这个文件是sun公司规范化格式的，用xml承载的web容器描述文件。

以工程中的实际配置为例，展示如下：
```xml
<?xml version="1.0" encoding="UTF-8"?>
<web-app version="2.5" xmlns="http://java.sun.com/xml/ns/javaee"
	xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://java.sun.com/xml/ns/javaee http://java.sun.com/xml/ns/javaee/web-app_2_5.xsd">


    <!-- 上下文参数 -->
    <!-- the details for log4j in Spring can be found in Spring's API doc for Log4jConfigListener -->
    <context-param>
        <param-name>log4jConfigLocation</param-name>
        <param-value>/WEB-INF/config/log4j.properties</param-value>
    </context-param>
    <!-- if we set the log4jRefreshInterval, the system will re-read the log4j configuration regularly-->
    <context-param>
        <param-name>log4jRefreshInterval</param-name>
        <param-value>60000</param-value>
    </context-param>
    <!-- the manegementtool.root can be used in log4j configuration to point to this application. -->
    <context-param>
        <param-name>webAppRootKey</param-name>  
        <param-value>SpringWeb.root</param-value>  
    </context-param>
    <!-- The definition of the Root Spring Container shared by all Servlets and Filters -->
	<context-param>
		<param-name>contextConfigLocation</param-name>
		<param-value>/WEB-INF/spring/appContext.xml</param-value>
	</context-param>
    
	<!-- 设置监听器 -->
	<listener>
        <listener-class>org.springframework.web.util.Log4jConfigListener</listener-class>
    </listener>
 	<!-- Creates the Spring Container shared by all Servlets and Filters -->
	<listener>
		<listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
	</listener>
	
	<!-- 设置过滤器 -->
	<filter>
		<filter-name>sysAccessFilter</filter-name>
        <filter-class>
            org.springframework.web.filter.DelegatingFilterProxy
        </filter-class>
    </filter>
    <filter-mapping>
        <filter-name>sysAccessFilter</filter-name>
        <url-pattern>/*</url-pattern>
    </filter-mapping>
	
	 <!-- Spring servlet相关设置 -->
    <servlet>
		<servlet-name>appServlet</servlet-name>
		<servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
		<init-param>
			<param-name>contextConfigLocation</param-name>
			<param-value>/WEB-INF/spring/servlet/servlet-context.xml</param-value>
		</init-param>
		<load-on-startup>2</load-on-startup>
	</servlet>
	<servlet-mapping>
		<servlet-name>appServlet</servlet-name>
		<url-pattern>/</url-pattern>
	</servlet-mapping>
    

	<!-- Processes application requests -->
    <session-config>
        <session-timeout>300</session-timeout>
    </session-config>
    <error-page>
        <exception-type>java.lang.Exception</exception-type>
        <location>/WEB-INF/views/error.jsp</location>
    </error-page>
    <error-page>
        <error-code>404</error-code>
        <location>/WEB-INF/views/error.jsp</location>
    </error-page>

</web-app>
```
首先我们先理解这个web.xml文档描述了那些内容，然后分析web容器怎么根据这个配置文件对整个web进行部署。

###### 1 web.xml文档内容分析 ######
（1）xml文件头：
因为web.xml也是xml标准格式，所以需要一个xml头来说明当前的xml承载格式的信息：
```xml
<?xml version="1.0" encoding="UTF-8"?>
```
指定了xml的版本号和所使用的编码。

（2）文件类型定义DTD：
DTD（Document Type Definition），即文档类型定义,是一种XML约束模式语言，是XML文件的验证机制，属于XML文件组成的一部分。
是一种保证XML文档格式正确的有效方法，可以通过比较XML文档和DTD文件来看文档是否符合规范，元素和标签使用是否正确。

一个典型的DTD文档包含：元素的定义规则，元素间关系的定义规则，元素可使用的属性，可使用的实体或符号规则。从而把要创作的XML文档的元素结构、属性类型、实体引用等预先进行规定。
用户既可以直接在XML文档中定义DTD，也可以通过URL引用外部的DTD。DTD为XML文档的编写者和处理者提供了共同遵循的原则，使得与文档相关的各种工作有了统一的标准。一般内容如下：
```xml
<!DOCTYPE
	web-app 
	PUBLIC
	"-//Sun Microsystems, Inc.//DTD Web Application 2.3//EN"
	http://java.sun.com/dtd/web-app_2_3.dtd >
```
其中包含四个元素，各自的含义如下：

>web-app定义该文档(部署描述符，不是DTD文件)的根元素

>PUBLIC意味着DTD文件可以被公开使用

>"-//Sun Microsystems, Inc.//DTD Web Application 2.3//EN"意味着DTD 由Sun Microsystems, Inc.维护。 该信息也表示它描述的文档类型是DTD Web Application 2.3，而且DTD是用英文书写的。

>URL"http://java.sun.com/dtd/web-app_2_3.dtd" 表示D文件的位置。

（3）文档类型定义XSD：
根据文档[web.xml - DTD and XSD](http://wiki.metawerx.net/wiki/Web.xmlDTDAndXSD)所述：
```text
There are two ways to specify the schema for a the web.xml file (Deployment Descriptor).
DTD - Document Type Definition
XSD - XML Schema Definition
```
自JSP 2.0 / Servlets 2.4规范之后，使用XSD（XML Schema Definition）的方式对当前xml文档进行规范描述，而不再使用DTD。

根据文档：[Why Use XML Schemas?](http://www.w3schools.com/schema/schema_why.asp)可知XSD也是为了描述了XML文档的结构。
XSD可以用一个指定的XML Schema来验证某个XML文档，以检查该XML文档是否符合其要求。文档设计者可以通过XML Schema指定一个XML文档所允许的结构和内容，并可据此检查一个XML文档是否是有效的。XML Schema本身是一个XML文档，它符合XML语法结构。可以用通用的XML解析器解析它。 

XSD是DTD替代者的原因：
**一是据将来的条件可扩展，二是比DTD丰富和有用，三是用XML书写，四是支持数据类型，五是支持命名空间。**

现在总结对比两者如下：
```text
DTD的缺点：
 1) DTD 是使用非 XML 语法编写的
 2) DTD 不可扩展,不支持命名空间,只提供非常有限的数据类型
相对于DTD，XSD的优点: 
 1) XML Schema基于XML,没有专门的语法 
 2) XML Schema可以象其他XML文件一样解析和处理 
 3) XML Schema比DTD提供了更丰富的数据类型
 4) XML Schema提供可扩充的数据模型
 5) XML Schema支持综合命名空间
 6) XML Schema支持属性组
```

根据上述内容介绍可知XSD可以通过读入不同的XSD配置来进行格式规范，给出一个 JSP 2.2 / Servlets 3.0 (Tomcat 7.0)规范的示例：

```xml
<web-app xmlns="http://java.sun.com/xml/ns/javaee"
      xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
      xsi:schemaLocation="http://java.sun.com/xml/ns/javaee http://java.sun.com/xml/ns/javaee/web-app_3_0.xsd"
      version="3.0">
```

本工程中通过引入web-app2.5的XSD规范来检查web.xml，所以具体的内容如下：

```xml
<web-app version="2.5" xmlns="http://java.sun.com/xml/ns/javaee"
	xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://java.sun.com/xml/ns/javaee http://java.sun.com/xml/ns/javaee/web-app_2_5.xsd">
```
>web-app_2_5.xsd这个规范，还是比较早的了（属于Java EE 5 Schema Resources）。
每个web.xml文件的根元素<web-app>中，都必须标明这个 web.xml使用的是哪个模式文件。所以XSD不再单独的放在web-app元素之前，而是作为web-app的一个配置属性存在了。

>参考文档：
[XML Schema 与 XML DTD的技术比较与分析](http://www.ibm.com/developerworks/cn/xml/x-sd/)
[XML中DTD,XSD的区别与应用](http://kim-miao.iteye.com/blog/1310963)
[Java EE: XML Schemas for Java EE Deployment Descriptors](http://www.oracle.com/webfolder/technetwork/jsc/xml/ns/javaee/index.html)
[web.xml - DTD and XSD](http://wiki.metawerx.net/wiki/Web.xmlDTDAndXSD)

（4）web-app元素：
部署描述符的根元素是web-app，其它的元素都放在<web-app></web-app>之中。所有和部署相关的配置都是作为web-app的子元素标签在web.xml中被描述。web-app作为他的子元素的统一声明存在。

（4）web-app的配置子元素：
现在将最常用的几个子元素进行分析。

1. <context-param>上下文参数：
context-param元素声明应用范围内的初始化参数。它用于向 ServletContext提供键值对，即应用程序上下文信息。
我们的listener,filter等在初始化时会用到这些上下文中的信息。在servlet里面可以通过getServletContext().getInitParameter("context/param")得到。
context-param元素含有一对参数名和参数值，用作应用的servlet上下文初始化参数。参数名在整个Web应用中必须是惟一的。
param-name子元素包含有参数名，而param-value子元素包含的是参数值。作为选择，可用description子元素来描述参数。
```xml
<context-param>
	<param-name>contextConfigLocation</param-name>
	<param-value>/WEB-INF/spring/appContext.xml</param-value>
</context-param>
```

2. <filter>过滤器：
filter元素用于指定Web容器中的过滤器。在请求和响应对象被servlet处理之前或之后，可以使用过滤器对这两个对象进行操作。
利用filter-mapping元素，过滤器被映射到一个servlet或一个URL模式。这个过滤器的filter元素和filter-mapping元素必须具有相同的名称。

```xml
<filter>
    <filter-name>sysAccessFilter</filter-name>
    <filter-class>
       org.springframework.web.filter.DelegatingFilterProxy
    </filter-class>
</filter>
```
**在请求和响应对象被servlet处理之前或之后，可以使用过滤器对这两个对象进行操作。**

3. <filter-mapping>关联元素：
filter-mapping元素用来声明Web应用中的过滤器映射。过滤器可被映射到一个servlet或一个URL模式。将过滤器映射到一个servlet中会造成过滤器作用于servlet上。将过滤器映射到一个URL模式中则可以将过滤器应用于任何资源，只要该资源的URL与URL模式匹配。过滤是按照部署描述符的filter-mapping元素出现的顺序执行的。

```xml
<filter-mapping>
    <filter-name>sysAccessFilter</filter-name>
    <url-pattern>/*</url-pattern>
</filter-mapping>
```
其中filter-name值必须对应filter元素中声明的其中一个过滤器名称。

4. <listener>监听器：
listener元素用来注册一个监听器类，可以在Web应用中包含该类。使用listener元素，可以收到事件什么时候发生以及用什么作为响应的通知。
也就是说，listener元素和servlet元素类似，用于在Web 应用启动时，启动某些后台程序，这些后台程序负责为系统运行提供支持。

使用Listener 只需要两个步骤:
 (1)创建Listener 实现类：
 创建Listener 类必须实现ServletContextListener 接口，该接口包含两个方法。
  • contextInitialized(ServletContextEvent sce): 启动Web 应用时，系统调用该Filter的方法。
  • contextDestroyed(ServletContextEvent sce): 关闭Web 应用时候，系统调用Filter的方法。
 (2)在web.xml 文件中配置Listener：
 **Listener 用于启动Web 应用的后台服务程序，但不负责处理及响应用户请求，因此无须配置URL。**
 若将Listener 配置在Web 容器中(如果Web 容器支持Listener)，则Listener 将随 Web 应用的启动而启动。配置Listener 时使用<listener/>元素，下面是配置Listener 的片段:
```xml
<listener>
    <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
</listener>
```

5. <servlet>元素：
<servlet></servlet> 用来声明一个servlet的数据，主要有以下子元素：

```xml
<servlet-name></servlet-name>		指定servlet的名称
<servlet-class></servlet-class>		指定servlet的类名称
<init-param></init-param>			用来定义servlet的初始化参数，可有多个init-param。在servlet类中通过getInitParamenter(String name)方法访问初始化参数
<jsp-file></jsp-file>				指定web站台中的某个JSP网页的完整路径
<load-on-startup></load-on-startup>	指定当Web应用启动时，装载Servlet的次序。当值为正数或零时：Servlet容器先加载数值小的servlet，再依次加载其他数值大的servlet。当值为负或未定义：Servlet容器将在Web客户首次访问这个servlet时加载它。
<servlet-mapping></servlet-mapping>	用来定义servlet所对应的URL，包含两个子元素
<url-pattern></url-pattern>			指定servlet所对应的URL
```
servlet元素必须含有servlet-name元素和servlet-class元素，或者servlet-name元素和jsp-file元素。描述如下：
 ● servlet-name元素用来定义servlet的名称，该名称在整个应用中必须是惟 一的。
 ● servlet-class元素用来指定servlet的完全限定的名称。
 ● jsp-file元素用来指定应用中JSP文件的完整路径。这个完整路径必须由a/ 开始。

当启动Web容器时，用load-on-startup元素自动将servlet加入内存。加载servlet就意味着实例化这个servlet， 并调用 它的init方法。可以使用这个元素来避免第一个servlet请求的响应因为servlet载入内存所导致的任何延迟。如果load-on- startup元素存在，而且也指定了jsp-file元素，则JSP文件会被重新编译成servlet，同时产生的servlet也被载入内存。  load-on-startup元素的内容可以为空，或者是一个整数。这个值表示由Web容器载入内存的顺序。举个例子，如果有两个servlet元素都 含有load-on-startup子元素，则load-on-startup子元素值较小的servlet将先被加载。如果load-on- startup子元素值为空或负值，则由Web容器决定什么时候加载servlet(在该servlet被选择时才加载)。如果两个servlet的load-on-startup子元素值相同，则由Web容器决定先加载哪一个servlet。

servlet-mapping元素将URL模式映射到某个servlet。根据规范，**servlet元素必须出现在所有servlet-mapping元素之前。**

6. <session-config>会话超时配置：
单位为分钟。项目中的配置如下：
```xml
<session-config>
    <session-timeout>300</session-timeout>
</session-config>
```

7. <error-page>错误页面
主要用于处理请求发生错误的时候跳转页面的设置，示例如下：
```xml
<!-- 1、通过错误码来配置error-page。当系统发生×××错误时，跳转到错误处理页面。 -->
<error-page>
    <error-code>404</error-code>
    <location>/NotFound.jsp</location>
</error-page>
<!-- 2、通过异常的类型配置error-page。当系统发生java.lang.NullException（即空指针异常）时，跳转到错误处理页面。 -->
<error-page>
    <exception-type>java.lang.NullException</exception-type>
    <location>/error.jsp</location>
</error-page>
```

上述就是web-app的常用元素，更多详细内容参看[官方文档](http://docs.oracle.com/cd/E13222_01/wls/docs81/webapp/web_xml.html)。
>参考文档：
[web.xml文件中的web-app元素](http://blog.sina.com.cn/s/blog_490cc6450100h54p.html)
[web.xml文件详解](http://www.cnblogs.com/hellojava/archive/2012/12/28/2835730.html)
[tomcat web.xml配置](http://www.blogjava.net/baoyaer/articles/107428.html)
[web.xml详解——元素含义及加载顺序](http://wenku.baidu.com/view/4d155464f5335a8102d22039.html)
[web.xml Reference Guide for Tomcat](http://wiki.metawerx.net/wiki/Web.xml)

###### 2 web容器加载web.xml的顺序 ######
tomcat等web容器在初始化web应用的时候首先会加载web.xml文件进行这个web应用的初始化配置，那么就需要根据web.xml中的配置进行加载，具体的加重顺序为：
** context-param -> listener -> filter -> servlet **

也就是说上述元素在web.xml中的定义顺序和加载顺序是无关的。但是**凡是涉及到mapping的都会顺序相关**，例如listener，filter和servlet的加载顺序就是按照定义顺序执行的。





### （3）Spring框架的使用 ###
根据（2）中的web一般配置，Spring也需要满足这些配置要求，但是Spring做了更多的工作来接管其中的各个组件，现在就针对Spring来看一下java web工程中的配置情况。

#### <1> Spring框架的web.xml配置 ####
工程中使用了Spring MVC框架进行前后台交互的封装，就需要在上述java web工程的web.xml文件中进行配置。
并且使用Spring的组件对基本web.xml中的Filter、Listener、Servlet等进行设置。下面就针对使用Spring框架的时候应该如何设置web.xml进行说明和分析。

##### (a) 前端servlet配置 #####
SpringMVC是一个基于DispatcherServlet的MVC框架，如果使用DispatcherServlet来替代传统的servlet，那么每一个前段请求最先访问的都是DispatcherServlet，DispatcherServlet负责转发每一个Request请求给相应的Handler，Handler处理以后再返回相应的视图(View)和模型(Model)，返回的视图和模型都可以不指定，即可以只返回Model或只返回View或都不返回。

###### 声明DispatcherServlet ######
也就是说DispatcherServlet是Spring MVC的入口，所有进入Spring Web的Request都经过DispatcherServlet，所以首先需要在 web.xml 中注册 DispatcherServlet并且进行配置。
>这儿和不使用SpringMVC而只使用最基本servlet的原理都是一样的，但是因为使用了Spring框架，所以这儿使用DispatcherServlet作为中间层对servlet进行管理。

在web.xml中的设置具体代码如下：

```xml
<servlet>
 <servlet-name>dispatherContext</servlet-name>
 <servlet-class>
  org.springframework.web.servlet.DispatcherServlet
  </servlet-class>
 <!-- 启动顺序 -->
 <load-on-startup>1</load-on-startup>
</servlet>
```
>注意：对于tomcat容器来说，在其自带的web.xml文件中最先初始化的是default，值是1；所以我们在实际工程中设置启动顺序的值为2，不过影响不大。

这样通过对web.xml中servlet内容的设置，引入了SpringMVC的servlet。

###### 设置DispatcherServlet初始化配置文件路径 ######
加载 DispatcherServlet 时 Spring 会尝试读取初始配置文件，默认的初始配置文件位于 web.xml 相同的路径下文件名与注册的Servlet名有关 Servlet注册名跟上 -servlet.xml。当然这种读入方式不灵活，所以可以通过指定参数的方式来对配置文件路径进行设置，具体配置如下：

```xml
<init-param>
 <param-name>contextConfigLocation</param-name>
 <param-value>
  <!-- 配置文件名 -->
 </param-value>
</init-param>
```
实际工程中的配置内容为：

```xml
<init-param>
	<param-name>contextConfigLocation</param-name>
	<param-value>/WEB-INF/spring/servlet/servlet-context.xml</param-value>
</init-param>
```

###### DispatcherServlet初始化配置文件分析 ######
接着我们根据这儿对DispatcherServlet的初始化参数配置文件进行分析，看看DispatcherServlet是怎么被设置的，打开上述设置的servlet-context.xml文件为：

```xml
<!-- xml文件头 -->
<?xml version="1.0" encoding="UTF-8"?>

<!-- 基本配置头 -->
<beans:beans xmlns="http://www.springframework.org/schema/mvc" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xmlns:beans="http://www.springframework.org/schema/beans" xmlns:context="http://www.springframework.org/schema/context"
  xsi:schemaLocation="http://www.springframework.org/schema/mvc http://www.springframework.org/schema/mvc/spring-mvc-3.0.xsd
		http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans-3.0.xsd
		http://www.springframework.org/schema/context http://www.springframework.org/schema/context/spring-context-3.0.xsd">


	<!-- Configurer that replaces ${...} placeholders with values from a properties file -->
	<context:property-placeholder location="/WEB-INF/config/configure.properties" />
	
	<!-- 设置使用注解的类所在的包,会自动扫描 -->
	<context:component-scan base-package="com.sys.mvc" />
	
	<!-- Handles HTTP GET requests for /resources/** by efficiently serving up static resources in the ${webappRoot}/resources 
    directory -->
	<resources mapping="/resources/**" location="/resources/" />
	
	<!-- 添加注解驱动 -->
	<!-- 
	真正作用是注册下面两个bean，完成注解映射 ：
    <bean class="org.springframework.web.servlet.mvc.annotation.DefaultAnnotationHandlerMapping">   这个bean是在使用注解@ResultMapping进行请求映射spring必须使用的解析类 
    <bean class="org.springframework.web.servlet.mvc.annotation.AnnotationMethodHandlerAdapter">  
	-->	
	<annotation-driven />

	<!-- DispatcherServlet Context: defines this servlet's request-processing infrastructure -->
	<beans:import resource="servlet-service.xml" />

	<!-- Using a MultipartResolver with Commons FileUpload to support file upload -->
	<beans:bean id="multipartResolver" class="org.springframework.web.multipart.commons.CommonsMultipartResolver">
	
	<!-- one of the properties available; the maximum file size in bytes -->
    <beans:property name="maxUploadSize" value="1000000" />
	</beans:bean>

	<!-- 对转向页面的路径解析。prefix：前缀， suffix：后缀 -->
	<!-- Resolves views selected for rendering by @Controllers to .jsp resources in the /WEB-INF/views directory -->
	<beans:bean class="org.springframework.web.servlet.view.InternalResourceViewResolver">
		<beans:property name="prefix" value="/WEB-INF/views/" />
		<beans:property name="suffix" value=".jsp" />
	</beans:bean>
	
	<beans:bean id="dataSechStuIfSechController"
		class="com.sys.mvc.controller.datasech.DataSechStuIfSechController">
		<beans:property name="resultPath" value="${path.for.generate.result}" />
	</beans:bean>

	<!-- Interceptor for internationalization -->
	
	<beans:bean id="localeChangeInterceptor" class="org.springframework.web.servlet.i18n.LocaleChangeInterceptor">
	</beans:bean>
	
	<beans:bean class="org.springframework.web.servlet.mvc.annotation.DefaultAnnotationHandlerMapping">
		<beans:property name="interceptors">
			<beans:list>
				<beans:ref bean="localeChangeInterceptor" />
			</beans:list>
		</beans:property>
	</beans:bean>

	<beans:bean id="stringHttpMessageConverter"
		class="org.springframework.http.converter.StringHttpMessageConverter" />

	<beans:bean id="castorMarshaller" class="org.springframework.oxm.castor.CastorMarshaller" />

	<beans:bean id="mappingJacksonHttpMessageConverter"
		class="org.springframework.http.converter.json.MappingJacksonHttpMessageConverter" />
	
	<!-- internationalization base on session -->
  
	<beans:bean id="localeResolver" class="org.springframework.web.servlet.i18n.SessionLocaleResolver">
		<beans:property name="defaultLocale" value="en" />
	</beans:bean>
	
	<!-- 完成请求和注解POJO的映射 -->
	<beans:bean class="org.springframework.web.servlet.mvc.annotation.AnnotationMethodHandlerAdapter">
		<beans:property name="messageConverters">
			<beans:list>
				<beans:ref bean="stringHttpMessageConverter" />
				<beans:ref bean="mappingJacksonHttpMessageConverter" />
			</beans:list>
		</beans:property>
	</beans:bean>

	<beans:bean class="org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter">
		<beans:property name="messageConverters">
			<beans:list>
				<beans:ref bean="stringHttpMessageConverter" />
				<beans:ref bean="mappingJacksonHttpMessageConverter" />
			</beans:list>
		</beans:property>
	</beans:bean>

	<!-- Internationalization end -->

</beans:beans>
```
这个配置文件中对于DispatcherServlet的行为进行了描述。
具体的内容还在分析。。。

###### 指定DispatcherServlet需要处理的URL ######
通过声明和初始化DispatcherServlet，就完成了对DispatcherServlet的注册，接着就必须指定哪些URL需要由DispatcherServlet来处理。
为DispatcherServlet指定需要处理的URL，需要在web.xml中添加如下配置代码：
```xml
<servlet-mapping>
 <servlet-name>dispatherContextServlet</servlet-name>
 <url-pattern>*.do</url-pattern>
</servlet-mapping>
```
其中url-pattern表示拦截地址，表示检测到设置的地址时候需要用为DispatcherServlet来进行处理。有以下几种处理方式：

```xml
<url-pattern>/</url-pattern>		：这种方式拦截了类似/user/login的地址；会导致静态资源无法访问。
<url-pattern>/*</url-pattern>		：错误的拦截方式，可以走到Action中，但转发到jsp时再次被拦截，不能访问到jsp。
<url-pattern>/path/*</url-pattern>	：拦截包含/path/路径的请求。
<url-pattern>*.do</url-pattern>		：简单，实用的拦截方式，不存在静态资源访问的问题。
```

所以上述的设置中表示SpringMVC将处理的对当前网址的所有.do请求。
在本项目中，Spring处理所有调用请求，将“.do”替换为“/”，表示Spring将处理所有的前端请求（至于为什么这么选没有想到原因）。

######  分解应用上下文 ######
DispatcherServlet 可以从以<servlet-name>命名的xml文件中载入应用上下文。但建议将应用上下文分散到应用系统的各个层中。例如：
```shell
web 层		test-servlet.xml（用于控制的bean，Spring MVC组件）
业务层		test-service.xml
持久层		test-data.xml
```
将不同的bean按照业务进行分组分类，从而将之前的单个xml按照分组和分类拆分为多个xml，但是这样之后DispatcherServlet就要管理多个配置文件了，为了保证所有配置文件都会被载入，你需要在web.xml中配置一个上下文载入器。
有两种上下文载入器：
>ContextLoaderListener
>ContextLoaderServlet

可以在web.xml中这样配置ContextLoaderListener:
```xml
<listener>
    <listener-class>org.springframework.web.context.ContextLoaderLisetener
    </listener-class>
</listener>
```
完成监听之后，还需指定配置文件的位置。如没有指定上下文，那么载入器将在/WEB-INF/application-Context.xml处寻找Spring配置文件。我们使用参数来明确上下文配置文件的路径：
```xml
<context-param>
	<param-name>contextConfigLocation</param-name>
	<param-value>/WEB-INF/spring/appContext.xml</param-value>
</context-param>
```
这样DispatcherServlet就可以自动的跟踪appContext.xml配置文件的变化来进行动态设置。

###### 总结 ######
通过上述步骤就将DispatcherServlet建立了起来，接下来就需要建立web层了。

##### (b) servlet map配置 #####

```xml
<servlet-mapping>
	<servlet-name>appServlet</servlet-name>
	<url-pattern>/</url-pattern>
</servlet-mapping>
```

##### (c) 上下文配置文件加载路径 #####
配置文件读取器注册成功后还需要设定配置文件列表，通过设置全局参数contextConfigLocation来对配置文件列表进行说明：
>多个配置文件列表以逗号分隔，注意路径

```xml
<context-param>
	<param-name>contextConfigLocation</param-name>
	<param-value>
		/WEB-INF/dispatcherContext-servlet.xml,
		<!-- classpath*: 指定编译后的class目录 在ide中 与src根目录相同 -->
		classpath*:hibernateContext.xml
	</param-value>
</context-param>
```



### （4）mybatis和Spring的集成 ###
单独使用mybatis是有很多限制的（比如无法实现跨越多个session的事务），而且很多业务系统本来就是使用spring来管理的事务，因此mybatis最好与spring集成起来使用。
在实际工程中，使用Spring管理session并且与mybatis集成的配置代码如下：

```xml
<beans:bean id="sqlSessionFactory" class="org.mybatis.spring.SqlSessionFactoryBean">
	<beans:property name="dataSource" ref="dataSource" />
	<beans:property name="mapperLocations" value="classpath*:com/sys/db/**/*.xml" />
</beans:bean>

<beans:bean id="sqlSession" class="org.mybatis.spring.SqlSessionTemplate">
	<beans:constructor-arg index="0" ref="sqlSessionFactory" />
</beans:bean>
```

这儿使用了

#### <2> Spring作为容器管理java class ####

### （5） javaScript前端开发 ###
本项目中使用html5、css3和jQuery来作为前端开发主要语言。其中：
> html和css主要用于静态网页展示；
> 然后通过后台java servlet的返回值使用js动态生成html的属性来完成页面的动态加载和生成。html和css主要用于静态网页展示。

前台页面的动态交互主要通过两种方式完成：
> （1）html网页的分解和动态加载；
> （2）html网页的动态生成；

当前工程中这两种方式都用到了，但是第二种主要为属性的动态生成，而不是html内容的动态生成。

#### <1> HTML基本概念 ####

#### <2> CSS基本概念 ####

#### <3> JavaScript基本概念 ####
JavaScript是和java没有任何关系的脚本语言，主要用在浏览器中对HTML的DOM元素进行操作的一门动态语言，
需要注意的是：函数内部声明变量的时候，一定要使用var命令。如果不用的话，你实际上声明了一个全局变量！例如：

```JavaScript
//可以在函数中直接使用全局变量
var n=999;
function f1(){
　alert(n);
}
alert(f1()); // 999
//不能在函数外使用内部变量
function f1(){
  var n=999;
}
alert(n); // error
//内部定义变量没有用var显示说明，默认为全局变量
function f1(){
  n=999;
}
f1();
alert(n); // 999
```

#### <4> 前端开发流程 ####

#### <5> 本工程中使用的js编写思路和方法 ####

##### (a) 动态加载js文件的调试方法 #####

在本框架中使用了将js文件路径动态的添加到header下的方式来加载js，这种方式会导致浏览器调试时候无法再当前domain下管理动态加载进来的js文件，那么如何调试这些代码？
根据网上帖子 [如何调试异步加载的js文件](http://blog.csdn.net/bruce128/article/details/40615233) 的说明：
> 在需要调试的js文件顶部加一行代码：
> //@ sourceURL=XXX.js
其中XXX表示当前js的文件名，这样就可以在浏览器调试中的"no domain"下面看到当前异步加载的js文件了，并且可以调试。

其中不管是用异步加载还是通过对header动态后缀js文件路径，都是对js的一种动态加载方式，根据 [Google Chrome调试js代码](http://www.open-open.com/lib/view/open1384785324165.html) ：
> 有时候一些非常 动态的代码是以字符串的形式通过 eval() 函数在当前 Javascript context 中创建出来，而不是作为一个独立的 js 文件加载的。这样你在左边的内容区就找不到这个文件，因此很难调试。其实我们只要在 eval 创建的代码末尾添加一行 “ //@ sourceURL=name“  就可以给这段代码命名（浏览器会特殊对待这种特殊形式的注释），这样它就会出现在左侧的内容区了，就好像你加载了一个指定名字的 js 文件一样，可以设置断点和调试了。

可以明白为什么js文件中加入了sourceURL才能够调试的原因，因为各种动态加载的js代码都是片段，需要一个虚拟的文件名来进行指定，这样才能方便调试。

##### (b) javaScript常用函数说明 #####
因为前台中目前都使用js开发，而且js自带了很多的原生函数来简化开发，但是自己不常使用，所以将遇到的不了解的js原生函数解释说明如下：

###### 1 join函数 ######
代码示例：

```JavaScript
setCookie("currentpage",names.join(".-/"),365);
```
用来将用户名构造为一个字符串，然后获取cookie。
####### 定义和用法 #######
join() 方法用于把数组中的所有元素放入一个字符串。
元素是通过指定的分隔符进行分隔的。
####### 语法 #######
arrayObject.join(separator)
参数 	    描述
separator	可选。指定要使用的分隔符。如果省略该参数，则使用逗号作为分隔符。
####### 返回值 #######
返回一个字符串。该字符串是通过把 arrayObject 的每个元素转换为字符串，然后把这些字符串连接起来，在两个元素之间插入 separator 字符串而生成的。
####### 实例 #######
```JavaScript
<script type="text/javascript">

var arr = new Array(3)
arr[0] = "George"
arr[1] = "John"
arr[2] = "Thomas"

document.write(arr.join())

</script>
```
返回值为：
```JavaScript
George,John,Thomas
```

###### 2 setTimeout()函数 ######
setTimeout() 方法用于在指定的毫秒数后调用函数或计算表达式。

语法
setTimeout(code,millisec)
参数	描述
code	必需。要调用的函数后要执行的 JavaScript 代码串。
millisec	必需。在执行代码前需等待的毫秒数。
提示和注释
提示：setTimeout() 只执行 code 一次。如果要多次调用，请使用 setInterval() 或者让 code 自身再次调用 setTimeout()。
实例
```JavaScript
<HTML>
  <HEAD>
    <SCRIPT type="text/javascript">
    function timedMsg()
    {
    var t=setTimeout("alert('5 seconds!')",5000)
    }
    </SCRIPT>
  </HEAD>
  
  <BODY>
    <FORM>
      <input type="button" value="Display timed alertbox!"
      onClick="timedMsg()">
    </FORM>
    <P>Click on the button above. An alert box will be
    displayed after 5 seconds.</P>
  </BODY>
</HTML>
```

###### 3 javaScript浏览器对象模型 ######
在代码中看到片段：

```JavaScript
function resetTheTimeout() {
	clearTimeout(timeOutSetting);
	timeOutSetting = setTimeout(function() {
		showAlertWindow(i18nmsg.text("text.alert.timeout"));
		location.assign(g_loginURL);
	}, g_millisecondsForTimeout);
}
```
其中的"location.assign()"方法加载新的文档，从而完成跳转。
这儿使用了JavaScript的浏览器对象模型（Browser Object Model），使 JavaScript 有能力与浏览器“对话”。
所有浏览器都支持 window 对象。它表示浏览器窗口。所有 JavaScript 全局对象、函数以及变量均自动成为 window 对象的成员。
全局变量是 window 对象的属性。全局函数是 window 对象的方法。
也就是说windows表示了当前js的执行环境，通过在不同的windows下进行切换，来完成交互的操作是需要js和windows进行沟通的。


##### (c) jQuery的选择符 #####
jQuery使用了CSS风格的选择符对DOM元素进行选择，这种方式是jQuery的核心，在使用了jQuery开发的前台代码中是非常重要的。现在就将常用的选择符说明总结。
###### 1 元素选择符 ######
jQuery 使用 CSS 选择器来选取 HTML 元素。
```text
$("p")        选取 <p> 元素。
$("p.intro")  选取所有 class="intro" 的 <p> 元素。
$("p#demo")   选取所有 id="demo" 的 <p> 元素。
```
###### 2 属性选择器 ######
jQuery 使用 XPath 表达式来选择带有给定属性的元素。
```text
$("[href]")          选取所有带有 href 属性的元素。
$("[href='#']")      选取所有带有 href 值等于 "#" 的元素。
$("[href!='#']")     选取所有带有 href 值不等于 "#" 的元素。
$("[href$='.jpg']")  选取所有 href 值以 ".jpg" 结尾的元素。
```

###### 3 CSS选择器 ######
jQuery CSS 选择器可用于改变 HTML 元素的 CSS 属性。
下面的例子把所有 p 元素的背景颜色更改为红色：
```JavaScript
$("p").css("background-color","red");
```

##### (d) 对jQuery的扩展使用 #####

jQuery插件的开发包括两种：
一种是类级别的插件开发，即给jQuery添加新的全局函数，相当于给jQuery类本身添加方法。jQuery的全局函数就是属于jQuery命名空间的函数，另一种是对象级别的插件开发，即给jQuery对象添加方法。
> 参考： [jQuery插件开发全解析](http://www.iteye.com/topic/545971)

举例说明，在分析menu调用点击事件的过程中，通过浏览器单步调试，发现了一个基本调用链：
> 

在这个调用链中发现了对jQuery方法的扩展：

```JavaScript
$.fn.loadPageToContent = function(names) {
	FileLoadTools.loadRqrFiles(names, $(this));
};
```
通过jQuery源代码可以知道：

```JavaScript
jQuery.fn = jQuery.prototype = ...
```
所以$.fn就是jQuery.fn，表示jQuery的原型，也是指jquery的命名空间。因此加在fn上的方法及属性，将会对jquery实例每一个有效，也就是说这个方法添加到了jQuery的整个命名空间中，成为了成员函数。 
例如：扩展$.fn.abc()，即$.fn.abc()是对jquery扩展了一个abc方法，那么后面你的每一个jquery实例都可以引用这个方法了。
> 参考：[What does jQuery.fn mean?](http://stackoverflow.com/questions/4083351/what-does-jquery-fn-mean)

但是这种扩展jQuery的方法为整个jQuery命名空间添加了一个静态函数或者对象，这样做容易造成命名污染，所以这种方式更适合开发一个完整的插件，而不是单个函数，从：
> [How to Create a Basic Plugin](http://learn.jquery.com/plugins/basic-plugin-creation/)
> [Advanced Plugin Concepts](http://learn.jquery.com/plugins/advanced-plugin-concepts/)
可以看到更多的细节。典型的例子就是$.AJAX()这个函数。

针对于单个函数的扩展，比较好的方法为使用jQuery提供的扩展函数来进行封装：
> jQuery.fn.extend();
> jQuery.extend();

（1）jQuery.extend(object)为jQuery类添加类方法，可以理解为添加静态方法。如：
```JavaScript
jQuery.extend({
min: function(a, b) { return a < b ? a : b; },
max: function(a, b) { return a > b ? a : b; }
});
jQuery.min(2,3); //  2 
jQuery.max(4,5); //  5
```
（2）jQuery.fn.extend(object)：对jQuery.prototype进得扩展，就是为jQuery类添加“成员函数”。jQuery类的实例可以使用这个“成员函数”。
比如我们要开发一个插件，做一个特殊的编辑框，当它被点击时，便alert 当前编辑框里的内容。可以这么做：

```JavaScript
$.fn.extend({
    alertWhileClick:function() {
          $(this).click(function(){
                 alert($(this).val());
           });
     }
});
$("#input1").alertWhileClick();
```
其中，$("#input1")　为一个jQuery实例，当它调用成员方法 alertWhileClick后，便实现了扩展，每次被点击时它会先弹出目前编辑里的内容。

对比这两种方法：
> jQuery.extend() 的调用并不会把方法扩展到对象的实例上，引用它的方法也需要通过jQuery类来实现，如jQuery.init()，而 jQuery.fn.extend()的调用把方法扩展到了对象的prototype上，所以实例化一个jQuery对象的时候，它就具有了这些方法，这 是很重要的，在jQuery.JS中到处体现这一点。


##### <e> 前台js代码的调试工具 #####
前台代码调试需要一个整体环境，包含HTML、CSS和JavaScript这三样，所以单纯的JavaScript解释器是没有用的，因此最好的调试器还是浏览器的开发者工具，推荐使用chrome的F12打开的Chrome developer tool。
网上有详细的 [官方文档](https://developer.chrome.com/devtools) 可与参看。
还有一个中文文档 [Chrome developer tool介绍（javascript调试）](http://www.cnblogs.com/wukenaihe/archive/2013/01/27/javascript%E8%B0%83%E8%AF%95.html) 也非常详细。

###### <f> 页面跳转方法 ######
在一个网站中，不同的页面之间跳转是一个非常重要的操作，通过页面的验证和跳转可以完成用户的登入等不同功能的展示，增加了网站页面内容的管理和丰富程度，HTML和JavaScript都可以实现页面跳转。
现在就JavaScript的页面跳转，总结不同的方式：
第一种：
```javaScript
<script language="javascript" type="text/javascript">
       window.location.href="login.jsp?backurl="+window.location.href;
</script>
```
使用 window.location.href 来重定位页面链接。根据stackoverflow的 []()：
> window.location.href is not a method, it's a property that will tell you the current URL location of the browser. 

第二种：
<script language="javascript">
       alert("返回");
       window.history.back(-1);
</script>


第三种：
<script language="javascript">
       window.navigate("top.jsp");
</script>


第四种：
<script language="JavaScript">
       self.location=’top.htm’;
</script>


第五种：
<script language="javascript">
       alert("非法访问！");
       top.location=’xx.jsp’;
</script>

