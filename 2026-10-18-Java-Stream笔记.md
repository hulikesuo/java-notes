# Java 学习笔记 · 2026-10-18

> 对应教程：《Java 后端求职实战教程（手把手版）》1.5 节「统计——先手写循环，再用 Stream 改写」
> 今日主线：把 Stream 的流水线、常用方法和 `Collectors` 弄明白，并与 `for` 循环写法对照。

---

## 一、今天要解决的代码

教程 1.5 节用 Stream 改写了分类统计：

```java
import java.util.Map;
import java.util.stream.Collectors;

private static void printStats() {
    double total = ledger.stream()
            .mapToDouble(Expense::getAmount)
            .sum();

    Map<String, Double> byCategory = ledger.stream().collect(
            Collectors.groupingBy(Expense::getCategory,
                                  Collectors.summingDouble(Expense::getAmount)));

    System.out.println("总支出：¥" + total);
    byCategory.forEach((cat, sum) -> System.out.println(cat + "：¥" + sum));
}
```

### 和循环写法对照

```java
double total = 0;
Map<String, Double> byCategory = new HashMap<>();
for (Expense e : ledger) {
    total += e.getAmount();
    byCategory.merge(e.getCategory(), e.getAmount(), Double::sum);
}
```

两段代码做的事完全一样，只是组织方式不同：

| 环节 | 循环写法 | Stream 写法 |
|---|---|---|
| 遍历 | `for (Expense e : ledger)` | `ledger.stream()` |
| 取金额 | `e.getAmount()` | `mapToDouble(Expense::getAmount)` |
| 累加 | `total += ...` | `sum()` |
| 按类别分组 | `merge(key, value, Double::sum)` | `groupingBy` + `summingDouble` |

实测两组结果一致：

```text
循环总支出 = 243.0
循环分类   = {交通=6.0, 购物=199.0, 餐饮=38.0}
Stream 总支出 = 243.0
Stream 分类   = {交通=6.0, 购物=199.0, 餐饮=38.0}
```

---

## 二、Stream 的基本结构

Stream 是一条**流水线**，分三段：

```java
ledger.stream()                          // ① 源头：List 转 Stream
      .mapToDouble(Expense::getAmount)   // ② 中间操作：元素逐个转换
      .sum();                            // ③ 终止操作：汇总出结果
```

### ① 源头

常见来源：

```java
ledger.stream()                       // 集合
Arrays.stream(array)                  // 数组
Stream.of("a", "b", "c")              // 直接给元素
IntStream.range(1, 10)                // 数字范围
```

### ② 中间操作

返回的仍然是 Stream，所以可以继续接下去。特点是**惰性**：登记要做什么，不立即执行。

### ③ 终止操作

返回具体结果（数字、List、Map、boolean、Optional 等）。**只有遇到终止操作，整条流水线才真正开始跑。**

### 判断方法

看返回值：返回 `Stream` 的是中间操作，返回其他类型的是终止操作。写完流水线必须接一个终止操作。

---

## 三、两个关键机制

### 1. 中间操作是惰性的

实测代码：

```java
var lazy = ledger.stream()
        .filter(e -> {
            System.out.println("执行 filter: " + e.category());
            return e.amount() >= 10;
        })
        .map(e -> {
            System.out.println("执行 map: " + e.category());
            return e.category();
        });

System.out.println("到这里为止，上面 filter/map 一次都没跑");
long n = lazy.count();   // 加了终止操作才开始执行
```

实际输出：

```text
到这里为止，上面 filter/map 一次都没跑
  执行 filter: 餐饮
  执行 map: 餐饮
  执行 filter: 交通
  执行 filter: 购物
  执行 map: 购物
  执行 filter: 餐饮
  执行 map: 餐饮
count = 3
```

注意 `交通` 只执行了 `filter` 没执行 `map`，因为它没通过筛选。这种「逐个元素走完整条链路」的执行方式，也是 Stream 能短路和优化遍历的原因。

### 2. 一个 Stream 只能消费一次

```java
var stream = ledger.stream();
stream.count();
stream.count();   // 报错
```

报错信息：

```text
stream has already been operated upon or closed
```

所以需要两次计算时，要写两次 `ledger.stream()`。教程里总支出和分类统计各开了一次流，不是重复啰嗦。

---

## 四、常用中间操作

