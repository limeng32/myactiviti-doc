---
layout: post
title: 使用 flying 解决 pojo 自动映射问题
description: 本节内容向您讲解如何使用 AutoMapperInterceptor 拦截器来实现pojo的自动映射。
category: blog
---
<a id="Index"></a>
## 目录
- [目录](#%E7%9B%AE%E5%BD%95)
- [Hello World](#hello-world)
- [flying 特征值描述](#flying-%E7%89%B9%E5%BE%81%E5%80%BC%E6%8F%8F%E8%BF%B0)
- [insert & delete](#insert--delete)
- [update & updatePersistent](#update--updatepersistent)
- [selectAll & count](#selectall--count)
- [foreign key](#foreign-key)
- [complex condition](#complex-condition)
- [limiter & sorter](#limiter--sorter)
- [分页](#%E5%88%86%E9%A1%B5)
- [乐观锁](#%E4%B9%90%E8%A7%82%E9%94%81)
- [其它](#%E5%85%B6%E5%AE%83)
  - [<font color="red">ignore tag</font>](#ignore-tag)
  - [复数外键](#%E5%A4%8D%E6%95%B0%E5%A4%96%E9%94%AE)
  - [跨数据源](#%E8%B7%A8%E6%95%B0%E6%8D%AE%E6%BA%90)
  - [<font color="red">兼容 JPA 标签</font>](#%E5%85%BC%E5%AE%B9-jpa-%E6%A0%87%E7%AD%BE)
- [附录](#%E9%99%84%E5%BD%95)
  - [常见问题](#%E5%B8%B8%E8%A7%81%E9%97%AE%E9%A2%98)
  - [代码示例](#%E4%BB%A3%E7%A0%81%E7%A4%BA%E4%BE%8B)
  - [account 表建表语句](#account-%E8%A1%A8%E5%BB%BA%E8%A1%A8%E8%AF%AD%E5%8F%A5)
  - [role 表建表语句](#role-%E8%A1%A8%E5%BB%BA%E8%A1%A8%E8%AF%AD%E5%8F%A5)
  - [AccountService 的实现方式](#accountservice-%E7%9A%84%E5%AE%9E%E7%8E%B0%E6%96%B9%E5%BC%8F)
  
## [Hello World](#Index)
上一篇文章中我们介绍了 flying 的基本情况，在展示第一个 demo 之前还需要做一些额外的工作，即描述您想让 mybatis 管理的数据的表结构。

无论是否使用 flying 插件，对于每一个由 mybatis 托管的表，都要有一个 <i>pojo_mapper</i>.xml 来告诉 mybatis 这个表的基本信息。在以往这个配置文件可能会因为 sql 片段而变得非常复杂，但加入 flying 插件后，这个配置文件中将不需要 sql 片段，变得精简而统一。下面是 [一个有代表性的 account 表](#AccountTableCreater) 以及对应它的配置文件 account.xml ：
``` 
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"  "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="myPackage.AccountMapper">
    <cache />
    <select id="select" resultMap="result">
        flying#{?}:select
    </select>
    <select id="selectOne" resultMap="result">
        flying:selectOne
    </select>
    <resultMap id="result" type="Account" autoMapping="true">
        <id property="id" column="account_id" />
    </resultMap>
</mapper>
``` 
在以上配置文件中，我们描述了一个接口 myPackage.AccountMapper，一个方法 select ，一个方法 selectOne，一个对象实体 Account，以及数据库表结构 resultMap。在 resultMap 中由于设置了 `autoMapping="true"`，我们只需要写出主键（以及外键，在稍后的章节会讲到），mybatis 会自动感知与 Account.java 中对应的变量名相同的字段，但与变量名有差异的字段仍需在 resultMap 中声明。

myPackage.AccountMapper 接口是 mybatis 本身需要的，里面的内容和 account.xml 中定义的方法相对应。如果您有使用 mybatis 的经验您就能立刻想到， AccountMapper.java 中的内容是：
```
package myPackage;
public interface AccountMapper {
    public Account select(Object id);
    public Account selectOne(Account t);
}
```
到目前为止一切都和不使用 flying 时一模一样，您可能奇怪的地方是：account.xml 中的 select 和 selectOne 方法描述中的 flying#{?}:select 是什么。这是这条查询的 flying 特征值描述，[在 flying 特征值描述部分会有解释。](#why-no-sql)马上我们就会在对象实体 Account 中看到更多不一样的地方，Account.java 的代码如下：
```
package myPackage;
import org.apache.ibatis.type.JdbcType;
import indi.mybatis.flying.annotations.FieldMapperAnnotation;
import indi.mybatis.flying.annotations.TableMapperAnnotation;
    
@TableMapperAnnotation(tableName = "account")
public class Account {
    @FieldMapperAnnotation(dbFieldName = "account_id", jdbcType = JdbcType.INTEGER, isUniqueKey = true)
    private Integer id;
	    
    @FieldMapperAnnotation(dbFieldName = "name", jdbcType = JdbcType.VARCHAR)
    private java.lang.String name;
	    
    public Integer getId() {
	    return id;
    }
    public void setId(Integer id) {
		this.id = id;
    }
    public String getName() {
		return name;
    }
    public void setName(String name) {
		this.name = name;
    }
}
```    
可见，和普通的 pojo 相比， Account.java 只是多了以下3行注解而已：
```
@TableMapperAnnotation(tableName = "account")
@FieldMapperAnnotation(dbFieldName = "id", jdbcType = JdbcType.INTEGER, isUniqueKey = true)
@FieldMapperAnnotation(dbFieldName = "name", jdbcType = JdbcType.VARCHAR) 
```
下面我们分别来解释它们的含义。

第1行 `@TableMapperAnnotation` 只能放在类定义之上，它声明这个类是一个表，它的属性 `tableName` 描述了这个表在数据库中的名字。

第2行 `@FieldMapperAnnotation` 只能放在变量定义之上，它声明这个变量是一个字段，它的属性 `dbFieldName` 描述了在数据库中这个字段的名称，它的属性 `jdbcType` 描述了在数据库中这个字段的类型，它的属性 `isUniqueKey = true` 描述了这个字段是一个主键。

第3行 `@FieldMapperAnnotation` 与第二行相同，它描述了另一个字段 name，值得注意的是这个字段的类型是 varchar 并且不是主键。

以上 3 个注解描述了表 account 的数据结构，然后我们就可以使用 AccountService 非常方便的操纵数据库的读取了。（AccountService 是 AccountMapper 的实现类，单独使用或在 spring 中使用都有多种方法进行配置，[本文档在附录部分提供了一种配置方法](#AccountService)）

使用以下代码，可以查询 id 为 1 的账户：
```
Account account = accountService.select(1);
```     
使用以下代码，可以查询 name 为 andy 的 1 条账户数据：
```
Account accountCondition = new Account();
accountCondition.setName("andy");
Account account = accountService.selectOne(accountCondition);
```
与以往的方式相比，这种方式是不是变得优雅了很多？关于 select 和 selectOne 之间的区别，我们在后面的章节会讲到。

## [flying 特征值描述](#Index)
<i>pojo_mapper</i>.xml 中的 flying#{?}:select 即是 flying 的特征值描述，如果您想用 flying 管理一个数据库操作，就用这行值替代原本应该写的 sql 语句，它的格式使用 linux 风格描述如下：
```
flying#{?}:select[:<ignoreTag>]
```
或者
```
flying:{selectOne|selectAll|count|insert|update|updatePersistent|delete}[:<ignoreTag>]
```
在第一个“:”之前的部分是 flying 的标识符，为避免复数外键时的缓存问题，当您使用 select 操作时需在 flying 后面加上 #{?}，当您使用其它类型操作时不需要加 #{?}。

第一个“:”和第二个“:”之间的部分是 flying 操作数据的方法，目前支持的方法有：
`select`：按主键查询，并返回结果集中的对象；
`selectOne`：按条件对象查询，只返回结果集中的第一个对象；
`selectAll`：按条件对象查询，返回结果集中所有对象组成的集合；
`count`：按条件对象查询，返回结果数量；
`insert`：按参数对象增加一条记录；
`update`：按参数对象中的非 null 属性更新一条记录，以参数主键为准；
`updatePersistent`：按参数对象中的所有属性更新一条记录，以参数主键为准，此操作会把参数对象为 null 属性在数据库中也更新为 null；
`delete`：按参数对象的主键删除一条记录；
本文为描述方便，大部分方法名（即方法配置中的 id）与其操作类型（即 flying 特征值的中间部分）相同，实际上方法名可以任意取，当您打算在同一个 <i>pojo_mapper</i>.xml 中定义多个操作类型相同的方法时就会用到。其它操作类型的开发还在评估之中，如果您有想法也可以告诉我们。

第二个“:”之后的部分是忽略标记，忽略标记是可选的。在 select、selectAll、selectOne 类型操作中如果配置了忽略标记，会使返回结果的类定义中配置了相同忽略标记的变量不被查询出来。在其它类型操作中配置忽略标记没有效果。

关于忽略标记更多的内容请见 [本文 ignore tag 部分。](#ignore-tag)

## [insert & delete](#Index)
在最基本的 select 之后，我们再看新增功能。但在此之前，需要先在 account.xml 中增加以下内容：
```
<insert id="insert" useGeneratedKeys="true" keyProperty="id">
    flying:insert
</insert>
```
上面的 `useGeneratedKeys="true"` 表示主键自增，如果您不使用主键自增策略此处可以省略，上面的语句和一般 mybatis 映射文件的区别在于具体 sql 语句变成了 flying 特征值描述。

同样在 AccountMapper.java 中我们需要加入：
```
public void insert(Account t);
```
然后使用以下代码，可以增加 1 条 name 为 “bob” 的账户数据（由于我们配置了主键自增，新增数据时不需要指定主键）：
```
Account newAccount = new Account();
newAccount.setName("bob");
accountService.insert(newAccount);
```
然后我们再看删除功能。先在 account.xml 中增加以下内容：
```
<delete id="delete">
    flying:delete
</delete>
```
然后在 `AccountMapper.java` 中加入：
```
public int delete(Account t);
```
然后使用以下代码，可以删掉 id 与 accountToDelete 的 id 一致的数据。
```
accountService.delete(accountToDelete);
```
delete 方法的返回值代表执行 sql 后产生影响的条数，一般来说，返回值为 0 表示 sql 执行后没有效果，返回值为 1 表示 sql 执行成功，在代码中可以通过判断 delete 方法的返回值来实现更复杂的事务逻辑。

## [update & updatePersistent](#Index)
接下来我们看看更新功能，这里我们要介绍两个方法：update（更新）和 updatePersistent（完全更新）。首先，在 `account.xml` 中增加以下内容：
```
<update id="update">
    flying:update
</update>
<update id="updatePersistent">
    flying:updatePersistent
</update>
```
上面的语句和一般 mybatis 映射文件的区别在于具体 sql 语句变成了 flying 特征值描述。

然后在 `AccountMapper.java` 中加入：
```
public int update(Account t);
public int updatePersistent(Account t);
```
然后使用以下代码，可以将 accountToUpdate 的 name 更新为 “duke” 。
```
accountToUpdate.setName("duke");
accountService.update(accountToUpdate);
```
update 和 updatePersistent 方法的返回值代表执行 sql 后产生影响的条数，一般来说，返回值为 0 表示 sql 执行后没有效果，返回值为 1 表示 sql 执行成功，在代码中可以通过判断 update 和 updatePersistent 方法的返回值来实现更复杂的事务逻辑。

下面我们来说明 update 和 updatePersistent 和关系。如果我们执行
```
accountToUpdate.setName(null);
accountService.update(accountToUpdate);
```
实际上数据库中这条数据的 name 字段不会改变，因为 flying 对为 null 的属性有保护措施。这在大多数情况下都是合理的，但如果我们真的需要在数据库中将这条数据的 name 字段设为 null，updatePersistent 就派上了用场。我们可以执行：
```
accountToUpdate.setName(null);
accountService.updatePersistent(accountToUpdate);
```
这样数据库中这条数据的 name 字段就会变为 null。可见 updatePersistent 会把 pojo 中所有的属性都更新到数据库中，而 update 只更新不为 null 的属性。在实际使用 updatePersistent 时，您需要特别小心慎重，因为当时 pojo 中为 null 的属性有可能比您想象的多。

## [selectAll & count](#Index)
在之前学习 select 和 selectOne 时，细心的您可能已经发现，这两个方法要完成的工作似乎是相同的。的确 select 和 selectOne 都返回 1 个绑定了数据的 pojo，但它们接受的参数不同：select 接受主键参数；selectOne 接受 pojo 参数，这个 pojo 中的所有被 `@FieldMapperAnnotation` 标记过的属性都会作为“相等”条件传递到 sql 语句中。之所以要这么设计，是因为我们有时会需要按照一组条件返回多条数据或者数量，即 selectAll 方法与 count 方法，这个时候以 pojo 作为入参最为合适。为了更清晰的讲述，我们先给 `Account.java` 再增加一个属性 address：
```
@FieldMapperAnnotation(dbFieldName = "address", jdbcType = JdbcType.VARCHAR)
private java.lang.String address;
/*相关的getter和setter方法请自行补充*/
```
然后我们在 `account.xml` 中增加以下内容：
```
<select id="selectAll" resultMap="result">
    flying:selectAll
</select>
<select id="count" resultType="int">
    flying:count
</select>
```
再在 `AccountMapper.java` 中加入
```
public Collection<Account> selectAll(Account t);
public int count(Account t);
```
就可以了。例如使用以下代码，可以查询所有 address 为 “beijing” 的数据和数量：
```
Account condition = new Account();
condition.setAddress("beijing");
Collection<Account> accountCollection = accountService.selectAll(condition);
int accountNumber = accountService.count(condition);
```
（当然一般来说执行 selectAll 后就不需要执行 count 了，我们取结果集的 size 即可，但如果我们只关心数量不关心具体数据集时，执行 count 比执行 selectAll 更节省时间）

如果我们想查询所有 address 为 “shanghai” 同时 name 为 “ella” 的账户，则执行以下代码：
```
Account condition = new Account();
condition.setAddress("shanghai");
condition.setName("ella");
Collection<Account> accountCollection = accountService.selectAll(condition);
```
如果我们知道 address 为 "shanghai" 同时  name 为 "ella" 的账户只有一个，并想直接返回这个数据绑定的 pojo，可以执行： 
```
Account account = accountService.selectOne(condition);
```
由此可见 selectOne 可以称作是 selectAll 的特殊形式，它只会返回一个 pojo 而不是 pojo 的集合。如果真的有多条数据符合给定的 codition ，也只会返回查询结果中排在最前面的数据。尽管如此，在合适的地方使用 selectOne 代替 selectAll，会让您的程序获得极大方便。

## [foreign key](#Index)
一般来说我们的 pojo 都是业务相关的，而这些相关性归纳起来无外乎一对一、一对多和多对多。其中一对一是一对多的特殊形式，多对多本质上是由两个一对多组成，所以我们只需要着重解决一对多关系，而 flying 就是为此而生的。

首先我们定义一个新的 pojo：角色（role）。角色和账户是一对多关系，即一个账户只能拥有一个角色，一个角色可以被多个账户拥有。为此我们要新建 [一个有代表性的 role 表](#RoleTableCreater)、`role.xml`、`RoleMapper.java` 以及 `Role.java`。`role.xml` 如下：
``` 
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"  "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="myPackage.RoleMapper">
    <cache />
    <select id="select" resultMap="result">
        flying#{?}:select
    </select>
    <select id="selectOne" resultMap="result">
        flying:selectOne
    </select>
    <select id="selectAll" resultMap="result">
        flying:selectAll
    </select>
    <select id="count" resultType="int">
        flying:count
    </select>
    <insert id="insert" useGeneratedKeys="true" keyProperty="id">
        flying:insert
    </insert>
    <update id="update">
        flying:update
    </update>
    <update id="updatePersistent">
        flying:updatePersistent
    </update>
    <delete id="delete">
        flying:delete
    </delete>
    <resultMap id="result" type="Role" autoMapping="true">
        <id property="id" column="role_id" />
    </resultMap>
</mapper>
```
`RoleMapper.java` 如下：
``` 
package myPackage;
public interface RoleMapper {
    public Role select(Object id);
    public Role selectOne(Role t);
    public Collection<Role> selectAll(Role t);
    public void insert(Role t);
    public int update(Role t);
    public int updatePersistent(Role t);
    public int delete(Role t);
    public int count(Role t);
}
```
`Role.java` 如下：
```  
package myPackage;
import org.apache.ibatis.type.JdbcType;
import indi.mybatis.flying.annotations.FieldMapperAnnotation;
import indi.mybatis.flying.annotations.TableMapperAnnotation;
    
@TableMapperAnnotation(tableName = "role")
public class Role {

    @FieldMapperAnnotation(dbFieldName = "role_id", jdbcType = JdbcType.INTEGER, isUniqueKey = true)
    private Integer id;
	    
    @FieldMapperAnnotation(dbFieldName = "role_name", jdbcType = JdbcType.VARCHAR)
    private String roleName;
    /*相关的getter和setter方法请自行补充*/
}
```
然后在 `Account.java` 中，加入以下内容：
```   
@FieldMapperAnnotation(dbFieldName = "fk_role_id", jdbcType = JdbcType.INTEGER, dbAssociationUniqueKey = "role_id")
private Role role;   
/*相关的getter和setter方法请自行补充*/
```
以上代码中，`dbFieldName` 的值为数据库表 account 中指向表 role 的外键名，`jdbcType` 的值为这个外键的类型，`dbAssociationUniqueKey` 的值为此外键对应的另一表的主键的名称，写出以上信息后，flying 在代码层面已经完全理解了数据结构。

最后在 `account.xml` 的 `resultMap` 元素中，加入以下内容
```   
<association property="role" javaType="Role" select="myPackage.RoleMapper.select" column="fk_role_id" /> 
```
写出以上信息后，flying 在配置文件层面已经完全理解了数据结构。（此处除 association 之外还有另一种使用 typeHandler 的解决方案，稍后您可以在 [跨数据源](#%E8%B7%A8%E6%95%B0%E6%8D%AE%E6%BA%90) 一节中看到。）

最后总结一下，完整版的 `account.xml` 如下：
``` 
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"  "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="myPackage.AccountMapper">
    <cache />
    <select id="select" resultMap="result">
        flying#{?}:select
    </select>
    <select id="selectOne" resultMap="result">
        flying:selectOne
    </select>
    <select id="selectAll" resultMap="result">
        flying:selectAll
    </select>
    <select id="count" resultType="int">
        flying:count
    </select>
    <insert id="insert" useGeneratedKeys="true" keyProperty="id">
        flying:insert
    </insert>
    <update id="update">
        flying:update
    </update>
    <update id="updatePersistent">
        flying:updatePersistent
    </update>
    <delete id="delete">
        flying:delete
    </delete>
    <resultMap id="result" type="Account" autoMapping="true">
        <id property="id" column="account_id" />
        <association property="role" javaType="Role" select="myPackage.RoleMapper.select" column="fk_role_id" />
    </resultMap>
</mapper>
```
在写完以上代码后，我们看看 flying 能做到什么。首先多对一关系中的<b>一</b>（也即父对象），是可以在多对一关系中的<b>多</b>（也即子对象）查询时自动查询的。为了说明接下来的例子，我们先以 dataset 的方式定义一个数据集
```
<dataset>
    <account account_id="1" fk_role_id="10" address="beijing" name="frank" />
    <account account_id="2" fk_role_id="11" address="tianjin" name="gale" />
    <account account_id="3" fk_role_id="11" address="beijing" name="hank" />
    <role role_id="10" role_name="user" />
    <role role_id="11" role_name="super_user" />
</dataset>
``` 
我们使用这个数据集进行测试，当我们输入以下代码时：
```
Account account1 = accountService.select(1);
/*此时account1的role属性也已经加载了真实数据*/
Role role1 = account1.getRole();
/*role1.getId()为10，role1.getRoleName()为"user"*/
```
这种传递是可以迭代的，即如果 Role 自己也有父对象，则 Role 的父对象也会一并加载，只要它的配置文件和代码正确。

不仅如此，我们可以在入参 pojo 中加入父对象，比如下面的代码查询的是角色名为 “super_user” 的所有帐户：
```
Role roleCondition = new Role();
roleCondition.setRoleName("super_user");
Account accountCondition = new Account();
accountCondition.setRole(roleCondition);
Collection<Account> accounts = accountService.selectAll(accountCondition);
/*accounts.seiz()为 2，里面包含的对象的 account_id 是 2 和 3*/

/*我们再给入参pojo加一个address限制*/
accountCondition.setAddress("beijing");
Collection<Account> accounts2 = accountService.selectAll(accountCondition);
/*accounts.size()为 1，里面包含的对象的 account_id 是 3，这说明 account 的条件和父对象 role 的条件同时生效*/
```
这个特性在 selectOne、count 中同样存在

最后，父对象同样可以参与子对象的 insert、update、updatePersistent，代码如下：
```
Account newAccount = new Account();

/*我们新建一个姓名为 iris，角色名称为 "user" 的账号*/
newAccount.setName("iris");

/*角色名称为 "user" 的数据的 role_id 是 10，由变量 role1 来加载它*/
Role role1 = roleService.select(10);
newAccount.setRole(role1);
accountService.insert(newAccount);
/*一个姓名为iris，角色名称为"user"的账号建立完成*/

/*我们用update方法将iris的角色变为"super_user"*/
/*角色名称为"super_user"的数据的role_id是11，由变量role2来加载了它*/
Role role2 = roleService.select(11);
newAccount.setRole(role2);
accountService.update(newAccount);
/*现在newAccount.getRole().getId()为11，newAccount.getRole().getRoleName为"super_user"*/

/*我们用updatePersistent方法将iris的角色变为null，即与Role对象不再关联*/
newAccount.setRole(null);
accountService.updatePersistent(newAccount);
/*现在 newAccount.getRole()为 null，在数据库中也不再有关联（注意在这里 update 方法起不到这种效果，因为 update 会忽略 null）*/
```

## [complex condition](#Index)
之前我们展示的例子中，条件只有“相等”一种，但在实际情况中我们会遇到各种各样的条件：大于、不等于、like、in、is not null 等等。这些情况 flying 也是能够处理的，但首先我们要引入一个“条件对象”的概念。条件对象是实体对象的子类，但它只为查询而存在，它拥有实体对象的全部属性，同时它还有一些专为查询服务的属性。例如下面是 Account 对象的条件对象 AccountCondition 的代码：
```
package myPackage;
import java.util.Collection;
import java.util.List;
import indi.mybatis.flying.annotations.ConditionMapperAnnotation;
import indi.mybatis.flying.annotations.QueryMapperAnnotation;
import indi.mybatis.flying.models.Conditionable;
import indi.mybatis.flying.statics.ConditionType;
@QueryMapperAnnotation(tableName = "account")
public class AccountCondition extends Account implements Conditionable {

    @ConditionMapperAnnotation(dbFieldName = "name", conditionType = ConditionType.Like)
    /*用作 name 全匹配的值*/
    private String nameLike;
	
    @ConditionMapperAnnotation(dbFieldName = "address", conditionType = ConditionType.HeadLike)
    /*用作 address 开头匹配的值*/
    private String addressHeadLike;
	
    @ConditionMapperAnnotation(dbFieldName = "address", conditionType = ConditionType.TailLike)
    /*用作 address 结尾匹配的值*/
    private String addressTailLike;
	
    @ConditionMapperAnnotation(dbFieldName = "address", conditionType = ConditionType.MultiLikeAND)
    /*用作 address 需要同时匹配的若干个值的集合（类型只能为List）*/
    private List<String> addressMultiLikeAND;
	
    @ConditionMapperAnnotation(dbFieldName = "address", conditionType = ConditionType.MultiLikeOR)
    /*用作 address*/ 需要至少匹配之一的若干个值的集合（类型只能为List）
    private List<String> addressMultiLikeOR;
	
    @ConditionMapperAnnotation(dbFieldName = "address", conditionType = ConditionType.In)
    /*用作 address*/ 可能等于的若干个值的集合（类型可为任意Collection）
    private Collection<String> addressIn;
	
    @ConditionMapperAnnotation(dbFieldName = "address", conditionType = ConditionType.NotIn)
    /*用作 address 不可能等于的若干个值的集合（类型可为任意Collection）*/
    private Collection<String> addressNotIn;
	
    @ConditionMapperAnnotation(dbFieldName = "address", conditionType = ConditionType.NullOrNot)
    /*用作 address 是否为 null 的判断（类型只能为Boolean）*/
    private Boolean addressIsNull;
	
    /*相关的getter和setter方法请自行补充*/
	
    /*以下四个方法是实现 Conditionable 接口后必须要定义的方法，我们这里只写出默认实现，在下一节中我们会详细介绍它们*/
    @Override
    public Limitable getLimiter() {
        return null;
    }
    @Override
    public void setLimiter(Limitable limiter) {
    }
    @Override
    public Sortable getSorter() {
        return null;
    }
    @Override
    public void setSorter(Sortable sorter) {
    }
}
```
以上各种条件并非要全部写出，您可以只写出业务需要的条件（变量名可以是任意的，只要条件标注准确即可）。在 flying 中进行复杂条件查询前需要先按需求写一些条件代码，但请您相信，从长远来看这种做法的回报率是极高的。然后我们可以进行测试：
```
/*查询名称中带有"a"的帐户数量*/
AccountCondition condition1 = new AccountCondition();
condition1.setNameLike("a");
int count1 = accountService.count(condition1);

/*查询地址以"bei"开头的帐户的数量*/
AccountCondition condition2 = new AccountCondition();
condition2.setAddressHeadLike("bei");
int count2 = accountService.count(condition2);

/*查询地址以"jing"结尾的帐户的数量*/
AccountCondition condition3 = new AccountCondition();
condition3.setAddressTailLike("jing");
int count3 = accountService.count(condition3);

/*查询地址同时包含"e"和"i"的账户的数量*/
List<String> listAddressMultiLikeAND = new ArrayList<>();
listAddressMultiLikeAND.add("e");
listAddressMultiLikeAND.add("i");
AccountCondition condition4 = new AccountCondition();
condition4.setAddressMultiLikeAND(listAddressMultiLikeAND);
int count4 = accountService.count(condition4);

/*查询地址至少包含"e"或"i"的账户的数量*/
List<String> listAddressMultiLikeOR = new ArrayList<>();
listAddressMultiLikeOR.add("e");
listAddressMultiLikeOR.add("i");
AccountCondition condition5 = new AccountCondition();
condition5.setAddressMultiLikeOR(listAddressMultiLikeOR);
int count5 = accountService.count(condition5);

/*查询地址等于"beijing"或"shanghai"中的一个的账户的数量*/
List<String> listAddressIn = new ArrayList<>();
listAddressIn.add("beijing");
listAddressIn.add("shanghai");
AccountCondition condition6 = new AccountCondition();
condition6.setAddressIn(listAddressIn);
int count6 = accountService.count(condition6);

/*查询地址不等于"beijing"或"shanghai"的账户的数量*/
List<String> listAddressNotIn = new ArrayList<>();
listAddressNotIn.add("beijing");
listAddressNotIn.add("shanghai");
AccountCondition condition7 = new AccountCondition();
condition7.setAddressNotIn(listAddressNotIn);
int count7 = accountService.count(condition7);

/*查询地址为null的账户的数量*/
AccountCondition condition8 = new AccountCondition();
condition8.setAddressIsNull(true);
int count8 = accountService.count(condition8);

/*最后我们查询名称中带有"a"且地址为"beijing"的帐户的数量*/
AccountCondition conditionX = new AccountCondition();
conditionX.setNameLike("a");
conditionX.setAddress("beijing");
int countX = accountService.count(conditionX);
/*这个用例说明条件变量也可以使用 pojo 本身的字段进行查询*/
```
## [limiter & sorter](#Index)
在之前的 selectAll 查询中我们都是取符合条件的所有值，但在实际业务需求中很少会这样做，更多的情况是我们会有一个数量限制。同时我们还会希望结果集是经过某种条件排序，甚至是经过多种条件排序的，幸运的是 flying 已经为此做好了准备。

一个可限制数量并可排序的查询也是由条件对象来实现的，代码如下：
```
package myPackage;
import indi.mybatis.flying.annotations.QueryMapperAnnotation;
import indi.mybatis.flying.models.Conditionable;
import indi.mybatis.flying.models.Limitable;
import indi.mybatis.flying.models.Sortable;
@QueryMapperAnnotation(tableName = "account")
public class AccountCondition extends Account implements Conditionable {
    private Limitable limiter;
    private Sortable sorter;
    @Override
    public Limitable getLimiter() {
        return limiter;
    }
    @Override
    public void setLimiter(Limitable limiter) {
        this.limiter = limiter;
    }
    @Override
    public Sortable getSorter() {
        return sorter;
    }
    @Override
    public void setSorter(Sortable sorter) {
        this.sorter = sorter;
    }
}
```
以上 limiter 和 sorter 变量名并非固定，只要类引入了 Conditionable 接口并实现相关方法，且在相关方法中对应上您定义的 limiter 和 sorter 即可。
然后可以采用如下代码进行测试：
```
import indi.mybatis.flying.models.Conditionable;
import indi.mybatis.flying.pagination.Order;
import indi.mybatis.flying.pagination.PageParam;
import indi.mybatis.flying.pagination.SortParam;

/*查询 account 表在默认排序下前 10 条数据*/
AccountCondition condition1 = new AccountCondition();
/*PageParam 的构造函数中第一个参数为起始页数，第二个参数为每页容量，new PageParam(0,10)即从头开始取 10 条数据*/
condition1.setLimiter(new PageParam(0, 10));
Collection<Account> collection1 = accountService.selectAll(codition1);

/*查询 account 表在默认排序下第 8 条数据*/
AccountCondition condition2 = new AccountCondition();
/*new PageParam(7,1)即从第 7 条开始取 1 条数据*/
condition2.setLimiter(new PageParam(7, 1));
/*因为结果只需要一条数据，我们可以使用 selectOne 方法*/
Account account2 = accountService.selectOne(condition2);

/*查询 account 表在 name 正序排序下的所有数据*/
AccountCondition condition3 = new AccountCondition();
/*new Order()的第一个参数是被排序的字段名，第二个参数是正序或倒序*/
condition3.setSorter(new SortParam(new Order("name", Conditionable.Sequence.asc)));
Collection<Account> collection3 = accountService.selectAll(codition3);

/*查询 account 表先在 name 正序排序，然后在 address 倒序排序下的所有数据*/
AccountCondition condition4 = new AccountCondition();
/*在new SortParam()中可以接受不定数量的 Order 参数，因此我们先新建一个 name 正序，再新建一个 address 倒序*/
condition4.setSorter(new SortParam(new Order("name", Conditionable.Sequence.asc),new Order("address", Conditionable.Sequence.desc)));
Collection<Account> collection4 = accountService.selectAll(codition4);

/*最后我们查询在 name 正序排序下的第 11 到 20 条数据*/
AccountCondition conditionX = new AccountCondition();
conditionX.setSorter(new SortParam(new Order("name", Conditionable.Sequence.asc)));
conditionX.setLimiter(new PageParam(1, 10));
Collection<Account> collectionX = accountService.selectAll(coditionX);
/*这个用例说明 limiter 和 sorter 是可以组合使用的*/
```
因为 limiter 和 sorter 也是以条件对象的方式定义，所以可以和复杂查询一起使用，只要在条件对象中既包含条件标注又包含 Limitable 和 Sortable 类型的变量即可。
## [分页](#Index)
在大多数实际业务需求中，我们的 limiter 和 sorter 都是为分页服务。在 flying 中，我们提供了一种泛型 Page&lt;?&gt; 来封装查询出的数据。使用 Page&lt;?&gt; 的好处是，它除了提供数据内容（pageItems）外还提供了全部数量（totalCount）、最大页数（maxPageNum）、当前页数（pageNo）等信息，这都是数据接收端希望了解的信息。并且这些数量信息是 flying 自动获取的，您只需执行下面这样的代码即可：
```
import indi.mybatis.flying.pagination.Page;

AccountCondition condition = new AccountCondition();
condition.setLimiter(new PageParam(0, 10));
Collection<Account> collection = accountService.selectAll(condition);

/*下面这句代码就将查询结果封装为了 Page<?> 对象*/
Page<Account> page = new Page<>(collection, condition.getLimiter());
/*需要注意的是上面的入参 condition.getLimiter() 是不能用其它任意 PageParam 对象代替的，因为在之前执行 selectAll 时已经将一些信息保存到 condition.getLimiter() 中*/
```
假设总的数据有 21 条，则 `page.getTotalCount()` 为 21，`pagegetMaxPageNum()` 为 3，`page.getPageNo()` 为 1，`page.getPageItems()` 为第一到第十条数据的集合。
## [乐观锁](#Index)
乐观锁是实际应用的数据库设计中重要的一环，而 flying 在设计之初就考虑到了这一点，
目前 flying 只支持版本号型乐观锁。在 flying 中使用乐观锁的方法如下：
在数据结构中增加一个表示乐观锁的 Integer 型字段 opLock 即可：
```
@FieldMapperAnnotation(dbFieldName = "opLock", jdbcType = JdbcType.INTEGER, opLockType = OpLockType.Version)
private Integer opLock;
/*乐观锁可以增加 getter 方法，不建议增加 setter 方法*/
```
以上实际上是给 `@FieldMapperAnnotation` 中的 `opLockType` 上赋予了 `OpLockType.Version`，这样 flying 就会明白这是一个起乐观锁作用的字段。当含有乐观锁的表 account 更新时，实际 sql 会变为：
```
update account ... and opLock = opLock + 1 where id = '${id}' and opLock = '${opLock}'
```
（上面 ... 中的内容是给其它的字段赋值）

每次更新时都会加入 opLock 的判断，并且更新数据时 opLock 自增 1 ，这样就可以保证多个线程对同一个 account 执行 update 或 updatePersistent 时只有一个能执行成功，即达到了我们需要的锁效果。

当含有乐观锁的表 account 删除时，实际 sql 会变为：
```
delete from account where id = '${id}' and opLock = '${opLock}'
```
即只有 opLock 和 id 都符合时才能被删除，这里乐观锁起到了保护数据的作用。

在实际应用中，可以借助 update、updatePersistent、delete 方法的返回值来判断是否变动了数据（一般来说返回 0 表示没变动，1 表示有变动），继而判断锁是否有效，是否合法（符合业务逻辑），最后决定整个事务是提交还是回滚。

最后我们再来谈谈为什么不建议给乐观锁字段加上 setter 方法。首先在代码中直接修改一个 pojo 的乐观锁值是很危险的事情，它会导致事务逻辑的不可靠；其次乐观锁不参与 select、selectAll、selectOne 方法，即便给它赋值在查询时也不会出现；最后乐观锁不参与 insert 方法，无论给它赋什么值在新增数据中此字段的值都是零，即乐观锁总是从零开始增长。
## [其它](#Index)
### [ignore tag](#Index)
有时候，我们希望在查询中忽略某个字段的值，但在作为查询条件和更新时要用到这个字段。一个典型的场景是 password 字段，出于安全考虑我们不想在 select 方法返回的结果中看到它的值，但我们需要在查询条件（如判断登录）和更新（如修改密码）时使用到它，这时我们可以在 Account.java 中加入以下代码：
```
@FieldMapperAnnotation(dbFieldName = "password", jdbcType = JdbcType.VARCHAR, ignoreTag = { "noPassword" })
private String password;
/*相关的getter和setter方法请自行补充*/
```
这样我们将 `password` 这个字段加上了一个忽略标记 `noPassword`，然后在查询 account 表时相关 flying 特征值最后加上 `:noPassword` 就不会再查找 password 字段，但作为查询条件和更新数据时 password 字段都可以参与进来，如下所示：
```
<select id="selectOne" resultMap="result">
    flying:selectOne:noPassword
</select>
```
然后，可通过代码验证 `password` 属性已被忽略
```
/*查找 name 为 "user" 且 password 为 "123456" 的一个账户*/
Account condition = new Account();
condition.setName("user");
condition.setPassword("123456");
Account account = accountService.selectOne(condition);
/*用以上方式是可以查出 passeord 为 "123456" 的账户的，然而结果中 account.getPassword()为 null*/

/* 但是仍然可以更新 password 的值 */
account.setPassword("654321");
accountService.update(account);
/*现在 account 对应的数据库中数据的 password 字段值变为 "654321"*/
```
另一种场景是查询对象中有一个长度很大的属性，例如我们在数据库中有一个类型为 varchar 长度为 3000 的属性 `detail`，为性能考虑，在不需要查看 `detail` 详情的情况下我们不想将其 select 出来，而忽略标记就可以做到这一点。我们在 account.java 中增加如下代码：
```
@FieldMapperAnnotation(dbFieldName = "detail", jdbcType = JdbcType.VARCHAR, ignoreTag = { "noDetail" })
private String detail;
/*相关的getter和setter方法请自行补充*/
```
此时用 flying 特征值为 `flying#{?}:select:noDetail` 的方法就不会查出 `detail` 字段；如果我们在某些情况又需要得到 `detail` 的内容，再增加一个特征值不带 `:noDetail` 的查询方法即可，例如直接用 `flying#{?}:select`。

如果我们想既不查询 `detail` 又不查询 `password`，可在 `password` 的注解上使用多个忽略标记，就像下面这样：
```
@FieldMapperAnnotation(dbFieldName = "password", jdbcType = JdbcType.VARCHAR, ignoreTag = { "noPassword", "noDetail" })
private String password;
```
这时特征值 `flying#{?}:select:noDetail` 就既忽略 `detail` 又忽略 `password`。
由此可见，在实体类中一个属性可配置多个忽略标记，其中一个被激活这个属性就不会参与查询；但是 flying 特征值每次只能激活一个忽略标记，所以如果您有多样化的忽略需求，您需要在实体类中仔细配置以满足需要。

最后，flying 特征值中的忽略标记没有传递性，只对当前查询对象有效而对自动查询的父对象无效。例如对 `Account` 对象的 `flying#{?}:select:noPassword` 查询，其忽略标记对自动查询的父对象 `Role` 无效，哪怕 `Role` 中有 `ignoreTag` 等于 'noPassword' 的属性也会查询出来。如果您需要激活自动查询的父对象中的忽略标记，您需要调整 `<resultMap>` 中的 `<association>` 的设置，让其指向一个激活了忽略标记的查询，例如：
```
<association property="role" javaType="Role" select="myPackage.RoleMapper.selectIgnore_" column="fk_role_id" />
```
如果您既需要带有激活忽略标记的自动查询父对象又需要不带激活忽略标记的自动查询父对象，那您为查询对象定义多个 `resultMap` 即可。

`最新版本新增` 忽略标记在 insert、update、updatePersist 中同样会生效，具体作用是激活后新增、修改时忽略此字段，某些情况下您会发现这种需要。另外，如果此字段受到 JPA 标签 `@Column` 修饰并且 `insertable = false` 或 `updateable = false`，则不论此字段上的 ignoreTag 为何，在任何情况下此字段都被认为是新增忽略或修改忽略。
### [复数外键](#Index)
有时候一个数据实体会有多个多对一关系指向另一个数据实体，例如考虑下面的情况：我们假设每个账户都有一个兼职角色，这样 account 表中就需要另一个字段 fk_second_role_id，而这个字段也是指向 role 表。为了满足这个需要，首先我们要在 account.xml 的 resultMap元素中，加入以下内容：
```
<association property="secondRole" javaType="Role" select="myPackage.RoleMapper.select" column="fk_second_role_id" />
```
然后在 Account.java 中还需要加入以下代码：
```
@FieldMapperAnnotation(dbFieldName = "fk_second_role_id", jdbcType = JdbcType.INTEGER, dbAssociationUniqueKey = "role_id")
private Role secondRole;   
/*相关的getter和setter方法请自行补充*/
```
如此一来表 account 和表 role 就构成了复数外键关系。flying 支持复数外键，您可以像操作普通外键一样操作它，代码如下：
```
/*查询角色名称为 "user",同时兼职角色名称为 "super_user" 的账户*/
Account condition = new Account();
Role role = new Role(), secondRole = new Role();
role.setName("user");
condition.setRole(role);
secondRole.setName("super_user");
condition.setSecondRole(secondRole);
Collection<Account> accounts = accountService.selectAll(condition);
```
可见，复数外键的增删改查等操作与普通外键是类似的，只需要注意虽然 secondRole 的类型为Role，但它的 getter、setter 是 getSecondRole()、setSecondRole()，而不是 getRole()、setRole()即可。
### [跨数据源](#Index)
在实际开发中，越来越多的系统采用分布式数据库设计，flying 对此也提供支持。flying 采用的是真实数据源配合自定义 TypeHandler 的方式，而非动态切换虚拟数据源方式，这样做的好处如下：
- 动态切换虚拟数据源方式需要频繁切换数据源，而真实数据源方式本身就是多个数据源无需切换，避免了这方面的开销。
- 动态切换虚拟数据源方式多数据源和单数据源实现差异较大，用户如果从单数据源升级至多数据源需要变更很多内容；而采用真实数据源配合自定义 TypeHandler 的方式，完全利用了 mybatis 自身支持多数据源特性，将单数据源看做多数据源的一种特例，每次新增数据源的配置都很少且易于理解。
- flying 对真实数据源配合自定义 TypeHandler 的方式进行了优化，当您在业务代码中调用数据时您不需要知道哪些是跨数据源调用哪些是单数据源调用，您也基本感知不到它们的不同。

为了更好的说明 flying 跨数据源实现方式，在本小节中，我们假定 Account 表和 Role 表处于不同的数据源内，前者的数据源为 dataSource1，后者的数据源为 dataSource2。因此 spring 中的配置如下：
```
    <bean id="dataSource1" class="org.apache.commons.dbcp.BasicDataSource" destroy-method="close" />
    <bean id="sqlSessionFactory" class="org.mybatis.spring.SqlSessionFactoryBean">
	<property name="configLocation" value="classpath:Configuration.xml" />
	<property name="dataSource" ref="dataSource1" />
	<property name="mapperLocations" value="classpath*:myPackage/mapper/*.xml" />
	<property name="typeAliasesPackage" value="myPackage" />
    </bean>
    <bean id="mapperScannerConfigurer" class="org.mybatis.spring.mapper.MapperScannerConfigurer">
	<property name="basePackage" value="myPackage" />
	<property name="sqlSessionFactoryBeanName" value="sqlSessionFactory" />
    </bean>

    <bean id="dataSource2" class="org.apache.commons.dbcp.BasicDataSource" destroy-method="close" />
    <bean id="sqlSessionFactory2" class="org.mybatis.spring.SqlSessionFactoryBean">
	<property name="configLocation" value="classpath:Configuration.xml" />
	<property name="dataSource" ref="dataSource2" />
	<property name="mapperLocations" value="classpath*:myPackage/mapper2/*.xml" />
	<property name="typeAliasesPackage" value="myPackage" />
    </bean>
    <bean id="mapperScannerConfigurer2" class="org.mybatis.spring.mapper.MapperScannerConfigurer">
	<property name="basePackage" value="myPackage" />
	<property name="sqlSessionFactoryBeanName" value="sqlSessionFactory2" />
    </bean>

    <!--因为TypeHandler并非第一时间初始化，不能以@Autowired方式调用Bean，所以增加ApplicationContextProvider方式来调用Bean-->
    <bean id="applicationContextProvder" class="indi.demo.flying.ApplicationContextProvider" />
```
以上配置文件中描述了两个数据源 `dataSource1` 和 `dataSource2` 以及它们对应的 `sqlSessionFactory` 和 `mapperScannerConfigurer`，至于最后的 `applicationContextProvder`，它的具体代码是：
```
package indi.demo.flying;
import org.springframework.beans.BeansException;
import org.springframework.context.ApplicationContext;
import org.springframework.context.ApplicationContextAware;

public class ApplicationContextProvider implements ApplicationContextAware {
	private static ApplicationContext context;

	public static ApplicationContext getApplicationContext() {
		return context;
	}

	@Override
	public void setApplicationContext(ApplicationContext ctx) throws BeansException {
		context = ctx;
	}
}
```
这个 `ApplicationContextProvider` 的作用我们后面就会看到。之后我们还要替换 Account 类中 role 属性的注解，如下所示：
```
    @FieldMapperAnnotation(dbFieldName = "fk_role_id", jdbcType = JdbcType.INTEGER, dbAssociationTypeHandler = myPackage.typeHandler.RoleTypeHandler.class)
    private Role role;
```
这实际上是将原本的 `dbAssociationUniqueKey` 替换为 `dbAssociationTypeHandler`，而 `dbAssociationTypeHandler` 需要指定一个类作为值，所以我们还要开发一个 RoleTypeHandler，如下：
```
package myPackage.typeHandler;
import java.sql.CallableStatement;
import java.sql.PreparedStatement;
import java.sql.ResultSet;
import java.sql.SQLException;
import org.apache.ibatis.type.BaseTypeHandler;
import org.apache.ibatis.type.JdbcType;
import org.apache.ibatis.type.MappedTypes;
import org.apache.ibatis.type.TypeHandler;
import indi.demo.flying.ApplicationContextProvider;
import myPackage.Role;
import myPackage.RoleService;

@MappedTypes({ Role.class })
public class RoleTypeHandler extends BaseTypeHandler<Role> implements TypeHandler<Role> {
	@Override
	public Role getNullableResult(ResultSet arg0, String arg1) throws SQLException {
		if (arg0.getString(arg1) == null) {
			return null;
		}
		return (getService().mySelect(arg0.getString(arg1)));
	}
	@Override
	public Role getNullableResult(ResultSet arg0, int arg1) throws SQLException {
		if (arg0.getString(arg1) == null) {
			return null;
		}
		return (getService().mySelect(arg0.getString(arg1)));
	}
	@Override
	public Role getNullableResult(CallableStatement arg0, int arg1) throws SQLException {
		if (arg0.getString(arg1) == null) {
			return null;
		}
		return (getService().mySelect(arg0.getString(arg1)));
	}
	@Override
	public void setNonNullParameter(PreparedStatement arg0, int arg1, Role arg2, JdbcType arg3) throws SQLException {
		if (arg2 != null) {
			arg0.setString(arg1, arg2.getId());
		}
	}

	/*
	 * 因为此TypeHandler并非第一时间初始化，不能以@Autowired方式调用RoleService，所以采用下面的方式
	 */
	private RoleService getService() {
		return (RoleService) ApplicationContextProvider.getApplicationContext().getBean(RoleService.class);
	}
}
```
如果您对这个类的代码不是很熟悉，您可以了解一下 mybatis 自定义 TypeHandler 的机制。另外，您可以看到我们之前开发的 `ApplicationContextProvider` 在此处发挥了作用。最后，我们还要修改 account.xml 中的 resultMap，如下所示：
```
    <resultMap id="result" type="Account" autoMapping="true">
        <id property="id" column="account_id" />
        <result property="role" typeHandler="myPackage.typeHandler.RoleTypeHandler" column="fk_role_id" />
    </resultMap>
```
以上是把 resultMap 中的 association 方式替换为 typeHandler 方式，关于 association 方式和 typeHandler 方式的区别，[您可以参考这里](#association-or-typeHandler)。

现在，您就可以使用如下代码来操作跨数据源的 Account 和 Role 对象了：
```
/* 跨数据源时，也可以新建关联了父对象的子对象 */
Account newAccount = new Account();
newAccount.setRole(role);
accountService.insert(newAccount);
/* 此时数据库中新增的newAccount已于另一数据源的role关联起来 */

/* 跨数据源时，子对象查询时也可自动加载父对象 */
Account account = accountService.select(newAccount.getId());
/* 此时account.getRole()的值即account关联的处于另一数据源的role */

/* 跨数据源时，子对象也可更新父对象 */
account.setRole(otherRole);
accountService.update(account);
/* 此时数据库中account和另一数据源的otherRole关联起来 */
```
然而跨数据源关联毕竟不同于单数据源，它无法做到将父对象除主键外的其它属性作为条件参与查询。实际上这是由于数据库的限制，目前大部分的数据库还不支持跨数据源的外键关联查询，更不用说是跨数据源异构数据库（例如一方是 oracle 另一方是 mysql）。当然对于支持跨数据源外键关联查询的数据库（例如使用了 federated 引擎的  mysql），我们在今后也会考虑支持它的特性。

<a id="flying-demo2"></a>
最后，这里有一个[跨数据源应用的代码示例](https://github.com/limeng32/flying-demo2/tree/use-flying-0.9.2)，相信您看完以后会对 flying 实现跨数据源的方法了然于胸。（同时这个例示还使用了 mybatis 的二级缓存，关于此方面内容我们会在下一篇文章中进行详细介绍）
### [兼容 JPA 标签](#Index)
`最新版本新增` flying 对部分常用的 JPA 标签进行了兼容，具体内容为：

- `@Id` 变量标签，表示此变量对应主键
- `@Column` 变量标签，描述字段定义，对 flying 有效属性：`name`（默认为该变量名）、`insertable`(若为 false 此字段在新增时永远被忽略)、`updateable`(若为 false 此字段在修改时永远被忽略)、`columnDefinition`（默认为该变量类型名，若指定则按值来推导类型，对应关系[见此](https://github.com/limeng32/mybatis.flying/blob/master/src/main/java/indi/mybatis/flying/utils/JdbcTypeEnum.java)）
- `@Table` 类标签，描述表定义，对 flying 有效属性：`name`（默认为类名）

和其它 JPA 实现不同的是，以上变量标签只在变量上有效，在 getter 方法上无效。在以上标签和 `FieldMapperAnnotation`、`TableMapperAnnotation` 同时出现时，优先级为（高优先级会覆盖低优先级的相关属性）：

- `@Id` 最高。
- `@FieldMapperAnnotation` 和 `TableMapperAnnotation` 其次。
- `@Column` 和 `@Table` 再次。

关于使用 JPA 的更多内容您可以参考这个[示例](https://github.com/limeng32/flying-demo/tree/use-flying-0.9.2)。
## [附录](#Index)
<a id="FAQ"></a>
### [常见问题](#Index)
<a id="why-no-sql"></a>
1、为何<i>pojo_mapper</i>.xml 中没有 sql 语句细节？

A：flying 的 sql 语句是动态生成的，只要您指定了正确的字段名，就绝对不会出现 sql 书写上的问题。并且 flying 采用了缓存机制，您无需担心动态生成 sql 的效率问题。

<a id="association-or-typeHandler"></a>
2、resultMap 中 association 和 typeHandler 两种方式的区别？

A：在单数据源情况下，这两种方式都可以实现“查询子对象时自动加载父对象”的需要；但在多数据源的情况下，只有 typeHandler 方式才能实现跨数据源关联。我们的建议是在任何情况下都只使用 typeHandler 方式，因为如果您打算在多数据源环境下并且使用 mybatis 的二级缓存，只有全部 resultMap 都采用 typeHandler 才能保证缓存的完整一致性。（在下一篇文章中会对此详细讲解）
<a id="AccountTableCreater"></a>
### [代码示例](#Index)
为了您更方便的使用 flying 进行开发，我们提供了一个[覆盖了本文大部分功能的单数据源的代码示例](https://github.com/limeng32/flying-demo/tree/use-flying-0.9.2)。如果您是对跨数据源感兴趣，则您应该关注[这里](#flying-demo2)。
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

<a id="AccountService"></a>
### [AccountService 的实现方式](#Index)
在 spring 3.x 及更高版本中，可以按以下方式构建一个 <i>pojoService</i>.java 类：
```
package myPackage;

import java.util.Collection;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;

@Service
public class AccountService implements AccountMapper {

	@Autowired
	private AccountMapper mapper;

	@Override
	public Account select(Object id) {
		return mapper.select(id);
	}

	@Override
	public Account selectOne(Account t) {
		return mapper.selectOne(t);
	}
	
	@Override
	public Collection<Account> selectAll(Account t) {
		return mapper.selectAll(t);
	}
	
	@Override
	public void insert(Account t) {
		mapper.insert(t);
	}

	@Override
	public int update(Account t) {
		return mapper.update(t);
	}

	@Override
	public int updatePersistent(Account t) {
		return mapper.updatePersistent(t);
	}

	@Override
	public int delete(Account t) {
		return mapper.delete(t);
	}

	@Override
	public int count(Account t) {
		return mapper.count(t);
	}
}
```
- - -