# 基本说明
```
<selectKey
  keyProperty="id"
  resultType="int"
  order="BEFORE"
  statementType="PREPARED">
```


| 属性 | 描述 |
| ------ | ------ |
|keyProperty | 	selectKey 语句结果应该被设置的目标属性。如果希望得到多个生成的列，也可以是逗号分隔的属性名称列表。|
|keyColumn |	匹配属性的返回结果集中的列名称。如果希望得到多个生成的列，也可以是逗号分隔的属性名称列表。|
|resultType|	结果的类型。MyBatis 通常可以推算出来，但是为了更加确定写上也不会有什么问题。MyBatis 允许任何简单类型用作主键的类型，包括字符串。如果希望作用于多个生成的列，则可以使用一个包含期望属性的 Object 或一个 Map。|
|order |	这可以被设置为 BEFORE 或 AFTER。如果设置为 BEFORE，那么它会首先选择主键，设置 keyProperty 然后执行插入语句。如果设置为 AFTER，那么先执行插入语句，然后是 selectKey 元素 - 这和像 Oracle 的数据库相似，在插入语句内部可能有嵌入索引调用。|
|statementType |	与前面相同，MyBatis 支持 STATEMENT，PREPARED 和 CALLABLE 语句的映射类型，分别代表 PreparedStatement 和 CallableStatement 类型。|

摘抄自官网: http://www.mybatis.org/mybatis-3/zh/sqlmap-xml.html#insert_update_and_delete
# 基本使用
> mapper.xml
```
<insert id="insertAuthor">
  <selectKey keyProperty="id" resultType="int" order="BEFORE">
    select CAST(RANDOM()*1000000 as INTEGER) a from SYSIBM.SYSDUMMY1
  </selectKey>
  insert into Author
    (id, username, password, email,bio, favourite_section)
  values
    (#{id}, #{username}, #{password}, #{email}, #{bio}, #{favouriteSection,jdbcType=VARCHAR})
</insert>
```
> main.java
```
    // 省略获得 mapper 对象的代码和实体类相关代码
    Author author = new Author();
    author.setTitle("11");
    author.setContent("22");
    mapper.insertAuthor(blog);
    System.out.println("id:"+author.getId());
```
执行后,控制台输出:`337856` 一段随机的数字

本文要说明的内容: selectKey 是如何工作的,为什么在 `author.getId()` 会有值.

# 主键生成
## 简要说明
首先来看主键生成的接口
```
public interface KeyGenerator {

  void processBefore(Executor executor, MappedStatement ms, Statement stmt, Object parameter);

  void processAfter(Executor executor, MappedStatement ms, Statement stmt, Object parameter);

}
```

接口 KeyGenerator 的方法 会被xml中selectKey的属性order所影响。
- BEFORE 执行 processBefore 方法
- AFTER  执行 processAfter 方法

KeyGenerator 有三种实现:
- NoKeyGenerator  什么都不做
- Jdbc3KeyGenerator   例如Mysql有自动增长
- SelectKeyGenerator  自己决定如何生成key

例如:在项目中,可以使用UUID作为主键,那么就可以用到 SelectKeyGenerator 的实现
## 调用说明
当order="BEFORE"时, 在执行insert语句之前执行selectKey的内容

对于在mybatis的代码中,则是在创建 `StatementHandler` 对象的时候

> org.apache.ibatis.executor.statement.BaseStatementHandler
```
  protected void generateKeys(Object parameter) {
      KeyGenerator keyGenerator = mappedStatement.getKeyGenerator();
      ErrorContext.instance().store();
      keyGenerator.processBefore(executor, mappedStatement, null, parameter);
      ErrorContext.instance().recall();
    }
```

当order="AFTER"时,在执行insert语句之后执行selectKey的内容

对应代码的 SqlSession 的update方法
```
 @Override
  public int update(Statement statement) throws SQLException {
    PreparedStatement ps = (PreparedStatement) statement;
    ps.execute();
    int rows = ps.getUpdateCount();
    Object parameterObject = boundSql.getParameterObject();
    KeyGenerator keyGenerator = mappedStatement.getKeyGenerator();
    keyGenerator.processAfter(executor, mappedStatement, ps, parameterObject);
    return rows;
  }
```


# 常见问题 
## dubbo调用selectKey的方法问题
当dubbo调用 insertAuthor 之后,通过序列化创建对象`(Author paramAuthor)`

insert之后执行selectKey的内容,  设置 paramAuthor.setId()。

无法改变调用者的author对象的值.




