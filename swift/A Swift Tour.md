[TOC]

# About Swift

- 变量一定是在使用前初始化的；
- 数组索引会检查越界错误；
- 整数会检查溢出；
- 可选项保证了 nil 值会显式处理；
- 内存自动管理；
- 错误处理允许从意外错误中恢复控制。

# Simple Value

- 在全局范围内编写的代码用作程序的入口点，不用写在主函数中，也不用再写分号

- `var`变量，`let`常量

- swift中值不能隐式转换，必须显示地声明它需要转换的数据类型

- 在字符串中可以通过`\()`的方式包涵value，支持运算操作

- 用`"""`将要占多行的字符串包括起来

- 用方括号`[]`创建数组和字典，并通过在方括号中写入索引或键来访问它们的元素。最后一个元素之后允许使用逗号。

  ```swift
  var shoppingList = ["catfish", "water", "tulips"]
  shoppingList[1] = "bottle of water"
  
  var occupations = [
      "Malcolm": "Captain",
      "Kaylee": "Mechanic",
  ]
  occupations["Jayne"] = "Public Relations"
  ```

- 数组都是可变类型

- 数组和字典的初始化格式：

  ```swift
  let emptyArray = [String]()
  let emptyDictionary = [String: Float]()
  ```

  如果可以推断类型信息，可以将空数组写成`[]`，将空字典写成`[:]`，例如，当您为变量设置新值或将参数传递给函数时。

  ```swift
  shoppingList = []
  occupations = [:]
  ```

# Control Flow

- 条件或循环变量的括号是可选的，但body的中括号是必需的。

  ```swift
  let individualScores = [75, 43, 103, 87, 12]
  var teamScore = 0
  for score in individualScores {
      if score > 50 {
          teamScore += 3
      } else {
          teamScore += 1
      }
  }
  print(teamScore)
  // Prints "11"
  ```

- if内的条件判断类型必须是`bool`类型

- 可以使用if和let一起使用可能缺少的值。这些值表示为可选值。可选值包含一个值或包含nil来指示缺少一个值。在值的类型后写一个问号（？），以将该值标记为可选。

  ```swift
  var optionalString: String? = "Hello"
  print(optionalString == nil)
  // Prints "false"
  
  var optionalName: String? = "John Appleseed"
  var greeting = "Hello!"
  if let name = optionalName {
      greeting = "Hello, \(name)"
  }
  ```

- 另一种处理可选值的方法是使用 ?? 运算符提供默认值。如果可选值丢失，默认值就会使用。

  ```swift
  let nickName: String? = nil
  let fullName: String = "John Appleseed"
  let informalGreeting = "Hi \(nickName ?? fullName)"
  ```

- Switch 选择语句支持任意类型的数据和各种类型的比较操作——它不再限制于整型和测试相等上。

  在执行完 switch 语句里匹配到的 case 之后，程序就会从 switch 语句中退出。执行并不会继续跳到下一个 case 里，所以完全没有必要显式地在每一个 case 后都标记 break 。

  ```swift
  let vegetable = "red pepper"
  switch vegetable {
  case "celery":
      print("Add some raisins and make ants on a log.")
  case "cucumber", "watercress":
      print("That would make a good tea sandwich.")
  case let x where x.hasSuffix("pepper"):
      print("Is it a spicy \(x)?")
  default:
      print("Everything tastes good in soup.")
  }
  ```

  

- 你可以使用 for-in来遍历字典中的项目，这需要提供一对变量名来储存键值对。字典使用无序集合，所以键值的遍历也是无序的。

  ```swift
  let interestingNumbers = [
      "Prime": [2, 3, 5, 7, 11, 13],
      "Fibonacci": [1, 1, 2, 3, 5, 8],
      "Square": [1, 4, 9, 16, 25],
  ]
  var largest = 0
  for (kind, numbers) in interestingNumbers {
      for number in numbers {
          if number > largest {
              largest = number
          }
      }
  }
  print(largest)
  ```

