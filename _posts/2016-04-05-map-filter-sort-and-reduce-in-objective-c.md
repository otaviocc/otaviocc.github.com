---
title:  Map, Filter, Sort, and Reduce in Objective-C
date:   2016-04-05 12:00:00 -0100
layout:   default
---

I strongly believe that the best way to learn something new is by experimenting and practicing it.

A few months ago I was studying *Functional Programming in Swift*. Swift is a modern, powerful, and safe language. Its syntax is so simple and elegante that it works as an invitation to dive into the Functional Programming world.

After practicing Functional Programming in Swift for a few weeks, I decided to try something a little bit different. I decided it was time to experiment with *curried functions*, *map*, *filter*, *sort*, and *reduce* in Objective-C.

Swift collections implement *map*, *sort*, *filter*, and *reduce*, so the first step on my experiment was to reimplement these in a `NSArray` category, trying to match the method signature that Swift implements.

* **Map**: given a *transform operation*, it returns a new `NSArray` where all its elements are transformed according to the operation. The *transform operation* takes an object and returns another object (that might be of a different type of the original object).
* **Filter**: given a *condition operation*, it returns a new `NSArray` where all its elements satisfy the condition. The *condition operation* takes an object and returns a boolean.
* **Sort**: given a *sort condition*, it returns a new `NSArray` where all its elements are sorted according to the condition. The *sort condition* takes two objects and returns a boolean.
* **Reduce**: given a *combine operation*, it returns a recombined object by recursively processing its constituent parts. The *combine operation* takes the initial state and an object and returns a recombined object which is of the same type of the initial state.

The `NSArray` category ended up with the following interface:

```objc
typedef id(^Transform)(id);
typedef BOOL(^Condition)(id);
typedef BOOL(^SortCondition)(id, id);
typedef id(^Combine)(id, id);

@interface NSArray (MFSR)

- (NSArray *)map:(Transform)transform;
- (NSArray *)filter:(Condition)condition;
- (NSArray *)sort:(SortCondition)isOrderedBefore;
- (id)reduce:(id)initial andCombine:(Combine)combine;

@end
```

To test the interface above and its implementation, I started with a immutable array of movies — James Bond movies — loaded from a *plist* file.

```objc
NSArray<Movie *> *movies = [[NSBundle mainBundle] moviesFromPlist];
```

Where each movie contains a title, the actor’s name who played Bond, and some additional information about about the flick. The movie interface is defined as:

```objc
@interface Movie : NSObject

@property (nonatomic, readonly) NSString *title;
@property (nonatomic, readonly) NSString *actor;
@property (nonatomic, readonly) NSInteger year;
@property (nonatomic, readonly) CGFloat boxOffice;
@property (nonatomic, readonly) CGFloat budget;
@property (nonatomic, readonly) CGFloat tomatometer;

@end
```

*Sorting* the array of movies requires a *sort condition* that takes two movies and returns a boolean that represents the relationship between them. So, for sorting all the British Secret Service agent movies by budget:

```objc
BOOL(^byBudget)(Movie *, Movie *) = ^BOOL(Movie *a, Movie *b) {
    return a.budget > b.budget;
};

NSArray<Movie *> *moviesByBudget = [movies sort:byBudget];
```

Simple.

For *filtering*, the method requires a movie and returns *true* when the movie matches the condition and *false* otherwise. Below, a simple way to create an immutable array containing the Sean Connery movies.

```objc
BOOL(^isConnery)(Movie *) = ^BOOL(Movie *a) {
    return [a.actor isEqual:@"Sean Connery"];
};

NSArray<Movie *> *conneryMovies = [movies filter:isConnery];
```

But here is the tricky part. To filter movies played by other actors, more blocks like the one above would be required. The *functional* way to address this is using curried functions — popular technique in Swift (and in other Functional Programming languages).

Since *filter* expects a block that takes a movie and returns a boolean, another function is required, where its output is a function matching this signature.

The new function takes a string (actor’s name) and returns a function that takes a movie and returns a boolean.

```objc
typedef BOOL(^FuncMovieToBool)(Movie *);

FuncMovieToBool(^isActor)(NSString *) = ^FuncMovieToBool(NSString *actor) {
    return ^BOOL(Movie *movie) {
        return [movie.actor isEqual:actor];
    };
};

NSArray<Movie *> *actorMovies = [movies filter:isActor(@"Daniel Craig")];
```

Without any changes to the *filter* method signature or implementation, it’s possible to filter the array of movies to get a list of movies played by any actor on the big screen.

Finally, it’s possible to combine all these functions to achieve the desired result. Let’s say I want the movies were Pierce Brosnan played James Bond, *sorted* by ratings and *reduced* to a list.

```objc
NSArray<Movie *> *moviesByRatings = [[movies filter:isActor(@"Pierce Brosnan")] sort:byRatings];
NSString *description = [moviesByRatings reduce:@"Brosnan movies sorted by ratings:"
                                     andCombine:^NSString *(NSString *initial, Movie *movie) {
                                         return [NSString stringWithFormat:@"%@\n* %@ (Tomatometer: %@)", initial, movie.title, @(movie.tomatometer)];
                                      }];
```

Voilà:

```
Brosnan movies sorted by ratings:
* GoldenEye (Tomatometer: 82)
* Die Another Day (Tomatometer: 57)
* Tomorrow Never Dies (Tomatometer: 57)
* The World Is Not Enough (Tomatometer: 51)
```

But putting all the fun and excitement aside, would I use these methods on an Objective-C project? Probably not — `NSArray` already implements methods for sorting and filtering arrays using `NSPredicate`.

My point here was to experiment and practice *map*, *filter*, *sort*, *reduce*, and *curried functions*. Programming requires practice. And practicing these techniques and methods (and reimplementing them) in Objective-C improved the way I use them in Swift.

Keep practicing. Keep hacking.
