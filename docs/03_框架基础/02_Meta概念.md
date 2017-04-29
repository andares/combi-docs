# Meta简述

Meta在Combi中的概念是一种结构化的数据载体。在业务逻辑的流转中，推荐使用Meta对象封装数据，尤其是结构型（Struct）的数据。在核心包提供的功能中，如请求与响应参数、数据模型等，都会用到Meta对象。

Meta提供的基础类有```Collection```和```Struct```两种，前者是一个列表，后者为一个结构体。同时为其提供了一系列Interfaces和Traits，可便捷统一地实现例如序列化，结构定义和检查等功能。

# Collection基类

Collection基类位于```Combi\Meta\Collection```，提供了一个列表式的可伸缩数据集合，也可以使用字串键值用作key/value结构存储。```Combi\Meta\Container```是collection的别名，Combi中的很多容器类对象都继承自Collection基类。

## 创建自己的Collection类

```php
use Combi\Meta\Collection;

class MyCollection extends Collection {
    // ... your code
}
```

Collection基类没有任何抽象方法需要实现，也不需要预先定义结构。

## 方法范例

### 基础操作

```php
$collection = new MyCollection();

$collection->set('name', 'triss');

echo $collection->get('name'); // print 'triss'

var_dump($collection->has('name')); // print true

echo $collection->count(); // print 1

$collection->remove('name'); // name is be removed

$collection->clear(); // collection data is empty array now
```

### 替换已存在键

```{php id:"iwtkik9g"}
$items = [
    'name'  => 'kerafy',
    'age'   => 2,
];

// 只有name被替换，age不会被写入
$collection->replace($items);
```

### 遍历数据

```{php id:"iwtkik9t"}
foreach ($collection->all() as $key => $value) {
    echo "$key = $value\n";
}
```

> 注意，all()方法将返回一个迭代器而非数组。

### 追加数据

提供了append()方法往collection中添加数据，就像使用数组那样。下面代码示例了往集合中添加了一个模型对象。

```{php id:"iwtkika3"}
$collection->append($model);
```

### 转换为数组

toArray()方法可将一个collection转换为数组。如果collection中包含的某些对象同样属于```Combi\Interfaces\Arrayable```的实现，将会一并转换为数组。

```{php id:"iwtkika7"}
var_dump($collection->toArray());
```

### 聚合迭代器

collection类默认实现了聚合迭代器，所以你也可以直接foreach遍历对象。

```{php id:"iwtkikaj"}
foreach ($collection as $key => $value) {
    echo "$key = $value\n";
}
```

此方法等同于调用了all()方法，和toArray()不同。

# Struct基类

Struct基类位于```Combi\Base\Struct```，提供一个确定的、具有自检特性的key/value数据结构。

## 创建自己的Struct

下面的例程定义了一个名为Person的结构类。相比Collection，Struct的定义要多出有关结构描述的内容。

```{php id:"iwtkikaq"}
use Combi\Base\Struct;

/**
 *
 * @property string $name
 * @property string $father
 * @property int $age
 */
class Person extends Struct {
    protected static $_defaults = [
        'name'      => null,
        'father'    => '',
        'age'       => 18,
    ];

    // ... your code
}
```

## Struct结构定义

如上面代码所示，基础的Struct的结构定义通过protected静态属性```$_defaults```来定义，通过一组键值定义每个字段名与其默认值。

Person类中有```name, father, age```三个字段，其默认值分别为null、空串，以及18。

> 对Struct类来说，当尝试访问某个属性时，如果该属性事先未被设置，会返回默认值，如同已经被设置过那样。默认值设null，一般情况下表示你希望**该值必须由外部赋予**方可正确运作，这在之后的confirm功能介绍中也会起到作用。

作为推荐，由于Struct属性使用重载实现，因此在类注释中标注每个属性会让类的使用更加方便。

### 获取Struct的结构定义

```{php id:"iwtkikau"}
var_dump(Person::defaults());
```

以上代码会输出```$_defaults```属性内容。

## 结构迁移

基础Struct的结构迁移基于deprecated配置，设置规则如下：

* 新增字段（属性）时，在```$_defaults```的数组末尾添加。
  * 不要重排```$_defaults```的定义顺序。
* 移除字段时，保留```$_defaults```中的字段定义，在```$_deprecated```中添加该字段。

```{php id:"iwtkikb3"}
use Combi\Base\Struct;

/**
 *
 * @property string $name
 * @property int $age
 * @property int gender
 */
class Person extends Struct {
    protected static $_defaults = [
        'name'      => null,
        'father'    => '',
        'age'       => 18,
        'gender'    => 0,
    ];

    protected static $_deprecated = [
        'father'    => 1,
    ];
}
```

