# 12 · 工具层 考题

## 🟢 概念题（5）

1. `EasyGrid<T>` 的越界访问行为是什么？
2. `Table<T>` 和 `TableIndex<K,V>` 的分工是什么？
3. `TableIndex` 内部用什么数据结构存一个 key 对应的多个 item？
4. LogKit 如何实现"发布期关闭普通日志"？
5. FluentAPI 的本质是什么？

<details><summary>参考答案要点</summary>

1. 越界时 `Debug.LogWarning` 并返回 `default`（getter）或忽略写入（setter），不抛异常（容错"告警失败"）。
2. `Table` 是抽象基类，提供 Add/Remove/Clear/遍历/Dispose 模板，子类声明并维护多个 `TableIndex`；`TableIndex` 是单个查询维度的索引（keyGetter + 分桶字典）。
3. `Dictionary<TKey, List<TDataItem>>`——每个 key 一个 List 桶（原版 List 来自 ListPool）。
4. 设 `LogLevel`，`I/W/E` 各自先判 `mLogLevel < LogLevel.Xxx` 提前 return；把等级调到 Error 即过滤掉 Normal/Warning。
5. 对已有类型（Unity/C#）添加扩展方法、返回 self 以支持链式调用——纯语法糖，不改变类型本身。
</details>

## 🟡 机制题（5）

1. `TableKit` 的"联合查询"是如何实现的？时间/空间权衡是什么？
2. `TableIndex.Dispose` 做了什么特别的事？
3. `EasyGrid.Resize` 如何处理旧数据与新增区域？
4. `EasyGrid.Clear` 的 `cleanupItem` 参数有什么用？
5. 为什么 `TableIndex.Get` 找不到 key 时返回 `Enumerable.Empty` 而非 null？

<details><summary>参考答案要点</summary>

1. 用某索引 `Get(key)` O(1) 取一个子集桶，再用 LINQ `.Where(...)` 在子集上二次过滤。空间换时间：对高频查询维度建索引（占内存），低频条件走 LINQ（省索引）。
2. 把每个桶 `List` `Release2Pool()` 还给 ListPool，再把索引字典本身 `Release2Pool`，最后置 null——连容器都池化回收。
3. 建新数组，对每个新位置：若在旧范围内（x<旧Width && y<旧Height）拷贝旧值，否则调 `onAdd(x,y)` 填新；原版分 x 方向和 y 方向两段循环分别处理拷贝与新增。
4. 清空时对每个格子内容调用清理回调（如 `Object.Destroy`），用于释放格子持有的资源，之后才置 default/null。
5. 返回空集合让调用方可以直接 `foreach`/LINK 链式而无需判 null，避免 NRE（空对象模式）。
</details>

## 🔴 架构陷阱题（6）

1. **跨模块对比**：TableKit 的 `Table<T>`/`TableIndex` 与 UIKit 的 `UIPanelTable`/`UIKitTableIndex`、ResKit 的 `ResTable` 几乎同构。为什么框架不让它们都复用 TableKit，而是各自抄一份？
2. 一个继承 `Table` 的子类在 `OnAdd` 维护了索引 A、B，但 `OnRemove` 只移除了 A 忘了 B。会出现什么 bug？容易被发现吗？
3. `TableIndex` 的桶 List 来自 ListPool，若 `Dispose` 后又继续 `Add`，会怎样？
4. `EasyGrid` 的 `this[x,y]` 越界返回 default 而不抛异常。这在什么场景下会掩盖真实 bug？
5. LogKit 的 `E`（Error）用 `string.Format(msg, args)`，若 msg 里含未转义的 `{` 但没有对应 args，会怎样？
6. FluentAPI 扩展方法是静态分发的。对一个声明类型为基类、实际是子类的对象调用扩展方法，会调到哪个版本？

<details><summary>参考答案要点</summary>

1. 复用 TableKit 会让 UIKit/ResKit 在 asmdef 层面依赖 TableKit，增加耦合、破坏模块自包含（每个 Kit 希望能独立裁剪/打包）。各自内联一份是"用代码重复换模块解耦"的工程权衡。代价是维护多份相似代码、修 bug 要改多处。
2. 按索引 B 查询会查到已被移除的"幽灵"item（B 桶里还留着引用），导致拿到已删除对象、内存泄漏（对象无法 GC）。不易发现：不报错，只在"用 B 索引查询时结果多了不该有的项"才暴露，且依赖具体查询路径。
3. `Dispose` 把桶 Release2Pool 并将 `mIndex=null`，之后 Add 会 NRE（访问 null 字典）。且若那个 List 已被池借给别处，继续用旧引用会数据错乱（释放后使用）。
4. 当算法逻辑本应保证坐标合法、却因 bug 算出越界坐标时，EasyGrid 静默返回 default 会让错误"看起来正常"（拿到默认值继续跑），掩盖了坐标计算的 bug，使问题更难定位。需要权衡"健壮容错"与"快速暴露错误"。
5. `string.Format` 遇到不匹配的 `{` 会抛 `FormatException`——日志调用本身崩溃。这是"日志里含 JSON/代码片段含花括号"的经典坑，应优先用无参重载或转义。
6. 调到声明类型对应的扩展方法版本（编译期按静态类型选择重载），不会按运行时实际类型多态分发。这是扩展方法的固有陷阱：扩展方法不是虚方法，不参与多态。
</details>

## ✍️ 实操题（3）

1. 用 `EasyGrid<int>` 实现一个 4×4 的扫雷雷区，随机布 3 个雷（值 -1），其余格子值为周围雷数。用 `ForEach` 打印。说明越界保护如何简化"统计周围 8 格"的边界处理。
2. 用 `Table<Item>` 建模背包，建立"按类型"和"按品质"两个索引，查询"武器类 + 品质≥稀有"的物品。指出维护两个索引时最易犯的错误。
3. 解释为什么 LogKit、TableKit 这类工具会大量使用"委托注入"（keyGetter/Func/Action），并对比"继承重写"方案的优劣。

<details><summary>参考答案要点</summary>

1. `var mine=new EasyGrid<int>(4,4); mine.Fill(0);` 随机设 3 格 -1；统计时对每格遍历周围 8 个偏移，`mine[x+dx,y+dy]` 越界自动返回 default(0)、不崩，免去手写 `if(nx>=0&&nx<w...)` 的边界判断（越界保护把边界处理内化进索引器）。`ForEach((x,y,c)=>print)`。
2. `TypeIndex=new(i=>i.Type); QualityIndex=new(i=>i.Quality)`，OnAdd/OnRemove 同步两者；查询 `TypeIndex.Get(Weapon).Where(i=>i.Quality>=Rare)`。最易犯错：OnRemove/OnClear 漏维护其中一个索引，导致幽灵数据。
3. 工具要对"未知的用户类型 T"工作，无法预知如何取 key/如何处理元素，故用委托把"取 key/逐项操作"的策略注入，保持工具对 T 完全泛化、零侵入（T 不需继承任何基类）。继承重写方案要求 T 实现接口/继承基类（侵入性强、占继承位），但能携带更多上下文、可多态。工具层选委托注入是为了最大通用性与零侵入。
</details>
