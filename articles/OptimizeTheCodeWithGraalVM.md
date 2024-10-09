# 背景

在上一篇文章《还在用JVM跑你的Java代码吗》中，我们简要介绍了GraalVM及其核心功能nativeimage，可以将java代码编译为原生二进制可执行文件，与基于 JVM 的 java 服务运行相比，经过编译得到的原生可执行文件的在启动速度与内存占用方面极具优势。

虽然GraalVM二进制编译功能有诸多优势，但是实际进行二进制编译时，还是会遇到一些问题。简单而言，因其编译二进制的工作原理需要对工程代码进行一定程度的改造，所以原先可以直接运行的代码很可能在编译出二进制后执行，会出现执行失败等问题，此时我们需要通过其他手段解决。

# 二进制编译出现问题，如何改造你的工程代码？

为了解决二进制编译问题，一个常用有效的方法是自动收集并配置可达性元数据。这可以通过在项目的 resource 目录下的 META-INF.native-image 文件夹中配置已收集的元数据来实现，从而有效解决部分编译难题。

但有时候通过元数据仍然无法解决问题，尤其是工程中引入的第三方依赖出现了二进制编译兼容的问题，在这种情况下，我们可以考虑使用Graal VM Substitution功能对工程代码做改造，完成对GraalVM二进制编译的适配。

## Graal VM Substitution功能介绍

本文作为Graal VM系列的第二篇，主要介绍nativeimage公共API（native Image public API for advanced use cases）中的 Substitution 功能，功能包含在 org.graalvm.nativeimage 模块下。

该功能主要能力在于不改变源码的基础上，对目标源码进行改造或增强，可以做到以下功能：

- 针对第三方依赖包进行改造。一般情况下我们是无法对第三方依赖组件进行修改的，但可以通过Substitution功能在一些基础类库上插入代码，实现特定功能的增强（例如对数据的收集或统计功能等等）。

- 解决二进制编译出现的问题。进行二进制编译时可能会遇到不兼容的项目和代码，需要对第三方类库进行小幅调整才能成功编译，可以通过Substitution功能解决此问题。

目前已经有一些开源项目使用了Substitution功能，例如 redisson 项目中为了适配二进制编译使用此功能对工程进行改造：

