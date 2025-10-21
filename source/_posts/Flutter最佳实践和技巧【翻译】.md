---
title: Flutter最佳实践和技巧【翻译】
link:
date: 2021-08-02 19:11:21
tags:
  - Flutter
---

翻译自 https://medium.com/flutter-community/flutter-best-practices-and-tips-7c2782c9ebb5

最佳实践是一个领域内可接受的专业标准，对于任何编程语言来说，提高代码质量、可读性、可维护性和健壮性都非常重要。
这是一些设计和开发 Flutter 应用程序的最佳实践。

## 命名规范

类名、枚举、`typedef`、扩展名应该使用大驼峰命名方式

```dart
class MainScreen { ... }
enum MainItem { .. }
typedef Predicate<T> = bool Function(T value);
extension MyList<T> on List<T> { ... }
```

libraries、包、目录、源文件名采用蛇形命名。

```dart
library firebase_dynamic_links;
import 'socket/socket_manager.dart';
```

变量、参数等使用蛇形命名

```dart
var item;
const bookPrice = 3.14;
final urlScheme = RegExp('^([a-z]+):');
void sum(int bookPrice) {
  // ...
}
```

## 总是指定变量类型

如果已经知道变量值的类型，那么应该指定变量的类型，尽量避免使用`var`

```dart
//Don't
var item = 10;
final car = Car();
const timeOut = 2000;


//Do
int item = 10;
final Car bar = Car();
String name = 'john';
const int timeOut = 20;
```

## 尽量用`is`替代`as`

`as`运算符会在无法进行类型转换的时候抛出异常，为了避免抛出异常，使用`is`操作符

```dart
//Don't
(item as Animal).name = 'Lion';


//Do
if (item is Animal)
  item.name = 'Lion';
```

## 使用`??`和`?.`操作符

尽量使用`??`和`?.`操作符替代`if-null`检测

```dart
//Don't
v = a == null ? b : a;

//Do
v = a ?? b;


//Don't
v = a == null ? null : a.b;

//Do
v = a?.b;
```

## 使用扩散集合（Spread Collections）

当items已经存在另外的集合中时，扩展集合语法可以简化代码

```dart
//Don't
var y = [4,5,6];
var x = [1,2];
x.addAll(y);


//Do
var y = [4,5,6];
var x = [1,2,...y];
```

## 使用级联操作符

在同一Object上的连续操作可以使用级联操作符`..`

```dart
// Don't
var path = Path();
path.lineTo(0, size.height);
path.lineTo(size.width, size.height);
path.lineTo(size.width, 0);
path.close();  


// Do
var path = Path()
..lineTo(0, size.height)
..lineTo(size.width, size.height)
..lineTo(size.width, 0)
..close(); 
```

## 使用原生字符串（Raw Strings）

原生字符串可以不对`$`和`\`进行转义

```dart
//Don't
var s = 'This is demo string \\ and \$';


//Do
var s = r'This is demo string \ and $';
```

## 不要使用`null`显示初始化变量

dart中的变量默认会被初始化为`null`，设置初始值为`null`是多余且不必要的。

```dart
//Don't
int _item = null;


//Do
int _item;
```

## 使用表达式函数体

对于只有一个表达式的函数，你可以使用表达式函数。`=>`标记着表达式函数。

```dart
//Don't
get width {
  return right - left;
}
Widget getProgressBar() {
  return CircularProgressIndicator(
    valueColor: AlwaysStoppedAnimation<Color>(Colors.blue),
  );
}


//Do
get width => right - left;
Widget getProgressBar() => CircularProgressIndicator(
      valueColor: AlwaysStoppedAnimation<Color>(Colors.blue),
    );
```

## 避免调用`print()`函数

`print()`和`debugPrint()`都用来向控制台打印消息。在`print()`函数中，如果输出过大，那么`Android`系统可能会丢弃一些日志。这种情况下，你可以使用`debugPrint`。

## 使用插值法来组织字符串

使用插值法组织字符串相对于用`+`拼接会让字符串更整洁、更短。

```dart
//Don’t
var description = 'Hello, ' + name + '! You are ' + (year - birth).toString() + ' years old.';


// Do
var description = 'Hello, $name! You are ${year - birth} years old.';
```

## 使用async/await而不是滥用future callback

异步代码难以阅读和调试，`async`和`await`语法增强了可读性

```dart
// Don’t
Future<int> countActiveUser() {
  return getActiveUser().then((users) {
    
    return users?.length ?? 0;

  }).catchError((e) {
    log.error(e);
    return 0;
  });
}


// Do
Future<int> countActiveUser() async {
  try {
    var users = await getActiveUser();
     
    return users?.length ?? 0;
  
  } catch (e) {
    log.error(e);
    return 0;
  }
}
```

## 在`widget`中使用`const`

如果我们把`widget`定义为`const`，那么该`widget`在`setState`的时候不会重建。通过避免重建来提升性能

```dart
Container(
      padding: const EdgeInsets.only(top: 10),
      color: Colors.black,
      child: const Center(
        child: const Text(
          "No Data found",
          style: const TextStyle(fontSize: 30, fontWeight: FontWeight.w800),
        ),
      ),
    );
```
