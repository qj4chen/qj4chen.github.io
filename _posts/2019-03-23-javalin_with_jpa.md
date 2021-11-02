---
layout: post
title: 短网址跳转：使用 Javalin 和 JPA 实现 
categories:
  - Java
  - Scala
  - Javalin
---

> 本文介绍了使用 Javalin 和 JPA 实现的简单的短网址跳转服务。Javalin 是一个 REST 的 HTTP 处理类库，类似于 SpringMVC 的功能作用。这个类库的名字非常的 —— 无厘头，Javalin 支持 Java 和 Kotlin，因此叫做 Java(Kot)lin

# Why Javalin

使用 Javalin 起一个服务非常简单：

```scala
Javalin.create().start(80).get("/", ctx => {
  ctx.result("Hello, World")
})
```

这种诱惑 —— 总让人为之着迷，就像 Spring Boot 带来的相比较 SSH/EE 的开发体验，或者像微服务之于传统服务给人带来的那种对于灵活性的幻想。

下面的这种两行实现 CRUD 的操作 —— 相比较 SpringMVC，实在是结构清晰流畅许多。

```scala
app.routes(() => {
  ApiBuilder.crud("users2/:user-id", new UserController)
})
class UserController extends CrudHandler {
  override def getAll(context: Context): Unit = {
    context.result("This is all User")
  }
  override def getOne(context: Context, s: String): Unit = {
    context.result("This is a User with id " + s)
  }
  override def create(context: Context): Unit = {
    context.result("This is a new User")
  }
  override def update(context: Context, s: String): Unit = {
    context.result("This is a User with id " + s)
  }
  override def delete(context: Context, s: String): Unit = {
    context.result("This is a Deleted User with id " + s)
  }
}
```

## Why not Javalin

下面介绍了使用 Javalin 实现的一个短连接跳转服务，正如我预期的，Javalin 足够的精致和小巧，但是，它并未提供让用户脱离 Spring 家族的强有力的理由。

- 缺失舒适自由的 YAML 和 property 配置系统，以及 SpEL 的表达式语言，是第一个问题，这意味着，所有的参数，都需要硬编码到 Java 中，或者 —— 自己使用 YAML 解析器去处理。

- CRUD Handler 固然方便，但是，实际上，完全遵守这种模式的数据处理恐怕不太多，总有各种各样的偏离最佳实践的意外。

- Router 将所有的 URL 放在一个地方管理，这种方式看似清晰简洁，但是，实际上是一种返祖现象，Java 的 Servlet 在一开始就是这样配置的，这种结构每次要处理一部分业务逻辑的时候，都要频繁的在 Router 和 其 Controller 之间切换，并不高效，相反，Spring MVC 提供的基于注解的处理方式很符合开发习惯 —— 尤其是大型项目。

- 缺失 IOC，服务不得不自行初始化，不得不自行注入到 Controller 中。

- Javalin 比不得 Spring 庞大家族的工具链 —— 尤其是手写 DAO —— 虽然有 JPA，但是，手写 DAO 还是很麻烦，要手动管理线程共享的 EntityManager，处理 Transition 和 Exception。而使用 Spring Data 则方便很多，只用定义 Service 层的 Interface，其被 Spring 实现并且存放在 IOC 中了。

但是，优点来说，也很方便，比如 Netty、Jetty 系统启动的是真的快，比 Spring Boot 快很多。但是，JPA 的加载还是要额外占用时间。Javalin 用于偶尔暴露个端口，提供简单的本地文件服务，或者一些路由不太多的服务，还是可以很好的应对的。Context 对象封装了大量的用于从 Path、Attr、FormParam 中获取参数的方式，提供了自动的 html、json 解析方式，提供了快速的 HTTP 代码处理、重定向、异常处理的方式 —— 虽然说，使用 Spring Boot 也很快，但是，后者总让人感觉是大象站在独木桥上跳舞 —— 笨重的灵活。

# DEMO：短网址跳转

## 运行示意图

下面是这个短连接服务的示意图：

