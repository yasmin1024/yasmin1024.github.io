---
title: SpringBoot集成Mybatis和Quartz实现配置多数据源的定时调度任务项目
categories:
  - Java学习
tags:
  - SpringBoot
  - Quartz
  - Mybatis
  - Druid
  - 多数据源配置
abbrlink: b0798327
date: 2023-04-26 17:54:28
---

这篇文章是为了记录自己实现这个Demo的过程。

涉及到SpringBoot，Mybatis，Quartz，Mysql，Druid，日志框架使用SpringBoot默认集成的logback

# 1.使用Maven构建多模块项目
Maven多模块项目通过合理的模块拆分，实现了代码的复用，便于维护和管理。
注意父Maven工程中的pom文件指定的打包类型必须为pom，子模块的打包类型需要指定为jar，而子模块也需要在父pom文件中声明

项目结构如下：

- MultDataSourceSchedule  父工程
  - common  公共模块，管理工具类、配置信息读取等
  - dao  持久化模块，与数据库交互
  - scheduleproject  定时任务调度业务模块
  - config  配置文件管理包，存放各种配置文件

其中common为基础模块，可根据实际需要引入redis，slf4j等。

dao与scheduleproject都需要依赖common，scheduleproject还需要依赖dao。

由于Maven具有传递依赖机制，dao引入依赖common，scheduleproject引入依赖dao，这时scheduleproject中会同样引入common，因此scheduleproject无需重复引入依赖common。

