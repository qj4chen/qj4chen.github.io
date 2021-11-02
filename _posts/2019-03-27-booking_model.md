---
layout: post
title: 预约系统模型设计 —— 基于 JPA 和 Java 实现
categories:
  - Java
  - JPA
---

> 为了提供一个实验室被试预约与管理的良好体验，基于 JPA 写了一个模型，这个模型定义了一些对象（实验、实验片段、主试、被试），以及对象的关系，这个模型支持在不影响整体架构设计的情况下扩容，为每个对象提供更多的功能。

预约系统大致分为两部分，预约和管理。预约前端使用 Vue + Bootstrap 实现，没有权限验证，可以从实验列表中选择要参加的实验，然后找到合适自己时间的实验片段，提交表单，提交实验请求。管理系统使用 Vue + Ant Design 实现，权限验证，可以修改用户信息，可以创建实验，定义多个实验片段，为被试提交的实验片段更改状态 —— 允许、不允许，可以为参加过实验的被试评级，备注。备注和评级对于每个主试都能进行，并且统一修改被试，这点可以通过扩容，为被试提供多对多的主试评价、评级，然后计算得出其平均评级来实现，本模型没有进一步实现这一功能。

## 概要

整体包含 Subject、Experience、Host、Experiment 四个 Bean：

其中 Host 和 Experiment 是双向一对多的关系，前者级联处理后者，懒加载，允许为空。后者不级联处理前者，急加载，并且不允许为空。

其中 Experiment 和 Experience 是双向一对多的关系，前者级联处理后者，懒加载，允许为空。后者不级联处理前者，急加载，不允许为空。

其中 Subject 和 Experience 是双向多对多的关系，使用中间表维护，双方均允许为空，Experience 级联处理 Subject。

这种设计适用于场景：Admin 创建 Host、Host 创建 Experiment，在 Experiment 中创建 Experience，Subject 在 Experience 中注册，Host 处理 Subject 的注册信息，修改 Subject 和 Experience 状态。


## 主试

```java
/**
 * 定义主试，包含其主持的实验（一对多）
 * 单独做表
 */
@Entity
@Table(name = "HOSTS")
public class Host {
    @Id @GeneratedValue(strategy = GenerationType.AUTO)
    private Long id;
    private String userName;
    private String nickName;
    private String passWord;
    @Temporal(TemporalType.TIMESTAMP)
    private Date regTime;
    private String information;
    private String email;
    private String phoneNumber;
    @Enumerated(EnumType.STRING)
    private HostType type;

    @OneToMany(mappedBy = "host", fetch = FetchType.LAZY, cascade = CascadeType.ALL)
    //Host 和 Experiment 关系不密切，不适用级联保存，懒加载
    private Set<Experiment> experiments = new HashSet<>();
```

```java
public enum HostType {
    COMMON,
    ADMIN
}
```



## 被试

```java
/**
 * 定义被试，包含其参加过的实验片段（多对多）
 */
@Entity
@Table(name = "SUBJECTS")
public class Subject {
    @Id @GeneratedValue(strategy = GenerationType.AUTO)
    private Long id;
    private String name;
    private String information;
    private String phoneNumber;
    private String email;
    private String cardNumber;
    @Temporal(TemporalType.TIMESTAMP)
    private Date addTime;
    private String qq;
    @Enumerated(EnumType.STRING)
    private SubjectType type;

    @ManyToMany(mappedBy = "subjects")
    private Set<Experience> joinedExperiments = new HashSet<>();
```

```java
public enum  SubjectType {
    VERY_GOOD,
    GOOD,
    COMMON,
    NOT_GOOD,
    BAD,
    VERY_BAD
}
```

## 实验