上面例程中为Person类添加了gender属性，并移除了father属性。

> 移除一个字段请在```$_deprecated```静态属性中以字段名为key设值为1

### deprecated字段的行为

废弃字段在Struct对象中对单个操作时，并不会报异常，例如：

```{php id:"iwtkikbi"}
$me = new Person();
$me->set('father', 'unamed);
echo $me->get('father');
```

即使father字段已经被弃用，以上例程并不会报错。

但批量输出和赋值（稍后会提到的```fill()```方法）时，会跳过弃用字段。比如：

```{php id:"iwtkikbs"}
$someone = new Person();
$someone
    ->set('name', 'triss)
    ->set('father', 'unamed')
    ->set('age', 50)
    ->set('gender', 2);
var_dump($someone->toArray());
```

上面例程不会输出father字段。

通过```defaults()```方法获取字段列表时，也会跳过deprecated字段。但下面方法可以获取全部字段：

```{php id:"iwtkikc6"}
var_dump(Person::defaults(true));
```

该访问支持接收一个```bool $include_deprecated```参数，用于返回全部字段结构。

> 要判断一个字段是否为弃用，可以使用```Person::isKeyDeprecated($key)```方法，返回类型为bool型。

## confirm机制

Struct提供了一套confirm机制，用于维护结构中数据规则。该机制通过显式调用触发。

```{php id:"iwtkikcg"}
$someone = new Person();
$someone
    ->set('age', 50)
    ->set('gender', 2)
    ->confirm(); // will throw an exception
```

上面的代码会抛出一个违例“field [name] can not be empty”，因为并未给默认值为null的name字段赋值。

confirm的作用是当对Struct中的数据进行了一系列变更后，将要对其进行下一步处理时（比如模型持久化，比如发送消息，或是确认数据输出），对结构中数据完整性做一次校验。

在默认情况下，仅会对默认值设置为null的字段检测是否有赋不为null的值。

> 同defaults()一样，如需对deprecated字段生效，请调```confirm(true)```

> 同toArray()一样，如果Struct中某些字段的值同样为一个Struct对象，这些对象的confirm()方法也会一并被调用。

### confirm勾子

可以自定义一些字段的校验规则，如下所示：

```{php id:"iwtkikcx"}
use Combi\Base\Struct;

/**
 *
 * @property string $name
 * @property int $age
 * @property int gender
 */
class Person extends Struct {
    protected static $_defaults = [
        'name'      => null,
        'father'    => '',
        'age'       => 18,
        'gender'    => 0,
    ];

    protected static $_deprecated = [
        'father'    => 1,
    ];

    protected function _confirm_name($val) {
        global $default_name;
        !$val &&
            $default_name &&
                $val = $default_name;
        if (!$val) {
            throw new \LogicException('Person name is empty');
        }
        return $val;
    }
}
```

按上例范所示，当confirm()时name的昵称为null或是空串时，就会尝试从全局变量$default_name中获取值。

> 使用自定义confirm勾子后，原有的null判定逻辑就不再有效，字段的可用性完全自行维护。

> confirm()勾子的机制可同时作为校验和过滤器使用。

### confirm整体勾子

Struct定义了一个空方法```afterConfirm()```，对于一些多字段相关联的判定，可以将相关处理逻辑复写在此方法中。

假设Person类有一个不合理的逻辑：所有大于40岁的只能是男性（gender=1），那么可以这么写：

```{php id:"iwtkikd7"}
use Combi\Base\Struct;

/**
 *
 * @property string $name
 * @property int $age
 * @property int gender
 */
class Person extends Struct {
    protected static $_defaults = [
        'name'      => null,
        'father'    => '',
        'age'       => 18,
        'gender'    => 0,
    ];

    protected static $_deprecated = [
        'father'    => 1,
    ];

    protected function _confirm_name($val) {
        global $default_name;
        !$val &&
            $default_name &&
                $val = $default_name;
        if (!$val) {
            throw new \LogicException('Person name is empty');
        }
        return $val;
    }

    protected function afterConfirm() {
        if ($this->get('age') > 40 & $this->get('gender') != 1) {
            throw new \LogicException('gender must be 1 when age > 40');
        }
    }
}
```

## 与Collection类似的方法

* set()
* get()
* has()
* remove()
* clear()
* all()

以上方法以及迭代器用法，与Collection基本相同。

> 其中```all()```方法与```defaults()```一样，可以接收一个```bool $include_deprecated```参数，为true时将忽略deprecated设置返回全部字段数据。默认为false

> Struct基类不支持```append()```方法。

# Meta之道

# Meta组件
