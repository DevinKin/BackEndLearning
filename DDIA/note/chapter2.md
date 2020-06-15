# 第二章-数据模型与查询语言

## 关系模型和文档模型

SQL是最著名的数据模型，是关系模型。

XML是文档模型。

## NoSQL的诞生

NoSQL是Not Only SQL。

采用NoSQL的几个驱动因素

- 比关系数据库更好的扩展性，包括更大的数据集和非常高的写入吞吐量。
- 相比商业数据库产品，免费和开源软件更受偏爱。
- 关系模型不能很好地支持一些特殊的查询操作。
- 受挫于关系模型的限制性，渴望一种更具多动态性与表现力的数据模型。

混合持久化（polyglot persistence）：关系数据库与各种非关系数据库一起使用

## 对象不匹配

面对对象模型和关系模型不匹配，需要进行一层转换。

模型之间的不连贯有时被称为阻抗不匹配（impedance mismatch）。

## 多对一和一对多关系

一个经验法则是，如果重复存储了可以存储在一个地方的值，则模式就不是规范化。

层次模型不支持多对多关系

## 网络模型

网络模型由一个称为数据系统语言会议（`CODASYL`） 的委员会进行了标准化，并被数个不同的数据库商实现;它也被称为`CODASYL`模型。

`CODASYL`模型是层次模型的推广。在层次模型的树结构中，每条记录只有一个父节点；在网络模式中，每条记录可能有多个父节点。

网络模型中记录之间的链接不是外键，而更像编程语言中的指针（同时仍然存储在磁盘上） 。访问记录的唯一方法是跟随从根记录起沿这些链路所形成的路径。这被称为访问路径（access path） 。

`CODASYL`中的查询是通过利用遍历记录列和跟随访问路径表在数据库中移动游标来执行的。

## 关系模型

关系模型做的就是将所有的数据放在光天化日之下：一个关系（表） 只是一个 元组（行） 的集合。

## 与文档数据库相比

文档数据库还原为层次模型：其父记录中存储嵌套记录，而不是单独的表中。

关系数据库表示多对一和多对多关系时，相关项都被一个唯一标识符应用，这个标识符在关系模型中称为外键，在文档模型中称为引用。

## 哪个数据模型更方便写代码？

如果应用程序中的数据具有类似文档的结构（即，一对多关系树，通常一次性加载整个树） ，那么使用文档模型可能是一个好主意。

文档模型有一定的局限性：

- 不能直接引用文档中的嵌套项目，只要嵌套不深，问题不大。
- 对连接的支持比较差

对于高度相关联的数据，选用文档模型是糟糕的，选用关系模型是可接受的，选用图形模型是最自然的。

## 文档模型中的架构灵活性

大多数文档数据库以及关系数据库中的JSON支持都不会强制文档中的数据采用何种模式。关系数据库的XML支持通常带有可选的模式验证。没有模式意味着可以将任意的键和值添加到文档中，并且当读取时，客户端对无法保证文档可能包含的字段。

文档数据库是读时模式（schema-on-read）（数据结构是隐含的，只有在数据读取时才被解释）。

传统的关系数据库是写时模式（schema-on-write）（传统的关系数据库方法中，模式明确，且数据库确保所有的数据都符合其模式） 

读时模式类似于编程语言中的动态（运行时）类型检查，而写时模式类似于静态（编译时）类型检查。

当由于某种原因（例如，数据是异构的） 集合中的项目并不都具有相同的结构时,读时模式更具优势。

## 查询的数据局部性

局部性仅仅适用于同时需要文档绝大部分内容的情况。数据库通常需要加载整个文档，即使只访问其中的一小部分，这对于大型文档来说是很浪费的。更新文档时，通常需要整个重写。只有不改变文档大小的修改才可以容易地原地执行。因此，通常建议保持相对小的文档，并避免增加文档大小的写入 。这些性能限制大大减少了文档数据库的实用场景。

## 文档和关系数据库的融合

文档模型和关系数据模型互补，关系模型和文档模型的混合是未来数据库一条很好的路线。