```java
/**
 * 定义一次完整的主试（多对一）主持的实验，包含多个实验片段（一对多）
 * 单独做表
 */
@Entity
@Table(name = "EXPERIMENTS")
public class Experiment {
    @Id @GeneratedValue(strategy = GenerationType.AUTO)
    private Long id;
    private String name;
    @Temporal(TemporalType.TIMESTAMP)
    private Date fromTime;
    @Temporal(TemporalType.TIMESTAMP)
    private Date endTime;

    @ManyToOne(fetch = FetchType.EAGER, optional = false)
    @JoinColumn(name = "HOST_ID", nullable = false)
    //一个 Experiment 必须有一个 Host，且通过 Experiment 要尽可能早的得到 Host
    private Host host;

    @OneToMany(fetch = FetchType.LAZY, cascade = CascadeType.ALL, mappedBy = "experiment", orphanRemoval = true)
    //级联托管 Experience，当删除一个实验，也删除其 Experience
    //这里不使用 Embeddable 的原因是，Subject 需要和 Experience 建立关系，而不是 Experiment
    private Set<Experience> experiences = new HashSet<>();
```

```java
public enum ExperienceType {
    CREATED,
    NOT_ALLOWED,
    JOINED,
    DONE
}
```

## 实验片段

```java
/**
 * 定义每组被试（多对多）参与的部分实验，是整体实验的组成部分（多对一）
 * 单独做表
 */
@Entity
@Table(name = "EXPERIENCES")
public class Experience {
    @Id @GeneratedValue(strategy = GenerationType.AUTO)
    private Long id;
    @Temporal(TemporalType.TIMESTAMP)
    private Date startTime;
    @Temporal(TemporalType.TIMESTAMP)
    private Date endTime;
    @Enumerated(EnumType.STRING)
    private ExperienceType type = ExperienceType.CREATED;
    private String name;
    private Integer allowedMaxSubject = 1;

    @ManyToOne(fetch = FetchType.EAGER, optional = false)
    @JoinColumn(name = "EXPERIMENT_ID", nullable = false)
    //基于链接表的方法，一个 Experience 必须对应一个 Experiment
    private Experiment experiment;

    //级联保存，但是不执行其余操作，当此 Experience 被删除，那么不应该影响 Subject
    @ManyToMany(cascade = CascadeType.PERSIST)
    @JoinTable(
            name = "SUBJECT_EXPERIENCES",
            joinColumns = @JoinColumn(name = "EXPERIENCE_ID"),
            inverseJoinColumns = @JoinColumn(name = "SUBJECT_ID")
    )
    private Set<Subject> subjects = new HashSet<>();
```

## 单元测试