| 方法 | 作用 | 示例 |
|---|---|---|
| `filter` | 筛选满足条件的元素 | `.filter(e -> e.getAmount() > 10)` |
| `map` | 转换成另一种类型 | `.map(Expense::getCategory)` |
| `mapToDouble` | 转成 `double` 流 | `.mapToDouble(Expense::getAmount)` |
| `mapToInt` | 转成 `int` 流 | `.mapToInt(e -> (int) e.getAmount())` |
| `distinct` | 去重 | `.map(Expense::getCategory).distinct()` |
| `sorted` | 排序 | `.sorted(Comparator.comparingDouble(Expense::getAmount))` |
| `limit` | 只取前 N 个 | `.limit(3)` |
| `skip` | 跳过前 N 个 | `.skip(2)` |
| `peek` | 调试时观察元素，不改变值 | `.peek(System.out::println)` |

### `map` 和 `mapToDouble` 的区别

`map` 可以转成任何类型，结果是 `Stream<R>`：

```java
Stream<String> categories = ledger.stream().map(Expense::getCategory);
```

`mapToDouble` 专用于转成 `double`，结果是 `DoubleStream`。数值流才直接提供 `sum()`、`average()`、`max()`、`min()`：

```java
double total = ledger.stream()
        .mapToDouble(Expense::getAmount)
        .sum();
```

普通 `Stream<Expense>` 没有 `sum()`，先转成数值流才能求和。

### `sorted` 的写法

对象默认没有大小关系，必须告诉 Stream 按什么排：

```java
// 金额升序
ledger.stream()
      .sorted(Comparator.comparingDouble(Expense::getAmount))
      .forEach(System.out::println);

// 金额降序
ledger.stream()
      .sorted(Comparator.comparingDouble(Expense::getAmount).reversed())
      .forEach(System.out::println);
```

---

## 五、常用终止操作

| 方法 | 作用 | 示例 |
|---|---|---|
| `sum` | 求和，只能用于数值流 | `.mapToDouble(Expense::getAmount).sum()` |
| `average` | 平均值，返回 `OptionalDouble` | `.mapToDouble(...).average()` |
| `max` / `min` | 最大 / 最小，返回 `Optional` | `.max(Comparator.comparingDouble(Expense::getAmount))` |
| `count` | 计数 | `.count()` |
| `forEach` | 逐个处理 | `.forEach(System.out::println)` |
| `anyMatch` | 是否至少有一个满足 | `.anyMatch(e -> e.getAmount() > 100)` |
| `allMatch` | 是否全部满足 | `.allMatch(e -> e.getAmount() > 0)` |
| `noneMatch` | 是否全都不满足 | `.noneMatch(e -> e.getAmount() > 500)` |
| `findFirst` | 取第一个 | `.findFirst()` |
| `collect` | 收集成 List、Map、字符串等 | `.collect(Collectors.toList())` |
| `toList` | 简化版收集成 List（JDK 16+） | `.toList()` |

### Optional 要取值

`findFirst()`、`max()`、`min()`、`average()` 都返回 Optional，因为结果可能不存在（比如列表为空）。取值用：

```java
double avg = ledger.stream()
        .mapToDouble(Expense::getAmount)
        .average()
        .orElse(0);

String first = ledger.stream()
        .findFirst()
        .map(Expense::getCategory)
        .orElse("没有记录");
```

---

## 六、`Collectors` 常用方法

`collect` 配合 `Collectors` 把流收集成各种结构。

| 方法 | 结果 | 示例 |
|---|---|---|
| `toList()` | List | `.collect(Collectors.toList())` |
| `toSet()` | Set | `.collect(Collectors.toSet())` |
| `joining()` | 拼接字符串 | `.collect(Collectors.joining(", "))` |
| `counting()` | 计数 | `.collect(Collectors.counting())` |
| `summingDouble()` | 求和 | `.collect(Collectors.summingDouble(Expense::getAmount))` |
| `averagingDouble()` | 平均值 | `.collect(Collectors.averagingDouble(Expense::getAmount))` |
| `maxBy()` / `minBy()` | 最大 / 最小 | `.collect(Collectors.maxBy(comparator))` |
| `groupingBy()` | 分组 | `.collect(Collectors.groupingBy(Expense::getCategory))` |
| `partitioningBy()` | 按 true/false 分成两组 | `.collect(Collectors.partitioningBy(e -> e.getAmount() >= 100))` |
| `toMap()` | 收集成 Map | `.collect(Collectors.toMap(Expense::getNote, Expense::getAmount))` |

### 最常用的四种

