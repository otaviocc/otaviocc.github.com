---
title:  Kotlin's distinctBy in Swift
date:   2018-03-08 14:00:00 -0100
category: Programming
---

A few months ago I was working on a new feature for the new iOS app we're building at work and when I finished it I decided to the check how the Android team had implemented the same functionality, to make sure both apps had exactly the same behavior implemented. 

And they did; both apps had the same behavior implemented, but the implementations were completely different. Since the Swift implementation was shorter and easier to read, I decided to give Kotlin a try and refactored the Android code to be *closer* to the code we have on iOS.

When working on the Android code I learned about Kotlin's [distinctBy](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.collections/distinct-by.html) method,

```kotlin
inline fun <T, K> Iterable<T>.distinctBy(selector: (T) -> K): List<T>
```

which, as the documentation states, returns a list containing only elements from the given array having distinct keys returned by the given selector function.

To illustrate it, a simple example from Kotlin's [tests](https://github.com/JetBrains/kotlin-native/blob/a69def42542cf250ee26d9c1c09544a720809df5/backend.native/tests/external/stdlib/collections/SetOperationsTest/distinctBy.kt) where the returned list contains the first word matching their unique length.

```kotlin
import kotlin.test.*

fun box() {
    assertEquals(
        listOf("some", "cat", "do"),
        arrayOf("some", "case", "cat", "do", "dog", "it").distinctBy { it.length }
    )
}
```

So I decided to reimplement it in Swift:

```swift
extension Sequence {
    func distinctBy<T: Hashable>(_ keyPath: KeyPath<Element, T>) -> [Element] {
        var set: Set<T> = []
        var list: [Element] = []

        forEach { element in
            let value = element[keyPath: keyPath]
            if set.insert(value).inserted {
                list.append(element)
            }
        }

        return list
    }
}
```

Below, the same example from Kotlin's tests, but this time in Swift:

```swift
let words = ["some", "case", "cat", "do", "dog", "it"].distinctBy(\.count)
// ["some", "cat", "do"]
```

Not complex, but a great tool to have in the utility belt.