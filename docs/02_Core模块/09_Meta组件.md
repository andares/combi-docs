# Meta简述

在很多情况下，我们需要在应用中处理一组有关联的数据。在其他语言中，可能会提供类似 Struct 之类的数据类型作为支持。

在Combi中，我们提供了Meta对象，用于实现操作一组数据的常用方法。在核心包提供的功能中，如通讯的消息体、数据模型等，都会用到Meta对象。

Meta提供的基础类有```Collection```和```Struct```两种，前者是一个列表或是一个动态的结构体，后者为一个确定的结构体。同时为其提供了一系列Interfaces和Traits，可便捷统一地实现例如序列化，结构定义和检查等功能。

# Collection基类

Collection基类位于```Combi\Core\Meta\Collection```，提供了一个列表式的可伸缩数据集合，也可以使用字串键值用作key/value结构存储。

## 创建自己的Collection类

```php
use Combi\Core;

class MyCollection extends Core\Meta\Collection {
    // ... your code
}
```

Collection基类没有任何抽象方法需要实现，也不需要预先定义结构。

## 方法范例

### 基础操作

```php
$collection = new MyCollection();

$collection->set('name', 'Triss');

echo $collection->get('name'); // print 'triss'

var_dump($collection->has('name')); // print true

echo count($collection); // print 1

$collection->remove('name'); // name is be removed

$collection->clear(); // collection data is empty array now
```

### 替换已存在键

注意，```replace()```方法对原选不存在的键值不会被加入。

```php
$items = [
    'name'  => 'kerafy',
    'age'   => 2,
];

// 只有name被替换，age不会被写入
$collection->replace($items);
```

### 获取全部数据

```php
foreach ($collection->all() as $key => $value) {
    echo "$key = $value\n";
}
```


### 追加数据

提供了```push()```方法往collection中添加数据，就像使用数组那样。下面代码示例了往集合中添加了一个模型对象。

```php
$collection->push($model);
```

### 转换为数组

```toArray()```方法可将一个collection转换为数组。如果collection中包含的某些对象同样属于```Combi\Core\Interfaces\Arrayable```的实现，将会一并转换为数组。

```php
var_dump($collection->toArray());
```

### 聚合迭代器

collection类默认实现了聚合迭代器，所以你也可以直接foreach遍历对象。

```php
foreach ($collection as $key => $value) {
    echo "$key = $value\n";
}
```

此方法等同于调用了```iterate()```方法，和toArray()不同。

## Container Trait

```Combi\Core\Meta\Container```是collection的功能的```trait```实现，在不方便使用继承的情况下可以使用```use \Combi\Core\Meta\Container```实现容器功能。

>   注意，如果仅使用```use Container```的形式实现容器类，建议加入```implements \Combi\Core\Interfaces\Collection, \IteratorAggregate```声明。

# Struct基类

Struct基类位于```Combi\Core\Meta\Struct```，提供一个确定的、具有自检特性的key/value数据结构。

## 创建自己的Struct

下面的例程定义了一个名为Person的结构类。相比Collection，Struct的定义要多出有关结构描述的内容。

```php
use Combi\Core;

/**
 *
 * @property string $name
 * @property string $father
 * @property int $age
 */
class Person extends Core\Meta\Struct {
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

此例中，Person类中有```name, father, age```三个字段，其默认值分别为null、空串，以及18。

>   对Struct类来说，当尝试访问某个属性时，如果该属性事先未被设置，会返回默认值，如同已经被设置过那样。默认值设null，一般情况下表示你希望**该值必须由外部赋予**方可正确运作，这在之后的confirm功能介绍中也会起到作用。

作为推荐，由于Struct属性使用重载实现，因此中使用```@property```注释标注每个属性会让类的使用更加方便。

### 获取Struct的结构定义

```php
var_dump(Person::defaults());
```

以上代码会输出```$_defaults```属性内容。

## 结构迁移

Struct的结构迁移基于deprecated配置，迁移一个结构体的规则如下：

* 新增字段（属性）时，在```$_defaults```的数组末尾添加。
  * 不要重排```$_defaults```的定义顺序。
* 移除字段时，保留```$_defaults```中的字段定义，在```$_deprecated```中添加该字段。

```php
use Combi\Core;

/**
 *
 * @property string $name
 * @property int $age
 * @property int gender
 */