- 循环的条件可以放在末尾，这样可以保证循环至少运行了一次。

  ```swift
  var n = 2
  while n < 100 {
      n = n * 2
  }
  print(n)
   
  var m = 2
  repeat {
      m = m * 2
  } while m < 100
  print(m)
  ```

- 可以使用 `..<`来创造一个序列区间：

  ```swift
  var total = 0
  for i in 0..<4 {
      total += i
  }
  print(total)
  ```

  使用 `..<`来创建一个不包含最大值的区间，使用` ...` 来创造一个包含最大值和最小值的区间。

# Functions and Closures

- 使用 func来声明一个函数。通过在名字之后在圆括号内添加一系列参数来调用这个方法。使用 ->来分隔形式参数名字类型和函数返回的类型。

  ```swift
  func greet(person: String, day: String) -> String {
      return "Hello \(person), today is \(day)."
  }
  greet(person: "Bob", day: "Tuesday")
  ```

- 默认情况下，函数使用其参数名称作为其参数的标签。在参数名称前写一个自定义参数标签，或写_以不使用任何参数标签。

  ```swift
  func greet(_ person: String, on day: String) -> String {
      return "Hello \(person), today is \(day)."
  }
  greet("John", on: "Wednesday")
  ```

- 使用元组(tuple)生成复合值，例如，从函数返回多个值。元组的元素可以通过名称或数字来引用。

  ```swift
  func calculateStatistics(scores: [Int]) -> (min: Int, max: Int, sum: Int) {
      var min = scores[0]
      var max = scores[0]
      var sum = 0
  
      for score in scores {
          if score > max {
              max = score
          } else if score < min {
              min = score
          }
          sum += score
      }
  
      return (min, max, sum)
  }
  let statistics = calculateStatistics(scores: [5, 3, 100, 3, 9])
  print(statistics.sum)
  // Prints "120"
  print(statistics.2)
  // Prints "120"
  ```

- 函数可以嵌套。嵌套函数可以访问在外部函数中声明的变量。您可以使用嵌套函数将代码组织为长函数或复杂函数。

  ```swift
  func returnFifteen() -> Int {
      var y = 10
      func add() {
          y += 5
      }
      add()
      return y
  }
  returnFifteen()
  ```

  一个函数可以返回另一个函数作为其值。

  ```swift
  func makeIncrementer() -> ((Int) -> Int) {
      func addOne(number: Int) -> Int {
          return 1 + number
      }
      return addOne
  }
  var increment = makeIncrementer()
  increment(7)
  ```

  一个函数可以将另一个函数作为其参数之一。

  ```swift
  func hasAnyMatches(list: [Int], condition: (Int) -> Bool) -> Bool {
      for item in list {
          if condition(item) {
              return true
          }
      }
      return false
  }
  func lessThanTen(number: Int) -> Bool {
      return number < 10
  }
  var numbers = [20, 19, 7, 12]
  hasAnyMatches(list: numbers, condition: lessThanTen)
  ```

- 函数实际上是闭包的一种特殊情况：可以稍后调用的代码块。闭包中的代码可以访问在创建闭包的范围内可用的变量和函数之类的东西，即使闭包在执行时位于不同的范围内。可以使用大括号`{}`将代码括起来，以编写没有名称的闭包。使用`in`分隔参数和返回类型。

  ```swift
  var numbers = [20, 19, 7, 12]
  print(numbers.map({ (number: Int) -> Int in //arr.map()返回一个数组，其中包含将给定闭包映射到序列元素上的结果。
      if number % 2 != 0 {
          return 0
      }
      return number
  }))
  //[20, 0, 0, 12]
  ```

  你有更多的选择来把闭包写的更加简洁。当一个闭包的类型已经可知，比如说某个委托的回调，你可以去掉它的参数类型，它的返回类型，或者都去掉。

  ```swift
  let mappedNumbers = numbers.map({ number in 3 * number })
  print(mappedNumbers)
  ```

  可以调用参数通过数字而非名字——这个特性在非常简短的闭包当中尤其有用。当一个闭包作为函数最后一个参数出入时，可以直接跟在圆括号后边。如果闭包是函数的唯一参数，可以去掉圆括号直接写闭包。

  ```swift
  let sortedNumbers = numbers.sorted { $0 > $1 }//通过$0和$1等来表示闭包中的第一个第二个参数，并且对应的参数类型会根据函数类型来进行判断
  //let sortedNumbers = numbers.sorted { (a, b) -> Bool in
  //    return a > b
  //}
  print(sortedNumbers)
  ```

