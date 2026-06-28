# Working with Numbers in Flutter: A Developer's Guide  

When you start building Flutter apps, numeric values appear in the most mundane corners—counters, prices, dimensions, animation offsets. Dart keeps the story simple with two core numeric types, but the way those types behave in arithmetic, parsing, and display can become a source of subtle bugs if you’re not careful. This article walks through the practical aspects of handling numbers in Dart and Flutter, showing how to avoid the common pitfalls that trip up even experienced engineers.

## Choosing the Right Numeric Type  

Dart defines `int` for whole numbers and `double` for values that may contain a fractional part. Both types inherit from the generic `num` superclass, which means you can assign either to a variable declared as `num`, but relying on that flexibility can obscure bugs.  

* `int` offers arbitrary precision—its size grows only with the amount of memory you allocate.  
* `double` follows the IEEE‑754 64‑bit floating‑point standard, giving you fast calculations at the cost of limited precision (roughly 15–17 significant digits).  

If you’re dealing with monetary values, the loss of precision in `double` is a red flag. A safer pattern is to store amounts as integer cents and only convert to a decimal representation when displaying the value. This eliminates rounding surprises without adding complexity.

```dart
int priceCents = 1999; // $19.99
double displayPrice = priceCents / 100; // 19.99
```

Choosing the appropriate type early saves you from having to retrofit a fix later in the codebase.

## Arithmetic Gotchas and Division  

The division operator behaves differently depending on the operand types. Using the standard `/` always yields a `double`, even when both operands are integers:

```dart
var result = 7 / 2; // 3.5, not 3
```

If you need integer division (the quotient without a remainder), Dart provides the truncating division operator `~/`:

```dart
var quotient = 7 ~/ 2; // 3
var remainder = 7 % 2; // 1
```

A subtle point is that `~/` truncates toward zero, so a negative dividend yields a result that may feel counterintuitive:

```dart
var neg = -7 ~/ 2; // -3
```

Beyond basic arithmetic, the `dart:math` library supplies constants (`pi`, `e`) and a suite of functions for trigonometry, logarithms, and random number generation:

```dart
import 'dart:math';

double area = pi * radius * radius;
int dice = Random().nextInt(6) + 1; // 1‑6
```

These tools are indispensable for any non‑trivial calculation, but remember to import the library before you can use them.

## Parsing and Formatting Numbers  

User input arrives as strings, and converting those strings to numeric types is a frequent source of runtime exceptions. Both `int.parse` and `double.parse` throw a `FormatException` when the string cannot be interpreted as a number, which can crash your app if left unhandled.

A more defensive approach is to use the `tryParse` methods, which return `null` instead of throwing:

```dart
int? qty = int.tryParse(userInput);
if (qty == null) {
  // handle invalid input gracefully
}
```

When you need to go the other way—turning a number into a string—`toString()` is the basic method, while `toStringAsFixed` lets you control the number of decimal places, which is especially useful for currency formatting:

```dart
double price = 19.9;
price.toString();          // "19.9"
price.toStringAsFixed(2);