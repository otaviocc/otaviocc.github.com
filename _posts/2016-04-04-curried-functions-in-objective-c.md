---
title:  Curried Function in Objective-C
date:   2016-04-04 12:00:00 -0100
category: Programming
---

[Function Currying](https://en.wikipedia.org/wiki/Currying) and *uncurrying* have been extensively discussed since Apple introduced Swift in 2014. I won’t extend myself on this topic because there are some great resources about it on the Internet.

But, shortly, what is currying? It’s a technique that transforms a function that takes several arguments into a sequence of functions, each with a single argument. The result is a chain of functions returning functions. It’s beautiful, it’s mathematics!

Below, a simple example in Swift:

```swift
func add(_ a: Int) -> (Int) -> (Int) -> Int {
    { b in { c in a + b + c } }
}

let addTwo = add(2)               // Int -> Int -> Int
let addFive = addTwo(3)           // Int -> Int
let result = addFive(4)           // 9

print("result = \(result)")       // result = 9
```

and the same example in Objective-C:

```objc
typedef NSInteger(^FuncInt2Int)(NSInteger);
typedef FuncInt2Int(^FuncInt2Int2Int)(NSInteger);

FuncInt2Int2Int(^add)(NSInteger) = ^FuncInt2Int2Int(NSInteger a) {
    return ^FuncInt2Int(NSInteger b) {
        return ^NSInteger(NSInteger c) {
            return a + b + c;
        };
    };
};

FuncInt2Int2Int addTwo = add(2);        // Int -> Int -> Int
FuncInt2Int addFive = addTwo(3);        // Int -> Int
NSInteger result = addFive(4);          // 9

NSLog(@"result = %ld", result);         // result = 9
```

The Swift version is minimal and elegant. But the Objective-C one has its charm, doesn’t it?

So, which one do you like better?