# Objects and Classes

- 通过在`class`后接类名称来创建一个类。在类里边声明属性与声明常量或者变量的方法是相同的，唯一的区别的它们在类环境下。同样的，方法和函数的声明也是相同的写法。

- 通过在类名字后边添加一对圆括号来创建一个类的实例。使用点语法来访问实例里的属性和方法。使用`init`来创建一个初始化器。

  ```swift
  class NamedShape {
      var numberOfSides: Int = 0
      var name: String
      
      init(name: String) {
          self.name = name
      }
      
      func simpleDescription() -> String {
          return "A shape with \(numberOfSides) sides."
      }
  }
  ```

- 如果需要在释放对象之前执行一些清理，请使用`deinit`创建一个反初始化程序。
- 子类的方法如果要重写父类的实现，则需要使用`override`——不使用 override关键字来标记则会导致编译器报错。编译器同样也会检测使用 override的方法是否存在于父类当中。

- 除了存储属性，你也可以拥有带有 getter 和 setter 的计算属性。

  ```swift
  class EquilateralTriangle: NamedShape {
      var sideLength: Double = 0.0
      
      init(sideLength: Double, name: String) {
          self.sideLength = sideLength
          super.init(name: name)
          numberOfSides = 3
      }
      
      var perimeter: Double {
          get {
              return 3.0 * sideLength
          }
          set {
            //在 perimeter的 setter 中，新值被隐式地命名为 newValue。你可以提供一个显式的名字放在 set 后边的圆括号里。
              sideLength = newValue / 3.0
          }
      }
      
      override func simpleDescription() -> String {
          return "An equilateral triangle with sides of length \(sideLength)."
      }
  }
  var triangle = EquilateralTriangle(sideLength: 3.1, name: "a triangle")
  print(triangle.perimeter)
  triangle.perimeter = 9.9
  print(triangle.sideLength)
  ```

- 如果你不需要计算属性但仍然需要在设置一个新值的前后执行代码，使用 willSet和 didSet。比如说，下面的类确保三角形的边长始终和正方形的边长相同。

  ```swift
  class TriangleAndSquare {
      var triangle: EquilateralTriangle {
          willSet {
              square.sideLength = newValue.sideLength
          }
      }
      var square: Square {
          willSet {
              triangle.sideLength = newValue.sideLength
          }
      }
      init(size: Double, name: String) {
          square = Square(sideLength: size, name: name)
          triangle = EquilateralTriangle(sideLength: size, name: name)
      }
  }
  var triangleAndSquare = TriangleAndSquare(size: 10, name: "another test shape")
  print(triangleAndSquare.square.sideLength)
  print(triangleAndSquare.triangle.sideLength)
  triangleAndSquare.square = Square(sideLength: 50, name: "larger square")
  print(triangleAndSquare.triangle.sideLength)
  ```

- 当你操作可选项的值的时候，你可以在可选项前边使用 ?比如方法，属性和下标。如果`?`前的值是`nil`，那`?`后的所有内容都会被忽略并且整个表达式的值都是`nil`。否则，可选项的值将被展开，然后 ?后边的代码根据展开的值执行。在这两种情况当中，表达式的值是一个可选的值。

  ```swift
  let optionalSquare: Square? = Square(sideLength: 2.5, name: "optional square")
  let sideLength = optionalSquare?.sideLength
  ```

