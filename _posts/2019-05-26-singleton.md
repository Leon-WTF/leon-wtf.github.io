---
title: 多线程安全的单例模式
category: Java
tags: [java, concurrency, design-pattern]
---
单例模式被认为是最简单的设计模式，属于创建型(设计模式又被分为：创建型、结构型和行为型)，经常被用到，下面以我在实际项目中用到的一个单例模式为例，看下如何利用经典的两次判空方法令其高效、安全得工作在多线程环境（见代码中注释）。

```java
package core;

import org.apache.ibatis.io.Resources;
import org.apache.ibatis.session.SqlSessionFactory;
import org.apache.ibatis.session.SqlSessionFactoryBuilder;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import java.io.IOException;
import java.io.Reader;
import java.util.Properties;

public class SqlSessionFactorySingleton {
    private static Logger logger = LoggerFactory.getLogger("SqlSessionFactorySingleton");
    private static final String MYBATIS_CONFIG_FILE = "mybatis-config.xml";
    // 使用volatile关键字令A线程的修改对B线程立即可见
    private static volatile SqlSessionFactory factory = null;
    
    // 屏蔽默认的公共构造函数
    private SqlSessionFactorySingleton() {
    }

    public static SqlSessionFactory getInstance() {
        if (factory == null) { // 第一次判空,如果不使用volatile，B线程可能看到的是A线程创建了一半的对象
            // 只有创建SqlSessionFactory实例时才需要同步，不直接在方法上加synchronized关键字可以避免在每次判断实例是否创建时加锁，极大得提高并发效率
            synchronized (SqlSessionFactorySingleton.class) {
                // 如果A、B两个线程同时通过第一次判空，A获得锁，B等待，等A创建完SqlSessionFactory实例释放锁，B获得锁，此时B需要再次判断实例是否已创建来避免重复创建
                if (factory == null) { // 第二次判空
                    String configFile = "config.properties";
                    try (Reader configReader = Resources.getResourceAsReader(configFile); Reader mybatisReader = Resources.getResourceAsReader(MYBATIS_CONFIG_FILE)) {
                        Properties properties = new Properties();
                        properties.load(configReader);
                        SqlSessionFactoryBuilder builder = new SqlSessionFactoryBuilder();
                        factory = builder.build(mybatisReader, properties);
                    } catch (IOException e) {
                        logger.error("Exception when reading {} and {}:", configFile, MYBATIS_CONFIG_FILE, e);
                    }
                }
            }
        }
        return factory;
    }
}
```
更好的可以利用如下方法, 利用JVM在类初始化时创建SqlSessionFactory，同时与JVM的延迟加载机制结合来实现延迟初始化。
```java
package core;

import org.apache.ibatis.io.Resources;
import org.apache.ibatis.session.SqlSessionFactory;
import org.apache.ibatis.session.SqlSessionFactoryBuilder;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import java.io.IOException;
import java.io.Reader;
import java.util.Properties;

public class SqlSessionFactorySingleton {
    private static Logger logger = LoggerFactory.getLogger("SqlSessionFactorySingleton");
    private static final String MYBATIS_CONFIG_FILE = "mybatis-config.xml";
    private static final Boolean IS_DEBUG = "yes".equalsIgnoreCase(System.getProperty("DEBUG"));
    private static SqlSessionFactory factory;
    static {
        String configFile = "config.properties";
        if (IS_DEBUG) {
            configFile = "config_test.properties";
        }
        try (Reader configReader = Resources.getResourceAsReader(configFile); Reader mybatisReader = Resources.getResourceAsReader(MYBATIS_CONFIG_FILE)) {
            Properties properties = new Properties();
            properties.load(configReader);
            SqlSessionFactoryBuilder builder = new SqlSessionFactoryBuilder();
            factory = builder.build(mybatisReader, properties);
        } catch (IOException e) {
            logger.error("Exception when reading {} and {}:", configFile, MYBATIS_CONFIG_FILE, e);
        }
    }
    
    private SqlSessionFactorySingleton() {
    }
    
    public static SqlSessionFactory getInstance() {
        return factory;
    }
}
```
或者还可以使用如下“延迟初始化占位类模式”：
```java
package core;

import org.apache.ibatis.io.Resources;
import org.apache.ibatis.session.SqlSessionFactory;
import org.apache.ibatis.session.SqlSessionFactoryBuilder;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import java.io.IOException;
import java.io.Reader;
import java.util.Properties;

public class SqlSessionFactorySingleton {
    private static Logger logger = LoggerFactory.getLogger("SqlSessionFactorySingleton");

    private static class SqlSessionFactoryHolder {
        private static final String MYBATIS_CONFIG_FILE = "mybatis-config.xml";
        private static final Boolean IS_DEBUG = "yes".equalsIgnoreCase(System.getProperty("DEBUG"));
        private static SqlSessionFactory factory;
        static {
            String configFile = "config.properties";
            if (IS_DEBUG) {
                configFile = "config_test.properties";
            }
            try (Reader configReader = Resources.getResourceAsReader(configFile); Reader mybatisReader = Resources.getResourceAsReader(MYBATIS_CONFIG_FILE)) {
                Properties properties = new Properties();
                properties.load(configReader);
                SqlSessionFactoryBuilder builder = new SqlSessionFactoryBuilder();
                factory = builder.build(mybatisReader, properties);
            } catch (IOException e) {
                logger.error("Exception when reading {} and {}:", configFile, MYBATIS_CONFIG_FILE, e);
            }
        }
    }

    private SqlSessionFactorySingleton() {
    }

    public static SqlSessionFactory getInstance() {
        return SqlSessionFactoryHolder.factory;
    }
}
```