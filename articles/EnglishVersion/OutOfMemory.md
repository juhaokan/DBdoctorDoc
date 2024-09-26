## foreword
OOM is well-known in the industry as one of the nightmares of Java programmers. Whether it is a novice or an experienced expert, it is difficult to avoid encountering this problem during the development process. Especially during peak business hours, the application is interrupted and restarted due to frequent OOM errors, which is really a headache. How to quickly locate and solve this problem, we will discuss with you in detail in this article.
## What is OOM?
OOM , the full name OutOfMemoryError, as the name suggests, is not enough memory. Java during the running of the program, the JVM (Java virtual machine) allocates memory for the program, but sometimes the program occupies more memory than the maximum memory allocated by the JVM, OOM error will occur.

### Common scenarios for OOM include:
1. Heap memory overflow : Usually occurs when the application creates a large number of objects, and these objects survive for a long time.

2. Method area memory overflow : related to the loading of classes and the storage of metadata, it may be caused by too much class loading or leakage of class loader.

3. Direct memory overflow : Usually related to thread execution and recursion calls, such as too deep recursion calls or too many thread creations.

Heap memory overflow is the most common For example, if you write a loop and accidentally generate countless objects, the memory bursts, and the JVM "strikes". Some colleagues here may ask, isn't there full GC, why does OOM happen? Here is a simple explanation. Full gc collects "garbage", that is, unreachable objects. If most of the objects in memory are reachable, and there are new objects that need to allocate Memory Space, OOM will appear when the available space is not enough.

## How to identify the root cause of OOM problems?

Locating the root cause of the OOM problem is nothing more than finding the code that generates a large number of objects. How to find it? At present, the most mainstream way is to analyze the JVM heap dump file through MAT (Memory Analyzer Tool) to obtain large object and thread stack information. The specific process is as follows:
#### 1. Get the heap dump file
To analyze an OOM problem using MAT, you first need to obtain the heap dump file generated by the JVM during OOM. It can be generated in the following ways:

- JVM startup parameters: Ensure that heap dump files are automatically generated when OOM occurs by adding the -XX: +HeapDumpOnOutOfMemoryError parameter when starting the JVM.

- Manual trigger: On systems with OOM problems, manually generate a heap dump file via the jmap command. For example: jmap -dump: live, format = b, file = heapdump.hprof.
#### 2. Use MAT to load and analyze heap dump files

After downloading and installing the MAT tool, start MAT and open the heap dump file (usually in .hprof format). MAT automatically parses the file and generates a report on memory usage.
![](https://mmbiz.qpic.cn/mmbiz_png/dFRFrFfpIZm2sicsmumryVDksSdCEKuxicLiaZ5HKXI2YLBGGDqufXfnjibCG9n196FY9ftWwuzzyIUGYtZN7S2Fiag/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

#### 3. Analyze large objects

Click to view dominator_tree view. Through this view, we can see that the ArrayList of http-nio-8080-exec-6 threads takes up almost 96.32% of the heap memory, and the ArrayList contains a large number of mysql query result set objects. Here we can basically confirm that the OOM is caused by the large result set returned when querying the database. But which SQL is the specific query logic?

![](https://mmbiz.qpic.cn/mmbiz_png/dFRFrFfpIZm2sicsmumryVDksSdCEKuxicSmXELYRq0iaGMicsOibFUO1lAjx5QiaMasTsGgUntkiaKhEqI5viaUx1nH8g/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

You can first take a look at the specific content of the MySQL query result set object as shown in the following figure:

![](https://mmbiz.qpic.cn/mmbiz_png/dFRFrFfpIZm2sicsmumryVDksSdCEKuxicZ4f4y1K8ESdiakQfolibWFh0kNbGzUsPdPDOccQ3bmtht4Cia476VtFcQ/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

If the business logic can be directly located through the result set content, it is over here. If not, you can continue to read.

#### 4. Analyze the thread stack

Click to view thread_overview, through this view we can see that the http-nio-8080-exec-6 thread does call the query logic in the business code when OOM, but this can only be said to be the last straw that crushed the camel, and cannot directly explain that it is the root cause.

![](https://mmbiz.qpic.cn/mmbiz_png/dFRFrFfpIZm2sicsmumryVDksSdCEKuxiczfEwrEDsyJldaRyMbofZcDpZ45Az6eic5AaBLfclqQRMiaSuiaq537sIg/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

So, what is its root cause? We need to analyze it in combination with DBdoctor.

### Combined with DBdoctor for quick analysis and repair
With DBdoctor performance insights, audit logs, and SQL auditing capabilities, we can more efficiently target root cause SQL and resolve OOM issues.

In the DBdoctor performance insight page, we can see that there is such a SQL in the OOM time period. The execution time of this SQL exceeds 10s and the network backpack is particularly large. Through the audit log, we can also see that the number of rows returned by this SQL is as high as 32818160 rows, which is consistent with the size of the Arraylist. At the same time, this SQL will only be called in the business logic of the previous step. Then we can confirm that the following SQL in the business logic caused the OOM problem.

![](https://mmbiz.qpic.cn/mmbiz_jpg/dFRFrFfpIZm2sicsmumryVDksSdCEKuxicOickxR2mnlzAj3PkLsYthibmhktKgqE89SCwoMu9HaG0VrjnEP76HjkQ/640?wx_fmt=jpeg&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

![](https://mmbiz.qpic.cn/mmbiz_jpg/dFRFrFfpIZm2sicsmumryVDksSdCEKuxic9hC99jyDIiakkdwvbDdEIoaPCicSIqfcxYPoxJ6D6ZlzPfmlY9klf9Cg/640?wx_fmt=jpeg&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

After locating the root cause of the problem, the next step is to fix it. Using DBdoctor's SQL audit function, you can quickly identify SQL problems and make repair suggestions with one click without complicated operations:

![](https://mmbiz.qpic.cn/mmbiz_jpg/dFRFrFfpIZm2sicsmumryVDksSdCEKuxicTgjicrkyws3NJgOFoiagfgibVzfLSx1tic6lUWbhsrf1cVeezxYzIOlehw/640?wx_fmt=jpeg&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

## summary
OOM is a common Java application problem, but through SQL audit and optimization in the development stage, we can effectively avoid and solve such problems. At the same time, for slow SQL or even full SQL, DBdoctor can automatically grab it and perform SQL audit, so that your stock SQL no longer has to worry about it! This feature will be available in the new version released next week, so stay tuned!