## 数据库查询语言

关系模型包含了一种查询数据的新方法：SQL是一种声明式查询语言，而IMS和CODASYL使用命令式代码来查询数据库。

声明式查询语言是迷人的，因为它通常比命令式API更加简洁和容易。但更重要的是，它还隐藏了数据库引擎的实现细节，这使得数据库系统可以在无需对查询做任何更改的情况下进行性能提升。

声明式语言往往适合并行执行。现在，CPU的速度通过内核的增加变得更快，而不是以比以前更高的时钟速度运行。

## MapReduce查询

MapReduce既不是一个声明式的查询语言，也不是一个完全命令式的查询API，而是处于两者之间：查询的逻辑用代码片断来表示，这些代码片段会被处理框架重复性调用。

map和reduce函数在功能上有所限制：它们必须是纯函数，这意味着它们只使用传递给它们的数据作为输入，它们不能执行额外的数据库查询，也不能有任何副作用。这些限制允许数据库以任何顺序运行任何功能，并在失败时重新运行它们

## 图数据模型

如果你的应用程序大多数的关系是一对多关系（树状结构化数据） ，或者大多数记录之间不存在关系，那么使用文档模型是合适的。

关系模型可以处理多对多关系的简单情况，但是随着数据之间的连接变得更加复杂，将数据建模为图形显得更加自然。

一个图由两种对象组成：

- 顶点（vertices）（也称节点（nodes）或实体（entites））
- 边（edges）（也称为关系（relationships）或弧（arcs））

图并不局限于同类数据：同样强大地是，图提供了一种一致的方式，用来在单个数据存储中存储完全不同类型的对象。

属性图模型（由Neo4j，Titan和InfiniteGraph实现） 和三元组存储（triple-store） 模型（由Datomic，AllegroGraph等实现） 。

图的三种声明式查询语言：

- Cypher
- SPARQL
- Datalog。

### 属性图

在属性图模型中，每个顶点（vertex）包括

- 唯一的标识符
- 一组出边（outgoing edges）
- 一组入边（ingoing edges）
- 一组属性（键值对）

每条边（edge）包括：

- 唯一标识符
- 边的起点
- 边的起点/尾部顶点（tail vertex）
- 边的中点/头部顶点（head vertex）
- 描述两个顶点之间关系类型的标签
- 一组属性（键值对）

使用关系模式来表示属性图

```sql
CREATE TABLE vertices (
    vertex_id INTEGER PRIMARY KEY,
    properties JSON
);
CREATE TABLE edges (
    edge_id INTEGER PRIMARY KEY,
    tail_vertex INTEGER REFERENCES vertices (vertex_id),
    head_vertex INTEGER REFERENCES vertices (vertex_id),
    label TEXT,
    properties JSON
);
CREATE INDEX edges_tails ON edges (tail_vertex);
CREATE INDEX edges_heads ON edges (head_vertex);
```

图表在可演化性是富有优势的：当向应用程序添加功能时，可以轻松扩展图以适应应用程序数据结构的变化。

## Cypher查询语言

Cypher是属性图的声明式查询语言，为Neo4j图形数据库而发明。

```cypher
CREATE
(NAmerica:Location {name:'North America', type:'continent'}),
(USA:Location {name:'United States', type:'country' }),
(Idaho:Location {name:'Idaho', type:'state' }),
(Lucy:Person {name:'Lucy' }),
(Idaho) -[:WITHIN]-> (USA) -[:WITHIN]-> (NAmerica),
(Lucy) -[:BORN_IN]-> (Idaho)
```

查找所有从美国移民到欧洲的人的Cypher查询：

```cypher
MATCH
(person) -[:BORN_IN]-> () -[:WITHIN*0..]-> (us:Location {name:'United States'}),
(person) -[:LIVES_IN]-> () -[:WITHIN*0..]-> (eu:Location {name:'Europe'})
RETURN person.name
```

## 三元组存储和SPARQL