class Person extends Core\Meta\Struct {
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

```php
$me = new Person();
$me->set('father', 'unamed');
echo $me->get('father');
```

即使father字段已经被弃用，以上例程并不会报错。

但批量输出和赋值（稍后会提到的```fill()```方法）时，会跳过弃用字段。比如：

```php
$someone = new Person();
$someone
    ->set('name', 'triss')
    ->set('father', 'unamed')
    ->set('age', 50)
    ->set('gender', 2);
var_dump($someone->toArray());
```

上面例程不会输出father字段。

通过```defaults()```方法获取字段列表时，也会跳过deprecated字段。但下面方法依然可以获取获取废弃部分的全部字段：

```php
var_dump(Person::defaults(true));
```

该方法支持接收一个```bool $includeDeprecated```参数，用于返回全部字段结构。

>   要判断一个字段是否为弃用，可以使用```Person::isKeyDeprecated($key)```方法，返回类型为bool型。

## confirm机制

Struct提供了一套confirm机制，用于维护结构中数据规则。该机制通过显式调用触发。

```php
$someone = new Person();
$someone
    ->set('age', 50)
    ->set('gender', 2)
    ->confirm(); // will throw an exception
```

上面的代码会抛出一个违例“field [name] can not be empty”，因为并未给默认值为null的name字段赋值。

confirm的作用是当对Struct中的数据进行了一系列变更后，将要对其进行下一步处理时（比如模型持久化，比如发送消息，或是确认数据输出），对结构中数据完整性做一次校验。在默认情况下，仅会对默认值设置为null的字段检测是否有赋值。

> 类似defaults()，如需对deprecated字段生效，请调```confirm(true)```

> 类似toArray()，如果Struct中某些字段的值同样为一个Struct对象，这些对象的confirm()方法也会一并被调用。

### confirm勾子

可以自定义一些字段的校验规则，如下所示：

```php
use Combi\Core;

/**
 *
 * @property string $name
 * @property int $age
 * @property int gender
 */
class Person extends Core\Meta\Struct {
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
        !$val && $val = $_REQUEST['name'] ?? null;
        if (!$val) {
            throw new \LogicException('Person name is empty');
        }
        return $val;
    }
}
```

如范例所示，当confirm时name的昵称为null或是空串时，就会尝试从$_REQUEST中获取值。

> 使用自定义confirm勾子后，原有的null判定逻辑就不再有效，字段的可用性完全自行维护。

> confirm()勾子的机制可同时作为校验和过滤器使用。

### confirm整体勾子

Struct定义了一个空方法```afterConfirm()```，对于一些多字段相关联的判定，可以将相关处理逻辑复写在此方法中。

假设Person类有一个不算合理的逻辑：所有大于40岁的只能是男性（gender=1），那么可以这么写：

```php
use Combi\Core;

/**
 *
 * @property string $name
 * @property int $age
 * @property int gender
 */
class Person extends Core\Meta\Struct {
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
        !$val && $val = $_REQUEST['name'] ?? null;
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

## 允许null字段

在某些情况下，需要接收一些可选字段，这些字段如果未接收到，则为```null```，但又不需要为这些字段定义confirm勾子方法。

这时可以设置可选字段```static $_nullable```，设置为可选的字段在confirm时为```null```时**不报错**：

```php
use Combi\Core;

/**
 *
 * @property string $name
 * @property int $age
 * @property int gender
 */
class Person extends Core\Meta\Struct {
    protected static $_defaults = [
        'name'          => null,
        'father'        => '',
        'age'           => 18,
        'gender'        => 0,
        'is_married'    => null,
    ];

    protected static $_nullable = [
        'is_married'    => 1,
    ];
}
```

## 与Collection类似的方法

* set()
* get()
* has()
* remove()
* clear()
* all()
* count()

以上方法以及迭代器用法，与Collection基本相同。

> 其中```all()```方法与```defaults()```一样，可以接收一个```bool $includeDeprecated```参数，为true时将忽略deprecated设置返回全部字段数据。默认为false

> Struct基类不支持```push()```方法。

# Meta组件

Meta系统除了2个抽象类，还提供了一系列扩展组件，为数据承载提供便利。

## ArrayAccess

可以像数组一样访问对象容器中的属性。此trait正常动作需要使用的类```implement \ArrayAccess```接口。

```php
use Combi\Core;

/**
 *
 * @property string $name
 * @property int $age
 * @property int gender
 */
class Person extends Core\Meta\Struct implement \ArrayAccess {
    use Core\Meta\Extensions\ArrayAccess;

