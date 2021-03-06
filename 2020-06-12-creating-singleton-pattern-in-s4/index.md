# Creating singleton pattern in S4

In my previous post, we have been creating a package with a connection to the database. However, this connection was exposed as a global object, which could then be freely accessed by anyone. I didn't like this approach, so I have decided to try to find a way how to encapsulate it and hide this object from the user of the package.

As someone, who used to work with Java and Scala before transitioning to R, singleton immediately came into my mind. However I have not worked with object-oriented programming in R. So I have studied a little bit about all the different class systems in R. From all the possible class systems, reference classes are the most similar to the Java classes as the objects are for example mutable. However, this mutability does not go hand in hand with the more functional design in R, so I have decided to create something similar to a singleton in the S4 system.

# Singleton

First of all, we need to create new S4 class.

```r
setClass("singleton")
```

Then we need to create a constructor for the class. Singleton pattern should create a new object only if it does not exist yet, otherwise, it should return an already existing instance of the object. For this, we can use function closure. In this closure, we will have a `list` which will keep all instances of the already created objects. We use a `list` because we might want to create a sub-class of the singleton. 


```r
closureFun <- function() {
  singleton <- list()
  function(.Object, ...) {
    arguments <- list(...)
    if (is.null(singleton[[class(.Object)]])) {
      for (slotName in intersect(slotNames(.Object), names(arguments))) {
        slot(.Object, slotName) <- arguments[[slotName]]
      }
      singleton[[class(.Object)]] <<- .Object
    }
    singleton[[class(.Object)]]
  }
}
setMethod(f="initialize", signature(.Object = "singleton"),closureFun())
```

And that's it. Now, we can create two subclasses, which will inherit from a singleton. These classes will save the object created with the first call of `new` function and each subsequent call will return the same object. We can try this with the following example.


```r
setClass("subClass1", slots = c(name = "character"), contains = "singleton")
setClass("subClass2", slots = c(age = "numeric"), contains = "singleton")

type1child1 <- new("subClass1", name = "abc")
type2child1 <- new("subClass2", age = 5)
type1child2 <- new("subClass1", name = "newName")
type2child2 <- new("subClass2", age = 55)
```

But as mentioned earlier, this object is not 100% true singleton, as the change on this object won't be propagated to the object saved in the list, so you can have desynchronized objects. To be safe, you should never change the object outside of the constructor function.