![](https://mmbiz.qpic.cn/mmbiz_png/dFRFrFfpIZlra7ZC7GRrCE4DsSbPeShVqicSGjz7lrLMh3uGBL1TDlrZk5upV2ibx3pcv6oTyGAfP0D6Nb9PQLRw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

项目代码地址：https://github.com/redisson/redisson/blob/569ed7711dfbe8e0368706befa9e0d51f91c1d55/redisson-quarkus/redisson-quarkus-30/cdi/runtime/src/main/java/io/quarkus/redisson/client/runtime/graal/CodecsSubstitutions.java#L16

# 使用Substitution功能动态改造工程代码

接下来我们用一个简单的例子来介绍Substitution。

假设有如下代码，功能是连接MySQL数据库并执行一个SQL。目标是如果执行了一个语法错误的SQL，在打印错误堆栈的同时把其他信息（例如拼接后的完整SQL、类信息等等）打印出来，方便排查问题。

```java
public static void main(String[] args) throws ClassNotFoundException, SQLException {
    Class.forName("com.mysql.cj.jdbc.Driver");
    String incorrectSQL = "DELETE FROM test.test_tx WHERE_WRONG id = ?"
    try (Connection connection = DriverManager.getConnection(url, user, pwd);
    PreparedStatement deleteStatement = connection.prepareStatement(incorrectSQL)) {
        deleteStatement.setString(1, "10086");
        deleteStatement.executeUpdate();
    }
}
```
通过debug代码，发现执行过程中会调用 ClientPreparedStatement 类中的 executeUpdate 方法，这个方法是一个很好的切入点。

```java

public class ClientPreparedStatement extends com.mysql.cj.jdbc.StatementImpl implements JdbcPreparedStatement {

@Override
public int executeUpdate() throws SQLException {
return Util.truncateAndConvertToInt(executeLargeUpdate());
   }
}
```
我们期望的是，修改executeUpdate方法，当发生异常时额外打印出一些信息，伪代码如下。

但是我们不能直接修改MySQL jdbc的相关代码。此时我们可以使用 Substitution功能完成目标。

```java

public class ClientPreparedStatement extends com.mysql.cj.jdbc.StatementImpl implements JdbcPreparedStatement {    
    
    @Override
    public int executeUpdate() throws SQLException {
       try {
           return Util.truncateAndConvertToInt(executeLargeUpdate());
       } catch (Exception ex) {
            //想要添加的部分, 但无法直接修改源码
            System.out.println("prepared sql: " + getPreparedSql());
            System.out.println("execute sql info: " + this.toString());
            throw ex;
       }
       
    }
}
```

## 工程代码
demo工程结构如下：

```bash
├── GraalVMSubstituteDemo
 │   ├── pom.xml
 │   └── src
 │       ├── main
 │       │   ├── java
 │       │   │   └── Main
 │       │   │   └── TargetClientPreparedStatement
 │       │   │   └── TargetUtil
 │       │   └── resources
```

首先在pom.xml中引入相关依赖:

```pom
<dependency>
    <groupId>org.graalvm.sdk</groupId>
    <artifactId>nativeimage</artifactId>
    <version>24.0.1</version>
</dependency>
```
创建一个TargetClientPreparedStatement类和TargetUtil类，完整代码如下:

TargetUtil.java（复制Util.truncateAndConvertToInt 方法）

```java
public class TargetUtil {

    public static int truncateAndConvertToInt(long longValue) {
        return longValue > Integer.MAX_VALUE
                ? Integer.MAX_VALUE : longValue < Integer.MIN_VALUE ? Integer.MIN_VALUE
                : (int) longValue;

   }
}
```
TargetClientPreparedStatement.java

```java
import com.oracle.svm.core.annotate.Alias;
import com.oracle.svm.core.annotate.Substitute;
import com.oracle.svm.core.annotate.TargetClass;

import java.sql.SQLException;

@TargetClass(className = "com.mysql.cj.jdbc.ClientPreparedStatement")
public final class TargetClientPreparedStatement {

    @Substitute
    public int executeUpdate() throws SQLException {
        try {
            return TargetUtil.truncateAndConvertToInt(executeLargeUpdate());
        } catch (Exception ex) {
            System.out.println("TargetClientPreparedStatement.executeUpdate method occurs exception");
            System.out.println("get prepared sql: " + getPreparedSql());
            System.out.println("execute sql info: " + this.toString());
            throw ex;
    }

    @Alias
    public long executeLargeUpdate() throws SQLException {
        return 0L;
    }

    @Alias
    public String getPreparedSql() {
        return null;
    }
}
```

在TargetClientPreparedStatement类中使用到了三个注解 :

@TargetClass：指示需要替换的java类

@Substitute：指示要替换的目标方法

@Alias：引用目标类的原始方法
通过以上三个注解的配合，就可以实现对原始代码方法的替换。但是有一些额外的点需要注意：

- 创建的 TargetClientPreparedStatement class 必须为 final 类型

- 使用 @Substitute、@Alias 注解的方法，方法签名必须与原始方法保持一致

- 使用 @Alias 注解的方法，只需保持签名与原方法一致即可，方法体留空或直接返回任意结果，此时GraalVM会引用目标类中的原始方法，AOT编译将进行正确的替换

main方法执行SQL逻辑

```java
public class Main {
    private static final String cname = "com.mysql.cj.jdbc.Driver";
    private static String user = "<USER>";
    private static String pwd = "<PASSWORD>";
    private static String url = "jdbc:mysql://127.0.0.1:3306/?useSSL=false&serverTimezone=UTC&useUnicode=true";

    public static void main(String[] args) throws ClassNotFoundException, SQLException {
        Class.forName(cname);
        String sql = args.length == 0 ? "DELETE FROM test.test_tx WHERE_WRONG id = ?" : args[0];
        try (Connection connection = DriverManager.getConnection(url, user, pwd);
             PreparedStatement deleteStatement = connection.prepareStatement(sql)) {
            deleteStatement.setString(1, "10086");
            deleteStatement.executeUpdate();
        }
    }
}
```
## 执行编译

执行编译命令 mvn -U clean package native:compile 进行二进制编译

```bash
========================================================================================================================
GraalVM Native Image: Generating 'GraalVMSubstituteDemo' (executable)...
========================================================================================================================
Recommendations:
 G1GC: Use the G1 GC ('--gc=G1') for improved latency and throughput.
 PGO:  Use Profile-Guided Optimizations ('--pgo') for improved throughput.
 HEAP: Set max heap for improved and more predictable memory usage.
 CPU:  Enable more CPU features with '-march=native' for improved performance.
 BRPT: Try out the new build reports with '-H:+BuildReport'.
------------------------------------------------------------------------------------------------------------------------
                        1.8s (3.6% of total time) in 31 GCs | Peak RSS: 3.54GB | CPU load: 11.30
------------------------------------------------------------------------------------------------------------------------
Produced artifacts:
 /usr/local/GraalVMSubstituteDemo/target/GraalVMSubstituteDemo (executable)
========================================================================================================================
Finished generating 'GraalVMSubstituteDemo' in 49.7s.
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time:  57.786 s
[INFO] Finished at: 2024-08-21T07:14:31Z
[INFO] ------------------------------------------------------------------------
```

## 编译结果

编译成功之后cd到target目录执行 ./GraalVMSubstituteDemo 可看到如下输出，可以看到打印了额外的信息（第1-3行）

```bash

TargetClientPreparedStatement.executeUpdate method occurs exception
get prepared sql: DELETE FROM test.test_tx WHERE_WRONG id = ?
execute sql info: com.mysql.cj.jdbc.ClientPreparedStatement: DELETE FROM test.test_tx WHERE_WRONG id = '10086'
Exception in thread "main" java.sql.SQLSyntaxErrorException: You have an error in your SQL syntax; check the manual that corresponds to your MySQL server version for the                  right syntax to use near 'id = '10086'' at line 1
        at com.mysql.cj.jdbc.exceptions.SQLError.createSQLException(SQLError.java:121)
        at com.mysql.cj.jdbc.exceptions.SQLExceptionsMapping.translateException(SQLExceptionsMapping.java:122)
        at com.mysql.cj.jdbc.ClientPreparedStatement.executeInternal(ClientPreparedStatement.java:916)
        at com.mysql.cj.jdbc.ClientPreparedStatement.executeUpdateInternal(ClientPreparedStatement.java:1061)
        at com.mysql.cj.jdbc.ClientPreparedStatement.executeUpdateInternal(ClientPreparedStatement.java:1009)
        at com.mysql.cj.jdbc.ClientPreparedStatement.executeLargeUpdate(ClientPreparedStatement.java:1320)
        at com.mysql.cj.jdbc.ClientPreparedStatement.executeUpdate(ClientPreparedStatement.java:13)
        at Main.main(Main.java:20)
```

# 结语
通过上述的demo演示了Substitution功能，可以看到通过此功能可以替换正常情况下无法修改的第三方库源码，实现我们自定义功能。

**但是Substitution功能也有其局限性：**

1. 可移植性：Substitution功能是GraalVM特有的，如果你的代码需要在其他JVM上运行，可能会遇到问题。

2. 维护难度：使用Substitution功能可能会增加代码的复杂性和维护难度。开发者需要确保替换的方法在所有情况下都能正确工作。

3. 开发难度：相对于其他工具（例如ebpf），Substitution功能相对局限，同时官方文档也较少提及此类功能。

尽管GraalVM当前版本的Substitution已经能够实现一些特殊的功能，但仍有其局限性。因此，我们期待在未来的GraalVM版本中，能够引入或增强这类特性，为Java开发带来新的技术和视角。



参考文档：

1.GraalVM的官方文档https://www.graalvm.org/latest/docs/