```java
@FixMethodOrder(MethodSorters.NAME_ASCENDING)
public class DataTest {

    private EntityManagerFactory factory;
    private EntityManager manager;
    private EntityTransaction transaction;

    @Before public void testBefore() {
        factory = Persistence.createEntityManagerFactory("jpa");
        manager = factory.createEntityManager();
        transaction = manager.getTransaction();
        transaction.begin();
    }

    @Test public void testASaveHost() {
        //测试管理员创建一个主试
        Host host = new Host();
        host.setRegTime(new Date());
        host.setEmail("corkine@live.com");
        host.setInformation("No Information");
        host.setNickName("Corkine Ma");
        host.setPhoneNumber("13084999445");
        host.setPassWord("mmm2333");
        host.setType(HostType.ADMIN);
        host.setUserName("corkine");
        manager.persist(host);
    }
    
    @Test public void testBGetHostAndCreateExperiment() {
        //模拟主试创建实验并且定义多个实验片段
        List<Host> hosts = manager.createQuery("select h from Host h", Host.class).getResultList();
        assertEquals("应该只能读取存储的唯一的数据",1, hosts.size());
        Host host = hosts.get(0);
        System.out.println("host = " + host);
        Experiment experiment = new Experiment();
        experiment.setFromTime(new Date());
        experiment.setEndTime(localToDate(LocalDate.now().plusDays(3)));
        experiment.setName("Experiment 1");
        experiment.setHost(host);

        Experience experience = new Experience();
        experience.setStartTime(new Date());
        experience.setEndTime(localToDate(LocalDate.now().plusDays(1)));
        experience.setAllowedMaxSubject(1);
        experience.setType(ExperienceType.CREATED);

        Experience experience2 = new Experience();
        experience2.setStartTime(new Date());
        experience2.setEndTime(localToDate(LocalDate.now().plusDays(2)));
        experience2.setAllowedMaxSubject(1);
        experience2.setType(ExperienceType.CREATED);

        experiment.getExperiences().add(experience);
        experiment.getExperiences().add(experience2);
        //Experience 会不会有 Experiment 呢？ 不会... Column 'EXPERIMENT_ID' cannot be null
        experience.setExperiment(experiment);
        experience2.setExperiment(experiment);
        //只用级联保存实验即可，注意要为其 Set 的实验片段设置 Experiment 引用
        manager.persist(experiment);
    }
    
    private Date localToDate(LocalDate localDate) {
        Instant instant = localDate.atStartOfDay().atZone(ZoneId.systemDefault()).toInstant();
        return Date.from(instant);
    }

    @Test public void testCSubjectCommitAExperiment() {
        //模拟测试填写表单的情况，根据实验获取其实验片段，然后将被试和实验片段绑定。最后保存实验片段，级联保存被试。
        Subject subject = new Subject();
        subject.setAddTime(new Date());
        subject.setInformation("No Info");
        subject.setName("Wang");
        subject.setType(SubjectType.COMMON);
        subject.setPhoneNumber("1000000000");
        List<Experiment> exps = manager.createQuery("select e from Experiment e", Experiment.class).getResultList();
        assertEquals("Experiment 应只有一个", 1, exps.size());
        Set<Experience> expers = exps.get(0).getExperiences();
        expers.forEach(ex -> {
            assertEquals("实验片段应该都是未处理的", ex.getType(), ExperienceType.CREATED);
        });
        //需要保存 Experience 来保存其和 Experiment 的多对一关系
        Experience experience = expers.stream().findFirst().get();
        experience.getSubjects().add(subject);
        //Experience 级联保存 Subject
        manager.persist(experience);
    }

    @Test public void testDSubjectInfo() {
        //模拟测试外部查询统计被试参与信息，当被试提交了实验片段，那么应该可以查询到期实验片段，以及对应实验
        assertEquals("保存后的 Subject 包含其实验信息，且只包含唯一的实验信息",
                1,
                manager.createQuery("select s from Subject s", Subject.class)
                        .getResultList().get(0)
                        .getJoinedExperiments().size());
    }

    @Test public void testEHostEditSubjectsCommit() {
        //模拟测试主试允许或者不允许被试的实验片段
        //如果不允许，则将其从实验片段中删除，否则，重置实验片段为同意
        TypedQuery<Experiment> exquery = manager.createQuery("select e from Experiment e", Experiment.class);
        List<Experiment> expList = exquery.getResultList();
        assertEquals("应该只有一个实验", 1,expList.size());
        Experiment exp = expList.get(0);
        Set<Experience> experiences = exp.getExperiences();
        System.out.println("experiences = " + experiences);
        assertTrue("实验的片段数大于1个",experiences.size() > 1);
        experiences.forEach(ex -> {
            if (ex.getSubjects().size() > 0) {
                //有被试报名的片段
                Set<Subject> subjects = ex.getSubjects();
                assertEquals("报名的被试应该只有一个", 1, subjects.size());
                ex.setType(ExperienceType.JOINED); //允许
                subjects.stream().findFirst().get().setType(SubjectType.VERY_GOOD);
                manager.persist(subjects.stream().findFirst().get());
                manager.persist(ex);
            }
        });
    }

    @Test public void testZClearAll() {
        //删除所有信息，同时不影响表结构
        List<Host> hosts = manager.createQuery("select h from Host h", Host.class).getResultList();
        hosts.forEach(host -> {
            manager.remove(host);
        });
        List<Experiment> exps = manager.createQuery("select e from Experiment e", Experiment.class).getResultList();
        exps.forEach(exp -> {
            manager.remove(exp);
        });
        List<Subject> subs = manager.createQuery("select s from Subject s", Subject.class).getResultList();
        subs.forEach(sub -> {
            manager.remove(sub);
        });
        assertEquals(0, manager.createQuery("select s from Subject s").getResultList().size());
        assertEquals(0, manager.createQuery("select h from Host h").getResultList().size());
        assertEquals(0, manager.createQuery("select e from Experiment e").getResultList().size());
        assertEquals(0, manager.createQuery("select ex from Experience ex").getResultList().size());
    }

    @After public void testAfter() {
        transaction.commit();
        manager.close();
        factory.close();
    }
}
```

