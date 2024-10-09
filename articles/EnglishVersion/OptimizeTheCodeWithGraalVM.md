# Background

In the previous article "Are you still using the JVM to run your Java code?", we briefly introduced GraalVM and its core function nativeimage, which can compile Java code into a native binary executable file. Compared with running Java services based on the JVM, the native executable file obtained by compiling has great advantages in startup speed and memory usage.

Although GraalVM's binary compile function has many advantages, there are still some problems encountered when actually performing binary compile. Simply put, due to the working principle of compiling binary, the engineering code needs to be modified to a certain extent. Therefore, the code that could have been run directly before may fail to execute after compiling the binary. At this time, we need to solve it through other means.

# How to transform your project code when there is a problem with binary compile?

To solve the binary compile problem, a commonly used and effective method is to automatically collect and configure reachable metadata. This can be achieved by configuring the collected metadata in the META-INF.native-image folder under the project's resource directory, thus effectively solving some of the compile problems.

However, sometimes problems cannot be solved through metadata, especially when third-party dependencies introduced in the project have binary compile compatibility issues. In this case, we can consider using the Graal VM Substitution function to modify the project code and complete the adaptation of the Graal VM binary compile.

## Graal VM Substitution Function Introduction

As the second article in the Graal VM series, this article mainly introduces the Substitution function in the nativeimage public API (native Image public API for advanced use cases), which is included in the org.graalvm.nativeimage module.

The main ability of this function is to modify or enhance the target source code without changing the source code, which can achieve the following functions:

- Modify third-party dependency packages. Generally, we cannot modify third-party dependency components, but we can insert code into some basic libraries through the Substitution function to enhance specific functions (such as data collection or statistical functions, etc.).

- Solve the problem of binary compile. When performing binary compile, you may encounter incompatible projects and code, and you need to make minor adjustments to third-party libraries to successfully compile. This problem can be solved by using the Substitution function.

Currently, some open source projects have used the Substitution function. For example, in the Redisson project, this function is used to modify the project to adapt to binary compile.

![](https://mmbiz.qpic.cn/mmbiz_png/dFRFrFfpIZlra7ZC7GRrCE4DsSbPeShVqicSGjz7lrLMh3uGBL1TDlrZk5upV2ibx3pcv6oTyGAfP0D6Nb9PQLRw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

Project code address: https://github.com/redisson/redisson/blob/569ed7711dfbe8e0368706befa9e0d51f91c1d55/redisson-quarkus/redisson-quarkus-30/cdi/runtime/src/main/java/io/quarkus/redisson/client/runtime/graal/CodecsSubstitutions.java#L16

# Use the Substitution function to dynamically transform project code

Next, we will use a simple example to introduce Substitution.

Assuming there is the following code, the function is to connect to the MySQL database and execute a SQL. The goal is to print out other information (such as the complete SQL after concatenation, class information, etc.) while printing the error stack if a syntax error SQL is executed, which is convenient for troubleshooting.
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
Through the debug code, it is found that the executeUpdate method in the ClientPreparedStatement class will be called during the execution process. This method is a good starting point.


```java

public class ClientPreparedStatement extends com.mysql.cj.jdbc.StatementImpl implements JdbcPreparedStatement {

@Override
public int executeUpdate() throws SQLException {
return Util.truncateAndConvertToInt(executeLargeUpdate());
   }
}
```
We expect to modify the executeUpdate method to print some additional information when an exception occurs. The pseudocode is as follows.

However, we cannot directly modify the relevant code of MySQL JDBC. At this time, we can use the Substitution function to achieve the goal.

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

## Engineering code
The demo project structure is as follows:

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

First, introduce relevant dependencies in pom.xml.

```pom
<dependency>
    <groupId>org.graalvm.sdk</groupId>
    <artifactId>nativeimage</artifactId>
    <version>24.0.1</version>
</dependency>
```
Create a TargetClientPreparedStatement class and TargetUtil class, the complete code is as follows:

TargetUtil.java (copy the Util.truncateAndConvertToInt method)

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

Three annotations are used in the TargetClientPreparedStatement class:

@TargetClass: Indicates the Java class to be replaced

@Substitute: Indicates the target method to be replaced

@Alias: Reference the original method of the target class

By combining the above three annotations, the replacement of the original code method can be achieved. However, there are some additional points to note.

- The created TargetClientPreparedStatement class must be of final type

- Methods annotated with @Substitute and @Alias must have the same method signature as the original method

- For methods annotated with @Alias, simply keep the signature consistent with the original method, leave the method body blank or return any result directly. At this time, GraalVM will reference the original method in the target class, and AOT compile will perform the correct replacement

The main method executes SQL logic

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
## Execute compile

Execute the compile command mvn -U clean package native: compile for binary compile

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

## Compile the results

After compile is successful, cd to the target directory to execute./GraalVMSubstituteDemo you can see the following output, you can see that additional information is printed (lines 1-3)

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

# Conclusion
The above demo demonstrates the Substitution function, which can replace the source code of third-party libraries that cannot be modified under normal circumstances and achieve our custom functions.

**However, the Substitution function also has its limitations.**

1. Portability: The Substitution feature is unique to GraalVM, and if your code needs to run on other JVMs, you may encounter problems.

2. Maintenance Difficulty: Using the Substitution function may increase the complexity and maintenance difficulty of the code. Developers need to ensure that the replacement method works correctly in all cases.

3. Development difficulty: Compared to other tools (such as ebpf), Substitution's functionality is relatively limited, and official documentation also mentions such functionality less.

Although the current version of GraalVM's Substitution has been able to implement some special features, it still has its limitations. Therefore, we look forward to introducing or enhancing such features in future versions of GraalVM, bringing new technologies and perspectives to Java development.


Reference Documentation:

1. GraalVM official documentation https://www.graalvm.org/latest/docs/