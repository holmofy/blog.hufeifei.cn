# 8. [CountedCompleter](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CountedCompleter.html)



# 9. [ManagedBlocker](https://docs.oracle.com/javase/10/docs/api/java/util/concurrent/ForkJoinPool.ManagedBlocker.html)



> https://stackoverflow.com/questions/37512662/is-there-anything-wrong-with-using-i-o-managedblocker-in-java8-parallelstream

# 10. ThreadPoolExecutor与ForkJoinPool的适用场景

Java7发布后，由于ForkJoinPool号称能充分利用CPU资源，许多开发人员决定将原来的ThreadPoolExecutor替换成ForkJoinPool，但实际上它们适用场景是不一样的。

ThreadPoolExecutor能让开发者控制生成的线程数并**控制任务执行的粒度**。ThreadPoolExecutor最佳用例就是**一个线程处理一个任务**。

ForkJoinPool主要是为了加快大任务的处理，将大任务递归地拆分成更小的子任务。



# 11. Spliterator



# 10. 通过Java8的Stream API使用ForkJoinPool

```java
int[] sortedInts = new Random().ints(10000, 0, 10000).parallel().sorted().toArray();
System.out.println(Arrays.toString(sortedInts));
```





```java
int sum = IntStream.range(0, 10000).parallel().sum();
System.out.println(sum);
```





> 参考链接：
>
> Javadoc文档：https://docs.oracle.com/javase/9/docs/api/java/util/concurrent/ForkJoinPool.html
>
> Fork/Join框架：http://gee.cs.oswego.edu/dl/papers/fj.pdf
>
> 初学者的F/J框架导论：https://homes.cs.washington.edu/~djg/teachingMaterials/spac/grossmanSPAC_forkJoinFramework.html
>
> 《Java8 In Action》：https://www.manning.com/books/java-8-in-action
>
> OpenMP并行编程简介: http://www.bowdoin.edu/~ltoma/teaching/cs3225-GIS/fall16/Lectures/openmp.html
>
> Cilk Plus与其他并行框架的比较：https://www.cilkplus.org/faq/24
>
> TBB与其他并行框架的比较：https://www.threadingbuildingblocks.org/compare
>
> F/J框架与Parallel Stream vs. ExecutorService：https://blog.takipi.com/forkjoin-framework-vs-parallel-streams-vs-executorservice-the-ultimate-benchmark/
>
> Fork/Join灾难：http://www.coopsoft.com/ar/CalamityArticle.html
>
> 使用Fork/Join框架的范例与反例：https://rmod.inria.fr/archives/papers/DeWa14a-PPPJ14-ForkJoin.pdf
>
> Fork/Join与MapReduce：http://www.macs.hw.ac.uk/cs/techreps/docs/files/HW-MACS-TR-0096.pdf
>
> 任务并行Wikipedia：https://en.wikipedia.org/wiki/Task_parallelism
>
> 关于Java你不知道的10件事：https://www.sitepoint.com/10-things-you-didnt-know-about-java/
>
> http://www.oracle.com/technetwork/articles/java/fork-join-422606.html
>
> http://ifeve.com/java7-concurrency-cookbook-4/
>
> http://jsr166-concurrency.10961.n7.nabble.com/CountedCompleters-td5213.html