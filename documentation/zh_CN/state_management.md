- [状态管理](#状态管理)
  - [响应式状态管理器](#响应式状态管理器)
    - [优势](#优势)
    - [最高性能](#最高性能)
    - [声明一个响应式变量](#声明一个响应式变量)
        - [有一个反应的状态，很容易。](#有一个反应的状态很容易)
    - [使用视图中的值](#使用视图中的值)
    - [什么时候重建](#什么时候重建)
    - [可以使用 .obs 的地方](#可以使用-obs-的地方)
    - [关于 List 的说明](#关于-list-的说明)
    - [为什么使用 .value](#为什么使用-value)
    - [Obx()](#obx)
    - [Workers](#workers)
  - [简单状态管理器](#简单状态管理器)
    - [优点](#优点)
    - [用法](#用法)
    - [如何处理 controller](#如何处理-controller)
    - [无需 StatefulWidgets](#无需-statefulwidgets)
    - [为什么它存在](#为什么它存在)
    - [其他使用方法](#其他使用方法)
    - [唯一的 ID](#唯一的-id)
  - [与其他状态管理器混用](#与其他状态管理器混用)
  - [GetBuilder vs GetX vs Obx vs MixinBuilder](#getbuilder-vs-getx-vs-obx-vs-mixinbuilder)

# 状态管理

目前，Flutter 有几种状态管理器。但是，它们中的大多数都涉及到使用 ChangeNotifier 来更新 widget，这对于中大型应用的性能来说是一个糟糕的方法。你可以在 Flutter 官方文档中查到，[ChangeNotifier 应该使用 1 个或最多 2 个监听器](https://api.flutter.dev/flutter/foundation/ChangeNotifier-class.html)，这使得它实际上无法用于任何中等或大型应用。

其他的状态管理器也不错，但有其细微的差别。

- BLoC 非常安全和高效，但是对于初学者来说非常复杂，这使得人们无法使用 Flutter 进行开发。
- MobX 比 BLoC 更容易，而且是响应式的，几乎是完美的，但是你需要使用一个代码生成器，对于大型应用来说，这降低了生产力，因为你需要喝很多咖啡，直到你的代码在 `flutter clean` 之后再次准备好 (这不是 MobX 的错，而是 codegen 真的很慢！)。
- Provider 使用 InheritedWidget 来传递相同的监听器，以此来解决上面报告的 ChangeNotifier 的问题，这意味着对其 ChangeNotifier 类的任何访问都必须在 widget 树内。

Get 并不是比任何其他状态管理器更好或更差，而是说你应该分析这些要点以及下面的要点来选择是只用 Get，还是与其他状态管理器结合使用。Get 不是其他状态管理器的敌人，因为 Get 是一个微框架，而不仅仅是一个状态管理器，它的状态管理功能既可以单独使用，也可以与其他状态管理器结合使用。

## 响应式状态管理器

响应式编程可能会让很多人感到陌生，因为它很复杂，但是 GetX 将响应式编程变得非常简单。

- 你不需要创建 StreamControllers。
- 你不需要为每个变量创建一个 StreamBuilder。
- 你不需要为每个状态创建一个类。
- 你不需要为一个初始值创建一个 get。

使用 Get 的响应式编程就像使用 setState 一样简单。

让我们想象一下，你有一个名称变量，并且希望每次你改变它时，所有使用它的小组件都会自动刷新。

这是你的计数变量：

```dart
var name = 'Jonatas Borges';
```

要想让它变得可观察，你只需要在它的末尾加上 “.obs”。

```dart
var name = 'Jonatas Borges'.obs;
```

就这么简单！

我们把这个 reactive-".obs" (ervables) 变量称为 _Rx_。

我们做了什么？我们创建了一个 “Stream” 的 “String”，分配了初始值 “Jonatas Borges”，我们通知所有使用 “Jonatas Borges” 的 widgets，它们现在 “属于” 这个变量，当 _Rx_ 的值发生变化时，它们也要随之改变。

这就是 GetX **的**魔力，这要归功于 Dart 的能力。

但是，我们知道，一个 `Widget` 只有在函数里面才能改变，因为静态类没有 “自动改变” 的能力。

你需要创建一个 `StreamBuilder`，订阅这个变量来监听变化，如果你想在同一个范围内改变几个变量，就需要创建一个 “级联” 的嵌套 `StreamBuilder`，对吧？

不，你不需要一个 `StreamBuilder`，但你对静态类的理解是对的。

在视图中，当我们想改变一个特定的 Widget 时，我们通常有很多 Flutter 方式的模板。
有了 **GetX**，你也可以忘记这些模板代码了。

`StreamBuilder( ... )`？`initialValue: ...`？`builder: ...`？不，你只需要把这个变量放在 `Obx()` 这个 Widget 里面就可以了。

```dart
Obx (() => Text (controller.name));
```

你只需记住 `Obx(()=>`

你只需将 Widget 通过一个箭头函数传递给 `Obx()` (_Rx_ 的 “观察者”)。

`Obx` 是相当聪明的，只有当 `controller.name` 的值发生变化时才会改变。

如果 `name` 是 `"John"`，你把它改成了 `"John"` (`name.value="John"`)，因为它和之前的 `value` 是一样的，所以界面上不会有任何变化，而 `Obx` 为了节省资源，会直接忽略新的值，不重建 Widget。**这是不是很神奇**？

> 那么，如果我在一个 `Obx` 里有 5 个 _Rx_ (可观察的) 变量呢？

当其中**任何**一个变量发生变化时，它就会更新。

> 如果我在一个类中有 30 个变量，当我更新其中一个变量时，它会更新该类中**所有**的变量吗？

不会，只会更新使用那个 _Rx_ 变量的**特定 Widget**。

所以，只有当 _Rx_ 变量的值发生变化时，**GetX** 才会更新界面。

```dart
final isOpen = false.obs;

// 什么都不会发生......相同的值。
void onButtonTap() => isOpen.value=false;
```
### 优势

**当你需要对更新的内容进行**精细的控制时，**GetX()** 可以帮助你。

如果你不需要 “unique IDs”，比如当你执行一个操作时，你的所有变量都会被修改，那么就使用 `GetBuilder`。
因为它是一个简单的状态更新器 (以块为单位，比如 `setState()`)，只用几行代码就能完成。
它做得很简单，对 CPU 的影响最小，只是为了完成一个单一的目的 (一个 _State_ Rebuild)，并尽可能地花费最少的资源。

如果你需要一个**强大的**状态管理器，用 **GetX** 是不会错的。

它不能和变量一起工作，除了 __flows__，它里面的东西本质都是 `Streams`。
你可以将 _rxDart_ 与它结合使用，因为所有的东西都是 `Streams`。
你可以监听每个 “_Rx_ 变量” 的 “事件”。
因为里面的所有东西都是 “Streams”。

这实际上是一种 _BLoC_ 方法，比 _MobX_ 更容易，而且没有代码生成器或装饰。
你可以把**任何东西**变成一个 _“Observable”_，只需要在它末尾加上 `.obs`。

### 最高性能

除了有一个智能的算法来实现最小化的重建，**GetX** 还使用了比较器以确保状态已经改变。

如果你的应用程序中遇到错误，并发送重复的状态变更，**GetX** 将确保它不会崩溃。

使用 **GetX**，只有当 `value` 改变时，状态才会改变。
这就是 **GetX**，和使用 MobX _的_ `computed` 的主要区别。
当加入两个 __observable__，其中一个发生变化时，该 _observable_ 的监听器也会发生变化。

使用 **GetX**，如果你连接了两个变量，`GetX()` (类似于 `Observer()`) 只有在它的状态真正变化时才会重建。

### 声明一个响应式变量

你有 3 种方法可以把一个变量变成是 “可观察的”。


1 - 第一种是使用 **`Rx{Type}`**。

```dart
// 建议使用初始值，但不是强制性的
final name = RxString('');
final isLogged = RxBool(false);
final count = RxInt(0);
final balance = RxDouble(0.0);
final items = RxList<String>([]);
final myMap = RxMap<String, int>({});
```

2 - 第二种是使用 **`Rx`**，规定泛型 `Rx<Type>`。

```dart
final name = Rx<String>('');
final isLogged = Rx<Bool>(false);
final count = Rx<Int>(0);
final balance = Rx<Double>(0.0);
final number = Rx<Num>(0)
final items = Rx<List<String>>([]);
final myMap = Rx<Map<String, int>>({});

// 自定义类 - 可以是任何类
final user = Rx<User>();
```

3 - 第三种更实用、更简单、更可取的方法，只需添加 **`.obs`** 作为 `value` 的属性。

```dart
final name = ''.obs;
final isLogged = false.obs;
final count = 0.obs;
final balance = 0.0.obs;
final number = 0.obs;
final items = <String>[].obs;
final myMap = <String, int>{}.obs;

// 自定义类 - 可以是任何类
final user = User().obs;
```

##### 有一个反应的状态，很容易。

我们知道，_Dart_ 现在正朝着 _null safety_ 的方向发展。
为了做好准备，从现在开始，你应该总是用一个**初始值**来开始你的 _Rx_ 变量。

> 用 **GetX** 将一个变量转化为一个 _observable_ + _initial value_ 是最简单，也是最实用的方法。

你只需在变量的末尾添加一个 “`.obs`”，即可把它变成可观察的变量，
然后它的 `.value` 就是_初始值_ )。


### 使用视图中的值

```dart
// controller
final count1 = 0.obs;
final count2 = 0.obs;
int get sum => count1.value + count2.value;
```

```dart
// 视图
GetX<Controller>(
  builder: (controller) {
    print("count 1 rebuild");
    return Text('${controller.count1.value}');
  },
),
GetX<Controller>(
  builder: (controller) {
    print("count 2 rebuild");
    return Text('${controller.count2.value}');
  },
),
GetX<Controller>(
  builder: (controller) {
    print("count 3 rebuild");
    return Text('${controller.sum}');
  },
),
```

如果我们把 `count1.value++` 递增，就会打印出来：
- `count 1 rebuild`
- `count 3 rebuild`


如果我们改变 `count2.value++`，就会打印出来。
- `count 2 rebuild`
- `count 3 rebuild`

因为 `count2.value` 改变了，`sum` 的结果现在是 `2`。

- 注意：默认情况下，第一个事件将重建小组件，即使是相同的 `值`。
 这种行为是由于布尔变量而存在的。

想象一下你这样做。

```dart
var isLogged = false.obs;
```

然后，你检查用户是否 “登录”，以触发 `ever` 的事件。

```dart
@override
onInit(){
  ever(isLogged, fireRoute);
  isLogged.value = await Preferences.hasToken();
}

fireRoute(logged) {
  if (logged) {
   Get.off(Home());
  } else {
   Get.off(Login());
  }
}
```

如果 “hasToken” 是 “false”，“isLogged” 就不会有任何变化，所以 “ever()” 永远不会被调用。
为了避免这种问题，_observable_ 的第一次变化将总是触发一个事件，即使它包含相同的 `.value`。

如果你想删除这种行为，你可以使用：
`isLogged.firstRebuild = false;`。

### 什么时候重建

此外，Get 还提供了精细的状态控制。你可以根据特定的条件对一个事件进行条件控制 (比如将一个对象添加到 List 中)。

```dart
// 第一个参数：条件，必须返回 true 或 false。
// 第二个参数：如果条件为真，则为新的值。
list.addIf(item < limit, item);
```

没有装饰，没有代码生成器，没有复杂的程序：爽歪歪！

你知道 Flutter 的计数器应用吗？你的 Controller 类可能是这样的。

```dart
class CountController extends GetxController {
  final count = 0.obs;
}
```

用一个简单的。

```dart
controller.count.value++
```

你可以在你的 UI 中更新计数器变量，不管它存储在哪里。

### 可以使用 .obs 的地方

你可以在 obs 上转换任何东西，这里有两种方法：

* 可以将你的类值转换为 obs
```dart
class RxUser {
  final name = "Camila".obs;
  final age = 18.obs;
}
```

* 或者可以将整个类转换为一个可观察的类。
```dart
class User {
  User({String name, int age});
  var name;
  var age;
}

// 实例化时。
final user = User(name: "Camila", age: 18).obs;
```

### 关于 List 的说明

List 和它里面的对象一样，是完全可以观察的。这样一来，如果你在 List 中添加一个值，它会自动重建使用它的 widget。

你也不需要在 List 中使用 “.value”，神奇的 dart api 允许我们删除它。
不幸的是，像 String 和 int 这样的原始类型不能被继承，使得 .value 的使用是强制性的，但是如果你使用 get 和 setter 来处理这些类型，这将不是一个问题。

```dart
// controller
final String title = 'User Info:'.obs
final list = List<User>().obs;

// view
Text(controller.title.value), // 字符串后面需要有 .value。
ListView.builder (
  itemCount: controller.list.length // List 不需要它
)
```

当你在使自己的类可观察时，有另外一种方式来更新它们：

```dart
// model
// 我们将使整个类成为可观察的，而不是每个属性。
class User{
    User({this.name = '', this.age = 0});
    String name;
    int age;
}

// controller
final user = User().obs;
// 当你需要更新user变量时。
user.update( (user) { // 这个参数是你要更新的类本身。
    user.name = 'Jonny';
    user.age = 18;
});
// 更新user变量的另一种方式。
user(User(name: 'João', age: 35));

// view
Obx(()=> Text("Name ${user.value.name}: Age: ${user.value.age}"));
// 你也可以不使用 .value 来访问模型值。
user().name; // 注意是 user 变量，而不是类变量（首字母是小写的）。
```

你可以使用 “assign” 和 “assignAll”。
“assign” 会清除你的 List，并添加一个单个对象。
“assignAll” 将清除现有的 List，并添加任何可迭代对象。

### 为什么使用 .value

我们可以通过一个简单的装饰和代码生成器来消除使用 “String” 和 “int” 的值的义务，但这个库的目的正是为了避免外部依赖。我们希望提供一个准备好的编程的环境 (路由、依赖和状态的管理)，以一种简单、轻量级和高性能的方式，而不需要一个外部包。

你可以在你的 pubspec 中添加 3 个字母 (get) 和一个冒号，然后开始编程。从路由管理到状态管理，所有的解决方案都是默认的，目的是为了方便、高效和高性能。

这个库的总大小还不如一个状态管理器，尽管它是一个完整的解决方案。

如果你被 `.value` 困扰，喜欢代码生成器，MobX 是一个很好的选择，也可以和 Get 一起使用。对于那些想在 pubspec 中添加一个单一的依赖，然后开始编程，而不用担心一个包的版本与另一个包不兼容，也不用担心状态更新的错误来自于状态管理器或依赖，还是不想担心控制器的可用性，Get 都是刚刚好。

如果你对 MobX 代码生成器或者 BLoC 模板熟悉，你可以直接用 Get 来做路由，而忘记它有状态管理器。如果有一个项目有 90 多个控制器，MobX 代码生成器在标准机器上进行 Flutter Clean 后，需要 30 多分钟才能完成任务。如果你的项目有 5 个、10 个、15 个控制器，任何一个状态管理器都能很好的满足你。如果你的项目大得离谱，代码生成器对你来说是个问题，但你已经获得了 Get 这个解决方案。

显然，如果有人想为项目做贡献，创建一个代码生成器，或者类似的东西，我将在这个 readme 中链接，作为一个替代方案，我的需求并不是所有开发者的需求，但现在我要说的是，已经有很好的解决方案，比如 MobX。

### Obx()

在 Get 中使用 Bindings 的类型是不必要的。你可以使用 Obx widget 代替 GetX，GetX 只接收创建 widget 的匿名函数。
如果你不使用类型，你将需要有一个控制器的实例来使用变量，或者使用 `Get.find<Controller>()`.value 或 Controller.to.value 来检索值。

### Workers

Workers 将协助你在事件发生时触发特定的回调。

```dart
// 每次`count1` 变化时调用。
ever(count1, (_) => print("$_ has been changed"));

// 只有在变量 $_ 第一次被改变时才会被调用。
once(count1, (_) => print("$_ was changed once"));

// 防 DDos - 每当用户停止输入1秒时调用，例如。
debounce(count1, (_) => print("debouce$_"), time: Duration(seconds: 1));

// 忽略 1 秒内的所有变化。
interval(count1, (_) => print("interval $_"), time: Duration(seconds: 1));
```
所有 Workers (除 “debounce” 外) 都有一个名为 “condition” 的参数，它可以是一个 “bool” 或一个返回 “bool” 的回调。
这个 `condition` 定义了 `callback` 函数何时执行。

所有 worker 都会返回一个 `Worker` 实例，你可以用它来取消 (通过 `dispose()`) worker。

- **`ever`**
 每当 _Rx_ 变量发出一个新的值时，就会被调用。

- **`everAll`**
 和 “ever” 很像，但它需要一个 _Rx_ 值的 “List”，每次它的变量被改变时都会被调用。就是这样。


- **`once`**
‘once’ 只在变量第一次被改变时被调用。

- **`debounce`**
‘debounce’ 在搜索函数中非常有用，你只希望 API 在用户完成输入时被调用。如果用户输入 “Jonny”，你将在 API 中进行 5 次搜索，分别是字母 J、o、n、n 和 y。使用 Get 不会发生这种情况，因为你将有一个 “debounce” Worker，它只会在输入结束时触发。

- **`interval`**
‘interval’ 与 debouce 不同，debouce 如果用户在1秒内对一个变量进行了 1000 次修改，他将在规定的计时器 (默认为 800 毫秒) 后只发送最后一次修改。Interval 则会忽略规定时间内的所有用户操作。如果你发送事件1分钟，每秒 1000 个，那么当用户停止 DDOS 事件时，debounce 将只发送最后一个事件。建议这样做是为了避免滥用，在用户可以快速点击某样东西并获得一些好处的功能中 (想象一下，用户点击某样东西可以赚取硬币，如果他在同一分钟内点击 300 次，他就会有 300 个硬币，使用间隔，你可以设置时间范围为3秒，无论是点击 300 次或 100 万次，1分钟内他最多获得 20 个硬币)。debounce 适用于防 DDOS，适用于搜索等功能，每次改变 onChange 都会调用你的 api 进行查询。Debounce 会等待用户停止输入名称，进行请求。如果在上面提到的投币场景中使用它，用户只会赢得 1 个硬币，因为只有当用户 “暂停” 到既定时间时，它才会被执行。

- 注意：Worker 应该总是在启动 Controller 或 Class 时使用，所以应该总是在 onInit (推荐)、Class 构造函数或 StatefulWidget 的 initState (大多数情况下不推荐这种做法，但应该不会有任何副作用)。

## 简单状态管理器

Get 有一个极其轻巧简单的状态管理器，它不使用 ChangeNotifier，可以满足特别是对 Flutter 新手的需求，而且不会给大型应用带来问题。

GetBuilder 正是针对多状态控制的。想象一下，你在购物车中添加了 30 个产品，你点击删除一个，同时 List 更新了，价格更新了，购物车中的徽章也更新为更小的数字。这种类型的方法使 GetBuilder 成为杀手锏，因为它将状态分组并一次性改变，而无需为此进行任何 “计算逻辑”。GetBuilder 就是考虑到这种情况而创建的，因为对于短暂的状态变化，你可以使用 setState，而不需要状态管理器。

这样一来，如果你想要一个单独的控制器，你可以为其分配 ID，或者使用 GetX。这取决于你，记住你有越多的 “单独” 部件，GetX 的性能就越突出，而当有多个状态变化时，GetBuilder 的性能应该更优越。

### 优点

1. 只更新需要的小部件。

2. 不使用 changeNotifier，状态管理器使用较少的内存 (接近 0mb)。

3. 忘掉 StatefulWidget！使用 Get 你永远不会需要它。对于其他的状态管理器，你可能需要使用 StatefulWidget 来获取你的 Provider、BLoC、MobX 控制器等的实例。但是你有没有停下来想一想，你的 appBar，你的脚手架，以及你的类中的大部分 widget 都是无状态的？那么如果你只能保存有状态的 Widget 的状态，为什么要保存整个类的状态呢？Get 也解决了这个问题。创建一个无状态类，让一切都成为无状态。如果你需要更新单个组件，就用 GetBuilder 把它包起来，它的状态就会被维护。

4. 真正的解耦你的项目！控制器一定不要在你的 UI 中，把你的 TextEditController，或者你使用的任何控制器放在你的 Controller 类中。

5. 你是否需要触发一个事件来更新一个 widget，一旦它被渲染？GetBuilder 有一个属性 “initState”，就像 StatefulWidget 一样，你可以从你的控制器中调用事件，直接从控制器中调用，不需要再在你的 initState 中放置事件。

6. 你是否需要触发一个动作，比如关闭流、定时器等？GetBuilder 也有 dispose 属性，只要该 widget 被销毁，你就可以调用事件。

7. 仅在必要时使用流。你可以在你的控制器里面正常使用你的 StreamControllers，也可以正常使用 StreamBuilder，但是请记住，一个流消耗合理的内存，响应式编程很美，但是你不应该滥用它。30 个流同时打开会比 changeNotifier 更糟糕 (而且 changeNotifier 非常糟糕)。

8. 更新 widgets 而不需要为此花费 ram。Get 只存储 GetBuilder 的创建者 ID，必要时更新该 GetBuilder。get ID 存储在内存中的消耗非常低，即使是成千上万的 GetBuilders。当你创建一个新的 GetBuilder 时，你实际上是在共享拥有创建者 ID 的 GetBuilder 的状态。不会为每个 GetBuilder 创建一个新的状态，这为大型应用节省了大量的内存。基本上你的应用程序将是完全无状态的，而少数有状态的 Widgets (在 GetBuilder 内) 将有一个单一的状态，因此更新一个状态将更新所有的状态。状态只是一个。

9. Get 是全知全能的，在大多数情况下，它很清楚地知道从内存中取出一个控制器的时机，你不需要担心什么时候移除一个控制器，Get 知道最佳的时机。

### 用法

```dart
// 创建控制器类并继承 GetxController。
class Controller extends GetxController {
  int counter = 0;
  void increment() {
    counter++;
    update(); // 当调用增量时，使用 update() 来更新用户界面上的计数器变量。
  }
}
// 在你的 Stateless/Stateful 类中，当调用 increment 时，使用 GetBuilder 来更新 Text。
GetBuilder<Controller>(
  init: Controller(), // 首次启动
  builder: (_) => Text(
    '${_.counter}',
  ),
)
// 只在第一次时初始化你的控制器。第二次使用 ReBuilder 时，不要再使用同一控制器。一旦将控制器标记为 "init" 的部件部署完毕，你的控制器将自动从内存中移除。你不必担心这个问题，Get 会自动做到这一点，只是要确保你不要两次启动同一个控制器。
```

**完成！**

- 你已经学会了如何使用 Get 管理状态。

- 注意：你可能想要一个更大的规模，而不是使用 init 属性。为此，你可以创建一个类并继承 Bindings 类，并在其中添加将在该路由中创建的控制器。控制器不会在那个时候被创建，相反，这只是一个声明，这样你第一次使用 Controller 时，Get 就会知道去哪里找。Get 会保持懒加载，当不再需要 Controller 时，会自动移除它们。请看 pub.dev 的例子来了解它是如何工作的。

如果你导航了很多路由，并且需要之前使用的 Controller 中的数据，你只需要再用一次 GetBuilder (没有 init)。

```dart
class OtherClass extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: Center(
        child: GetBuilder<Controller>(
          builder: (s) => Text('${s.counter}'),
        ),
      ),
    );
  }

```

如果你需要在许多其他地方使用你的控制器，并且在 GetBuilder 之外，只需在你的控制器中创建一个 get，就可以轻松地拥有它。(或者使用 `Get.find<Controller>()`)

```dart
class Controller extends GetxController {

  // 你不需要这个，我推荐使用它只是为了方便语法。
  // 用静态方法：Controller.to.increment()。
  // 没有静态方法的情况下：Get.find<Controller>().increment();
  // 使用这两种语法在性能上没有区别，也没有任何副作用。一个不需要类型，另一个 IDE 会自动完成。
  static Controller get to => Get.find(); // 添加这一行

  int counter = 0;
  void increment() {
    counter++;
    update();
  }
}
```

然后你可以直接访问你的控制器，这样：

```dart
FloatingActionButton(
  onPressed: () {
    Controller.to.increment(),
  } // 是不是贼简单！
  child: Text("${Controller.to.counter}"),
),
```

当你按下 FloatingActionButton 时，所有监听 ‘counter’ 变量的 widget 都会自动更新。

### 如何处理 controller

比方说，我们有这样的情况。

`Class a => Class B (has controller X) => Class C (has controller X)`

在 A 类中，控制器还没有进入内存，因为你还没有使用它 (Get 是懒加载)。在类 B 中，你使用了控制器，并且它进入了内存。在 C 类中，你使用了与 B 类相同的控制器，Get 会将控制器 B 的状态与控制器 C 共享，同一个控制器还在内存中。如果你关闭 C 屏和 B 屏，Get 会自动将控制器 X 从内存中移除，释放资源，因为 a 类没有使用该控制器。如果你再次导航到 B，控制器 X 将再次进入内存，如果你没有去 C 类，而是再次回到 a 类，Get 将以同样的方式将控制器从内存中移除。如果类 C 没有使用控制器，你把类 B 从内存中移除，就没有类在使用控制器 X，同样也会被处理掉。唯一能让 Get 乱了阵脚的例外情况，是如果你意外地从路由中删除了 B，并试图使用 C 中的控制器，在这种情况下，B 中的控制器的创建者 ID 被删除了，Get 被设计为从内存中删除每一个没有创建者 ID 的控制器。如果你打算这样做，在 B 类的 GetBuilder 中添加 “autoRemove：false” 标志，并在 C 类的 GetBuilder 中使用 adopID = true；

### 无需 StatefulWidgets

使用 StatefulWidgets 意味着不必要地存储整个界面的状态，甚至因为如果你需要最小化地重建一个 widget，你会把它嵌入一个 Consumer/Observer/BlocProvider/GetBuilder/GetX/Obx 中，这将是另一个 StatefulWidget。
StatefulWidget 类是一个比 StatelessWidget 大的类，它将分配更多的 RAM，只使用一两个类之间可能不会有明显的区别，但当你有 100 个类时，它肯定会有区别！
除非你需要使用混合器，比如 TickerProviderStateMixin，否则完全没有必要使用 StatefulWidget 与 Get。

你可以直接从 GetBuilder 中调用 StatefulWidget 的所有方法。
例如，如果你需要调用 initState() 或 dispose() 方法，你可以直接调用它们。

```dart
GetBuilder<Controller>(
  initState: (_) => Controller.to.fetchApi(),
  dispose: (_) => Controller.to.closeStreams(),
  builder: (s) => Text('${s.username}'),
),
```

比这更好的方法是直接从控制器中使用 onInit() 和 onClose() 方法。

```dart
@override
void onInit() {
  fetchApi();
  super.onInit();
}
```

- 注意：如果你想在控制器第一次被调用的那一刻启动一个方法，你不需要为此使用构造函数，使用像 Get 这样面向性能的包，这样做反而是糟糕的做法，因为它偏离了控制器被创建或分配的逻辑 (如果你创建了这个控制器的实例，构造函数会立即被调用，你会在控制器还没有被使用之前就填充了一个控制器，你在没有被使用的情况下就分配了内存，这绝对违背这个库的原则)。onInit()；和 onClose()；方法就是为此而创建的，它们会在 Controller 被创建，或者第一次使用时被调用，这取决于你是否使用 Get.lazyPut。例如，如果你想调用你的 API 来填充数据，你可以忘掉老式的 initState/dispose 方法，只需在 onInit 中开始调用 api，如果你需要执行任何命令，如关闭流，使用 onClose() 来实现。

### 为什么它存在

这个包的目的正是为了给你提供一个完整的解决方案，用于导航路线，管理依赖和状态，使用尽可能少的依赖，高度解耦。Get 将所有高低级别的 Flutter API 都纳入自身，以确保你在工作中尽可能减少耦合。我们将所有的东西集中在一个包中，以确保你在你的项目中没有任何形式的耦合。这样一来，你就可以只在视图中放置小组件，而让你的团队中负责业务逻辑的那部分人自由地工作，不需要依赖视图中的任何元素来处理业务逻辑。这样就提供了一个更加干净的工作环境，这样你的团队中的一部分人只用 widget 工作，而不用担心将数据发送到你的 controller，你的团队中的一部分人只用业务逻辑工作，而不依赖于视图的任何元素。

所以为了简化这个问题。
你不需要在 initState 中调用方法，然后通过参数发送给你的控制器，也不需要使用你的控制器构造函数，你有 onInit() 方法，在合适的时间被调用，以启动你的服务。
你不需要调用设备，你有 onClose() 方法，它将在确切的时刻被调用，当你的控制器不再需要时，将从内存中删除。这样一来，只给 widget 留下视图，不要从中进行任何形式的业务逻辑。

不要在 GetxController 里面调用 dispose 方法，它不会有任何作用，记住控制器不是 Widget，你不应该 “dispose” 它，它会被 Get 自动智能地从内存中删除。如果你在上面使用了任何流，想关闭它，只要把它插入到 close 方法中就可以了。例如

```dart
class Controller extends GetxController {
  StreamController<User> user = StreamController<User>();
  StreamController<String> name = StreamController<String>();

  // 关闭流用onClose方法，而不是dispose
  @override
  void onClose() {
    user.close();
    name.close();
    super.onClose();
  }
}
```

控制器的生命周期。

- onInit() 是创建控制器的地方。
- onClose()，关闭控制器，为删除方法做准备。
- deleted：你不能访问这个 API，因为它实际上是将控制器从内存中删除。它真的被删除了，不留任何痕迹。

### 其他使用方法

你可以直接在 GetBuilder 值上使用 Controller 实例。

```dart
GetBuilder<Controller>(
  init: Controller(),
  builder: (value) => Text(
    '${value.counter}',  // 这里
  ),
),
```

你可能还需要在 GetBuilder 之外的控制器实例，你可以使用这些方法来实现。

```dart
class Controller extends GetxController {
  static Controller get to => Get.find();
[...]
}
//view
GetBuilder<Controller>(  
  init: Controller(), // 每个控制器只用一次
  builder: (_) => Text(
    '${Controller.to.counter}', //这里
  )
),
```

或者

```dart
class Controller extends GetxController {
 // static Controller get to => Get.find(); // with no static get
[...]
}
// on stateful/stateless class
GetBuilder<Controller>(  
  init: Controller(), // 每个控制器只用一次
  builder: (_) => Text(
    '${Get.find<Controller>().counter}', // 这里
  ),
),
```

- 你可以使用 “非规范” 的方法来做这件事。如果你正在使用一些其他的依赖管理器，比如 get_it、modular 等，并且只想交付控制器实例，你可以这样做。

```dart
Controller controller = Controller();
[...]
GetBuilder<Controller>(
  init: controller, // 这里
  builder: (_) => Text(
    '${controller.counter}', // 这里
  ),
),

```

### 唯一的 ID

如果你想用 GetBuilder 完善一个 widget 的更新控件，你可以给它们分配唯一的 ID。

```dart
GetBuilder<Controller>(
  id: 'text', // 这里
  init: Controller(), // 每个控制器只用一次
  builder: (_) => Text(
    '${Get.find<Controller>().counter}', // 这里
  ),
),
```

并更新它：

```dart
update(['text']);
```

您还可以为更新设置条件。

```dart
update(['text'], counter < 10);
```

GetX 会自动进行重建，并且只重建使用被更改的变量的小组件，如果您将一个变量更改为与之前相同的变量，并且不意味着状态的更改，GetX 不会重建小组件以节省内存和 CPU 周期 (界面上正在显示 3，而您再次将变量更改为 3。在大多数状态管理器中，这将导致一个新的重建，但在 GetX 中，如果事实上他的状态已经改变，那么 widget 将只被再次重建)

## 与其他状态管理器混用

有人开了一个功能请求，因为他们只想使用一种类型的响应式变量，而其他的则手动去更新，需要为此在 GetBuilder 中插入一个 Obx。思来想去，MixinBuilder 应运而生。它既可以通过改变 “.obs” 变量进行响应式改变，也可以通过 update() 进行手动更新。然而，在 4 个 widget 中，他是消耗资源最多的一个，因为除了有一个 Subscription 来接收来自他的子代的变化事件外，他还订阅了他的控制器的 update 方法。

继承 GetxController 是很重要的，因为它们有生命周期，可以在它们的 onInit() 和 onClose() 方法中 “开始” 和 “结束” 事件。你可以使用任何类来实现这一点，但我强烈建议你使用 GetxController 类来放置你的变量，无论它们是否是可观察的。


## GetBuilder vs GetX vs Obx vs MixinBuilder

在十年的编程工作中，我能够学到一些宝贵的经验。

我第一次接触到响应式编程的时候，是那么的 “哇，这太不可思议了”，事实上响应式编程是不可思议的。
但是，它并不适合所有情况。通常情况下，你需要的是同时改变 2、3 个 widget 的状态，或者是短暂的状态变化，这种情况下，响应式编程不是不好，而是不适合。

响应式编程对 RAM 的消耗比较大，可以通过单独的工作流来弥补，这样可以保证只有一个 widget 被重建，而且是在必要的时候，但是创建一个有 80 个对象的 List，每个对象都有几个流，这不是一个好的想法。打开 dart inspect，查看一个 StreamBuilder 的消耗量，你就会明白我想告诉你什么。

考虑到这一点，我创建了简单的状态管理器。它很简单，这正是你应该对它提出的要求：以一种简单的方式，并且以最高效的方式更新块中的状态。

GetBuilder 在 RAM 中是非常高效的，几乎没有比他更高效的方法 (至少我无法想象，如果存在，请告诉我们)。

然而，GetBuilder 仍然是一个手动的状态管理器，你需要调用 update()，就像你需要调用 Provider 的 notifyListeners() 一样。

还有一些情况下，响应式编程真的很有趣，不使用它就等于重新发明轮子。考虑到这一点，GetX 的创建是为了提供状态管理器中最现代和先进的一切。它只在必要的时候更新必要的东西，如果你出现了错误，同时发送了 300 个状态变化，GetX 只在状态真正发生变化时才会过滤并更新界面。

GetX 比其他响应式状态管理器还是比较高效的，但它比 GetBuilder 多消耗一点内存。思前想后，以最大限度地消耗资源为目标，Obx 应运而生。与 GetX 和 GetBuilder 不同的是，你将无法在 Obx 内部初始化一个控制器，它只是一个带有 StreamSubscription 的 Widget，接收来自你的子代的变化事件，仅此而已。它比 GetX 更高效，但输给了 GetBuilder，这是意料之中的，因为它是响应式的，而且 GetBuilder 有最简单的方法，即存储 widget 的 hashcode 和它的 StateSetter。使用 Obx，你不需要写你的控制器类型，你可以从多个不同的控制器中监听到变化，但它需要在之前进行初始化，或者使用本 readme 开头的示例方法，或者使用 Bindings 类。