```java
// 1. 收集成 List
List<String> categories = ledger.stream()
        .map(Expense::getCategory)
        .collect(Collectors.toList());

// 2. 拼接字符串
String joined = ledger.stream()
        .map(Expense::getCategory)
        .collect(Collectors.joining(", "));

// 3. 按类别分组求和
Map<String, Double> byCategory = ledger.stream().collect(
        Collectors.groupingBy(Expense::getCategory,
                              Collectors.summingDouble(Expense::getAmount)));

// 4. 收集成 Map
Map<String, Double> byNote = ledger.stream()
        .collect(Collectors.toMap(Expense::getNote, Expense::getAmount));
```

---

## 七、重点拆解：`groupingBy` + `summingDouble`

```java
Map<String, Double> byCategory = ledger.stream().collect(
        Collectors.groupingBy(Expense::getCategory,
                              Collectors.summingDouble(Expense::getAmount)));
```

这段只做一件事：**按类别分组，再把每组的金额加起来**。

| 部分 | 作用 |
|---|---|
| `collect(...)` | 把流收集成某个数据结构，这里是 Map |
| `groupingBy(Expense::getCategory, ...)` | 第一个参数决定 Map 的键：类别名 |
| `summingDouble(Expense::getAmount)` | 第二个参数决定每组的值怎么算：金额求和 |

两笔餐饮账 25.5 和 12.5 会被分到同一组，`summingDouble` 相加得到 38.0，Map 里就是 `"餐饮" → 38.0`。

它等价于循环版的：

```java
byCategory.merge(e.getCategory(), e.getAmount(), Double::sum);
```

`groupingBy` 负责「查表找组」，`summingDouble` 负责「组内累加」，`merge` 在循环里同时做了这两件事。

---

## 八、方法引用在 Stream 里的用法

Stream 里大量使用 `类名::方法` 这种写法：

```java
Expense::getAmount
```

等价于 Lambda：

```java
e -> e.getAmount()
```

`Expense::getCategory` 等价于 `e -> e.getCategory()`，`System.out::println` 等价于 `x -> System.out.println(x)`。

方法引用只适合「参数原样传给方法」的场景。需要额外计算时，就写 Lambda：

```java
// 方法引用
.map(Expense::getCategory)

// Lambda：需要拼接或计算
.map(e -> e.getCategory() + "：" + e.getAmount())
```

---

## 九、实际练习过的结果

用四笔账做实验：

```text
餐饮 25.5
交通 6.0
购物 199.0
餐饮 12.5
```

| 操作 | 结果 |
|---|---|
| `filter(amount > 20).map(category).toList()` | `[餐饮, 购物, 餐饮]` |
| `map(category).distinct().toList()` | `[餐饮, 交通, 购物]` |
| `sorted(comparingDouble(...).reversed()).map(category)` | `[购物, 餐饮, 餐饮, 交通]` |
| `limit(2).map(category)` | `[餐饮, 交通]` |
| `skip(2).map(category)` | `[购物, 餐饮]` |
| `mapToDouble(amount).sum()` | `243.0` |
| `mapToDouble(amount).average()` | `60.75` |
| `count()` | `4` |
| `anyMatch(amount > 100)` | `true` |
| `allMatch(amount > 0)` | `true` |
| `noneMatch(amount > 500)` | `true` |
| `groupingBy(category, summingDouble(amount))` | `{餐饮=38.0, 交通=6.0, 购物=199.0}` |
| `partitioningBy(amount >= 100)` | `{false=[餐饮 25.5, 交通 6.0, 餐饮 12.5], true=[购物 199.0]}` |

---

## 十、常见坑

| 坑 | 说明 | 解决 |
|---|---|---|
| 普通 Stream 调用 `sum()` | `Stream<Expense>` 没有 `sum()` | 先用 `mapToDouble` 转成数值流 |
| 忘记写终止操作 | 中间操作是惰性的，不执行 | 末尾接 `sum`、`collect`、`forEach` 等 |
| 同一个 Stream 用两次 | 第二次报 `stream has already been operated upon or closed` | 重新调用 `ledger.stream()` |
| `map` 和 `mapToDouble` 混用 | `map` 返回 `Stream<Double>`，没有直接 `sum()` | 需要求和时直接用 `mapToDouble` |
| `sort` 对象不传比较器 | 对象没有默认大小关系，编译失败 | 用 `Comparator.comparingDouble(...)` |
| 直接对 Optional 做运算 | `average()` 等返回 Optional | 用 `orElse(...)` 取值 |
| 用 `int` 累加 `double` | 小数被截断，结果偏小 | 声明成 `double`，或用 `mapToDouble(...).sum()` |