## 体会

JPA 大致流程：先根据业务确定 Java Bean 结构（类、字段、实体的单向双向一对多、多对多、嵌入类的嵌入、单向双向一对多关系），然后根据需求进一步确定多个实体类的关系数据库实现：

- 根据是否需要验证、是否允许**空值**使用不同的 JPA 注解定义外键约束、中间表表结构
- 根据是否需要**级联**、级联的最常用方向，使用不同的级联注解定义字段属性特性、DAO 需要处理的依赖）。

在理清业务逻辑的时候，通过 JPA 一边创建测试，一边定义业务单元所需要的 Java Bean 和数据库表结构。

最后，mappedBy 不意味着不能级联，persist 会自动为每个集合内对象进行 persist。

即：

对于集合映射，需要注意的一点是，@OneToMany、@ManyToOne 和 @ManyToMany 的 mappedBy 属性和 cascade 属性是不冲突的。在上面，介绍过，对于 OneToMany 的情况，One 的一侧总是放弃维护双方关系，而 Many 的一侧进行维护。这里问题就来了，那么对于 @OneToMany 注解，cascade 还有意义吗？既然总是在 Many 的一侧保存 Many Bean 以及其依赖的 One Bean。**即这里的问题是：对于 @OneToMany 来说，CASCADE 和 MAPPEDBY** 冲突吗？

下面提供了一个一对多的例子：

```java
@Entity
@Table(name = "ROCKSTAR")
public class RockStar {
    @Id @GeneratedValue(strategy = GenerationType.AUTO)
    private Long id;
    private String name;
    private Integer age;
    @OneToMany(mappedBy = "star", cascade = CascadeType.ALL)
    private Set<Fans> fans = new HashSet<>();
}
@Entity
@Table(name = "FANS")
public class Fans {
    @Id @GeneratedValue(strategy = GenerationType.AUTO)
    private Long id;
    private String name;
    private Integer age;

    @ManyToOne(optional = false, fetch = FetchType.EAGER)
    private RockStar star;
}
```

总的来说，**JPA 在底层总是不管多的一侧，而是保存少的一侧，即集合中元素 Many 的一侧，而不是集合持有者 One 的一侧**。如果少的一侧有多的引用和依赖，那么就处理其依赖。

**级联的意思是，当你从某个方向保存，那么其依赖将被自动处理，级联定义了这个依赖被处理的规则，比如保存级联、删除级联、更新级联**。很多人不建议使用级联，觉得它增加了模型的复杂性，而强调手动再 DAO 中保存所有可能的依赖，这种把复杂留给编程人员的做法也可以，但是，也可以通过一两句话避免在 DAO 层的复杂度 —— 级联是个很好用，很方便的工具，如果认识清除的话。

### 当双方均未启用级联时

这种情况需要保存双方的 Bean，包括 Many 的一侧和 One 的一侧。

需要注意的是，如果是双向关系，那么这不意味着，你将 Fans 添加到 RockStar 的集合中就可以了，而也需要为 Fans 的 RockStar 添加引用。换句话说，因为 JPA 总是不处理 mappedBy 一侧的集合，而是对集合中每个元素进行处理，这意味着，如果对于 Fan 和 Star 双向关系，当你保存 Star 的话，那么 JPA 会去 Fans 集合中寻找 Fan Bean，逐个执行保存。这时，如果 Fan Bean 中没有 Star 的引用，就会出错。错误是 Transient instance must be saved before current operation。

