---
title:  On deleted function in C++
author: xp
---
In C++, when you don't want people to call any specific class method, you would probably define the method as private. But sometimes, it's not always possible to do, or it's not convenient. Sometimes, you have a method with a certain parameter, but the caller would call the method by passing in a different type of parameter, with a different intention. By different type of parameter here, I mean a different type of parameter from the one you intended, when you defined the method.

However, given the C baggage in C++, the compiler would not able to differentiate. For example, we have the following class definition:

```
class MyClass
{
public:
    void withIntOnly(int x) {
        // Do something with x
    }
};
```

Your intention was to have caller pass in an integer only. But there's no way for you to prevent people from passing in a different type of parameter. The following codes are all valid:

```
enum Color { Blue, Red, Green };
MyClass c;
c.withIntOnly(42);
c.withIntOnly('x');
c.withIntOnly(true);
c.withIntOnly(false);
c.withIntOnly(Blue);
c.withIntOnly(3.1415);
```

As we can see, the intentions of the caller are very different for each call, but to the compiler, they are perfectly valid method calls, except maybe the last one which needs narrowing conversion, but it is still a valid method call.

To prevent this, we had to overload the method with every single type that could be converted to an integer and make the overloaded methods private, such as the following codes:

```
class MyClass
{
public:
    void withIntOnly(int x) {
        // Do something with x
    }
private:
    void withIntOnly(char c) {}
    void withIntOnly(bool b) {}
    void withIntOnly(float f) {}
};
```

This would make the method calls above fail to compile. With C++11, there's another way to filter out the unwanted overloads with the `delete` keyword. So the class definition would become:

```
class MyClass
{
public:
    void withIntOnly(int x) {
        // Do something with x
    }
    void withIntOnly(char) = delete;
    void withIntOnly(bool) = delete;
    void withIntOnly(float) = delete;
};
```

This would achieve the same effects as the private overloaded methods. You could do this with the copy constructor, if you want to make your class uncopyable. And this latter way is the recommended way, because it is supposedly better. Honestly, this is just a question of personal taste. From a technical point of view, I don't see any difference at all.

Only at this point that you start to really appreciate the strict static type checking and/or type inference in other programming languages. We should only need to define our class as the first definition above, and the compiler should handle the rest of the work, and make sure that the method can't be invoked with undefined effect.

You might argue that this is not a C++ problem, because it's an issue carried over from the C baggage. True, but since the C++ designers decided to make it "C-compatible", we got to live with the consequences. All I'm saying here is that, from a programming point of view, that really sucks.

Therefore, even with this feature of deleted function, we only shuffle the problem around, we have not really resolved it.