### 当前项目里的真实 bug

`test2/src/Main.java` 的统计方法是：

```java
private static void printStats() {
    int sum = 0;
    for (Expense expense : ledger) {
        sum += expense.getAmount();
    }
    System.out.println("总计消耗：" + sum);
}
```

`sum` 是 `int`，`getAmount()` 是 `double`。编译能过，但小数会被直接截掉。

实测三笔账 25.5、6.9、199.7：

```text
int sum      = 230
double total = 232.1
```

修正方式：

```java
private static void printStats() {
    double total = ledger.stream()
            .mapToDouble(Expense::getAmount)
            .sum();
    System.out.println("总计消耗：¥" + total);
}
```

---

## 十一、今天真正掌握的内容

- 能说出 Stream 的三段结构：源头、中间操作、终止操作
- 知道中间操作是惰性的，必须靠终止操作触发
- 知道一个 Stream 只能消费一次，用完要重新开
- 理解 `mapToDouble` 的作用是转成数值流，转完才能 `sum()` / `average()`
- 会用 `filter`、`map`、`sorted`、`distinct`、`limit`、`skip`
- 会用 `sum`、`average`、`count`、`forEach`、`anyMatch`、`findFirst`、`collect`
- 理解 `groupingBy` + `summingDouble` 就是「按类别分组再求和」
- 能说出 Stream 版和循环版是同一种逻辑的两种写法

---

## 十二、待办

- [ ] 把 `test2/src/Main.java` 的 `printStats()` 改成 `double` 或 Stream 版，修掉小数截断
- [ ] 用 `filter` 做一个「只看金额大于 50 的账目」功能
- [ ] 用 `sorted` 做一个「按金额从高到低显示」功能
- [ ] 用 `groupingBy` 的 `counting()` 统计每个类别各记了几笔
- [ ] 复习教程 1.5 节检查点：两种统计写法输出必须完全一致

---

## 十三、追问补充：`groupingBy` 的两个参数怎么区分

```java
Map<String, Double> result = employees.stream()
        .collect(Collectors.groupingBy(
                Employee::getDept,
                Collectors.averagingDouble(Employee::getSalary)
        ));
```

这里的两个参数有固定位置，不是从两个元素里挑一个当 key：

| 参数位置 | 代码 | 作用 | 对应结果 |
|---|---|---|---|
| 第 1 个参数 | `Employee::getDept` | 分类函数（classifier），生成 Map 的 key | `String` 类型的部门名 |
| 第 2 个参数 | `Collectors.averagingDouble(...)` | 下游收集器（downstream），计算每组的值 | `Double` 类型的平均工资 |

对应的方法签名可以简化为：

```java
groupingBy(
    Function classifier,   // 生成 key
    Collector downstream   // 生成 value
)
```

因此：

```text
"研发部" -> 15000.0
```

其中 `"研发部"` 是 key，`15000.0` 是该组所有员工工资的平均值。

组里的员工不是 key，也不会有一个员工被选出来当 key；他们只是参与 `averagingDouble` 计算的数据。

### 只有一个参数时

```java
Map<String, List<Employee>> byDept = employees.stream()
        .collect(Collectors.groupingBy(Employee::getDept));
```

这里只有分类函数，所以 value 使用默认收集器，每组收集成 `List`：

```text
"研发部" -> [张三, 李四]
```

### 三个参数时

```java
Map<String, Set<Employee>> result = employees.stream()
        .collect(Collectors.groupingBy(
                Employee::getDept,       // key
                TreeMap::new,             // Map 工厂
                Collectors.toSet()        // 每组如何收集
        ));
```

第二个参数是 Map 工厂，第三个参数才是下游收集器。

### 注意

- 默认返回 `HashMap`，不保证分组顺序；需要插入顺序时可用 `LinkedHashMap::new`
- 分类函数返回 `null` 会抛出 `NullPointerException`
- 常用下游收集器：`counting()`、`mapping()`、`summingDouble()`、`averagingDouble()`、`toSet()`
- 嵌套分组可以在下游再放一个 `groupingBy`

```java
Map<String, Map<String, List<Employee>>> result =
        employees.stream()
                .collect(Collectors.groupingBy(
                        Employee::getDept,
                        Collectors.groupingBy(Employee::getLevel)
                ));
```

---

## 十四、`BufferedWriter` 是什么

```java
try (BufferedWriter w = Files.newBufferedWriter(FILE)) {
    w.write("hello");
    w.newLine();
    w.write("world");
}
```

