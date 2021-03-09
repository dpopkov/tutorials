Liquibase для эволюции схемы базы данных
========================================

Содержание
----------
* [Обзор](#1--обзор)
* [Change Log базы данных](#2---change-log-базы-данных)
* [Запуск Liquibase через Spring Bean](#3---запуск-liquibase-через-spring-bean)
* [Использование Liquibase вместе со Spring Boot](#4---использование-liquibase-вместе-со-spring-boot)
* [Отключение Liquibase в Spring Boot](#5---отключение-liquibase-в-spring-boot)
* [Генерация changeLog с помощью Maven Plugin](#6---генерация-changelog-с-помощью-maven-plugin)
    * [Конфигурация плагина](#61---конфигурация-плагина)
    * [Генерация ChangeLog из существующей базы данных](#62---генерация-changelog-из-существующей-базы-данных)
    * [Генерация ChangeLog из разницы между двумя базами данных](#63---генерация-changelog-из-разницы-между-двумя-базами-данных)
* [Использования Liquibase Hibernate Plugin](#7---использования-liquibase-hibernate-plugin)
    * [Конфигурация плагина](#71---конфигурация-плагина)
    * [Генерация changeLog из разницы между Базой Данных и Persistence Entities](#72---генерация-changelog-из-разницы-между-базой-данных-и-persistence-entities)
* [Заключение](#8---заключение)

1 - Обзор
---------
В этом руководстве Liquibase будет использоваться для эволюционирования схемы базы данных 
Java веб приложения.  
Сначала рассмотрим само приложение, а затем сфокусируемся на некоторых интересных возможностях
доступных для Spring и Hibernate.  
В основе использования Liquibase лежит __changeLog__ файл - это XML файл, который отслеживает все 
изменения необходимые для обновления базы данных.  
В проект необходимо добавить Maven зависимость:
```xml
<dependency>
    <groupId>org.liquibase</groupId>
     <artifactId>liquibase-core</artifactId>
      <version>4.3.1</version>
</dependency>
```
[Тут](https://mvnrepository.com/artifact/org.liquibase/liquibase-core) можно поискать последнюю версию.

[Вверх](#содержание)

2 - Change Log базы данных
-------------------------- 
Взглянем на простой changeLog файл, который добавляет столбец `address` в таблицу `users`:
```xml
<databaseChangeLog 
  xmlns="http://www.liquibase.org/xml/ns/dbchangelog" 
  xmlns:ext="http://www.liquibase.org/xml/ns/dbchangelog-ext" 
  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" 
  xsi:schemaLocation="http://www.liquibase.org/xml/ns/dbchangelog-ext
   http://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-ext.xsd 
   http://www.liquibase.org/xml/ns/dbchangelog 
   http://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-3.4.xsd">
    
    <changeSet author="John" id="someUniqueId">
        <addColumn tableName="users">
            <column name="address" type="varchar(255)" />
        </addColumn>
    </changeSet>
    
</databaseChangeLog>
```
Change set идентифицируется по атрибутам _id_ и _author_ - чтобы сделать его уникальным и применимым только 
один раз.  
Теперь посмотрим как добавить это в приложение и удостовериться, 
что он отрабатывает во время старта приложения. 

[Вверх](#содержание)

3 - Запуск Liquibase через Spring Bean
--------------------------------------
Первым вариантом запуска изменений во время старта приложения является Spring bean.
Существуют и другие способы, но если речь идет о Spring приложении, то это хороший и простой способ:
```Java
@Bean
public SpringLiquibase liquibase() {
    SpringLiquibase liquibase = new SpringLiquibase();
    liquibase.setChangeLog("classpath:liquibase-changeLog.xml");
    liquibase.setDataSource(dataSource());
    return liquibase;
}
```
Файл changeLog должен присутствовать на classpath.

[Вверх](#содержание)

4 - Использование Liquibase вместе со Spring Boot
-------------------------------------------------
Если используется Spring Boot, то нет нужды определять Bean для Liquibase.
Всё что нужно это положить change log в __db/changelog/db.changelog-master.yaml__
и миграции Liquibase начнуться автоматически при старте.  
Следует обратить внимание:
* Добавить __liquibase-core__ зависимость.
* Можно изменить default change log файл используя __liquibase.change-log__ свойство.
Например:
```properties
liquibase.change-log=classpath:liquibase-changeLog.xml
```

[Вверх](#содержание)

5 - Отключение Liquibase в Spring Boot
--------------------------------------
Иногда требуется отключить миграции Liquibase на старте.
Простейшим вариантом является использование свойства __spring.liquibase.enabled__.
Таким образом вся остальная конфигурация Liquibase остается нетронутой.  
Пример для Spring Boot 2:
```properties
spring.liquibase.enabled=false
```
Для Spring Boot 1.x следует использовать свойство _liquibase.enabled_:
```properties
liquibase.enabled=false
```

[Вверх](#содержание)

6 - Генерация changeLog с помощью Maven Plugin
----------------------------------------------
Вместо написания changeLong файла вручную можно использовать Maven plugin для его генерации.

### 6.1 - Конфигурация плагина

Ниже приведены изменения для _pom.xml_:
```xml
<dependency>
    <groupId>org.liquibase</groupId>
    <artifactId>liquibase-maven-plugin</artifactId>
    <version>4.3.1</version>
</dependency> 
...
<plugins>
    <plugin>
        <groupId>org.liquibase</groupId>
        <artifactId>liquibase-maven-plugin</artifactId>
        <version>4.3.1</version>
        <configuration>                  
            <propertyFile>src/main/resources/liquibase.properties</propertyFile>
        </configuration>                
    </plugin> 
</plugins>
```

[Вверх](#содержание)

### 6.2 - Генерация ChangeLog из существующей базы данных

Мы можем использовать плагин для генерации ChangeLog из существующей базы данных:
```xml
mvn liquibase:generateChangeLog
```
Ниже _liquibase properties_:
```properties
url=jdbc:mysql://localhost:3306/database_name
username=applicationuser
password=applicationmy5ql
driver=com.mysql.jdbc.Driver
outputChangeLogFile=src/main/resources/liquibase-outputChangeLog.xml
```
Результат генерации может быть использован либо для создания начальной схемы базы данных или
для наполнения данных.  
Ниже пример:
```xml
<?xml version="1.0" encoding="UTF-8" standalone="no"?>
<databaseChangeLog ...>
    
    <changeSet author="John (generated)" id="1439225004329-1">
        <createTable tableName="APP_USER">
            <column autoIncrement="true" name="id" type="BIGINT">
                <constraints primaryKey="true"/>
            </column>
            <column name="accessToken" type="VARCHAR(255)"/>
            <column name="needCaptcha" type="BIT(1)">
                <constraints nullable="false"/>
            </column>
            <column name="password" type="VARCHAR(255)"/>
            <column name="refreshToken" type="VARCHAR(255)"/>
            <column name="tokenExpiration" type="datetime"/>
            <column name="username" type="VARCHAR(255)">
                <constraints nullable="false"/>
            </column>
            <column name="preference_id" type="BIGINT"/>
            <column name="address" type="VARCHAR(255)"/>
        </createTable>
    </changeSet>
    ...
</databaseChangeLog>
```

[Вверх](#содержание)

### 6.3 - Генерация ChangeLog из разницы между двумя базами данных

Можно использовать плагин для генерации changeLog из разницы между двумя существующими
базами данных, например development и production:
```
mvn liquibase:diff
```
Ниже properties:
```properties
changeLogFile=src/main/resources/liquibase-changeLog.xml
url=jdbc:mysql://localhost:3306/database_name
username=dbuser
password=dbmy5ql
driver=com.mysql.jdbc.Driver
referenceUrl=jdbc:h2:mem:database_name
diffChangeLogFile=src/main/resources/liquibase-diff-changeLog.xml
referenceDriver=org.h2.Driver
referenceUsername=sa
referencePassword=
```
Ниже фрагмент из сгенерированного changeLog:
```xml
<?xml version="1.0" encoding="UTF-8" standalone="no"?>
<databaseChangeLog ...>
    <changeSet author="John" id="1439227853089-1">
        <dropColumn columnName="address" tableName="APP_USER"/>
    </changeSet>
</databaseChangeLog>
```
Это супер-мощный способ для эволюционирования базы данных. 
Например можно позволить Hibernate авто-сгенерировать новую схему для development,
а потом использовать ее как reference point против старой схемы.

[Вверх](#содержание)

7 - Использования Liquibase Hibernate Plugin
--------------------------------------------
Если приложение использует Hibernate, то может быть применен очень полезный способ 
генерации _changeLog_.  
Для начала вот как [ the liquibase-hibernate plugin](https://github.com/liquibase/liquibase-hibernate/wiki) 
должен быть сконфигурирован в Maven:

### 7.1 - Конфигурация плагина

Конфигурация и зависимость:
```xml
<plugins>
    <plugin>
        <groupId>org.liquibase</groupId>
        <artifactId>liquibase-maven-plugin</artifactId>
        <version>4.3.1</version>
        <configuration>                  
            <propertyFile>src/main/resources/liquibase.properties</propertyFile>
        </configuration> 
        <dependencies>
            <dependency>
                <groupId>org.liquibase.ext</groupId>
                <artifactId>liquibase-hibernate4</artifactId>
                <version>3.5</version>
            </dependency>
            <dependency>
                <groupId>org.springframework</groupId>
                <artifactId>spring-beans</artifactId>
                <version>4.1.7.RELEASE</version>
            </dependency>
            <dependency>
                <groupId>org.springframework.data</groupId>
                <artifactId>spring-data-jpa</artifactId>
                <version>1.7.3.RELEASE</version>
            </dependency>
        </dependencies>               
    </plugin> 
</plugins>
```

[Вверх](#содержание)

### 7.2 - Генерация changeLog из разницы между Базой Данных и Persistence Entities

Теперь самое интересное. Мы можем использовать этот плагин для генерации changeLog файла
из разницы между существующей базой данных (например production) и новыми persistence entities.

Как только Entity модифицируется, мы просто можем сгенерировать изменения против старой БД схемы,
получая __чистый, мощный способ эволюционировать схему в production__.

Ниже liquibase properties:
```properties
changeLogFile=classpath:liquibase-changeLog.xml
url=jdbc:mysql://localhost:3306/database_name
username=dbuser
password=dbmy5ql
driver=com.mysql.jdbc.Driver
referenceUrl=hibernate:spring:org.example.persistence.model
  ?dialect=org.hibernate.dialect.MySQLDialect
diffChangeLogFile=src/main/resources/liquibase-diff-changeLog.xml
```
Важно: _referenceUrl_ использует package scan, поэтому обязателен параметр _dialect_.

[Вверх](#содержание)

8 - Заключение
--------------
В этом руководстве было рассмотрено несколько способов использования Liquibase и получения
безопасного и практичного способа эволюционирования и рефакторинга схемы Базы Данных в Java приложении.

[Вверх](#содержание)
