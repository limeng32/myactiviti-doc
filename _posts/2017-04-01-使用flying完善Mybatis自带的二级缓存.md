---
layout: post
title: 使用 flying 完善Mybatis 自带的二级缓存
description: 本节内容向您讲解如何使用 EnhancedCachingInterceptor 拦截器来改造Mybatis的二级缓存使其可用。
category: blog
---
<a id="Index"></a>
## 目录
- [上手指南](#%E4%B8%8A%E6%89%8B%E6%8C%87%E5%8D%97)
- [观察者 & 触发者](#%E8%A7%82%E5%AF%9F%E8%80%85--%E8%A7%A6%E5%8F%91%E8%80%85)
- [跨数据源缓存](#%E8%B7%A8%E6%95%B0%E6%8D%AE%E6%BA%90%E7%BC%93%E5%AD%98)
- [注意事项](#%E6%B3%A8%E6%84%8F%E4%BA%8B%E9%A1%B9)
  - [flying 如何判断缓存是否命中](#flying-%E5%A6%82%E4%BD%95%E5%88%A4%E6%96%AD%E7%BC%93%E5%AD%98%E6%98%AF%E5%90%A6%E5%91%BD%E4%B8%AD)
  - [flying 已回滚的操作是否会生成缓存](#flying-%E5%B7%B2%E5%9B%9E%E6%BB%9A%E7%9A%84%E6%93%8D%E4%BD%9C%E6%98%AF%E5%90%A6%E4%BC%9A%E7%94%9F%E6%88%90%E7%BC%93%E5%AD%98)
- [附录](#%E9%99%84%E5%BD%95)
  - [代码示例](#%E4%BB%A3%E7%A0%81%E7%A4%BA%E4%BE%8B)
  - [account 表建表语句](#account-%E8%A1%A8%E5%BB%BA%E8%A1%A8%E8%AF%AD%E5%8F%A5)
  - [role 表建表语句](#role-%E8%A1%A8%E5%BB%BA%E8%A1%A8%E8%AF%AD%E5%8F%A5)

## [上手指南](#Index)
上一篇文章中我们介绍了使用 flying 解决 pojo 自动映射问题，在本篇文章中我们介绍如何使用 flying 优化后的 mybatis 自带二级缓存。mybatis 自带的二级缓存方便上手容易使用，并且如果您把它和 redis 等第三方缓存结合使用，它就能成为可扩展的强大缓存；如果您的系统用户数不到一千并且您不想为缓存配置额外的服务器，那您也可以只使用 mybatis 的二级缓存，因为它的使用成本极低。

mybatis 的二级缓存只存在于内存中，不会写入硬盘，但可以通过 redis 来实现数据落地，在使用前请确保您的业务可以接受这些特性。

为使用 flying 优化后的 mybatis 自带二级缓存，您只需要在 Configuration.xml 和 <i>pojo_mapper</i>.java 中进行简单的配置。在 Configuration.xml 中配置的内容是：
```
<settings>
	<setting name="cacheEnabled" value="true" />
	<setting name="lazyLoadingEnabled" value="false" />
	<setting name="aggressiveLazyLoading" value="false" />
	<setting name="localCacheScope" value="SESSION" />
	<setting name="autoMappingBehavior" value="PARTIAL" />
	<setting name="lazyLoadTriggerMethods" value="" />
</settings>
```
需要注意的是 `settings` 中第一行 `cacheEnabled` 的值为 `true`，其它内容都和 [《flying-是什么》](http://flying-doc.limeng32.com/flying-%E6%98%AF%E4%BB%80%E4%B9%88) 中介绍的配置完全一致。

在完成上述工作后，mybatis 的二级缓存就可以使用了。现在启动项目后，每次对数据库发起的查询请求都会被缓存进行拦截，如果在缓存中能找到结果（包括单个 pojo、pojo集合、数量等）就直接返回结果，不会对数据库产生压力；如果找不到结果再去数据库中进行查询，查询后返回结果的同时把此次结果记入缓存中，这样下次再进行相同条件查询时如果相关记录没进行过刷新型操作（如 update、delete），就会返回缓存中的结果；如果对某条数据进行了 update、delete 操作，会使得相关缓存失效，其中的机制在后面会有详细介绍。

由于服务容器刚启动时缓存中没有任何内容，因此所有查询第一次都会经过数据库，在此之后缓存将发挥作用。为了更好的说明，我们需要新建[一个账号表](#AccountTableCreater)和[一个角色表](#RoleTableCreater)、相关的 `account.xml`、`role.xml`、`AccountMapper.java`、`RoleMapper.java`、`AccountService.java`、`RoleService.java`、`Account.java` 以及 `Role.java`。`AccountMapper.java` 如下：
```
package myPackage;
public interface AccountMapper {
    public Account select(Object id);
    public Account selectOne(Account t);
    public Collection<Account> selectAll(Account t);
    public int count(Account t);
    public void insert(Account t);
    public int update(Account t);
    public int updatePersistent(Account t);
    public int delete(Account t);
}
```
`RoleMapper.java` 如下：
```
package myPackage;
public interface RoleMapper {
    public Role select(Object id);
    public Role selectOne(Role t);
    public Collection<Role> selectAll(Role t);
    public int count(Role t);
    public void insert(Role t);
    public int update(Role t);
    public int updatePersistent(Role t);
    public int delete(Role t);
}
```
`account.xml`、`role.xml`、`AccountService.java`、`RoleService.java`、`Account.java` 和 `Role.java` 与 [《使用 flying 解决 pojo 自动映射问题》](http://flying-doc.limeng32.com/%E4%BD%BF%E7%94%A8flying%E8%A7%A3%E5%86%B3pojo%E8%87%AA%E5%8A%A8%E6%98%A0%E5%B0%84%E9%97%AE%E9%A2%98) 中介绍的完全一致且不会在本文中进行修改，因此这里不再累述。

之后，我们再定义一个用于测试的数据集：
```
<dataset>
    <account account_id="1" fk_role_id="10" address="beijing" name="frank" />
    <account account_id="2" fk_role_id="11" address="tianjin" name="gale" />
    <account account_id="3" fk_role_id="11" address="beijing" name="hank" />
    <role role_id="10" role_name="user" />
    <role role_id="11" role_name="super_user" />
</dataset>
```
当我们第一次执行 `accountService.select(1)` 时，会从数据库中得到一个 `name` 为 “frank” 的 Account 对象，当再次执行 `accountService.select(1)` 时，会不经数据库而从缓存中得到这个 Account 对象。如果我们通过数据库管理工具直接修改了这条记录，比如将 `name` 改为 “iris”，缓存也不会知道我们做出了改动，再次执行 `accountService.select(1)` 得到的 pojo 的 `name` 依然为 “frank”。但接下来如果我们在程序中执行 `accountService.update(account)` 将此对象的 `name` 改为 “hank”，则在数据库中的记录变更后缓存中的对象也会刷新，下一次再执行 `accountService.select(1)` 得到的 Account 对象的 `name` 就是 “hank” 了。

最后，我们强烈不建议在开启缓存的项目正在运行的情况下，通过数据库管理工具直接修改数据，这会导致数据库和缓存不一致，就像上一段中演示的那样。当您得到缓存的好处时，您也只能使用已经定义好的方法来操作数据库。（实际上，我们不建议通过数据库管理工具直接修改任何正在运行中的系统，无论它是否使用了缓存。）
## [观察者 & 触发者](#Index)
在上一节中我们了解了 mybatis 二级缓存的原理，但它本身还有需要优化的地方。例如当您缓存了一个 Account 对象后，当这个对象对应的 Role 对象发生改变时，我们当然希望缓存中的 Account 对象会知道它的多对一关系 Role 对象已经失效，然后在查询时会到数据库中查找最新数据（因为查询子对象时会自动加载父对象）。然而遗憾的是，mybatis 原始的二级缓存并没有这个功能，但是 flying 提供了解决方案。您只需将 flying 优化缓存的插件配置好并了解 “观察者” 和 “触发者” 的概念，就可以解决这一问题。

首先在 `Configuration.xml` 中的 `plugins` 中加入以下内容：
```
<plugins>
    <plugin interceptor="indi.mybatis.flying.interceptors.EnhancedCachingInterceptor">
        <property name="cacheEnabled" value="true" />
        <property name="annotationPackage" value="myPackage" />
    </plugin>
</plugins>
```
其中 `cacheEnabled` 为 `true` 表示让优化插件生效，`annotationPackage` 的值则是放置您所有 <i>pojo_mapper</i>.java 接口的包路径。如果您的 <i>pojo_mapper</i>.java 在多个包下（例如您在项目中用 mybatis 管理多个数据源，每个数据源的 <i>pojo_mapper</i>.java 放在不同的包内），这里可以配置多个包路径，只需用英文逗号分开即可，如 `value="myPackage,myPackage2"` 这样。如果您有未声明的 <i>pojo_mapper</i>.java，则该 pojo 对应的缓存不会被优化。

然后我们来介绍<b>观察者</b>和<b>触发者</b>的概念。在有父子关系的两个 pojo 中，父对象自身的改变可以使子对象的缓存失效，因此父对象可以称作子对象的<b>触发者</b>，而因为子对象只能参考它的父对象，子对象又可称为父对象的<b>观察者</b>。值得一提的是，这两种称呼都是基于 pojo 的关系的。所以会出现这种情况：pojo A 是 B 的触发者，pojo B 是 A 的观察者；同时 pojo B 是 C 的触发者，pojo C 是 B 的观察者。触发-观察关系可以传递，因此可以称 pojo A 是 C 的<b>间接触发者</b>，pojo C 是 A 的<b>间接观察者</b>。（如果两个 pojo 间有 触发-观察 关系但又不是直接关联，那就称为 间接触发-间接观察 关系，稍后您就后明白为什么要这么定义）。

接下来我们还需要了解触发者有哪些方法可以“触发”观察者，由之前的文章可以知道，`update`、`delete` 会使观察者改变，`select` 不会改变观察者，而 `insert` 是增加新的父对象，也不会改变观察者。了解以上概念后，我们就需要让程序了解我们的对象关系模型，而 flying 选择在 <i>pojo_mapper</i>.java 接口中接收您描述的对象关系模型。

我们只需在 AccountMapper.java 中增加一些注解：
```
package myPackage;
import indi.mybatis.flying.annotations.CacheAnnotation;
import indi.mybatis.flying.annotations.CacheRoleAnnotation;
import indi.mybatis.flying.statics.CacheRoleType;

@CacheRoleAnnotation(ObserverClass = { Role.class }, TriggerClass = { Account.class })
public interface AccountMapper {

    @CacheAnnotation(role = CacheRoleType.Observer)
    public Account select(Object id);
    
    @CacheAnnotation(role = CacheRoleType.Observer)
    public Account selectOne(Account t);
    
    @CacheAnnotation(role = CacheRoleType.Observer)
    public Collection<Account> selectAll(Account t);
    
    @CacheAnnotation(role = CacheRoleType.Observer)
    public int count(Account t);
    
    public void insert(Account t);
    
    @CacheAnnotation(role = CacheRoleType.Trigger)
    public int update(Account t);
    
    @CacheAnnotation(role = CacheRoleType.Trigger)
    public int updatePersistent(Account t);
    
    @CacheAnnotation(role = CacheRoleType.Trigger)
    public int delete(Account t);
}
```
接下来我们对这些注解依次说明：
`@CacheRoleAnnotation` 只能放在类定义之上，它声明当前 <i>pojo_mapper</i>.java 接口对应的 pojo 和其它 pojo 的关系，具体来说，`ObserverClass` 描述的是当前 pojo 是哪些 pojo 的观察者，因此需要填写当前 pojo 的所有<b>直接触发者</b>的类名（无需填写间接触发者，因为 flying 在运行期可以推导出），`TriggerClass` 则<b>只需填写当前接口对应的 pojo 类名</b>。填好以后 `@CacheRoleAnnotation` 就配置完毕。

`@CacheAnnotation` 只能放在方法定义之上，它声明当前这个方法是具有“触发”能力还是具有“观察”能力。具有触发能力的方法可以刷新这个类的观察者的缓存，具有观察能力的方法可以发现自己的缓存已被刷新，因为 触发－观察 关系可以传递，最终所有 pojo 的缓存关系都会被 flying 所获知。

在 flying 给定的方法中，`update`、`updatePersistent`、`delete` 具有触发能力，`select`、`selectOne`、`selectAll`、`count` 具有观察能力，所以要在相应的方法上配置 `@CacheRoleAnnotation` 注解。`insert` 既不具有触发能力也不具有观察能力。如果您在 <i>pojo_mapper</i>.java 中还有自己定义的方法，需要看这个方法对应的 sql 语句类型来进行配置，如果是 update 或 delete 类型就具有触发能力，如果是 select 类型就具有观察能力。

同样，RoleMapper.java 中也要增加一些注解：
```
package myPackage;
import indi.mybatis.flying.annotations.CacheAnnotation;
import indi.mybatis.flying.annotations.CacheRoleAnnotation;
import indi.mybatis.flying.statics.CacheRoleType;

@CacheRoleAnnotation(ObserverClass = {}, TriggerClass = { Role.class })
public interface RoleMapper {

	@CacheAnnotation(role = CacheRoleType.Observer)
	public Role select(Object id);

	@CacheAnnotation(role = CacheRoleType.Observer)
	public Collection<Role> selectAll(Role t);

	@CacheAnnotation(role = CacheRoleType.Observer)
	public Role selectOne(Role t);

	@CacheAnnotation(role = CacheRoleType.Observer)
	public int count(Role t);

	public void insert(Role t);

	@CacheAnnotation(role = CacheRoleType.Trigger)
	public int update(Role t);

	@CacheAnnotation(role = CacheRoleType.Trigger)
	public int updatePersistent(Role t);

	@CacheAnnotation(role = CacheRoleType.Trigger)
	public int delete(Role t);
}
```
（如上可见，当前系统中 Role 对象已无父对象，也就是 Role 不是任何对象的观察者，因此它的 `@CacheRoleAnnotation` 中的 `ObserverClass` 为空，但在它的方法中，仍然建议将具有观察能力的方法上的注解写全，这样当您以后扩展项目需要为 Role 类时增加父对象时，只需要在 `@CacheRoleAnnotation` 中的 `ObserverClass` 上增加类名即可，不需要再修改方法上面的注解）

当您所有的 <i>pojo_mapper</i>.java 都按以上情况配置好后，flying 优化 mybatis 二级缓存的工作就完成了，现在您可以放心的使用 mybatis 的自带缓存，而不用担心任何缓存与数据库不匹配的问题。

## [跨数据源缓存](#Index)
flying采用真实数据源配合自定义 TypeHandler 的方式来实现跨数据源，并可以将多个数据源的数据保存至同一缓存中，这一切都是自动完成的，您只需要给每个 <i>pojo_mapper</i>.java 都配置上合适的注解就能实现这一点，最终在缓存层面实现数据统一。

有一点需要注意的是，在跨数据源且使用二级缓存时，您在 <i>pojo_mapper</i>.xml 中的 resultMap 只能以 typeHandler 方式实现多对一，而不能以 association 方式。关于这一点，您可以阅读 [《使用 flying 解决 pojo 自动映射问题》中的跨数据源](http://flying-doc.limeng32.com/%E4%BD%BF%E7%94%A8flying%E8%A7%A3%E5%86%B3pojo%E8%87%AA%E5%8A%A8%E6%98%A0%E5%B0%84%E9%97%AE%E9%A2%98#%E8%B7%A8%E6%95%B0%E6%8D%AE%E6%BA%90) 部分。

## [注意事项](#Index)

### [flying 如何判断缓存是否命中](#Index)
flying 的缓存的 key 值是由查询条件 pojo 按业务属性序列化后再取 md5 生成，这样可保证不同线程上相同的查询条件 pojo 能够互相命中。因此使用缓存插件的前提条件是使用了<b>我们上一节介绍的 pojo 自动映射插件</b>，缓存插件无法单独使用。

### [flying 已回滚的操作是否会生成缓存](#Index)
不会生成。因为在您正确设计事务的情况下，未提交的（即被回滚）的数据变化是不会被查询到的，没有查询就不会生成缓存，所以您的缓存永远会和您的数据库保持一致，您可以放心的在有事务逻辑的系统中使用 flying 优化过的 mybatis 二级缓存。

## [附录](#Index)
<a id="flying-demo2"></a>
### [代码示例](#Index)
为了您更方便的使用 flying 进行优化 mybatis 二级缓存，我们提供了一个 [跨数据源并使用二级缓存的代码示例](https://github.com/limeng32/flying-demo2/tree/use-flying-0.9.2) 供您查看。
<a id="AccountTableCreater"></a>
### [account 表建表语句](#Index)
```
CREATE TABLE account (
  account_id int(11) NOT NULL AUTO_INCREMENT,
  name varchar(20) DEFAULT NULL,
  address varchar(100) DEFAULT NULL,
  fk_role_id int(11) DEFAULT NULL,
  fk_second_role_id int(11) DEFAULT NULL,
  PRIMARY KEY (account_id)
)
```

<a id="RoleTableCreater"></a>
### [role 表建表语句](#Index)
```
CREATE TABLE role (
  role_id int(11) NOT NULL AUTO_INCREMENT,
  role_name varchar(30) DEFAULT NULL,
  PRIMARY KEY (role_id)
)
```
- - -
