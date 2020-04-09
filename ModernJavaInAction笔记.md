# Modern Java in Action笔记


## 第8章：Collection API Enhancement

### 8.3 Working with Map

#### 8.3.3 getOrdefault

便面空指针异常

```java
Map<String, String> favouriteMovies = Map.ofEntries(entry("Raphael", "Star Wars"),
entry("Olivia", "James Bond"));

System.out.println(favouriteMovies.getOrDefault("Olivia", "Matrix"));
System.out.println(favouriteMovies.getOrDefault("Thibaut", "Matrix"));
```

#### 8.3.4 Compute Patterns

1. computeIfAbsent 若不存在则计算，并作为key存在map中
2. computeIfPresent 若存在则计算并存在mapzhong、
3. compute 计算并存在map中

```java
lines.forEach(line ->
dataToHash.computeIfAbsent(line,
this::calculateDigest));
private byte[] calculateDigest(String key) {
return messageDigest.digest(key.getBytes(StandardCharsets.UTF_8));
}
```

#### 8.3.5 Remove Patterns

favouriteMovies.remove(key, value);

#### 8.3.6 Replacement Patterns

1. replaceAll 替换所有
2. replace

#### 8.3.7

putAll方法可以将两个Map合并在一起，但若出现重复的则会出现问题。Merge可以通过BiFunction处理重复的键。

```java
Map<String, String> family = Map.ofEntries(
entry("Teo", "Star Wars"), entry("Cristina", "James Bond"));
Map<String, String> friends = Map.ofEntries(
entry("Raphael", "Star Wars"), entry("Cristina", "Matrix"));

Map<String, String> everyone = new HashMap<>(family);
friends.forEach((k, v) ->
everyone.merge(k, v, (movie1, movie2) -> movie1 + " & " + movie2));
System.out.println(everyone);
```

merge方法用一种非常复杂的方式处理空值
> If the specified key is not already associated with a value or is associated with null,
>[merge] associates it with the given non-null value. Otherwise, [merge] replaces the
>associated value with the [result] of the given remapping function, or removes [it] if the
>result is null.

可以使用初始化验证避免这种

```java
Map<String, Long> moviesToCount = new HashMap<>();
String movieName = "JamesBond";
long count = moviesToCount.get(movieName);
if(count == null) {
moviesToCount.put(movieName, 1);
}
else {
moviesToCount.put(moviename, count + 1);
}
```

可以重写成：

```java
moviesToCount.merge(movieName, 1L, (key, count) -> count + 1L);
```

### 8.4 Improved ConcurrentHashMap

#### 8.4.1 Reduce and search

stream操作

1. forEach
2. reduce
3. search

#### 8.4.2 counting

#### 8.4.3 set views