    // ...your code
}
```

使用时：

```php
$person = new Person;
$person['name'] = 'triss';
echo $person['name']; // print triss
```

## Fillable

实现了```fill()```方法，允许填充一组数组进Meta对象。所填充的变量可以是数组也可以是另一个Meta对象。

```php
use Combi\Core;

/**
 *
 * @property string $name
 * @property int $age
 * @property int gender
 */
class Person extends Core\Meta\Struct {
    use Core\Meta\Extensions\Fillable;
}

```

使用时：

```php

$person->fill([
    'name'      => 'triss',
    'gender'    => 2,
]);
```

在填充时可以指定哪些字段不允许被填充赋值。

```php
$person->exclude('gender', 'age')
    ->fill([
        'name'      => 'triss',
        'gender'    => 2,
        'age'       => 48,
    ]); // 这里只有name字段被填充
```

需要注意exclude()的设置为对象属性，在一次填充后不会被清除。如果要清除之前的设定，需要重新调用一次。

```php
$person->exclude('gender', 'age')
    ->fill([
        'name'      => 'triss',
        'gender'    => 2,
        'age'       => 48,
    ]) // 这里只有name字段被填充
    ->exclude();
```

## IteratorAggregate

提供基于```iterate()```方法的迭代器简化访问方式。Collection和Struct基类已经支持。

用法范例：

```php
foreach ($person as $key => $value) {
    echo "$key = $value\n";
}
```

## ToArray

提供```toArray()```方法。Collection与Struct基类已经支持。

使用此类时，建议同时```implement Combi\Core\Interfaces\Arrayable```。如果Meta对象中包括的字段同样实现了Arrayable接口，也会被调用转为数组。

```php
$arr = $person->toArray();
```

>   注意，此方法会移除值为null的字段。

## JsonSerializable

提供json序列化的相关方法。Collection和Struct基类已经支持。

用法：

```php
// 以下三行代码等同
var_dump(json_encode($person));
var_dump(json_encode($person->toArray()));
var_dump("$person");
```

## Overloaded

实现属性重载，方便访问属性。

```php
use Combi\Core;

/**
 *
 * @property string $name
 * @property int $age
 * @property int gender
 */
class Person extends Combi\Core\Struct {
    use Combi\Core\Extensions\Overloaded;
}


```

范例如下：

```php
// 以下两行代码等价
$person->set('name', 'triss');
$person->name = 'triss';

// 以下两行代码等价
var_dump($person->get('name'));
var_dump($person->name);

// 以下两行代码等价
var_dump($person->has('name'));
var_dump(isset($person->name));

// 以下两行代码等价
$person->remove('name');
unset($person->name);
```

## ToBin

提供```toBin()```方法，使用msgpack对将Meta对象的数据转为二进制。可以当成是toArray()的二进制版本。

```php
use Combi\Core;

/**
 *
 * @property string $name
 * @property int $age
 * @property int gender
 */
class Person extends Combi\Core\Struct {
    use Combi\Core\Extensions\ToBin;
}
```

范例：

```php
var_dump($person->toBin());
```


## Serializable

对象可序列化实现，目前仅针对Struct基类可用。对于对象序列化，serialzable是相对于```__wakeup()/__sleep()```机制更底层的对象序列化触发机制。

该trait支持将对象序列化，使用msgpack后的二进制数据保存主要数据，占用空间更少，并提供了简单的版本控制。

```php
use Combi\Core;

/**
 *
 * @property string $name
 * @property int $age
 * @property int gender
 */
class Person extends Combi\Core\Struct {
    use Combi\Core\Extensions\Serializable;
}
```

使用：

```php
$packed = serialize($person);
$person = unserialize($packed);
```

serializable提供了简单的版本控制，默认打包版本为1。如果需要升级版本，比如把版本号升到2，可以像下面这样通过复写方法实现。

```php
class Person extends Combi\Core\Struct {
    use Combi\Core\Extensions\Serializable;

    protected static function version() {
        return 2;
    }
}
```

当unserialize时检测到打包版本与当前版本不符，比如打包版本为1，会调用```renew()```方法进行处理。在默认情况下，renew方法不会对数据做任何兼容性处理。

在某些情况下，我们可能需要对老版本的打包数据进行一些转换，才能适应更新后的模型结构。除了业务逻辑上的需求外，另一个可能的原因是为了减小打包后的占用空间，serializable打包时会把结构的字段名去掉，如果前后模型的字段有过删减或调整（虽然建议不要这么做），那么还是需要代码对数据进行迁移的。

这里假设Person在version=1时有三个字段：

-   name = null
-   father = ''
-   gender = 1

之后因为某些原因，在version=2时father被移除了，增加了age字段在末尾，默认值是0：

-   name = null
-   gender = 1
-   age = 0

这时如果需要迁移打包数据，renew方法可能是这样：

```php
class Person extends Combi\Core\Struct {
    use Combi\Core\Extensions\Serializable;

    protected static function renew(array $data, $lastVersion) {
        if ($lastVersion == 1) {
            $data[1] = $data[2]; // 原gender位置移至father位置
            $data[2] = 0; // 原gender位置变为age，给默认值0
        }
        return $data;
    }
}
```

## Accessors

实现了Accessors存取器勾子。

```php
use Combi\Core;

/**
 *
 * @property string $name
 * @property int $age
 * @property int gender
 */
class Person extends Core\Meta\Struct {
    use Core\Meta\Extensions\Accessors;

    protected _get_age($value) {
        return $value > 40 ? '??' : $value;
    }

    protected _set_name($value) {
        $pos = strpos($value, ' ');
        return substr($value, 0, $pos);
    }
}
```

在上面的代码中，当调用```echo $person->age```时，会隐藏大于40岁人的真实年龄。而当调用```$person->name = 'Triss Merigold'```时，只会将```Triss```赋值给name字段。

## Configurable

待实现

## Mask

待实现
