When you start building Flutter apps, numbers show up everywhere—counters, prices, dimensions, animation values. Dart keeps this simple with just two numeric types, but the details around operations, parsing, and display can trip you up if you're not careful. Let's walk through what you actually need to know.

## The Two Types You'll Use

Dart gives you `int` for whole numbers and `double` for anything with a decimal point. Both inherit from `num`, which means you can treat them interchangeably in most contexts—but you shouldn't rely on that too heavily.

```dart
int userCount = 142;
double temperature = 23.5;
```

The distinction matters. An `int` has arbitrary precision, so it grows as large as memory allows. A `double` is a 64-bit IEEE 754 float, which means precision drops off around 15–17 significant digits. If you're doing financial calculations, that's a problem waiting to happen.

## Arithmetic That Behaves Differently Than You Expect

The division operator `/` always returns a `double`, even when both operands are integers:

```dart
var result = 7 / 2;  // 3.5, not 3
```

If you need integer division, use the truncating division operator `~/`:

```dart
var quotient = 7 ~/ 2;  // 3
var remainder = 7 % 2;  // 1
```

This catches people off guard constantly. The `~/` operator isn't just "integer division"—it truncates toward zero, so `-7 ~/ 2` gives `-3`, not `-4`.

For anything beyond basic arithmetic, pull in `dart:math`. You get constants like `pi` and `e`, plus functions for trigonometry, logarithms, and random number generation:

```dart
import 'dart:math';

double area = pi * radius * radius;
int diceRoll = Random().nextInt(6) + 1;  // 1-6
```

## Parsing Input Safely

User input arrives as strings. Converting to numbers means `int.parse()` and `double.parse()`, but both throw `FormatException` on invalid input:

```dart
// These work
int.parse('42');
double.parse('3.14159');

// This crashes your app
int.parse('not-a-number');
```

Always wrap parsing in try-catch, or use the `tryParse` variants that return `null` instead of throwing:

```dart
int? quantity = int.tryParse(userInput);
if (quantity == null) {
  // Handle invalid input gracefully
}
```

The same goes for the reverse direction. Every number has `toString()`, but `double` adds `toStringAsFixed()` for controlling decimal places:

```dart
double price = 19.9;
price.toString();           // "19.9"
price.toStringAsFixed(2);   // "19.90"
```

This is essential for currency display—nobody wants to see `$19.9` in a receipt.

## Displaying Numbers in Widgets

Flutter's `Text` widget only accepts strings, so conversion is mandatory:

```dart
Text('Items: $count');                    // int interpolates fine
Text('Total: \$${price.toStringAsFixed(2)}');  // format the double
```

A small optimization: if you're passing numeric values to const constructors, make the conversion const too:

```dart
const Text('Count: 42');  // fully const, no runtime allocation
```

But `toStringAsFixed()` isn't const, so you can't use it in const contexts. Plan accordingly.

## Precision Traps

Floating-point comparison is the classic footgun:

```dart
// Dangerous
if (price == 19.99) { ... }

// Safer
if ((price - 19.99).abs() < 0.001) { ... }
```

For money, consider storing values as integer cents instead of dollars. It eliminates an entire class of rounding bugs:

```dart
int priceCents = 1999;  // $19.99
double displayPrice = priceCents / 100;
```

Large integers have their own quirk. While `int` has arbitrary precision, converting to `double` loses precision past 2^53. If you're handling IDs or timestamps that exceed that range, keep them as `int` or `String`—never cast to `double`.

## Putting It Together

A practical pattern for a shopping cart item might look like:

```dart
class CartItem {
  final String name;
  final int quantity;
  final int unitPriceCents;  // store as cents
  
  int get totalCents => quantity * unitPriceCents;
  
  String get formattedTotal => 
      '\$${(totalCents / 100).toStringAsFixed(2)}';
}
```

Notice the integer arithmetic for calculations, conversion only at the display layer. This pattern scales.

---

Numbers in Dart are straightforward once you internalize the type split and the division operators. The real skill is knowing where precision matters—parsing input, comparing floats, formatting output—and handling those boundaries deliberately. Everything else is just arithmetic.