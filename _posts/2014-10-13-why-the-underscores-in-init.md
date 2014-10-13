---
layout: post
title: Why the Underscores in init?
---

You have probably seen this before:

```objc

- (instancetype) init
{
  self = [super init];
  if (self)
  {
    _friends = [[NSArray alloc] init];
  }
  return self;
}
```

The little underscores in front of variables. If you pay closer attention you will notice that they only occur in two possible places. Either in accessor methods or in the `init` method. The underscore is a convention that means this variable is an [instance variable](http://blog.ablepear.com/2010/04/objective-c-tuesdays-instance-variables.html). Accessing the instance variable in an accessor makes sense...that's kind of their job. Accessing them from `init` makes a lot less sense. As up standing object oriented programmers we know that you should only access instance variables through accessor methods specifically made for that purpose. In Objective-C this means we generally access the instance variables using the `@property` directive with code like `self.firstName`. Lets take a dive into why we use instance variables in our `init` methods.

# Because Apple Said So

> The only places you shouldn’t use accessor methods to set an instance variable are in initializer methods and dealloc. 

[Apple said](https://developer.apple.com/library/mac/documentation/Cocoa/Conceptual/MemoryMgmt/Articles/mmPractical.html) so is a really great reason to do things. While I'm not advocating blindly following someone for any reason, Apple generally knows what they are doing when it comes to iOS. Either way, just following what someone says isn't good enough for me. I wanted to go deeper.

# Nobody Expects The Subclass's Setter

The main reason we use instance variables instead of the actual setter is because of subclassing. Here is an example of things going poorly by using setters instead of directly accessing the instance variable. If we think of a class called `Animal` with a property called `noise` we would initialize this class like this:

```objc
- (instancetype) init
{
  self = [super init];
  if (self)
  {
    self.noise = @"rawr";
  }
  return self;
}
```

This will probably work without any odd side-effects or anything going wrong. That is until we subclass our `Animal` class with `Dog`. Let's say in `Dog` we modify the setter to do a bit more work then just setting the variable. Let's make the `Dog` class's noise setter to grab it's name and include that in the noise it makes. Something like this:

```objc
- (void) setNoise: (NSString *)noise
{
  NSString *dogNoise = [NSString stringWithFormat:@"%@ %@ %@", noise, self.name, noise]; // for fido outputs "noise fido noise"
  [super setNoise:dogNoise];  
}
```

Now let's write the `init` method for our `Dog` class.

```objc
- (instancetype) init
{
  self = [super init];
  if (self)
  {
    self.name = @"Fido";
  }
  return self;
}
```

Now in our code let's do `Dog *fido = [[Dog alloc] init];`. If we take a look at what the `noise` property is of `fido` is we see it's `@"rawr (null) rawr"`. That's unexpected! This can be a bit hard to understand what happened but let's follow the chain of methods that we created. First things first we call the `init` instance method on the Dog class. We are then encountered with this line of code: `self = [super init]`. This kicks us up to the `init` instance method in the `Animal` class which calls the `noise` setter on `self`. The question is...what is `self` in this case? `Self` is an instance of the `Dog` class! So that brings us to the setter created in the `Dog` class. The setter in the `Dog` class grabs the `name` property...which hasn't been set yet. That's how you get the `nil` in for the name.

## Semi-Completed Self

So the first issue here is you are never sure when a class you write will get subclassed. We need to be aware of those possible issues like the one above. To do this we just never use `self` until `self` is completely finished being created. During our `init` method we are in a state of flux. Our instance has just started being created, but hasn't finished. Because of this, we can't rely on `self` being completely set up like we can elsewhere in our class. So, in the `init` method, we *must never* use a method on `self`. 

There is also a ton of design considerations about writing a setter such as that...but that's another issue. You never know who's subclassing your code!