`BufferedWriter` 是 Java 的**带缓冲区的字符输出流**，用来高效地写文本文件。

可以把它理解成：

```text
程序写入字符
    ↓
BufferedWriter 的内存缓冲区
    ↓ 缓冲区满 / flush() / close()
文件
```

### 为什么需要缓冲区

如果每次 `write()` 都直接访问磁盘，系统调用会很多，速度较慢。`BufferedWriter` 会先把字符攒在内存中，攒到一定量后再批量写入文件，从而减少 I/O 次数。

注意：它不是写到一行就立刻落盘。数据在以下时机才会真正写到文件：

- 缓冲区满了
- 调用 `flush()`
- 调用 `close()`

### `flush()` 和 `close()`

```java
w.write("hello");
w.flush();   // 立即把缓冲区内容写出去，但不关闭流
```

```java
w.close();   // 先 flush，再关闭流
```

使用 `try-with-resources` 时，代码块结束会自动调用 `close()`：

```java
try (BufferedWriter w = Files.newBufferedWriter(FILE)) {
    w.write("hello");
}
```

即使代码块中抛出异常，也会自动关闭流，所以通常不需要手动调用 `flush()`。

### `Files.newBufferedWriter(FILE)` 做了什么

它直接创建一个 `BufferedWriter`，默认使用 UTF-8，并且默认会：

- 文件不存在时创建文件
- 文件已存在时覆盖原内容

大致等价于：

```java
BufferedWriter w = new BufferedWriter(
        new OutputStreamWriter(
                Files.newOutputStream(FILE),
                StandardCharsets.UTF_8
        )
);
```

### 追加写入

默认写法会覆盖文件。如果要保留原内容并在末尾追加：

```java
try (BufferedWriter w = Files.newBufferedWriter(
        FILE,
        StandardCharsets.UTF_8,
        StandardOpenOption.CREATE,
        StandardOpenOption.APPEND)) {
    w.write("新增一行");
    w.newLine();
}
```

### 注意

- `BufferedWriter` 处理的是字符，不是字节
- `newLine()` 会写入当前系统的换行符，比手写 `"\n"` 更通用
- 不要 `close()` 之后继续调用 `write()`，否则会抛出 `IOException: Stream closed`
- 如果只是一次性写入整个字符串，也可以直接用 `Files.writeString(...)`

---

## 十五、`Path` 和 `FILE` 常量是什么

```java
private static final Path FILE = Path.of("ledger.csv");
```

`FILE` 是一个 **`java.nio.file.Path` 对象**，表示一个路径，不是文件内容，也不是已经打开的文件。

逐部分理解：

| 部分 | 含义 |
|---|---|
| `private` | 只在当前类中可见 |
| `static` | 属于类，不属于某个实例 |
| `final` | 引用不能再指向其他 `Path` 对象 |
| `Path.of("ledger.csv")` | 创建一个表示 `ledger.csv` 路径的对象 |

`Path` 只保存路径信息，不会自动创建文件。真正操作文件要配合 `Files`：

```java
Files.exists(FILE);                  // 判断文件是否存在
Files.createFile(FILE);              // 创建空文件
Files.writeString(FILE, "hello");    // 写入文本
Files.readString(FILE);              // 读取文本
Files.newBufferedWriter(FILE);       // 打开字符输出流
```

### 相对路径

```java
Path.of("ledger.csv")
```

是相对路径，相对于**程序启动时的工作目录**，不是相对于 `.java` 源文件所在目录。

如果程序从 `C:\Users\GF76\Desktop\后端` 启动，实际路径通常是：

```text
C:\Users\GF76\Desktop\后端\ledger.csv
```

可以打印绝对路径确认：

```java
System.out.println(FILE.toAbsolutePath());
```

### `final` 表示什么

```java
private static final Path FILE = Path.of("ledger.csv");
```

`final` 只表示 `FILE` 这个引用不能再指向别的 `Path`，不表示磁盘上的文件不能修改。

下面这种重新赋值会编译失败：

```java
FILE = Path.of("other.csv");   // 编译错误
```

但文件内容仍然可以通过 `Files` 或 `BufferedWriter` 修改。

### 和 `java.io.File` 的区别

旧写法：

```java
File file = new File("ledger.csv");
```

新写法更推荐：

```java
Path path = Path.of("ledger.csv");
```

两者都能表示路径，但 `Path` 配合 `Files` 使用更现代，API 也更清晰。

