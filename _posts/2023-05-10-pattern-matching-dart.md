---
author: Biplab
title: An introduction to Records and Pattern Matching in Dart & Flutter
date: 2023-05-10 3:00:00 +545
categories: [flutter]
tags: [flutter, dart, records, pattern matching, object matching, sealed classes]
image:
  path: https://raw.githubusercontent.com/Biplab-Dutta/personal_site/main/preview_images/pattern_matching_and_records.png
  alt: Preview Image Credit - Sandro Maglione
---

With the arrival of Dart 3, it brings many new features to the Dart language.  Features like records, pattern matching, guard clauses, logical and relational operators in switch cases, if-case statements, pattern/object destructuring, multiple returns, and many more will be added. This article aims to familiarize the readers with these upcoming features in the Dart language.

## Table of Contents
- [Record](#record)
- [Why/When would I want to use a record? 🤔](#whywhen-would-i-want-to-use-a-record-)
- [Record Type Annotations](#record-type-annotations)
  - [Record with positional fields](#record-with-positional-fields)
  - [Record with named fields](#record-with-named-fields)
  - [Record with named and positional fields](#record-with-named-and-positional-fields)
- [Destructuring](#destructuring)
  - [Destructuring positional fields](#destructuring-positional-fields)
  - [Destructuring named fields](#destructuring-named-fields)
  - [Destructuring positional and named fields](#destructuring-positional-and-named-fields)
  - [JSON Destructuring](#json-destructuring)
  - [Object Destructuring](#object-destructuring)
- [Guard Clauses](#guard-clauses)
- [Sealed class and pattern matching](#sealed-class-and-pattern-matching)
- [Logical \& Relational operatora in switch case](#logical--relational-operatora-in-switch-case)
- [If-case Statements](#if-case-statements)
- [Control Flow in Argument Lists](#control-flow-in-argument-lists)
- [Conclusion](#conclusion)
- [Credit](#credit)

> _All the code in this blog has been tested in the Dart's master channel. You can try running the examples shown in this blog on [Dartpad][Dartpad master branch] and switching the channel from stable (by defualt) to master._
{: .prompt-info }

Now, let us get started with the major introductions in this new segment.

## Record
**Record** is a new first-class object defined in the `dart:core` library. It is able to hold single datum or multiple data of same or different types. A record is like a list in a sense that it can hold multiple data but unlike list, a record is immutable and have value equality. To use records, wrap your data with a pair of parenthesis. For example:

```dart
final x = (1, 2, 'a'); // Here, x is a record with the type (int, int, String)

final y = (x: 'value', 5); // Here, y is a record having the type ({String x}, int) with the first field being named and second being positional
```

> _It is important to realize that if our record contains only one positional field, then there **MUST** be a trailing comma before the closing parenthesis._
{: .prompt-info }

```dart
final x = ('a'); // Compile-time error
final y = ('a',) // 👌
```

> _The expression `()` refers to the constant empty record with no fields._
{: .prompt-info }

## Why/When would I want to use a record? 🤔
Imagine a situation where you'd want to bundle multiple objects into a single value. At the point of writing this article, Dart offers mainly two ways to do so. Firstly, by storing the multiple data into a list. While this is doable, if the data that are to be stored, happen to be of different types, the best you can use is a `List<dynamic>` or a `List<Object>`. This easily loses the type safety feature.

```dart
void main() {
  final json = <String, dynamic>{
    'name': 'Neko',
    'age': 22,
  };

  final info = studentInfo(json); // info is of type `List<Object>`
  final name = info[0] as String; // Manual casting 🤮
  final age = info[1] as int; // Manual casting 🤮
}

List<Object> studentInfo(Map<String, dynamic> json) {
  return [
    json['name'] as String,
    json['age'] as int,
  ];
}
```

Another way of accomplishing our goal would be by creating a class that would contain fields that are to be stored. This is okay if the class has some responsibilty. But if it is just about storing data, then creating a new class and instantiating an object would become verbose.

```dart
void main() {
  final json = {    
    'name': 'Neko',
    'age': 22,
  };

  final info = Student.info(json);
  final name = info.name; // Neko
  final age = info.age; // 22
}

class Student {
  final String name;
  final int age;

  Student({
    required this.name,
    required this.age,
  });

  factory Student.info(Map<String, dynamic> json) {
    return Student(
      name: json['name'] as String,
      age: json['age'] as int,
    );
  }
}
```

Thus, a new solution was proposed in the form of Records. Records bring all the goodness of existing collections (`List`, `Set`, etc) while promoting type safety and helping to obtain concise code.

```dart
void main() {
  final json = {
    'name': 'Neko',
    'age': 22,
  };

  final info = studentInfo(json);
  final name = info.$1; // Neko
  final age = info.$2; // 22
}

(String, int) studentInfo(Map<String, dynamic> json) {
  return (
    json['name'] as String,
    json['age'] as int,
  );
}
```

> _In the above method `studentInfo()`, its return type is `(String, int)` i.e. with the arrival of Records, it is possible to return multiple values from a function/method._
{: .prompt-info }

It is a compile-time error if a record has any of:
- The same field name more than once.
- Only one positional field and no trailing comma.
- A field named `hashCode`, `runtimeType`, `noSuchMethod` or `toString`.
- A field name that starts with an underscore.
- A field name that collides with the synthesized getter name of a positional field. For example: (int, $1: int) since the named field `$1` collides with the getter for the first positional field.

## Record Type Annotations
A record can either have positional or named field(s) or a combination of both.

### Record with positional fields
```dart
(int, String) value = (1, 'a');
```

### Record with named fields
```dart
({int i, String str}) value = (i: 1, str: 'a');
```

### Record with named and positional fields
```dart
(int, {String str}) value1 = (5, str: 'a');

// - - - - - - - - - - - - - - - - - - - - - - - - - -

(int, {String str}) value2 = (str: 'a', 5);

// - - - - - - - - - - - - - - - - - - - - - - - - - -

(int, {String s, int i}) value3 = (s: 'data', i: 1, 5);
                        // OR
(int, {int i, String s}) value3 = (s: 'data', i: 1, 5);
```

## Destructuring

Since, a record is a collection, there has to be a way of accessing its individual field. That is known as destructuring. Once a record has been created, its fields can be accessed using getters. Every named field exposes a getter with the same name, and positional fields expose getters named `$1`, `$2`, etc.

### Destructuring positional fields

```dart
final value = ('Neko', 22);
print(value.$1); // Neko
print(value.$2); // 22
```
Invoking `$1` on a record gives the first element of that record and so on.

> _Using `$0` on a record to obtain the first element will result in a compile-time error._
{: .prompt-danger }

### Destructuring named fields

```dart
final value = (name: 'Neko', age: 22);
print(value.name); // Neko
print(value.age); // 22
```

### Destructuring positional and named fields

```dart
final value = ('Neko', age: 22, address: 'Nepal', 7);
print(value.$1); // Neko
print(value.age); // 22
print(value.address); // Nepal
print(value.$2); // 7
```

If we take a closer look at destructuring our records, it is still pretty verbose. That's where **Patterns** come into play. With the help of patterns, we can destructure records inline.

```dart
final (name, age) = ('Neko', 22);
print(name); // Neko
print(age); // 22

// - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - 

final (name, age, address: addr) = ('Neko', 22, address: 'Nepal');
print(name); // Neko
print(age); // 22
print(addr); // Nepal
```

In the above example, the data type for the variables `name`, `age` and `addr` is implicitly assigned by Dart. Though not necessary, we can also explicitly assign the data type for each field in the Record.

```dart
final (String name, int age, address: String addr) = ('Neko', 22, address: 'Nepal');
print(name); // Neko
print(age); // 22
print(addr); // Nepal
```

### JSON Destructuring
In addition to records, patterns allow destructuring on JSON objects too.
```dart
final map = {'first': 1, 'second': 2};
final {'first': a, 'second': b} = map;
print(a); // 1
print(b); // 2

// - - - - - - - - - - - - - - - - - - - - - -

final map = {
  'events': ['event-1', 'event-2']
};
final {'events': [firstEvent, secondEvent]} = map;
print(firstEvent); // event-1
print(secondEvent); // event-2
```

Most of the times, the obtained data in JSON representation is provided by the backend, and before we work on it, we need to validate if the obtained response is to our liking.
Let's consider the following JSON:

```dart
final map = {
  'events': ['event-1', 'event-2']
};
```
If we were to validate this JSON in a traditional way. We'd have to something like this.

```dart
final json = <String, dynamic>{
  'events': ['event-1', 'event-2']
};

if (json.length == 1 && json.containsKey('events')) {
  final events = json['events'];
  if (events != null &&
      events.isNotEmpty &&
      events is List<dynamic> &&
      events.length == 2) {
    final firstEvent = events[0] as String; // Manual type-casting 🤮
    final secondEvent = events[1] as String; // Manual type-casting 🤮

    print(firstEvent); // event-1
    print(secondEvent); // event-2
  }
}
```
This code is clearly verbose and error-prone. With the help of pattern, we can easily perform JSON validation and obtain desired result.

```dart
final json = <String, dynamic>{
  'events': ['event-1', 'event-2']
};

switch (json) {
  case {'events': [String firstEvent, String secondEvent]}:
    print(firstEvent); // event-1
    print(secondEvent); // event-2
  default:
    throw 'Invalid JSON';
}
```
And that's it. Patterns matches the JSON structure that we want and if it statisfies, the code inside of our case gets executed.

> _When the pattern doesn't match the value, then it is said as the pattern refutes the value. Thus, irrefutable always match._
{: .prompt-info }

> _Also, in the above snippet, I didn't add break in the switch-case explicitly. That's because Dart 3 brings a new feature called [implicit break][Implicit break]._
{: .prompt-info }

### Object Destructuring
Pattern Matching and destructuring is not only limited to Records and collections. We can also use the same set of tools for objects.

```dart
abstract class Shape {}

class Rectangle extends Shape {
  Rectangle(this.length, this.breadth);

  final double length, breadth;
}

class Circle extends Shape {
  Circle(this.radius);

  final int radius;
}

void display(Shape shape) {
  switch (shape) {
    case Rectangle(length: final l, breadth: final b):
      print('Area of rectangle: ${l * b} sq. units');

    case Circle(radius: final r):
      print('Area of circle: ${22/7 * r * r} sq. units');

    default:
      print(shape);
  }
}

void main() {
  display(Rectangle(14, 2));
  display(Circle(7));
}
```
The getter names inside the switch can be omitted and inferred from the variable pattern such that the above `display()` method could be re-written as:

```dart
void display(Shape shape) {
  switch (shape) {
    case Rectangle(:final length, :final breadth): // Inferred getter names
      print('Area of rectangle: ${length * breadth} sq. units');

    case Circle(:final radius):
      print('Area of circle: ${22 / 7 * radius * radius} sq. units');

    default:
      print(shape);
  }
}
```

## Guard Clauses
Guard clauses allow us to use arbitrary expression and see if the case should be matched. We use `when` keyword when using guard clause in switch cases.

```dart
void main() {
  final list = <int>[1, 2];

  switch (list) {
    case [int a, int b] when a + b > 10: // Guard clause
      print('the sum is greater than 10');

    default:
      print('The sum is less than 10');
  }
}
```

> _Using guard clause is different from using `if` inside case body because if the guard is false, then the execution will continue in the next case instead of coming out of the entire switch case._
{: .prompt-info }


## Sealed class and pattern matching
A new addition is introduced in Dart 3 and that's sealed class. It can be declared with the keyword sealed. Sealed classes can't be directly constructed and are implicitly abstract.

```dart
sealed class Shape {}
```

We can represent class hierarchies with sealed classes. The subtypes of a sealed class can be normal classes or sealed classes.

```dart
sealed class Shape {}
class Circle extends Shape {}
sealed class Quadrilateral extends Shape {}
class Square extends Quadrilateral {}
class Rectangle extends Quadrilateral {}
```

The above class declarations can be used to visualize a class hierarchy that looks something like this.

```
Shape       
  ├─ Circle       
  └─ Quadrilateral
        ├─ Square
        └─ Rectangle      
```

> _All direct subtypes of the type must be defined in the same library where the sealed class is defined._
{: .prompt-info }

One advantage of having a sealed class is that the compiler is able to tell if we missed any subtype of that sealed class in the switch statement. This way, we get compile time error if our switch case isn't exhaustive.

```dart
sealed class Shape {}

class Square implements Shape {}
class Circle implements Shape {}
class Sphere implements Shape {}

String display(Shape shape) => switch (shape) {
      Square() => 'Square',
      Circle() => 'Circle',
      Sphere() => 'Sphere',
    };

void main() {
  print(display(Square()));
}
```

The `display()` method above could be re-written as:

```dart
String display(Shape shape) {
  switch (shape) {
    case Square():
      return 'Square';
    case Circle():
      return 'Circle';
    case Sphere():
      return 'Sphere';
  }
}
        // OR

String display(Shape shape) {
  return switch (shape) {
    Square() => 'Square',
    Circle() => 'Circle',
    Sphere() => 'Sphere',
  };
}
```

## Logical & Relational operatora in switch case

```dart
void main() {
  int a = 5;

  display(a);
}

void display(int value) {
  switch (value) {
    case 1 || 2 || 3: // Logical Operator
      print('Top Three');

    default:
      print('Not top three');
  }
}
```

Dart 3 also supports switch expressions.

```dart
void main() {
int obtainedMarks = 9;

final reaction = switch(obtainedMarks) {
    0 => 'Really?',
    1 || 2 || 3 => 'Call your parents',
    >= 4 && <=6 => 'Good',
    7 || 8 || 9 => 'Noice',
    10 => 'You are OP',
    _ => 'Invalid marks'
};

print(reaction); // Noice
}
```

> _The underscore in above switch expression aka wildcard acts as a default case._
{: .prompt-info }

## If-case Statements
Always using switch cases can be verbose. So, Dart 3 also provides us with the new if-case statements. It allows us to add a single pattern inside the if check and perform certain actions.

```dart
void main() {
  final json = <String, dynamic>{
    'events': ['event-1', 'event-2']
  };

  if (json case {'events': [String firstEvent, String secondEvent]}) {
    print(firstEvent); // event-1
    print(secondEvent); // event-2
  } else {
    print('Invalid JSON');
  }
}
```

The above code is readed as if the json follows the given pattern inside the if statement, then print firstEvent and secondEvent else print Invalid JSON.


## Control Flow in Argument Lists
While writing Flutter code, we often write if checks inside collection literals such as inside the children property of a Row or a Column. This helps in writing clean code and avoid ugly imperative code.
But sometimes, there are child widgets that need some conditional behaviour are inside named arguments and the best we can do is use ternary operator. Such as:

```dart
ListTile(
  leading: const Icon(Icons.weekend),
  title: const Text('Hello'),
  enabled: hasNextStep,
  subtitle: hasNextStep ? const Text('Tap to advance') : null,
  onTap: hasNextStep ? () { advance(); } : null,
  trailing: hasNextStep ? null : const Icon(Icons.stop),
)
```

While it's okay to use ternary operator as shown above for conditional behaviour, it can be made more elegant and easier with `if` inside arguments lists. Then the same code can be written as:

```dart
ListTile(
  leading: const Icon(Icons.weekend),
  title: const Text('Hello'),
  enabled: hasNextStep,
  if(hasNextStep) ...(
    subtitle: const Text('Tap to advance'),
    onTap: advance,
  ) else ...(
    trailing: const Icon(Icons.stop),
  )
)
```

This way we can make use of `if` statement inside argument lists with spread operator like syntax.

## Conclusion
Records and Patterns bring so much new to Dart and with the addition of so many new options, Dart as a programming language will only prosper. For more depth overview on these topics, I highly encourage the readers to go through the Feature Specification on Github for [records][Record Feature Specification], [patterns][Pattern Feature Specification], [sealed classes][Sealed Class Feature Specification], etc.

If you wish to see some Flutter projects, follow me on [GitHub][GitHub-Biplab]. I am also active on Twitter [@b_plab][Twitter-Biplab] where I tweet about Flutter and Android.

**My Socials:**

|[GitHub][GitHub-Biplab]|[LinkedIn][LinkedIn-Biplab]|[Twitter][Twitter-Biplab]|

Until next time, happy coding!!! 👨‍💻

— Biplab Dutta

## Credit

[Sandro Maglione][] for the preview image.

<!-- Hyperlinks -->
[Dartpad master branch]: <https://dartpad.dev/?channel=master>
[Implicit break]: <https://github.com/dart-lang/language/blob/main/accepted/future-releases/0546-patterns/feature-specification.md#implicit-break>
[Record Feature Specification]: <https://github.com/dart-lang/language/blob/main/accepted/future-releases/records/records-feature-specification.md>
[Pattern Feature Specification]: <https://github.com/dart-lang/language/blob/main/accepted/future-releases/0546-patterns/feature-specification.md>
[Sealed Class Feature Specification]: <https://github.com/dart-lang/language/blob/main/accepted/future-releases/sealed-types/feature-specification.md>
[Twitter-Biplab]: https://twitter.com/b_plab98
[GitHub-Biplab]: https://github.com/Biplab-Dutta/
[LinkedIn-Biplab]: https://www.linkedin.com/in/biplab-dutta-43774717a/
[Sandro Maglione]: https://twitter.com/SandroMaglione