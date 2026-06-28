
# Working with Numbers in Flutter: Common Pitfalls and Best Practices  

Numbers are everywhere in mobile apps—user counts, prices, scores, percentages. Yet mishandling them leads to bugs that range from minor display glitches to catastrophic calculation errors. Here's how to work with Dart's numeric types effectively in Flutter.

## Dart's Numeric Types: int, double, and num  

Dart provides two concrete numeric types: `int` for integers and `double` for floating-point numbers. Both inherit from the abstract `num` type, which means you can declare variables as `num` when you need to accept either:

```dart
int userCount = 42;
double price = 19.99;
num value = userCount;  // OK
value = price;          // Also OK
```

However, relying on `num` too heavily reduces type safety. Prefer specific types when possible.

The `int` type supports arbitrary precision, growing based on available memory. `double` follows IEEE 754 64-bit floating-point standards with roughly 15-17 significant digits of precision. This matters significantly for financial calculations.

## Mathematical Operations and the dart:math Library  

Basic arithmetic works intuitively, but division behaves differently than expected:

```dart
var result = 7 / 2;   // 3.5 (always returns double)
var quotient = 7 ~/ 2; // 3 (truncating division)
```

The `/` operator always produces a `double`, even with integer operands. Use `~/` when you need integer division, but remember it truncates toward zero (so `-7 ~/ 2` equals `-3`, not `-4`).

For advanced math, import `dart:math`:

```dart
import 'dart:math';

double area = pi * pow(radius, 2);  // pi and pow() available
int randomNum = Random().nextInt(100); // generates 0-99
```

## Parsing Strings and Type Conversion  

Converting between strings and numbers requires careful error handling:

```dart
// These throw FormatException on invalid input
int count = int.parse('42');
double price = double.parse('19.99');

// Safer alternatives return null instead of throwing
int? maybeCount = int.tryParse(userInput);
if (maybeCount != null) {
  // Process valid number
}
```

Converting numbers to strings uses `toString()` or `toStringAsFixed()` for formatting:

```dart
double price = 19.9;
price.toString();            // "19.9"
price.toStringAsFixed(2);    // "19.90"
```

## Displaying Numbers in Flutter Widgets  

Flutter's `Text` widget requires string input, so convert numbers before display:

```dart
Text('Users: $userCount');              // int interpolates directly
Text('Price: \$${price.toStringAsFixed(2)}'); // format double
```

Optimize static numeric values with const constructors:

```dart
const Text('Score: 100');  // No runtime allocation
```

Note that `toStringAsFixed()` isn't const, so dynamic formatting can't benefit from compile-time optimization.

## Common Mistakes to Avoid  

**Floating-point comparison**: Never use `==` directly with doubles due to precision issues:

```dart
// Wrong
if (price == 19.99) { ... }

// Right
if ((price - 19.99).abs() < 0.001) { ... }
```

**Division confusion**: Remember that `/` returns `double` while `~/` truncates:

```dart
7 / 2;   // 3.5
7 ~/ 2;  // 3
```

**Parsing without error handling**: Always wrap `parse()` calls or use `tryParse()`:

```dart
// Risky - crashes on invalid input
int value = int.parse(userInput);

// Safer
int? value = int.tryParse(userInput);
```

**Precision loss with large numbers**: Be careful converting large `int` values to `double`:

```dart
int bigId = 9007199254740993;
double imprecise = bigId.toDouble();  // Loses precision
```

## Building Robust Numeric Logic  

For financial calculations, store amounts as integer cents:

```dart
class Product {
  final String name;
  final int priceCents;
  
  Product(this.name, this.priceCents);
  
  String get formattedPrice => '\$${(priceCents / 100).toStringAsFixed(2)}';
}
```

This approach eliminates floating-point rounding errors while maintaining clean display formatting.

When building counters or progress indicators, consider using `int` for state management and converting to `double` only when calculating ratios:

```dart
int completedSteps = 3;
int totalSteps = 5;
double progress = completedSteps / totalSteps;  // 0.6
```

---

Mastering Flutter's numeric handling comes down to understanding type behaviors, using appropriate conversion methods, and avoiding common comparison pitfalls. Keep these patterns in mind, and your number-crunching will be both reliable and efficient.