另外，`.csv` 只是文件名后缀。`Path` 和 `BufferedWriter` 都不会自动识别 CSV 格式，内容仍需要自己按逗号分隔的规则读写。

---

## 十六、按月筛选流水的 Stream 写法

```java
List<Expense> inMonth = ledger.stream()
        .filter(e -> e.getDate().toString().startsWith(month))
        .sorted(Comparator.comparing(Expense::getDate).reversed())
        .toList();
```

功能是：

1. 筛选日期字符串以 `month` 开头的记录
2. 按日期从新到旧排序
3. 收集成一个新的 `List`

### 为什么 `startsWith` 有风险

如果 `month` 是 `"2026-1"`：

```java
"2026-10-18".startsWith("2026-1") // true
```

它会把 `2026-10`、`2026-11`、`2026-12` 也误认为是一月的记录。

更稳妥的方式是解析成年月再比较：

```java
YearMonth target = YearMonth.parse(month);   // 例如 "2026-08"

List<Expense> inMonth = ledger.stream()
        .filter(e -> YearMonth.from(e.getDate()).equals(target))
        .sorted(Comparator.comparing(Expense::getDate).reversed())
        .toList();
```

需要导入：

```java
import java.time.YearMonth;
```

如果 `getDate()` 返回 `LocalDate`，`YearMonth.from(...)` 可以直接使用；如果返回 `LocalDateTime`，也可以正常提取年月。

### 关于 `toList()`

Java 16+ 的 `Stream.toList()` 返回**不可修改列表**。下面操作会抛出 `UnsupportedOperationException`：

```java
inMonth.add(new Expense(...));   // 不允许
```

如果后面还要增删元素，改用：

```java
.collect(Collectors.toCollection(ArrayList::new))
```

---

## 十七、月份输入应该捕获什么异常

原代码：

```java
try {
    List<Expense> inMonth = ledger.stream()
            .filter(e -> e.getDate().toString().startsWith(sc))
            .sorted(Comparator.comparing(Expense::getDate).reversed())
            .toList();

    System.out.println(sc + " 月，共 " + inMonth.size() + " 笔：");
    inMonth.forEach(System.out::println);
} catch (NumberFormatException e) {
    System.out.println("请输入正确格式的日期" + e.getMessage());
}
```

问题在于：这段 `try` 中没有任何代码会把字符串解析成数字。

- `scanner.nextLine()` 不解析数字
- `trim()` 不解析数字
- `filter()`、`sorted()`、`toList()` 也不会抛出 `NumberFormatException`

所以这个 `catch` 基本不会执行。

正确做法是先把月份解析成 `YearMonth`，捕获 `DateTimeParseException`：

```java
System.out.println("您想查询几年几月的流水？如：2026-08");
String sc = scanner.nextLine().trim();

YearMonth targetMonth;
try {
    targetMonth = YearMonth.parse(sc);
} catch (DateTimeParseException e) {
    System.out.println("请输入正确格式的月份，例如：2026-08");
    return;
}

List<Expense> inMonth = ledger.stream()
        .filter(e -> YearMonth.from(e.getDate()).equals(targetMonth))
        .sorted(Comparator.comparing(Expense::getDate).reversed())
        .toList();

System.out.println(sc + " 月，共 " + inMonth.size() + " 笔：");
inMonth.forEach(System.out::println);
```

需要导入：

```java
import java.time.YearMonth;
import java.time.format.DateTimeParseException;
```

### 为什么只捕获这个异常

`DateTimeParseException` 专门表示日期或年月字符串格式不合法，例如：

```text
abc
2026-13
2026/08
```

不建议直接写：

```java
catch (Exception e) {
    // ...
}
```

因为它会把空指针、类型错误、集合异常等所有问题一起吞掉，反而不容易发现真正的 bug。

### 在菜单循环中的处理

如果这段代码位于菜单的 `while` 循环中，不希望输入错误后直接退出程序，可以用 `continue`：

```java
YearMonth targetMonth;
try {
    targetMonth = YearMonth.parse(sc);
} catch (DateTimeParseException e) {
    System.out.println("请输入正确格式的月份，例如：2026-08");
    continue;
}
```

### 小结

| 写法 | 是否合适 | 原因 |
|---|---|---|
| `catch (NumberFormatException)` | 不合适 | 当前代码没有数字解析，不会触发 |
| `catch (Exception)` | 不推荐 | 会掩盖所有异常 |
| `catch (DateTimeParseException)` | 推荐 | 精确对应年月格式错误 |

---

*记录于 2026-10-18*