![](http://static2.mazhangjing.com/20190323/a477939_2019-03-2319.21.09.gif)

代码如下：

## 控制器和路由部分

```scala
object Main {

  val service = new SimpleRowService()

  def main(args: Array[String]): Unit = {
    val app = Javalin.create().enableCaseSensitiveUrls().start(80)
    app.get("/", ctx => {
      ctx.html("Welcome to Muninn URL Service by Scala and Javaln & JPA")
    })
    //添加
    app.get("/add", ctx => {
      ctx.html(
        """
          |<html>
          |<form method="post" action="/add">
          |Name: <br/>
          |<input type="text" name="name"><br/>
          |LocalUrl: <br/>
          |<input type="text" name="localUrl"><br/>
          |FullUrl: <br/>
          |<input type="text" name="fullUrl"><br/><br/>
          |<button type="submit">Add to Database</button>
          |</form>
          |</html>
        """.stripMargin
      )
    })
    app.post("/add", UrlController.create)
    //查询
    app.get("/show", UrlController.handleAll)
    app.get("/show/:id", UrlController.handleOne)
    //删除
    app.get("/delete/:id", UrlController.delete)
    //跳转
    app.get("/:localUrl", ctx => {
      service.getRowByLocalUrl(ctx.pathParam("localUrl")) match {
        case Some(value) => ctx.redirect(value.getFullUrl)
        case None => ctx.html("Can't find This Url in Database.")
      }
    })
  }

  object UrlController {

    val doSimpleAuth: (Context, Context => Unit) => Unit = (context, op) => {
      if (context.anyQueryParamNull("password")) {
        context.html("No Auth")
      } else if (context.queryParam("password").toInt % LocalDate.now().getDayOfMonth == 0) {
        op(context)
      } else {
        context.html("Auth Failed")
      }
    }

    //显示所有的 URL 信息
    def handleAll(context: Context): Unit = {
      doSimpleAuth(context, context => {
        service.getAllRows match {
          case Some(value) => context.json(value)
          case None => context.html("None Row Found.")
        }
      })
    }

    def handleOne(context: Context): Unit = {
      doSimpleAuth(context, context => {
        val s = context.pathParam("id")
        service.findRow(s.toLong) match {
          case None => context.html("Can't find a Row In database with Id " + s)
          case Some(row) => context.json(row)
        }
      })
    }

    def create(context: Context): Unit = {
      val row = new data.Row()
      row.setAddTime(new Date())
      row.setName(context.formParam("name"))
      row.setLocalUrl(context.formParam("localUrl"))
      row.setFullUrl(context.formParam("fullUrl"))
      service.saveRow(row) match {
        case Some(value) => context.json(value)
        case None => context.html("Add Failed.")
      }
    }

    def delete(context: Context): Unit = {
      doSimpleAuth(context, context => {
        val s = context.pathParam("id")
        service.removeRow(s.toLong) match {
          case Some(value) => context.json(value)
          case None => context.html("Can't find This Row with Id " + s)
        }
      })
    }
  }
}
```

## JPA 服务部分

```scala
trait RowService {
  def getRowByLocalUrl(localUrl: String): Option[Row]
  def saveRow(row: Row): Option[Row]
  def findRow(id: Long): Option[Row]
  def removeRow(id: Long): Option[Row]
  def getAllRows: Option[java.util.List[Row]]
}

class SimpleRowService extends RowService {

  import JPAUtils._

  val getRowByLocalUrlMethod: (String, EntityManager) => Option[Row] = (localUrl, manager) => {
    val res = manager.createQuery("select r from Row r where r.localUrl = ?1", classOf[Row])
      .setParameter(1,localUrl).getResultList
    if (res.size() > 0) Some(res.get(0))
    else None
  }

  val findWithId: (Long, EntityManager) => Option[Row] = (id, manager) =>
    Option(manager.find(classOf[Row], id))

  override def getRowByLocalUrl(localUrl: String): Option[Row] = {
    doWithEntityManager(manager => {
      getRowByLocalUrlMethod(localUrl, manager)
    })
  }

  override def saveRow(row: Row): Option[Row] = {
    doWithEntityManager(manager => {
      //如果本地数据库存在，则不保存，否则保存
      getRowByLocalUrlMethod(row.getLocalUrl, manager) match {
        case Some(_) => None
        case None =>
          manager.persist(row)
          Some(row)
      }
    })
  }

  //这里的每一此调用都是一个单独的 EntityManager，切记不要嵌套调用此结构，或隐式嵌套调用
  //比如在一个 API 中调用另一个 API，而它们使用了不同的 EM，应该抽象为函数
  //对于 remove 方法，问题可能尤其严重，因为如果在一个 EM 中不存在，则不能删除
  private def doWithEntityManager[T](op: EntityManager => Option[T]): Option[T] = {
    var manager: EntityManager = null
    try {
      manager = getManager
      val trans = manager.getTransaction
      trans.begin()
      val res = op(manager)
      trans.commit()
      res
    } catch {
      case e :Throwable => e.printStackTrace(System.err); None
    }
  }

  override def findRow(id: Long): Option[Row] = {
   doWithEntityManager(manager => {
     val res = manager.find(classOf[Row], id)
     Option(res)
   })
  }

  override def removeRow(id: Long): Option[Row] = {
    doWithEntityManager(manager => {
      //不要在这里调用 findRow，remove 只能删除存在于 EM 中的值
      findWithId(id, manager) match {
        case Some(row) =>
          manager.remove(row)
          Some(row)
        case None => None
      }
    })
  }

  override def getAllRows: Option[util.List[Row]] = {
    doWithEntityManager(manager => {
      val list = manager.createQuery("select r from Row r", classOf[Row]).getResultList
      if (list.size() > 0) Option(list)
      else None
    })
  }
}

object JPAUtils {
  private val factory = Persistence.createEntityManagerFactory("jpa-a")
  private val local = new ThreadLocal[EntityManager]()
  def getManager: EntityManager = {
    val em = local.get()
    if (em == null) {
      val newManager = factory.createEntityManager()
      local.set(newManager)
      newManager
    } else { em }
  }
}
```

## JPA Entity 部分

```java
@Entity
@Table(name = "LINKS")
public class Row {
    @Id @GeneratedValue(strategy = GenerationType.AUTO)
    private Long id;

    private String fullUrl;

    private String localUrl;

    private String name;

    @Temporal(TemporalType.TIMESTAMP)
    private Date addTime;

    public Row(String name, String localUrl, String fullUrl, Date addTime) {
        this.fullUrl = fullUrl;
        this.localUrl = localUrl;
        this.name = name;
        this.addTime = addTime;
    }

    public Row() {
    }

    public String getFullUrl() {
        return fullUrl;
    }

    public void setFullUrl(String fullUrl) {
        this.fullUrl = fullUrl;
    }

    public String getLocalUrl() {
        return localUrl;
    }

    public void setLocalUrl(String localUrl) {
        this.localUrl = localUrl;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public Date getAddTime() {
        return addTime;
    }

    public void setAddTime(Date addTime) {
        this.addTime = addTime;
    }

    public Long getId() {
        return id;
    }

    public void setId(Long id) {
        this.id = id;
    }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (o == null || getClass() != o.getClass()) return false;
        Row row = (Row) o;
        return Objects.equals(id, row.id) &&
                Objects.equals(fullUrl, row.fullUrl) &&
                Objects.equals(localUrl, row.localUrl) &&
                Objects.equals(name, row.name) &&
                Objects.equals(addTime, row.addTime);
    }

    @Override
    public int hashCode() {
        return Objects.hash(id, fullUrl, localUrl, name, addTime);
    }

    @Override
    public String toString() {
        return "Row{" +
                "id=" + id +
                ", fullUrl='" + fullUrl + '\'' +
                ", localUrl='" + localUrl + '\'' +
                ", name='" + name + '\'' +
                ", addTime=" + addTime +
                '}';
    }
}
```

## JPA 配置部分

```xml
<?xml version="1.0" encoding="UTF-8"?>
<persistence xmlns="http://java.sun.com/xml/ns/persistence" version="2.0">
    <!--需要提供在 META-INF/persistence.xml 中写入：
    persistence-unit，包含 名称、事务类型、实现提供者、对应的映射 POJO、详细的数据库和调试连接。-->
    <persistence-unit name="jpa-a" transaction-type="RESOURCE_LOCAL">
        <!--如果只有一个实现，则可以不用写-->
        <provider>org.hibernate.jpa.HibernatePersistenceProvider</provider>

        <shared-cache-mode>ENABLE_SELECTIVE</shared-cache-mode>
        <!--提供验证，AUTO：如果没有相应实现，则不验证，如果有，则验证。 CALLBACK 始终进行验证-->
        <validation-mode>AUTO</validation-mode>
        <!--提供数据库信息、调试信息，注意，提供数据库试用的方言和更新策略。-->
        <properties>
            <property name="javax.persistence.jdbc.driver" value="com.mysql.cj.jdbc.Driver"/>
            <property name="javax.persistence.jdbc.url" value="jdbc:mysql:///orm?serverTimezone=GMT%2B8" />
            <property name="javax.persistence.jdbc.user" value="xxxxx" />
            <property name="javax.persistence.jdbc.password" value="xxxxx" />

            <property name="hibernate.format_sql" value="false" />
            <property name="hibernate.show_sql" value="true" />
            <property name="hibernate.hbm2ddl.auto" value="update" />
            <property name="hibernate.dialect" value="org.hibernate.dialect.MySQL8Dialect" />
            <property name="hibernate.cache.use_second_level_cache" value="true" />
            <property name="hibernate.cache.region.factory_class" value="org.hibernate.cache.ehcache.internal.EhcacheRegionFactory" />
            <property name="hibernate.cache.use_query_cache" value="false"/>
        </properties>
    </persistence-unit>
</persistence>
```

## JUNIT 单元测试部分

```java
@Ignore("测试本地数据库是否准备就绪，可供连接")
@FixMethodOrder(MethodSorters.NAME_ASCENDING)
public class TestJPA {

    private EntityManagerFactory factory;
    private EntityManager manager;
    private EntityTransaction trans;

    @Before public void start() {
        factory = Persistence.createEntityManagerFactory("jpa-a");
        manager = factory.createEntityManager();
        trans = manager.getTransaction();
        trans.begin();
    }

    @Test public void testAWithSave() {
        Row row = new Row();
        row.setName("testName");
        row.setLocalUrl("/testLocal");
        row.setFullUrl("http://www.fullurl.com");
        manager.persist(row);
    }

    @Test public void testBWithRead() {
        TypedQuery<Row> select = manager.createQuery("select r from Row r", Row.class);
        List<Row> resultList = select.getResultList();
        Assert.assertNotNull(resultList);
            Assert.assertNotEquals(0, resultList.size());
        System.out.println("resultList = " + resultList);
    }

    @Test public void testZWithClearAll() {
        TypedQuery<Row> all = manager.createQuery("select r from Row r", Row.class);
        all.getResultList().forEach(row -> {
            manager.remove(row);
        });
        manager.flush();
        TypedQuery<Row> all2 = manager.createQuery("select r from Row r", Row.class);
        List<Row> list2 = all2.getResultList();
        Assert.assertEquals(0, list2.size());
    }

    @After public void end() {
        trans.commit();
        manager.close();
        factory.close();
    }
}
public class RowServiceTest {

    private static String LOCAL_URL = "/3434/343234";

    @BeforeClass public static void fakeSomeData() {
        Row row = new Row();
        row.setFullUrl("http://testFullUrl.com");
        row.setLocalUrl(LOCAL_URL);
        row.setName("TestName");
        row.setAddTime(new Date());

        Row row2 = new Row();
        row2.setFullUrl("http://testFullUrl2.com");
        row2.setLocalUrl("/34342/343234");
        row2.setName("TestName2");
        row2.setAddTime(new Date());

        doBetweenTrans(manager -> {
            manager.persist(row);
            manager.persist(row2);
        });
    }

    private static void doBetweenTrans(Consumer<EntityManager> op) {
        EntityManagerFactory factory;
        EntityManager manager;
        EntityTransaction trans;
        factory = Persistence.createEntityManagerFactory("jpa-a");
        manager = factory.createEntityManager();
        trans = manager.getTransaction();
        trans.begin();
        op.accept(manager);
        trans.commit();
        manager.close();
        factory.close();
    }

    private RowService service;

    @Before public void before() {
        service = new SimpleRowService();
    }

    @Test
    public void getRowByLocalUrl() {
        Option<Row> row = service.getRowByLocalUrl(LOCAL_URL);
        Assert.assertFalse("不应该从一个已存储到数据库中的数据的遍历中无法重新找回此字段",row.isEmpty());
    }

    @Test
    public void findRow() {
        Option<Row> row = service.getRowByLocalUrl(LOCAL_URL);
        Assert.assertFalse("不应该从一个已存储到数据库中的数据的遍历中无法重新找回此字段",row.isEmpty());
        Row realRow = row.get();
        Long realId = realRow.getId();
        Option<Row> findRow = service.findRow(realId);
        Assert.assertFalse("不应该从数据库获取的一个 Id 无法执行查找",findRow.isEmpty());
    }

    @Test
    public void removeRow() {
        Option<Row> row = service.getRowByLocalUrl(LOCAL_URL);
        Assert.assertFalse("不应该从一个已存储到数据库中的数据的遍历中无法重新找回此字段",row.isEmpty());
        Row realRow = row.get();
        System.out.println("realrow id is " + realRow.getId());
        Option<Row> removedRow = service.removeRow(realRow.getId());
        Assert.assertFalse("存在于数据库中的值，且被删除后，其值不应该是空的",removedRow.isEmpty());
        Option<Row> fakeRow = service.removeRow(Integer.MAX_VALUE);
        Assert.assertTrue("一个不存在数据库中的值不应该被删除", fakeRow.isEmpty());
    }

    @Test
    public void saveRow() {
        Row testRow = new Row();
        testRow.setName("TestRow");
        testRow.setAddTime(new Date());
        testRow.setLocalUrl("local_url");
        testRow.setFullUrl("http://fullurl.com");
        Option<Row> row = service.saveRow(testRow);
        Assert.assertFalse("保存到数据库中的数据不应该无法获取",row.isEmpty());
        Assert.assertEquals("保存后，应该有三条记录",3, service.getAllRows().get().size());
    }

    @Test
    public void getAllRow() {
        Option<List<Row>> allRows = service.getAllRows();
        Assert.assertFalse("数据库不应该为空",allRows.isEmpty());
        Assert.assertEquals("数据库中应该存在 2 条数据",2,allRows.get().size());
    }

    @AfterClass public static void clearAllData() {
        doBetweenTrans(manager -> {
            List<Row> list = manager.createQuery("select r from Row r", Row.class).getResultList();
            list.forEach(manager::remove);
            manager.flush();
        });
    }
}
```

## 小结

这是我第一次完整的对 JPA 服务使用 JUNIT 进行测试 —— 结果发现了很多问题，测试 —— 尤其是第一次测试，并不好写。要定义很多模拟的数据，还要对这些数据进行清理。此外，还要处理单元测试的顺序问题，为每个 Assert 提供完整的定义说明。很麻烦。

但是，却很有用。我对于 EntityManager 和服务的整合了解并不好，之前都是直接学完 JPA 直接上了 Spring Data —— 就像现在的学生直接学 Spring Boot，那自然丧失了许多本来应该属于 Spring Hibernate 和 SpringMVC 整合的经验，甚至 JSP、Servlet 的使用经验。比如第一个愚蠢的错误就是对数据库所有数据取出来，然后判断某个 Bean 的 property，然后返回 —— 这是极其愚蠢的 —— 使用 JPQL 即可对条件查询，效率高很多。此外，我在一开始并不知道如何处理 EM 和线程绑定的问题，导致了低效的 Service API，多个 EM 的问题也暴露了出来，比如 delete 一个不在本 EM 中的 Bean 会报错。最后，忘记写 translation，或者 translation 忘记 begin，导致无法保存。

很幸运的是，这些错误，都没有，或者说几乎没有通过 JUNIT 的单元测试。因此，我得以很快的发现错误并进行更正。JUNIT 必须是细粒度的，针对 API 的，如果 API 有套用，那么要细心的定义其检验顺序，以保证能通过单元测试很快的定义到问题，解决 BUG。

# 总结

没什么可总结的。Javalin 虽然比不得 Spring Boot 以及其附属的 Spring IOC、AOP、MVC、Data with JPA 家族，但胜在小巧灵活 —— 尽管这对于 Spring Boot 而言并不是一个关键的优点。

但是，函数式，尤其是 Scala 实现的 Javalin 的函数式，其表现力还是让人感到惊艳的。100 行的控制器和路由代码相比较任何宣称灵活、动态、高效的语言实现的 Web 框架都丝毫不逊色。Javalin 倒是可以和 Python 的 Django 比较一下，不过也赢得绰绰有余。

-----

2019-03-23 撰写本文