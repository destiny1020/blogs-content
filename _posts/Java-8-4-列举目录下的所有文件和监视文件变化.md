---
title: '[Java 8] (4) 列举目录下的所有文件和监视文件变化'
date: 2014-10-24 10:18:00
categories: [编程语言, Java]
tags: [Java, Java 8, 函数式编程, Lambda, 文件]
---

## 列举目录中的所有文件

首先给出代码：

```java
Files.list(Paths.get(".")).forEach(System.out::println);
```

Files.list方法得到的是一个Stream类型的对象，它代表了目标路径下所有的文件。如果只想获取目标路径下的所有目录文件：

```java
Files.list(Paths.get("."))
    .filter(Files::isDirectory)
    .forEach(System.out::println);
```

<!-- More -->

在以前的Java版本中，如果需要实现一个自定义的过滤器，那么通常会选择使用FilenameFilter结合匿名类的方式：

```java
final String[] files =
    new File("target_dir").list(new java.io.FilenameFilter() {
    public boolean accept(final File dir, final String name) {
        return name.endsWith(".java");
    }
});
System.out.println(files);
```

我们说过，当遇见了匿名内部类的时候，如果被实现的接口是一个函数式接口，那么可以考虑将该匿名内部类以Lambda表达式的形式重新实现，再结合Java 8中新添加的DirectoryStream，可以将上述代码重新实现为：

```java
Files.newDirectoryStream(
    Paths.get("target_dir"), 
    path -> path.toString().endsWith(".java"))
        .forEach(System.out::println);
```

当目标目录下含有大量的文件或者子目录时，使用DirectoryStream往往会具有更好的性能。因为它实际上是一个Iterator用来遍历目标目录，而直接使用listFiles方法时，得到的是一个代表了所有文件和目录的数组，意味着内存的开销会更大。

### 使用flatMap列举所有直接子目录

所谓的直接子目录(Immediate Subdirectory)，指的就是目标目录下一级的所有目录。对于这样一个任务，最直观的实现方式恐怕是这样的：

```java
public static void listTheHardWay() {
    List<File> files = new ArrayList<>();
    File[] filesInCurerentDir = new File(".").listFiles();
    for(File file : filesInCurerentDir) {
        File[] filesInSubDir = file.listFiles();
        if(filesInSubDir != null) {
            files.addAll(Arrays.asList(filesInSubDir));
        } else {
            files.add(file);
        }
    }
    System.out.println("Count: " + files.size());
}
```

很显然，此段代码噪声太多，没有清晰地反映出代码的整体目标。下面就用flatMap方法来简化它：

```java
public static void betterWay() {
    List<File> files = Stream.of(new File(".").listFiles())
        .flatMap(file -> file.listFiles() == null ?
            Stream.of(file) : Stream.of(file.listFiles()))
        .collect(toList());
    System.out.println("Count: " + files.size());
}

// flatMap
<R> Stream<R> flatMap(Function<? super T, ? extends Stream<? extends R>> mapper);
```

从flatMap方法的签名来看，它接受了一个Function接口作为参数，将一种类型转换为另一种类型的Stream类型。而从flatMap方法的命令来看，它的执行过程主要包含两个步骤：

1. 首先是会对当前Stream的每个元素执行一次map操作，根据传入的mapper对象将一个元素转换为对应的Stream对象
2. 将第一步中得到的若干个Stream对象汇集成一个Stream对象

从上面的代码来看，签名中的T类型就是File类型，而R类型同样也是File类型。当一个File对象不含有任何的子目录或者子文件时，那么通过Stream.of(file)来仅仅包含它自身，否则使用Stream.of(file.listFiles())来包含其下的所有子目录和子文件。

### 监视文件变化

WatchService是Java 7中新添加的一个特性，用来监视一某个路径下的文件或者目录是否发生了变化。

```java
final Path path = Paths.get(".");
final WatchService watchService = path.getFileSystem().newWatchService();

path.register(watchService, StandardWatchEventKinds.ENTRY_MODIFY);

System.out.println("Report any file changed within next 1 minutes...");
```

注册了需要监视的目录后，需要使用WatchKey来得到一段时间内的，该目录的变化情况：

```java
final WatchKey watchKey = watchService.poll(1, TimeUnit.MINUTES);
if(watchKey != null) {
    watchKey.pollEvents().stream().forEach(event ->
    System.out.println(event.context()));
}
```

这里使用了Java 8中的内部遍历器forEach来完成对于事件的遍历。这也算是一个Java 7和Java 8特性的联合使用吧。