## 1.1 创建父maven工程
创建完成后，删除掉src包，父工程中无需写代码，只需要配置pom文件。
![](https://raw.githubusercontent.com/yasmin1024/hexoBlogImgRepository/main/postImg/01/createParentMaven.png)

## 1.2 创建子模块
父工程上右键 → new → Module  选择Spring Initaializr ,注意JDK版本与java版本要对应。

![](https://raw.githubusercontent.com/yasmin1024/hexoBlogImgRepository/main/postImg/01/creatSonModule.png)

依赖项选择Spring Web。重点注意❗❗❗ ***<font color= red>SpringBoot版本的选择，3.x的版本不支持jdk1.8，因此使用jdk1.8需要选择2.x的版本来使用</font>***

![](https://raw.githubusercontent.com/yasmin1024/hexoBlogImgRepository/main/postImg/01/creat-important.png)

继续创建common模块和dao模块，此时可以不用再创建Spring项目了，父工程上右键 → new → Module  选择  新建模块

![](https://raw.githubusercontent.com/yasmin1024/hexoBlogImgRepository/main/postImg/01/creatcommon.png)

接下来在父工程下创建一个folder，父工程上右键 → new → Folder，输入名称config，此时项目结构如下

![](https://raw.githubusercontent.com/yasmin1024/hexoBlogImgRepository/main/postImg/01/createResult.png)
最后删除掉common模块和dao模块中的Application启动项，删除common、dao、scheduleproject模块中resource下的application.properties

在config下新建文件，命名为application.properties，相关配置信息写在这个文件中

## 1.3 建立依赖

### 1.3.1 父工程pom.xml

`<parent>.....</parent>`这部分内容可以直接从scheduleproject/pom.xml中复制过来

`<packaging>pom</packaging>`父工程需要指定打包方式为pom


 ```xml  
  <?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.7.11</version>
        <relativePath/> <!-- lookup parent from repository -->
    </parent>

    <groupId>com.example</groupId>
    <artifactId>MultDataSourceSchedule</artifactId>
    <packaging>pom</packaging>
    <version>1.0-SNAPSHOT</version>

    <!--声明子模块-->
    <modules>
        <module>common</module>
        <module>ScheduleProject</module>
        <module>DAO</module>
    </modules>

    <build>
        <!--项目构建后的名称-->
        <finalName>MultDataSourceSchedule</finalName>
        <plugins>
            <plugin>
                <!--spring boot提供的maven打包插件，可直接打包成jar或war-->
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
                <!--如果这里报错，就找本地maven仓库下 org\springframework\boot\spring-boot-starter-web 的版本号-->
                <version>2.7.11</version>
                <configuration>
                    <!--项目的启动类-->
                    <mainClass>
                        com.example.SchedualProject.SchedualProjectApplication
                    </mainClass>
                </configuration>
            </plugin>
        </plugins>
    </build>

</project>
 ```

### 1.3.2 common子模块pom.xml

`<parent>.....</parent>`声明父模块为MultDataSourceSchedule

`<packaging>jar</packaging>`指定打包方式为jar

`<classifier>exec</classifier>`这里需要注意，不能漏掉，否则会报错

***<font color= red>maven多项目打包报错---子模块相互依赖打包时所遇到的问题:依赖的程序包找不到 package xxx does not exist</font>***

**原因：spring-boot-maven-plugin打包出的jar是不可依赖的**

官方给出的解决方式就是添加这一行配置，那么打包时会生成两个jar包，一个用来执行即common-exec.jar，一个用来依赖即common.jar


```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <parent>
        <artifactId>MultDataSourceSchedule</artifactId>
        <groupId>com.example</groupId>
        <version>1.0-SNAPSHOT</version>
    </parent>
    <modelVersion>4.0.0</modelVersion>

    <name>common</name>
    <artifactId>common</artifactId>
    <packaging>jar</packaging>

    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter</artifactId>
        </dependency>

        <dependency>
            <groupId>org.slf4j</groupId>
            <artifactId>slf4j-api</artifactId>
            <version>1.7.25</version>
        </dependency>
        <dependency>
            <groupId>ch.qos.logback</groupId>
            <artifactId>logback-classic</artifactId>
            <version>1.2.10</version>
        </dependency>

    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
                <!--如果这里报错，就找本地maven仓库下 org\springframework\boot\spring-boot-starter-web 的版本号-->
                <version>2.7.11</version>
                <configuration>
                    <classifier>exec</classifier>
                </configuration>
            </plugin>
        </plugins>
    </build>

</project>

```

### 1.3.3 dao子模块pom.xml

dao模块是持久化模块，需要引入Mysql，Mybatis，Druid

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xmlns="http://maven.apache.org/POM/4.0.0"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <parent>
        <artifactId>MultDataSourceSchedule</artifactId>
        <groupId>com.example</groupId>
        <version>1.0-SNAPSHOT</version>
    </parent>
    <modelVersion>4.0.0</modelVersion>

    <artifactId>DAO</artifactId>
    <packaging>jar</packaging>

    <name>DAO</name>

    <dependencies>
        <!-- https://mvnrepository.com/artifact/mysql/mysql-connector-java -->
        <dependency>
            <groupId>mysql</groupId>
            <artifactId>mysql-connector-java</artifactId>
            <version>8.0.33</version>
        </dependency>
        <!-- https://mvnrepository.com/artifact/com.alibaba/druid -->
        <dependency>
            <groupId>com.alibaba</groupId>
            <artifactId>druid</artifactId>
            <version>1.2.17</version>
        </dependency>

        <!-- https://mvnrepository.com/artifact/org.mybatis.spring.boot/mybatis-spring-boot-starter -->
        <dependency>
            <groupId>org.mybatis</groupId>
            <artifactId>mybatis</artifactId>
            <version>3.4.6</version>
        </dependency>

        <dependency>
            <groupId>com.example</groupId>
            <artifactId>common</artifactId>
            <version>1.0-SNAPSHOT</version>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
                <!--如果这里报错，就找本地maven仓库下 org\springframework\boot\spring-boot-starter-web 的版本号-->
                <version>2.7.11</version>
                <configuration>
                    <classifier>exec</classifier>
                </configuration>
            </plugin>
        </plugins>
    </build>

</project>
```

### 1.3.4 ScheduleProject模块pom.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <parent>
        <artifactId>MultDataSourceSchedule</artifactId>
        <groupId>com.example</groupId>
        <version>1.0-SNAPSHOT</version>
    </parent>

    <artifactId>ScheduleProject</artifactId>
    <packaging>jar</packaging>

    <version>0.0.1-SNAPSHOT</version>
    <name>ScheduleProject</name>
    <description>ScheduleProject</description>

    <properties>
        <java.version>8</java.version>
    </properties>
    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-quartz</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>

        <!-- https://mvnrepository.com/artifact/org.springframework.boot/spring-boot-starter-aop -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-aop</artifactId>
        </dependency>

        <dependency>
            <groupId>com.example</groupId>
            <artifactId>DAO</artifactId>
            <version>1.0-SNAPSHOT</version>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
                <!--如果这里报错，就找本地maven仓库下 org\springframework\boot\spring-boot-starter-web 的版本号-->
                <version>2.7.11</version>
            </plugin>
        </plugins>
    </build>

</project>

```

# 2.Mybatis+Druid实现多数据源的动态切换
在开发过程中，不可避免需要对多个数据库进行读写的请求，不论是使用备库做读写分离，还是不同业务数据存储在不同的库中，都需要配置多个数据源。

[Druid](https://github.com/alibaba/druid)是来源于阿里的一个开源的连接池，提供了非常优秀的监控和扩展功能，而且监控不影响性能。

本项目对Mybatis和Druid的配置全部放在了xml文件中，以配置两个数据源为例。

## 2.1 Mybatis配置
首先在config包下新建mybatis-config.xml和mybatis-config-business.xml，每一个文件中都配置一个数据库的信息

`<settings>...</settings>` 对Mybatis非常重要的配置，会改变Mybatis的运行时行为，不能出现拼写错误，否则会报错，具体配置项可以查看org.apache.ibatis.session.Configuration

```xml
    <typeAliases>
        <typeAlias type="com.example.DruidDataSourceFactory" alias="Druid"/>
    </typeAliases>
    <environments default="development">
        <environment id="development">
            <transactionManager type="JDBC"/>
            <dataSource type="Druid">
            .......
```
dataSource表示Mybatis获取连接的方式，内置三个值unPooled,pooled和JNDI。

- unpooled：非连接池，直接返回一个连接
- pooled：连接池
- JNDI：使用了JNDI技术，每个服务器对应的连接池都不一样，tomcat使用的dpcp连接池。这个属性只能在web工程中使用


本项目使用Druid连接池，因此需要自定义一个Druid的工厂类，这个会在后面
[实现druid数据源工厂类](#span-id-jump22-实现druid数据源工厂类span) 部分完成。

这里只列出mybatis-config.xml的配置信息，mybatis-config-business.xml可以参照进行修改
```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE configuration
        PUBLIC "-//mybatis.org//DTD Config 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-config.dtd">
<configuration>
    <settings>
      <setting name="defaultStatementTimeout" value="10"/>
      <setting name="defaultFetchSize" value="300"/>
      <setting name="jdbcTypeForNull" value="NULL"/>
    </settings>
    <typeAliases>
        <typeAlias type="com.example.DruidDataSourceFactory" alias="Druid"/>
    </typeAliases>
    <environments default="development">
        <environment id="development">
            <transactionManager type="JDBC"/>
            <dataSource type="Druid"><!--指定第三方数据源Druid-->
                <property name="id" value="business"/><!--指定数据源id-->
                <property name="driver" value="com.mysql.cj.jdbc.Driver"/>
                <property name="url" value="jdbc:mysql://127.0.0.1:3306/mysimpledemodb"/>
                <property name="username" value="root"/>
                <property name="password" value="123456"/>
                <!--最大连接数-->
                <property name="maxActive" value="5"/>
                <!--初始连接数-->
                <property name="initSize" value="5"/>
                <!--最小空闲连接数-->
                <property name="minIdle" value="5"/>
                <!--从连接池获取连接等待超时的时间  单位毫秒-->
                <property name="maxWait" value="5000"/>
                <!--配置连接在池中最小空闲时间  单位毫秒-->
                <property name="minIdleTime" value="300000"/>
                <!--检查连接有效性，true-空闲时间过长时检查  false-不检查-->
                <property name="checkWhileIdle" value="true"/>
                <!--检查连接有效性的sql   只对MySQL有效  Oracle使用select 1 from dual-->
                <property name="validSQL" value="select 1"/>
                <!--true-检查minIdle内的空闲连接数，每次检查强制验证连接有效性-->
                <property name="keepAlive" value="true"/>
                <!--解决网络异常时业务线程阻塞在请求连接的过程-->
                <property name="failFast" value="true"/>
            </dataSource>
        </environment>
    </environments>
    <mappers>
        <mapper resource="StudentMapper.xml"/>
    </mappers>
</configuration>

```



## <span id ="jump">2.2 实现Druid数据源工厂类</span>
由于Mybatis配置中指定了第三方数据源Druid，因此需要实现一个Druid数据源的工厂类。
关于自定义DruidDataSourceFactory的方式有很多种：
- **<font color = darkred>extends DruidDataSourceFactory implements DataSourceFactory</font>** ：这种方式继承了Druid中已经实现的DruidDataSourceFactory并且实现接口DataSourceFactory，这种方式可以对Druid中的DruidDataSourceFactory进行一些个性化定制或者扩展，会额外增加代码量，可直接配置web.xml来使用Druid监控，后面两种方式可能需要自定义监控逻辑
- **<font color = darkred>extends UnpooledDataSourceFactory</font>** ：这种方式简单明了,因为UnpooledDataSourceFactory已经实现了获取UnpooledDataSource 的属性，因此只需覆盖掉需要的属性即可
- **<font color = darkred>implements DataSourceFactory</font>** ：这种方式需要实现setProperties和getDataSource方法，可以很灵活地配置DruidDataSource

这里要注意的是，第一种方式在mybatis-config.xml文件中，property的name如果没有与Druid规定的属性名字保持一致，则需要对该属性进行覆盖，否则会报错；第二种方式也同样，如果没有与Mybatis规定的属性名一致且没有对属性进行覆盖，也会报错。

第三种方式由于是自定义getDataSource的实现逻辑，因此property的name可以自定义。

这里采用第三种方式，实现DataSourceFactory来自定义DruidDataSourceFactory。

```java
/**
 * 自定义Druid数据源工厂类
 * new SqlSessionFactoryBuilder().build的时候会进入这里
 */
public class DruidDataSourceFactory implements DataSourceFactory {

    //配置信息
    private Properties properties;
UnpooledDataSourceFactory
    /**
     * 已配置的数据源集合
     */
    private static ConcurrentHashMap<String, DruidDataSource> dataSourceMap = new ConcurrentHashMap<>();

    @Override
    public void setProperties(Properties properties) {
        this.properties = properties;
    }

    @Override
    public DataSource getDataSource() {
        DruidDataSource druidDataSource = new DruidDataSource();

        //获取xml配置内容
        druidDataSource.setDriverClassName(this.properties.getProperty("driver"));
        String key = this.properties.getProperty("id");
        if (dataSourceMap.containsKey(key)){
            dataSourceMap.get(key).close();
        }
        druidDataSource.setUrl(this.properties.getProperty("url"));
        druidDataSource.setUsername(this.properties.getProperty("username"));
        druidDataSource.setPassword(this.properties.getProperty("password"));

        int maxActive = Integer.parseInt(this.properties.getProperty("maxActive"));
        druidDataSource.setMaxActive(maxActive);

        int initSize = Integer.parseInt(this.properties.getProperty("initSize"));
        druidDataSource.setInitialSize(initSize);

        int minIdle = Integer.parseInt(this.properties.getProperty("minIdle"));
        druidDataSource.setMinIdle(minIdle);

        long maxWait = Long.parseLong(this.properties.getProperty("maxWait"));
        druidDataSource.setMaxWait(maxWait);

        boolean failFast = Boolean.parseBoolean(this.properties.getProperty("failFast"));
        druidDataSource.setFailFast(failFast);

        try {
            //添加stat过滤器，用于统计监控信息，例如SQL执行时间、访问次数、连接池的活跃连接数、请求连接数、最大连接数等
            druidDataSource.addFilters("stat");
            //Spring Boot 默认的日志系统是 logback，并没有依赖 log4j，这里配置的 log4j 会报错。要想使用 log4j filter 要把日志系统切换为 log4j。
            //druidDataSource.addFilters("log4j");
            //添加config过滤器，用于配置连接池的一些参数，例如初始连接数、最大连接数、最小连接数等
            druidDataSource.addFilters("config");
            druidDataSource.init();
            dataSourceMap.put(key,druidDataSource);
        } catch (SQLException ex){
            ex.printStackTrace();
            druidDataSource.close();
            return null;
        }
        return druidDataSource;
    }
}
```

## 2.3 初始化Mybatis

项目中每一个数据源配置信息都分别存在不同的mybatis-config.xml文件中，需要在创建SqlSessionFactory实例时将每一个配置文件都传入，随着业务的发展可能会新增数据源，因此不能在代码中全部罗列出来再创建SqlSessionFactory对象。

可以采用这样一种解决方案：

- 1.创建一个常量类DataBaseConfigKey用于存储datasourceId
- 2.配置datasourceId与mybatis-config.xml文件的映射关系
- 3.通过反射遍历常量类DataBaseConfigKey的属性来获取配置文件并读取


在common模块下新建常量类DataBaseConfigKey
```java
public class DataBaseConfigKey {
    /**
     * 业务类数据库1
     */
    public static final String BUSINESSDATASOURCE = "datasource.business";

    /**
     * 业务类数据库2
     */
    public static final String BUSINESSDATASOURCE2 = "datasource.business2";
}
```

在config/application.properties中添加mybatis配置文件，用来指定每个mybatis配置文件的datasourceId
```properties
# =============DataSource Config===========
datasource.business = mybatis-config.xml
datasource.business2 = mybatis-config-business2.xml
```

在dao模块下创建一个MybatisHelper类，用于管理SqlSession，初始化的逻辑如下：
- 通过反射获取DataBaseConfigKey的成员变量的值，即dataSourceId
- application.properties文件中通过dataSourceId找到对应的mybatis-config.xml配置文件
- 读取xml文件内容，建立SqlSessionFactory并放入Mybatis集合中

首先创建一个工具类，common/CommonUtil，读取applicaition.properties内容的方法就在这个工具类中实现

```java
public class CommonUtil {


    private static Properties properties = null;

    /**
     * 初始化配置信息
     */
    private static void initConfiguration(){

        try {
            //读取配置文件
            Resource resource = new FileSystemResource("config/application.properties");
            if (!resource.exists()){
                resource = new ClassPathResource("application.properties");
            }
            //载入配置文件至properties
            properties = PropertiesLoaderUtils.loadProperties(resource);
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    /**
     * 根据配置文件中的key获取对应的value
     * @param key
     * @return
     */
    public static String getProperties(String key){
        if (properties == null){
            //还未载入配置文件
            initConfiguration();
        }
        return properties.getProperty(key);
    }
}
```

接下来按照上面的初始化逻辑来实现对Mybtais的初始化

```java
public class MybatisHelper {

    /**
     * 已经建好的sqlsessionfactory集合
     */
    public static Map<String, SqlSessionFactory> sqlSessionFactoryMap = new ConcurrentHashMap<>();

    /**
     * 根据application.properties配置文件中的配置key获取对应的sqlsessionfactory
     * @param dataSourceName 配置文件中的配置key
     * @return
     */
    public static SqlSessionFactory getSqlSessionFactory(String dataSourceName){
        //在配置文件中根据datasourcename获取对应的mybatis配置文件名
        String configFileName = ConfigUtil.getProperties(dataSourceName);
        //根据mybatis配置文件名创建对应的sqlsessionfactory
        return getSqlSessionFactoryByFileName(configFileName);
    }

    /**
     * 根据指定文件名获取对应的sqlsessionfactory
     * @param fileName  mybatis配置文件名
     * @return
     */
    public static SqlSessionFactory getSqlSessionFactoryByFileName(String fileName){
        SqlSessionFactory sqlSessionFactory = null;
        Resource resource;
        if (StringUtils.hasText(fileName)){
            resource = new FileSystemResource("config/"+fileName);
            if (!resource.exists()){
                resource = new ClassPathResource(fileName);
            }
            try {
                //通过sqlsessionfactorybuilder建立SqlSessionFactory
                //build的时候会执行自定义的DruidDataSourceFactory里面对mybatis-config文件的的读取
                sqlSessionFactory = new SqlSessionFactoryBuilder().build(resource.getInputStream());
            } catch (IOException e) {
                e.printStackTrace();
            }
        } else {
            System.out.println("文件"+fileName+"未配置");
        }
        return sqlSessionFactory;
    }

    /**
     * 初始化mybatis集合
     */
    public static void InitMybatisMap() throws NoSuchMethodException, InvocationTargetException, InstantiationException, IllegalAccessException {
        //通过反射获取所有mybatis数据库配置
        Class<DataBaseConfigKey> clazz = DataBaseConfigKey.class;
        Constructor constructor = clazz.getConstructor();
        Object obj = constructor.newInstance();//获取DataBaseConfigKey对象的实例

        Field[] fields = clazz.getFields();//获取public变量
        for (Field f :fields){
            long begin = System.currentTimeMillis();
            String objValue = (String) f.get(obj);//获取实例obj的成员变量的值
            if (objValue != null){
                if (!sqlSessionFactoryMap.containsKey(objValue)){
                    //objValue数据源对应的sqlSessionFactory还未建立
                    SqlSessionFactory sqlSessionFactory = getSqlSessionFactory(objValue);
                    if (sqlSessionFactory != null){
                        sqlSessionFactoryMap.put(objValue,sqlSessionFactory);
                        LogUtil.LogError(String.format("Mybatis %s 初始化成功，耗时: %s!",objValue,System.currentTimeMillis()-begin));
                    } else {
                        LogUtil.LogError(String.format("Mybatis %s 初始化失败，耗时: %s!",objValue,System.currentTimeMillis()-begin));
                    }
                }
            }
        }
    }
}
```

需要在启动类中手动添加初始化Mybatis逻辑

```java
@SpringBootApplication
public class SchedualProjectApplication {


    public static void main(String[] args) {

        try {
            SpringApplication.run(SchedualProjectApplication.class, args);

            //可以根据需要配置不同的日志
            try {
                LogUtil.Init();
            } catch (Exception e){
                LogUtil.LogError(e.getMessage(),e);
            }

            //初始化mybatis集合
            MybatisHelper.InitMybatisMap();


        } catch (Exception e) {
            e.printStackTrace();
        }
    }

}
```

## 2.4 实现多数据源手动切换
上面已经实现了  加载Myatis配置->构建SqlSessionFactory   的过程，现在就需要通过SqlSessionFactory中获取SqlSession。

SqlSession 提供了在数据库执行 SQL 命令所需的所有方法。可以通过 SqlSession 实例来直接执行已映射的 SQL 语句。

首先实现获取SqlSession的方法，这个方法添加到MybatisHelper中

```java
/**
     * 根据datasoureid获取sqlsession
     * @param datasourceId
     * @return
     */
    public static SqlSession getSqlSession(String datasourceId){
        if (sqlSessionFactoryMap.containsKey(datasourceId)){
            if (sqlSessionFactoryMap.get(datasourceId) == null){
                //mybatis中对应的sqlsessionfactory为空，手动获取并放入集合中
                SqlSessionFactory sessionFactory = getSqlSessionFactory(datasourceId);
                if (sessionFactory != null){
                    sqlSessionFactoryMap.put(datasourceId,sessionFactory);
                    return sessionFactory.openSession();
                } else {
                    return null;
                }
            } else {
                //获取到了sqlsessionFactory实例，获取对应的SqlSession
                return sqlSessionFactoryMap.get(datasourceId).openSession();
            }
        } else {
            //没有获取到datasoruce对应的sqlsessionFactory，手动加载
            SqlSessionFactory sessionFactory = getSqlSessionFactory(datasourceId);
            if (sessionFactory != null){
                sqlSessionFactoryMap.put(datasourceId,sessionFactory);
                return sessionFactory.openSession();
            } else {
                return null;
            }
        }
    }
```
数据表的创建以及初始数据的插入不再赘述

接下来创建映射器mapper，mapper.xml包含了SQL代码和映射定义信息，mapper.xml文件放在config下，mapper接口类放在Dao中，映射器需要添加到对应数据源配置文件的<mappers></mappers>元素下

``` xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE mapper
        PUBLIC "-//mybatis.org//DTD mapper 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="com.example.scheduleproject.mappers.StudentMapper">

    <select id="getStudentInfoByStudentNum" resultType="com.example.scheduleproject.model.Student">
        select stuNum,name,classNum,gender,age from student where stuNum = #{studentNum}
    </select>

</mapper>
```
```java
@Mapper
public interface StudentMapper {

    /**
     * 根据学号查询学生信息
     * @param studentNum
     * @return
     */
    public Student getStudentInfoByStudentNum(@Param("studentNum") String studentNum);
}
```

现在可以在启动类中测试是否可以成功执行了

```java
    public static void main(String[] args) {

        try {
            SpringApplication.run(SchedualProjectApplication.class, args);
            try {
                Logger.Init();
            } catch (Exception e){
                Logger.LogError(e.getMessage(),e);
            }


            //初始化mybatis集合
            MybatisHelper.InitMybatisMap();

            SqlSession sqlSession = MybatisHelper.getSqlSession(DataBaseConfigKey.BUSINESSDATASOURCE);
            StudentMapper mapper = sqlSession.getMapper(StudentMapper.class);
            Student student = mapper.getStudentInfoByStudentNum("20220101");
            System.out.println(student.toString());
            sqlSession.close();

            SqlSession session = MybatisHelper.getSqlSession(DataBaseConfigKey.BUSINESSDATASOURCE2);
            CityMapper cityMapper = session.getMapper(CityMapper.class);
            City city = cityMapper.getCityProvince("武汉");
            System.out.println(city.toString());
            session.close();


        } catch (Exception e) {
            e.printStackTrace();
        }
    }
```

执行结果如下，实现了手动切换数据源的功能

![](https://raw.githubusercontent.com/yasmin1024/hexoBlogImgRepository/main/postImg/01/%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_20230428184758.png)

## 2.5 实现多数据源动态切换


# 3.Quartz定时任务
