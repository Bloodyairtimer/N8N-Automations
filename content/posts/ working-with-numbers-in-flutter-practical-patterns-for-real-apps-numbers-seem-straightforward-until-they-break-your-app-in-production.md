
# Working with Numbers in Flutter: Practical Patterns for Real Apps  

Numbers seem straightforward until they break your app in production. Countless developers have shipped features that work perfectly in the simulator, only to discover their pricing calculations were off by pennies or their animation values jittered oddly on real devices. The root cause is usually the subtle way Dart handles numeric types, which can trip up even experienced Flutter engineers.

## Understanding Dart's Numeric Types  

Dart keeps things intentionally simple with just two core numeric types: `int` for whole numbers and `double` for values with decimal points. Both inherit from the generic `num` type, though you'll rarely need to work at that level. What's important is recognizing that `int` supports arbitrary precision—meaning it grows to match your device's memory—while `double` follows the IEEE 754 standard with its familiar precision limitations.  

This distinction becomes critical when handling currency, large counters, or IDs. Store monetary values as `int` cents rather than `double` dollars, and you'll sidestep most rounding headaches. Keep user IDs in `int` or `String` format, and you won't lose precision on massive identifiers.

## Arithmetic Operations That Catch People Off Guard  

The division operator behaves differently than many developers expect. Even when dividing two integers, Dart returns a `double`:

```dart
var quotient = 7 / 2;  // 3.5, not 3
```

If you need integer division, use the truncating division operator `~/`:

```dart
var result = 7 ~/ 2;  // 3
var negative = -7 ~/ 2;  // -3 (truncates toward zero)
```

This catches everyone eventually. The `~/` operator isn't just "integer division"—it specifically truncates toward zero, which differs from floor division for negative numbers.

For more complex calculations, pull in `dart:math`. It provides constants like `pi` and `e`, plus trigonometric, logarithmic, and random number functions:

```dart
import 'dart:math';

double area = pi * radius * radius;
int diceRoll = Random().nextInt(6) + 1;  // 1-6
```

## Converting Numbers Safely  

Parsing user input into numbers happens everywhere in apps, and it's also where many crashes originate. Both `int.parse()` and `double.parse()` throw `FormatException` when given malformed strings, so always wrap them in try-catch or use the safer alternatives:

```dart
int? userAge = int.tryParse(ageInput);
if (userAge == null) {
  // Handle invalid input gracefully
}
```

Converting numbers back to strings for display follows a similar pattern. Every numeric type supports `toString()`, but `double` offers `toStringAsFixed()` for controlling decimal representation:

```dart
double price = 19.9;
price.toString();           // "19.9"
price.toStringAsFixed(2);   // "19.90"
```

This is essential for consistent currency formatting—nobody wants to see "$19.9" in their checkout flow.

## Working with Numbers in Widget Trees  

Flutter's `Text` widget accepts strings, so numeric values need conversion before display. Basic interpolation works fine for integers, but doubles often need formatting:

```dart
Text('Items: $count');                    // Fine for ints
Text('Total: \$${price.toStringAsFixed(2)}');  // Format doubles properly
```

A small but useful optimization involves const constructors. Hardcoded numbers can be const, eliminating runtime allocations:

```dart
const Text('Count: 42');  // Fully const, no runtime allocation
```

However, `toStringAsFixed()` isn't const, so formatted numbers can't benefit from this optimization. Plan your widget construction accordingly.

## Avoiding Common Pitfalls  

Floating-point comparison remains the classic trap. Direct equality checks fail due to inherent precision limitations:

```dart
// Problematic
if (price == 19.99) { ... }

// Better approach
if ((price - 19.99).abs() < 0.001) { ... }
```

Similarly, be mindful of implicit conversions. Dart allows mixing `int` and `double` in expressions, but the results always favor `double`. For precise calculations, stick to integer math until the final display step:

```dart
int priceCents = 1999;  // $19.99
double displayPrice = priceCents / 100;  // 19.99
```

Very large integers present another gotcha. While Dart's `int` handles arbitrary precision, converting beyond 2^53 to `double` loses accuracy. Keep large IDs and timestamps in their native types.

---

Numbers in Flutter aren't complicated—they're just detail-oriented. Master the division operators, handle parsing defensively, and format with intention. These practices prevent the subtle bugs that waste hours of debugging time and keep your users from seeing incorrect prices or jittery animations.
