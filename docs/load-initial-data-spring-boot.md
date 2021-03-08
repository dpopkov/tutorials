Загрузка начальных данных с помощью Spring Boot
===============================================

Содержание
----------
* [Обзор](#1---обзор)
* [data.sql](#2---datasql)
* [schema.sql](#3---schemasql)
* [Контроль создания Базы Данных используя Hibernate](#4---контроль-создания-базы-данных-используя-hibernate)
* [@Sql](#5---sql)
* [@SqlConfig](#6---sqlconfig)
* [@SqlGroup](#7---sqlgroup)
* [Заключение](#8---заключение)

1 - Обзор
---------
Spring Boot делает легким изменения базы данных. Если оставить конфигурацию по умолчанию, 
то он будет производить поиск в пакетах и создавать соответствующие таблицы автоматически.
Но иногда нужен более тонкий контроль над изменениями в базе данных. 
Для этого можно использовать `data.sql` и `schema.sql` файлы.

[Вверх](#содержание)

2 - data.sql
------------
Предположим, мы работаем с JPA. Создадим простую сущность Country:
```Java
@Entity
public class Country {
    @Id
    @GeneratedValue(strategy = IDENTITY)
    private Integer id;

    @Column(nulable = false)
    private String name;

    //..
}
```
При запуске приложения Spring Boot создаст пустую таблицу, но ничем ее не наполнит. 
Можно легко заполнить ее если создать файл `data.sql`.
```sql
INSERT INTO country (name) VALUES ('India');
INSERT INTO country (name) VALUES ('Brazil');
INSERT INTO country (name) VALUES ('USA');
INSERT INTO country (name) VALUES ('Italy');
```
При запуске проекта содержащего данный файл в classpath, Spring использует его для наполнения таблицы.

[Вверх](#содержание)

3 - schema.sql
--------------
Иногда, если не использовать механизм создания схемы базы данных по умолчанию, можно создать файл `schema.sql`.
```sql
CREATE TABLE country (
    id   INTEGER      NOT NULL AUTO_INCREMENT,
    name VARCHAR(128) NOT NULL,
    PRIMARY KEY (id)
);
```
Spring использует этот файл для создания схемы базы данных.  
__Важно помнить что в таком случае следует отключить автоматическое создание схемы ради избежания конфликтов:__
```properties
spring.jpa.hibernate.ddl-auto=none
```

[Вверх](#содержание)

4 - Контроль создания Базы Данных используя Hibernate
-----------------------------------------------------
Spring использует JPA свойство, которое применяется Hibernate для DDL генерации: 
    `spring.jpa.hibernate.ddl-auto`.
Стандартные значения: create, update, create-drop, validate и none:
* create - существующие таблицы удаляются, затем создаются новые
* update - объектная модель созданная на основе аннотаций или xml сравнивается с существующей схемой,
и затем Hibernate обновляет схему согласно обнаруженной разнице. Не используемые таблицы и столбцы не удаляются.
* create-drop - подобно create, но Hibernate удаляет базу данных после завершения всех операций. 
Обычно используется для юнит-тестов. 
* validate - производится только проверка существования таблиц и столбцов, в противном случае выбрасывается исключение
* none - никакая генерация или проверка не выполняется.
Spring Boot по умолчанию использует `create-drop` если не обнаруживает менеджера схемы,
иначе используется `none`.

[Вверх](#содержание)

5 - @Sql
--------
Spring также предоставляет аннотацию @Sql - это декларативный способ проинициализировать и наполнить тестовую схему.
```Java
@Sql({"/employees_schema.sql", "/import_employees.sql"})
public class SpringBootInitialLoadIntegrationTest {
    @Autowired
    private EmployeeRepository employeeRepository;

    @Test
    public void testLoadDataForTestClass() {
        assertEquals(3, employeeRepository.findAll().size());
    }
}
```
Аттрибутами @Sql являются:
* config - локальная конфигурация sql скриптов (описана в следующей секции)
* executionPhase - фаза когда выполнять скрипты, либо BEFORE_TEST_METHOD или AFTER_TEST_METHOD
* statements - можно буквально написать SQL для выполнения
* scripts - пути для исполняемых SQL файлов, это то же самое что и атрибут value.

Аннотация @Sql __может быть использована на уровне класса или метода__. 
Можно загрузить дополнительные данные для отдельных тест кейсов добавляя аннотацию @Sql методу.
```Java
@Test
@Sql({"/import_senior_employees.sql"})
public void testLoadDataForTestCase() {
    assertEquals(5, employeeRepository.findAll().size());
}
```

[Вверх](#содержание)

6 - @SqlConfig
--------------
Эта аннотация служит для конфигурирования способа парсинга и запуска SQL скриптов.
@SqlConfig может быть использована на уровне класса, где служит для глобального конфигурирования.
Либо она может конфигурировать отдельные @Sql аннотации.

Пример указанию кодировки SQL скриптов, а также режима транзакции для выполнения скриптов:
```Java
@Test
@Sql(scripts = {"/import_senior_employees.sql"}, 
  config = @SqlConfig(encoding = "utf-8", transactionMode = TransactionMode.ISOLATED))
public void testLoadDataForTestCase() {
    assertEquals(5, employeeRepository.findAll().size());
}
```  
Аттрибуты @SqlConfig:
* blockCommentStartDelimiter – разделитель обозначающий начала блока комментариев
* blockCommentEndDelimiter – разделитель обозначающий конец блока комментариев
* commentPrefix – префикс обозначающий однострочные комментарии 
* dataSource – имя javax.sql.DataSource бина с которым будут выполнятся скрипты и команды 
* encoding – кодировка SQL скрипта, по умолчанию это кодировка платформы
* errorMode – режим, который будет использован при ошибке во время выполнения скрипта 
* separator – строка разделяющая отдельные команды (statements), по умолчанию это "-" 
* transactionManager – имя бина PlatformTransactionManager который будет использоваться для транзакций
* transactionMode – режим, который будет использоваться при выполнении скриптов в транзации 

[Вверх](#содержание)

7 - @SqlGroup
-------------
Java 8 и выше позволяют использовать повторяющиеся аннотации. Это может быть применено и для @Sql.
Для Java 7 и ниже существует аннотация-контейнер - @SqlGroup. 
С ее помощью можно объявить несколько @Sql аннотаций.
```Java
@SqlGroup({
  @Sql(scripts = "/employees_schema.sql", 
    config = @SqlConfig(transactionMode = TransactionMode.ISOLATED)),
  @Sql("/import_employees.sql")})
public class SpringBootSqlGroupAnnotationIntegrationTest {
    @Autowired
    private EmployeeRepository employeeRepository;

    @Test
    public void testLoadDataForTestCase() {
        assertEquals(3, employeeRepository.findAll().size());
    }
}
```

[Вверх](#содержание)

8 - Заключение
--------------
В этом руководстве показано использование `schema.sql` и `data.sql` для создания и наполнения схемы.
Также показано использование @Sql, @SqlConfig, @SqlGroup аннотаций для загрузки тестовых данных.

Эти средства походят для простых сценариев. Более сложные случе нуждаются в использование таких инструментов
как Liquibase или Flyway.

[Вверх](#содержание)
