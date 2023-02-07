---
title: "Applications of Fpdart - functional programming with Flutter"
date: 2023-02-01T16:00:00-08:00
draft: false
---

Functional programming has been garnering more attention in recent years, for good reason: it's a tool for writing more predicatable and readable code. I won't go into the details here - many, *many*, articles have covered that already. If you are looking for an intro, though, [this reddit comment](https://www.reddit.com/r/explainlikeimfive/comments/msmot/comment/c33msb8/?utm_source=share&utm_medium=web2x&context=3) may prove helpful.

This post explains how I've incorporated aspects of functional programming into my Flutter projects with [Fpdart](https://pub.dev/packages/fpdart). Each section includes an example of how a feature from the package can be used, as well as any potential downsides and considerations.

## Option 
### More control of null values


If you've used [Riverpod](https://riverpod.dev/), the following code will feel very familiar - let's start with an immutable `Person` class that includes a name and, optionally, an age:

```dart
class Person {
    final String name;
    final int? age;

    Person(this.name, this.age);
}   
```

Next, we'll add a `copyWith` method to easily create a new instance of `Person` with some of the fields changed:

```dart {hl_lines=["7-12"]}
class Person {
    final String name;
    final int? age;

    Person(this.name, this.age);

    Person copyWith({String? name, int? age}) {
        return Person(
            name ?? this.name,
            age ?? this.age,
        );
    }
}
```

Now, take a look at the following code:

```dart
var steve = Person('Steve');

steve = steve.copyWith(age: 30);
```

This creates a person, `steve`, with no specified age, then creates a new instance of `Person` with the same name but an age of 30. This is a common pattern in Flutter.

What if we wantted to change the age back to null? Your first thought may be the following:

```dart
steve = steve.copyWith(age: null);
```

However, `age` will remain 30, because the `copyWith` method above only changes the value if the new value is *not* null. 

```dart
print(steve.age); // prints "30", even after the copyWith call
```

The `Option` type from Fpdart helps to address this issue. It's a wrapper around a value that can either be `Some` or `None`, which can be used instead of nullable types.

```dart
// Creating an Option with a value of `None`
Option<String> data = None();

// Updating the value to `Some`
data = Some('Hello, world!');
```
If we were to use `Option` in the `Person` class above, we could change the `age` field to be of type `Option<int>`:

```dart {hl_lines=["3"]}
class Person {
    final String name;
    final Option<int> age;

    Person(this.name, [this.age = None()]);
}
```

Next, we can add the `copyWith` method again, using the `Option` type:

```dart {hl_lines=["7-12"]}
class Person {
    final String name;
    final Option<int> age;

    Person(this.name, [this.age = None()]);

    Person copyWith({String? name, Option<int>? age}) {
        return Person(
            name ?? this.name,
            age ?? this.age,
        );
    }
}
```

Now, it is possible to change the age from a value to `None`:

```dart
var steve = Person('Steve');

steve = steve.copyWith(age: Some(30));
print(steve.age); // prints "Some(30)"

steve = steve.copyWith(age: None());
print(steve.age); // prints "None"
```

To actually use the value contained in an `Option`, there are a few methods available:

```dart
final option = Some(5); // Implicit `Option<int>`

// Returns the value if it is `Some`, otherwise returns the default value
// Similar to the `??` operator
final value = option.getOrElse(() => 0); // value = 5

final triple = option.match(
    () => 0, // Called if `None`
    (value) => value * 3, // Called if `Some`
); // triple = 15

final nullable = option.toNullable(); // nullable is an `int?` with a value of 5

```

`Option` can also be converted to an `Either`, another type from Fpdart ([explained in more detail below](#either)):

```dart
// Converts to an `Either<String, int> with a value of `Right(5)`
// If the value was `None`, the `Either` would be `Left('No value')`
final either = option.toEither(() => 'No value'); 
```

Many more methods can be found in the [Fpdart docs](https://pub.dev/documentation/fpdart/latest/fpdart/Option-class.html).

### Downsides

Using `Option` can be a bit more verbose when compared to using nullable types. It adds a small amount of boilerplate and means you can not use the `??` operator to set a default value (though `getOrElse` is a similar alternative). In my experience, though, the benefits far outweigh the downsides.

## Either
### Error handling without exceptions

Error handling in Dart is painful. The main problem for me is that it's unclear as to whether a function can throw an error or not unless you read the implementation or see it mentioned in the docs. Combined with the somewhat awkward syntax for `try`/`catch` blocks, it is far from a great experience.

`Either` is a type from Fpdart that can be used to handle errors in a more functional way. It's a wrapper around a value that can either be `Left` or `Right`, which can be used instead of throwing exceptions. Typically, `Left` is used to represent an error, and `Right` is used to represent a successful result.

Let's say we have a function that can throw an error called `fetchFromDatabase`. Instead of just hoping the error is caught, we can wrap the result in an `Either`, where `Left` will be a string, used to represent an error, and `Right` will be an int, used to represent a successful result that was fetched from the database:

```dart
Either<String, int> fetchFromDatabase() async {
    try {
        // This could be a firebase call, a network request, 
        // or anything else that could throw an error
        final int data = await database.fetchData();

        // Assuming the data was successfully fetched, 
        // we return it wrapped in `Right`
        return Right(data);
    } catch (error) {
        // If an error was thrown, we return it wrapped in `Left`
        return Left(error.toString());
    }
}
```

It's now very clear that `fetchFromDatabase` can throw an error, and it's also much easier to handle the with pattern matching:

```dart
final result = await fetchFromDatabase();

// Pattern matching
result.match(
    (error) => print(error), // Called if `Left`
    (data) => print('Data found: $data'), // Called if `Right`
);
```

`Either` can also be converted to an `Option` ([explained in more detail above](#option)):

```dart
final either = Either<String, int>.of(5);

// Converts to an `Option<int>` with a value of `Some(5)`
// If the value was `Left`, the `Option` would be `None`,
// no matter what the value was
final option = either.toOption(); 
```

### Downsides

Due to exceptions being built into the language and found in many libraries, it can be difficult to be consistent with how you handle them when you introduce `Either`. It is possible to wrap the result of any throwing functions in an `Either`, but this will add more boilerplate to your code.

## Tuple
### Multiple values without a class

Returning multiple values from a function can be accomplished by creating a convenience class, but this tends to add unneeded verbosity. `Tuple2` (because it holds 2 values) is a type from Fpdart that can be used to return multiple values from a function without creating a class.

In this example, we are fetching data from a database, and we want to return the data and the time it took to fetch it. We can use `Tuple2` to do this:

```dart
Tuple2<int, Duration> fetchFromDatabase() async {
    final stopwatch = Stopwatch()..start();

    // This could be a firebase call, a network request, etc.
    final int data = await database.fetchData();

    stopwatch.stop();

    return Tuple2(data, stopwatch.elapsed);
}
```

The values can then be accessed using the `first` and `second` properties:

```dart
final result = await fetchFromDatabase();

print(result.first); // Prints the data
print(result.second); // Prints the duration
```

### Downsides

It can be unclear as to what the values represent, so it's best to use a class if you need named parameters. `Tuple2` is also limited to holding 2 values, so it's not a good fit for more complex use cases.

*Note that [Dart 3](https://medium.com/dartlang/dart-3-alpha-f1458fb9d232) will introduce the [`Record`](https://github.com/dart-lang/language/blob/master/accepted/future-releases/records/records-feature-specification.md) type, which provides similar functionality to `Tuple2` but with more flexibility, including more than two values and named parameters.*

## The Rest

Fpdart includes over 20 types and extensions, only three of which have been covered here. While the other types certainly have uses, I haven't found many situations in which my code has been signifigantly improved. Either way (pun intended), you can view examples for all of the types in the [Fpdart docs](https://pub.dev/documentation/fpdart/latest/fpdart/fpdart-library.html) and in the [example folder](https://github.com/SandroMaglione/fpdart/tree/main/example) of the GitHub repository. 🐟