即，对于双向关系，**你必须为每个 Bean 完全设置对方的引用，然后再进行保存，最起码，要为 Many 的一侧添加 One 一侧的引用。**。当然，如果你在 One 的一侧没有将 Many 添加到集合，那么逻辑也不会错，因为这时候，One - Star 会先持久化，然后因为没有启用级联，那么不会处理 Many - Fans。当 Many - Fans 进行保存的时候，会找到其引用的 One - Star，发现是一个持久化类，因此建立其关系，然后持久化 Many - Fans。但是，反过来就不对了，如果你在 One 侧保存了 Many 到其集合，但是在 Many 没有保存 One 的引用，那么因为 mappedBy 不会处理 One 侧的集合，并且 One 侧集合也没有启用级联，所以不会处理这部分 Fans，而是单独保存 Star，而在处理 Fans 的时候，因为找不到 Star，就报了错。

```java
RockStar star = new RockStar();
star.setName("Corkine");
Fans fans = new Fans();
fans.setName("Fans1");
Fans fans2 = new Fans();
fans2.setName("Fans2");
fans.setStar(star);
fans2.setStar(star);
manager.persist(star);
manager.persist(fans);
manager.persist(fans2);
```

### 当 Many 一侧启用级联时

如果启用了 Fans 的级联保存，那么就不需要先保存 Star，当然，需要为 Fans 写入 Star 的引用，如果引用不存在，则会级联自动的进行依赖引用的保存，之后再保存这个 Fans。**即，先需要保存 One 在 Many 一侧的引用，然后只需要持久化 Many 一侧即可。**

```java
RockStar star = new RockStar();
star.setName("Corkine");
Fans fans = new Fans();
fans.setName("Fans1");
Fans fans2 = new Fans();
fans2.setName("Fans2");
fans.setStar(star);
fans2.setStar(star);
manager.persist(fans);
manager.persist(fans2);
```

### 当 One 一侧启用级联时

这个是看起来最让人糊涂的，但是，也很简单。当启用了 One - Star 一侧的级联，那么你只需要将 Many - Fans 添加到 Star 的集合中，然后保存 Star 即可。看起来很美好，但是，在测试的时候很容易就出错了。当 Many - Fans 没有 One - Star 的引用，还是会出错，这是为什么呢？

很简单，因为，JPA 在底层从来不去处理 mappedBy 部分，它只会去找到 mappedBy 的 Bean，如果像上面那样，没有启用级联，那么什么也不处理，而如果启用了级联，则去先保存这些集合中的 Bean。JPA 不会去自动解决这些 Bean 的依赖，因此，如果 Fans 的 Star 引用不存在，那么就会出错。

因此，当在 One 一侧启用级联时，尤其需要注意这一点，需要我们手动解决其级联保存的 Bean 的内部依赖。

```java
RockStar star = new RockStar();
star.setName("Corkine");
Fans fans = new Fans();
fans.setName("Fans1");
Fans fans2 = new Fans();
fans2.setName("Fans2");
fans.setStar(star);
fans2.setStar(star);
star.getFans().add(fans);
star.getFans().add(fans2);
manager.persist(star);
```

### 当 One 和 Many 一侧均启用级联时

在这种情况下，我本来以为这需要执行更多的 SQL 语句，但实际上并不是。当两边启用级联，两边都可以进行单独的保存，保存会解决其依赖。

但问题也一样，如果是从 Many 的一侧进行单独保存，那么需要为 Many 侧处理其 One 的依赖，此依赖如果没持久化，会自动持久化。如果从 One 的一侧进行单独保存，那么需要将 Many 侧注入 One 的集合中，在这个时候，mappedBy 会遍历所有的 Bean，如果发现 Bean 没有处理，那么级联调用其保存。

**即，你最好永远为 Many 一侧添加 One 引用，只在需要从 One 一侧进行级联保存的时候，才为 One 一侧的集合添加 Many 引用。**



