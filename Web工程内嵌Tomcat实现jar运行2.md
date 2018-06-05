
# Web工程内嵌Tomcat实现jar运行

自从Spring boot出来后，他的嵌入式Tomcat，Jetty容器，无需部署WAR包的方式，也是受很开发运维人员的喜爱，但是碰到旧的web项目进行部署，习惯了spring boot启动方式，还是感 觉到有点狗血的，所以如何将web工程改成java -jar xx.jar 方式启动，可以使用Embeded-Tomcat,内嵌入Tomcat来达到我们的目的， 下面就分享一下内嵌tomcat的方式。

---


## 1.加入内嵌Tomcat依赖

```
<dependency>
	<groupId>org.apache.tomcat.embed</groupId>
	<artifactId>tomcat-embed-core</artifactId>
	<version>8.5.5</version>
</dependency>
<dependency>
	<groupId>org.apache.tomcat.embed</groupId>
	<artifactId>tomcat-embed-el</artifactId>
	<version>8.5.5</version>
</dependency>
<dependency>
	<groupId>org.apache.tomcat.embed</groupId>
	<artifactId>tomcat-embed-jasper</artifactId>
	<version>8.5.5</version>
</dependency>
```

## 2.配置文件

![](https://i.imgur.com/duhVjYz.png)


application-local.properties
```
# Tomcat settings
tomcat.port=28080
tomcat.basedir=E:/fhcb10/news-web/tomcat/basedir


```
application-rc.properties

```
#Tomcat settings
tomcat.port=28080
tomcat.basedir=/data/fhcb10/news-web/tomcat/

```

## 3.添加加载配置的类

```
import org.apache.commons.configuration.Configuration;
import org.apache.commons.configuration.PropertiesConfiguration;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

/**  
 *    
 * 环境配置加载类
 * @author huangyan 
 * @date 2018/6/5 11:38  
 */
public class EnvConfig {
    private final static Logger log = LoggerFactory.getLogger(EnvConfig.class);

    public static String port = null;
    public static String basedir = null;
    public static String filepath = null;

   /**
    *    
    * 初始化加载配置
    * @author
    * @date 2018/6/5 11:25  
    * @param []  
    * @return boolean  
    */  
    public static boolean init() {

        Configuration config;
        try {
            String env = System.getProperty("env");
            if (env == null) {
                log.info("没有配置环境，使用本地配置local");
                env = "local";
            }
            log.info("当前的环境是: " + env);
            String fileName = "application" + "-" + env + ".properties";

            config = new PropertiesConfiguration(fileName);
            port = config.getString("tomcat.port");
            if(port == null || port.isEmpty()) {
                port = "8080";
            }
            basedir = config.getString("tomcat.basedir");
            filepath = config.getString("filepath");

            log.info("==========================================");
            log.info("                    CONFIG                ");
            log.info("==========================================");
            log.info("port: " + port);
            log.info("docbase : " + basedir);
            log.info("filepath : " + filepath);

            return true;
        } catch (Exception e) {
            log.error(e.getMessage(), e);
            return false;
        }
    }
}

```
# 4.内嵌Tomcat启动类

```
package com.twsm.embededtomcat;


import com.twsm.embededtomcat.config.EnvConfig;
import org.apache.catalina.core.StandardContext;
import org.apache.catalina.startup.Tomcat;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import java.io.File;

/**  
 *    
 *  内嵌Tomcat配置启动主类
 * @author huangyan 
 * @date 2018/6/5 11:38  
 */
public class NewsWebMain {
    private static Logger log = LoggerFactory.getLogger(NewsWebMain.class);

    /**  
     *    
     *  Tomcat 启动主类方法
     * @author huangyan 
     * @date 2018/6/5 11:39
     * @param [args]  
     * @return void  
     */  
    public static void main(String[] args) throws Exception {
        try {
            if (!EnvConfig.init()) {
                log.info("加載配置文件失敗。");
                System.exit(0);
            }
            // 1.创建一个内嵌的Tomcat
            Tomcat tomcat = new Tomcat();

            // 2.设置Tomcat端口默认为8080
            final Integer webPort = Integer.parseInt(EnvConfig.port);
            tomcat.setPort(Integer.valueOf(webPort));

            // 3.设置工作目录,tomcat需要使用这个目录进行写一些东西
            final String baseDir = EnvConfig.basedir;
            tomcat.setBaseDir(baseDir);
            tomcat.getHost().setAutoDeploy(false);

            // 4. 设置webapp资源路径
            String webappDirLocation = "webapp/";
            StandardContext ctx = (StandardContext) tomcat.addWebapp("/", new File(webappDirLocation).getAbsolutePath());
            log.info("configuring app with basedir: " + new File("" + webappDirLocation).getAbsolutePath());
            log.info("project dir:"+new File("").getAbsolutePath());

            // 5. 设置上下文路每径
            String contextPath = "";
            ctx.setPath(contextPath);
            ctx.addLifecycleListener(new Tomcat.FixContextListener());
            ctx.setName("news-web");

            System.out.println("child Name:" + ctx.getName());
            tomcat.getHost().addChild(ctx);

           /* File additionWebInfClasses = new File("");
            WebResourceRoot resources = new StandardRoot(ctx);
            resources.addPreResources(new DirResourceSet(resources, "/WEB-INF/classes",
                    additionWebInfClasses.getAbsolutePath() + "/classes", "/"));
            ctx.setResources(resources);
            */

            log.info("服务器加载完配置，正在启动中……");
            tomcat.start();
            log.info("服务器启动成功");
            tomcat.getServer().await();
        } catch (Exception exception) {
            log.error("服务器启动失敗", exception);
        }
    }
}

```

5.Maven打包插件配置和资源配置

```
<build>
		<finalName>${artifactId}</finalName>
		<plugins>
            <!-- 配置启动类 -->
			<plugin>
				<groupId>org.apache.maven.plugins</groupId>
				<artifactId>maven-jar-plugin</artifactId>
				<version>3.0.2</version>
				<configuration>
					<archive>
						<manifest>
							<addClasspath>true</addClasspath>
							<classpathPrefix>lib/</classpathPrefix>	<mainClass>com.twsm.embededtomcat.NewsWebMain</mainClass>
						</manifest>
					</archive>
				</configuration>
			</plugin>

            <!-- 打包加入依赖的包 -->
			<plugin>
				<groupId>org.apache.maven.plugins</groupId>
				<artifactId>maven-dependency-plugin</artifactId>
				<version>2.10</version>
				<executions>
					<execution>
						<id>copy-dependencies</id>
						<phase>package</phase>
						<goals>
							<goal>copy-dependencies</goal>
						</goals>
						<configuration>
							<outputDirectory>${project.build.directory}/lib</outputDirectory>

						</configuration>
					</execution>
				</executions>
			</plugin>

		</plugins>

        <!-- 资源输出路径配置 -->
		<resources>
			<resource>
				<directory>src/main/java</directory>
				<includes>
					<include>**/*.properties</include>
				</includes>
				<!-- 是否替换资源中的属性 -->
				<filtering>false</filtering>
			</resource>
			<resource>
				<directory>src/main/resources</directory>
				<includes>
					<include>**/*.*</include>
				</includes>
				<filtering>false</filtering>
			</resource>
			<resource>
				<directory>src/main/webapp</directory>
				<targetPath>${basedir}/target/webapp</targetPath>
				<includes>
					<include>**/*.*</include>
				</includes>
				<filtering>false</filtering>
			</resource>
		</resources>
	</build>
```

## 5.修改pom文件中的打包类型 

```
<packaging>jar</packaging>
```

## 6.在父工程目录编译打包

```
mvn clean package -Dmaven.test.skip=true
```

打包完后的target目录结构如下：
![](https://i.imgur.com/zGA4jtc.png)


将target目录下的，lib,webapp,news-web.jar和classes下的所有配置文件拷贝到任一目录下，如下图：

![](https://i.imgur.com/ezfUbSY.png)

## 7.启动运行应用

```
java -jar Denv=rc news-web.jar 
```
执行以上命令就可以看到启动的日志和启动成功，如下图：
![](https://i.imgur.com/GkhVxXH.png)
![](https://i.imgur.com/sfQUR1Q.png)

## 8.在linux的启动脚本

```
nohup java -Xmx500m -Xss64m -jar -Denv=rc  news-web.jar > log.txt &
tail -f log.txt
```
