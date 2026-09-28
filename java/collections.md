# Java 集合框架

## 目录

- [Collection 是接口、Map 是例外：JCF 全景与"框架"一词的含义](#collection-是接口map-是例外jcf-全景与框架一词的含义)
- [集合的三维分类：语义、实现、并发](#集合的三维分类语义实现并发)
- [forEach()：内部迭代的声明式遍历](#foreach-内部迭代的声明式遍历)
- [增强 for、Consumer super T、方法引用：一次 forEach 调用的三个零件](#增强-forconsumer-super-t方法引用一次-foreach-调用的三个零件)
- [Collections 工具类高频方法](#collections-工具类高频方法)

---

## Collection 是接口、Map 是例外：JCF 全景与"框架"一词的含义
- 2026-09-23

**结论**：Collection 是**接口**不是类，List/Set/Queue 都实现自它；**Map 是独立体系**，不继承 Collection（元素模型是键值对，`add(E)` 语义没法定义），靠 `keySet()/values()/entrySet()` 三个 Collection 视图与体系互通。整个体系叫集合框架（JCF）——接口（契约）+ 实现（数据结构）+ 算法（Collections 工具）的统一架构。

**细节**：

```
Iterable                        ← 最顶层接口，能被 for-each 遍历的原因
  └── Collection
        ├── List    → ArrayList, LinkedList, Vector → Stack
        ├── Set     → HashSet → LinkedHashSet, TreeSet, EnumSet
        └── Queue   → PriorityQueue, ArrayDeque, LinkedList
              └── Deque

Map（平行独立体系）→ HashMap → LinkedHashMap, TreeMap, Hashtable
```

统一顶层接口的三层收益：

1. **多态解耦**：`void process(Collection<Order> orders)` 面向抽象，换实现不改代码；
2. **算法只写一遍全体受益**：`Collections.sort()` 等只针对接口编程，所有实现直接能用；
3. **for-each 通行证**：Collection extends Iterable → `for (X x : collection)` 语法糖（底层调 iterator()）。

关于"家族"：不是官方术语，是口语，正式叫**继承体系（hierarchy）**；判断归属唯一依据是继承链（is-a）。Java 类型归属是显式声明的（不像鸭子类型）。

**"框架"为什么叫框架（框架 vs 库）**：

| | 库（library） | 框架（framework） |
|---|---|---|
| 关系 | 你调用它 | 它调用你 |
| 控制流 | 你写主流程，需要时调库 | 框架搭骨架，你在指定位置填代码 |

经典口诀：好莱坞原则 "Don't call us, we'll call you"。JCF 配得上"框架"的证据（控制反转）：自定义集合 `extends AbstractCollection` 实现 iterator() 后，for-each 语法和 Collections 算法会**反过来调用你的代码**——你写的 iterator() 被不认识你的 JDK 代码调用。

重要的思辨结论：**好莱坞原则主要是设计者视角，消费式使用下体感不到框架/库差异**。差异的触发条件是"从消费方变成提供方"，三个感知信号：①我的代码没被调用却在跑（Spring @Controller/@Test）；②自定义实现被体系自动调用（iterator()/Comparator）；③代码必须顺着别人的骨架摆放（契约约束）。更深一层：控制反转只是一种 API 设计手法（回调），框架/库是**光谱不是二分**——JCF 只在迭代器/比较器几个点用回调（反转浓度低、框架感弱），Spring 把控制反转作为全局组织原则（浓度高到叫 IoC 容器）。

历史：JDK 1.0 只有 Vector、Stack、Hashtable 孤立老类；JDK 1.2（Joshua Bloch 主导）建立接口体系并收编老类（Vector implements List、Hashtable implements Map）——所以老类里新旧方法并存，是活化石。

## 集合的三维分类：语义、实现、并发
- 2026-09-23

**结论**：集合的分类是多维正交的——按接口分 = 按元素语义分；按实现分 = 按数据结构性能分；按并发分 = 按场景分。选集合就是三个维度各做一次选择：先定语义，再定性能，最后看并发。

**细节**：

维度一：按顶层接口（官方分法）：

| 接口 | 特征 | 语义 | 代表实现 |
|---|---|---|---|
| List | 有序、**可重复**、有索引 | "第几个是谁" | ArrayList、LinkedList |
| Set | **不可重复** | "在不在名单里"（归属） | HashSet、TreeSet |
| Queue/Deque | 按处理顺序进出 | "谁先被处理" | ArrayDeque、PriorityQueue |
| Map（独立） | 键值对、键不重复 | "用 key 查 value" | HashMap、TreeMap |

分类依据是元素语义约束：允许重复吗（购物车 vs 黑名单）、在乎顺序吗（[A,B]≠[B,A] vs 无序承诺）、怎么检索（索引/成员/队头/key）——同一份数据放不同家族语义完全不同。

维度二：按底层数据结构（性能特征）：

| 结构 | 实现 | 强项 | 弱项 |
|---|---|---|---|
| 动态数组 | ArrayList | 随机访问 O(1) | 中间插删 O(n) |
| 双向链表 | LinkedList | 头尾插删 O(1) | 随机访问 O(n) |
| 哈希表 | HashSet/HashMap | 增删查 O(1) | 无序、最坏退化 |
| 红黑树 | TreeSet/TreeMap | O(log n)、有序遍历、范围查询 | 比哈希慢 |

语义与性能正交组合（接口与实现分离的落地）：遍历多选 ArrayList，头尾操作多选 ArrayDeque。

维度三：按线程安全性：非线程安全（ArrayList/HashMap，默认最快）→ 遗留同步（Vector/Hashtable，方法级 synchronized(this) 粒度粗）→ juc 并发容器（ConcurrentHashMap、CopyOnWriteArrayList，专门并发设计）。

细分修饰维度：插入序（LinkedHashSet）vs 排序序（TreeSet）vs 无序（HashSet）；可变 vs 不可变（List.of）。

## forEach()：内部迭代的声明式遍历
- 2026-09-23

**结论**：forEach 是 JDK 8 加在 Iterable 接口上的**内部迭代**方法——传统循环是你来转、逐个拿元素（外部迭代），forEach 是集合来转、逐个把元素喂给你给的动作（内部迭代）。性能与普通 for 无差别（就是语法糖），价值在把"怎么做（循环）"和"做什么（动作）"分离。

**细节**：

三个住所：

```java
list.forEach(s -> System.out.println(s));       // 1. Iterable.forEach —— 所有集合
list.forEach(System.out::println);              //    方法引用写法
map.forEach((k, v) -> System.out.println(k));   // 2. Map.forEach —— BiConsumer<K,V>
list.stream().filter(s -> s.length() > 3).forEach(...);  // 3. Stream 终端操作
```

本体（Iterable 的 default 方法，JDK 8 接口第一次能带方法体的标志性产物）：

```java
default void forEach(Consumer<? super T> action) {
    for (T t : this) {
        action.accept(t);      // 每个元素调用一次你给的 lambda
    }
}
```

forEach 也是"框架调用你"的回调点：把行为登记进去，集合在循环里逐个调用它。

与 for-each 循环怎么选：

| | `for (X x : list)` | `list.forEach(x -> ...)` |
|---|---|---|
| 迭代控制权 | 外部迭代 | 内部迭代 |
| 中途退出 | break/continue 随便用 | **不能 break**；lambda 里 return ≈ continue |
| 局部变量 | 随意 | 只能捕获 effectively final |

经验法则：逐元素消费且不需中途退出 → forEach；需要下标/break/复杂控制流 → 传统 for；需要转换/过滤/聚合出结果 → Stream 的 map/filter/collect（别用 forEach 收集数据）。

**坑**：没有 break（要"找到第一个就停"用 for-each 循环）；遍历中改集合照样 CME（删除用 removeIf）；lambda 里 return ≠ 返回方法（只是结束本轮）；parallelStream().forEach 不保证顺序（要顺序用 forEachOrdered）；list 为 null 时调用 NPE。

## 增强 for、Consumer super T、方法引用：一次 forEach 调用的三个零件
- 2026-09-23

**结论**：`names.forEach(System.out::println)` 内部发生的事 = 三个语法的组装：①集合内部跑着 `for (String x : names)`（语法糖展开的迭代器循环）；②每轮调 `action.accept(x)`，action 类型是 `Consumer<? super String>`；③`System.out::println` 就是那个 Consumer——每收到一个元素就执行 `System.out.println(x)`。与手写 `for (String x : names) System.out.println(x)` 零差别。

**细节**：

**① 增强 for（JDK 5）**：`for (X x : list)` 读作"对 list 里的每个 x"，是编译器语法糖，对集合展开为迭代器循环：

```java
for (X x : list) { ... }
// 编译后等价于：
for (Iterator<X> it = list.iterator(); it.hasNext(); ) {
    X x = it.next();
    ...
}
```

能用 for-each 的资格：实现 Iterable（或是数组）。"Collection extends Iterable 所以所有集合都能 for-each"机关在此。

**② `Consumer<? super T>`**：两截语法。`Consumer<T>` 是 JDK 8 函数式接口（java.util.function），代表"接受一个 T、无返回值的动作"，唯一抽象方法 `void accept(T t)`，lambda 可直接赋值。`<? super T>` 是下界通配符（"T 或 T 的任意父类型"），出现在参数位置是为了兼容更宽泛的消费者：

```java
Consumer<Object> printer = o -> System.out.println(o);  // "打印任何东西"
List<String> names = List.of("a", "b");
names.forEach(printer);    // ✓ 若签名写死 Consumer<String>，这个合理调用就被拒绝
```

直觉：能消费"任何对象"的消费者当然能消费 String。PECS 原则：参数在输入/消费位置用 super，输出/生产位置用 extends。

**③ 方法引用 `System.out::println`（JDK 8）**：读作"System.out 这个对象的 println 方法"，把现成方法**当作值**传递。规则：lambda 体只是"调用一个现成方法"时即可缩写为方法引用。四种形态：

| 形态 | 例子 | 等价 lambda |
|---|---|---|
| 对象::实例方法 | `System.out::println` | `x -> System.out.println(x)` |
| 类::静态方法 | `Integer::parseInt` | `s -> Integer.parseInt(s)` |
| 类::实例方法 | `String::toUpperCase` | `s -> s.toUpperCase()` |
| 类::new | `ArrayList::new` | `() -> new ArrayList<>()` |

**已验证的理解修正**：":: 把一个方法传给一个需要重写的方法" → 准确说法是**用一个现成方法充当函数式接口那个抽象方法的实现体**（顶替实现，不是传给谁）。两处术语/机制精化：(a) "重写"（override，子类覆盖父类方法）应为"实现"（implement，实现接口抽象方法）；(b) Java 的方法**不是一等公民**，无法当值传——`::` 的产物是函数式接口的一个实例（底层 invokedynamic + LambdaMetafactory），"记住"该调谁，accept 被调用时转调它。这解释了方法引用必须出现在能推断目标接口的上下文里。

四级等价演化：

```java
list.forEach(new Consumer<String>() {              // 匿名内部类时代
    public void accept(String t) { System.out.println(t); }
});
list.forEach(t -> System.out.println(t));          // lambda
list.forEach(System.out::println);                 // 方法引用
```

## Collections 工具类高频方法
- 2026-09-23

**结论**：`java.util.Collections`（注意有 s）是纯静态工具类（私有构造器、全静态方法），是 JCF"算法层"的主体——**Collections 是工具箱，Collection 是族谱**，两码事。

**细节**：

排序与顺序（只吃 List，Set 要排序先转 List 或用 TreeSet）：

| 方法 | 作用 |
|---|---|
| `sort(list)` / `sort(list, cmp)` | 自然排序 / 定制排序 |
| `reverse(list)` | 反转 |
| `shuffle(list)` | 随机打乱（洗牌） |
| `swap(list, i, j)` | 交换两位置元素 |

查找与统计：

| 方法 | 作用 | 备注 |
|---|---|---|
| `max(coll)` / `min(coll)` | 最大/最小 | 有 Comparator 重载 |
| `binarySearch(list, key)` | 二分查找 | **前提已排序**；找不到返回 `-(插入点)-1` |
| `frequency(coll, obj)` | 元素出现次数 | 依赖 equals |

批量与填充：`addAll(coll, e1, e2...)`（可变参数批量加）、`fill(list, obj)`、`nCopies(n, obj)`（n 份拷贝的只读假列表）、`copy(dest, src)`（dest 长度须 ≥ src）。

空集合与只读包装（工程防御利器）：

```java
// 返回"没有结果"时返回空集合而不是 null——调用方直接遍历不炸
return r == null ? Collections.emptyList() : r;
```

`emptyList()/emptySet()/emptyMap()`（共享空常量）、`singletonList(o)/singleton(o)`（单元素）、`unmodifiableList(list)` 等（只读视图）。

线程安全包装：`Collections.synchronizedList(new ArrayList<>())` 把任意集合包成方法级 synchronized 版本。**坑**：单方法原子但迭代不是，遍历必须手动 `synchronized (list) { for ... }`；现代高并发已被 juc 的 ConcurrentHashMap/CopyOnWriteArrayList 取代。

其他：`reverseOrder()` / `reverseOrder(cmp)`（反序比较器，降序排序钥匙）。

**坑速查**：Collections ≠ Collection（面试送命题）；binarySearch 前必须 sort 否则垃圾值；unmodifiableXxx 是**视图不是拷贝**（原集合变、视图跟着变）；copy 要求 dest 已有足够 size（不是 capacity）。