在三元组存储中，所有信息都以非常简单的三部分表示形式存储（主语，谓语，宾语） 。例如，三元组 (吉姆, 喜欢 ,香蕉) 中，吉姆 是主语，喜欢 是谓语（动词） ，香蕉 是对象。

三元组的主语相当于图中的一个顶点。而宾语是下面两者之一：

1. **原始数据类型中的值，例如字符串或数字。在这种情况下，三元组的谓语和宾语相当于主语顶点上的属性的键和值**。例如， (lucy, age, 33) 就像属性 {“age”：33} 的顶点lucy。
2. **图中的另一个顶点。在这种情况下，谓语是图中的一条边，主语是其尾部顶点，而宾语是其头部顶点**。例如，在 (lucy, marriedTo, alain) 中主语和宾语 lucy 和 alain 都是顶点，并且谓语 marriedTo 是连接他们的边的标签。

### SPARQL查询语言

SPARQL变量以问号开头

```SPARQL
PREFIX : <urn:example:>
SELECT ?personName WHERE {
?person :name ?personName.
?person :bornIn / :within* / :name "United States".
?person :livesIn / :within* / :name "Europe".
}
```

变量 usa 被绑定到任意具有值为字符串 "United States" 的 name 属性的顶点

```SPARQL
?usa :name "United States".
```

## 图形数据库和网络模型相比较

- 在CODASYL中，数据库有一个模式，用于指定哪种记录类型可以嵌套在其他记录类型中。在图形数据库中，不存在这样的限制：任何顶点都可以具有到其他任何顶点的边。这为应用程序适应不断变化的需求提供了更大的灵活性。

- 在CODASYL中，达到特定记录的唯一方法是遍历其中的一个访问路径。在图形数据库中，可以通过其唯一ID直接引用任何顶点，也可以使用索引来查找具有特定值的顶点。
- 在CODASYL，记录的后续是一个有序集合，所以数据库的人不得不维持排序（这会影响存储布局） ，并且插入新记录到数据库的应用程序不得不担心的新记录在这些集合中的位置。在图形数据库中，顶点和边不是有序的（只能在查询时对结果进行排序）。
- 在CODASYL中，所有查询都是命令式的，难以编写，并且很容易因架构中的变化而受到破坏。在图形数据库中，如果需要，可以在命令式代码中编写遍历，但大多数图形数据库也支持高级声明式查询语言，如Cypher或SPARQL。

## 基础：Datalog

在实践中，Datalog被用于少数的数据系统中：例如，它是Datomic的查询语言，Cascalog是一种用于查询Hadoop大数据集的Datalog实现。

Datalog的数据模型类似于三元组模式，但进行了一点泛化。把三元组写成**谓语（主语，宾语）** ，而不是写三元语**（主语，谓语，宾语）** 。

```datalog
within_recursive(Location, Name) :- name(Location, Name). /* Rule 1 */
within_recursive(Location, Name) :- within(Location, Via), /* Rule 2 */
within_recursive(Via, Name).
migrated(Name, BornIn, LivingIn) :- name(Person, Name), /* Rule 3 */
born_in(Person, BornLoc),
within_recursive(BornLoc, BornIn),
lives_in(Person, LivingLoc),
within_recursive(LivingLoc, LivingIn).
?- migrated(Who, 'United States', 'Europe'). /* Who = 'Lucy'. */
```

在这里，我们定义了两个新的谓语， `within_recursive`和 `migrated `。这些谓语不是存储在数据库中的三元组中，而是它们是从数据或其他规则派生而来的。规则可以引用其他规则，就像函数可以调用其他函数或者递归地调用自己一样。像这样，复杂的查询可以一次构建其中的一小块。

在规则中，以大写字母开头的单词是变量，谓语则用Cypher和SPARQL的方式一样来匹配。例如， name(Location, Name) 通过变量绑定 Location = namerica 和 Name ='North America' 可以匹配三元组 name(namerica, 'North America')。

要是系统可以在`:-`操作符的右侧找到与所有谓语的一个匹配，就运用该规则。当规则运用时，就好像通过`:-`的左侧将其添加到数据库（将变量替换成它们匹配的值） 。