- 使用`struct`来创建结构体。结构体提供很多类似与类的行为，包括方法和初始化器。其中最重要的一点区别就是结构体总是会在传递的时候拷贝其自身，而类则会传递引用。

# 协议和扩展

- 使用 protocol来声明协议。类，枚举以及结构体都兼容协议。

  ```swift
  protocol ExampleProtocol {
      var simpleDescription: String { get }
      mutating func adjust()
  }
  ```

  ```swift
  class SimpleClass: ExampleProtocol {
      var simpleDescription: String = "A very simple class."
      var anotherProperty: Int = 69105
      func adjust() {
          simpleDescription += "  Now 100% adjusted."
      }
  }
  var a = SimpleClass()
  a.adjust()
  let aDescription = a.simpleDescription
   
  struct SimpleStructure: ExampleProtocol {
      var simpleDescription: String = "A simple structure"
      mutating func adjust() {
          simpleDescription += " (adjusted)"
      }
  }
  var b = SimpleStructure()
  b.adjust()
  let bDescription = b.simpleDescription
  ```

  

- 注意使用`mutating`关键字来声明在 SimpleStructure 中使方法可以修改结构体。在 SimpleClass 中则不需要这样声明，因为类里的方法总是可以修改其自身属性的。

- 使用`extension`来给现存的类型增加功能，比如说新的方法和计算属性。你可以使用扩展来使协议来别处定义的类型，或者你导入的其他库或框架。

# Structures and Classes

- 在 Swift 中类和结构体有很多共同之处，它们都能：
  - 定义属性用来存储值；
  - 定义方法用于提供功能；
  - 定义下标脚本用来允许使用下标语法访问值；
  - 定义初始化器用于初始化状态；
  - 可以被扩展来默认所没有的功能；
  - 遵循协议来针对特定类型提供标准功能。

- 类有而结构体没有的额外功能：
  - 继承允许一个类继承另一个类的特征;
  - 类型转换允许你在运行检查和解释一个类实例的类型；
  - 反初始化器允许一个类实例释放任何其所被分配的资源；
  - 引用计数允许不止一个对类实例的引用。

- 结构体和枚举是值类型

- 类是引用类型

## 特征运算符

因为类是引用类型，在后台有可能有很多常量和变量都是引用到了同一个类的实例。(相同这词对结构体和枚举来说并不是真的相同，因为它们在赋予给常量，变量或者被传递给一个函数时总是被拷贝过去的。)

有时候找出两个常量或者变量是否引用自同一个类实例非常有用，为了允许这样，Swift提供了两个特点运算符：

- 相同于 ( ===)
- 不相同于( !==)

利用这两个恒等运算符来检查两个常量或者变量是否引用相同的实例：

注意”相同于”(用三个等于号表示，或者说 ===)这与”等于”的意义不同(用两个等于号表示，或者说 ==)。

- “相同于”意味着两个类类型常量或者变量引用自相同的实例；
- “等于”意味着两个实例的在值上被视作“相等”或者“等价”，某种意义上的“相等”，就如同类设计者定义的那样。

## 字符串，数组和字典的赋值与拷贝行为

Swift 的 String , Array 和 Dictionary类型是作为结构体来实现的，这意味着字符串，数组和字典在它们被赋值到一个新的常量或者变量，亦或者它们本身被传递到一个函数或方法中的时候，其实是传递了拷贝。

这种行为不同于基础库中的 NSString, NSArray和 NSDictionary，它们是作为类来实现的，而不是结构体。 NSString , NSArray 和 NSDictionary实例总是作为一个已存在实例的引用而不是拷贝来赋值和传递。

# 访问权限

[访问权限 **由大到小** 依次为：open，public，internal（默认），fileprivate，private](http://www.cnblogs.com/yajunLi/p/6626509.html)

