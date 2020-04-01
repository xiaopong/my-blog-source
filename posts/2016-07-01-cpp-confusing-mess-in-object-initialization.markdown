---
title:  C++'s confusing mess in object initialization
author: xp
---
C++ is a powerful programming language, but it is a hugely complex and convoluted programming language. Everyday, we see people struggle to learn the syntax, the semantics and the little quirks to try to cross the learning chasm. It's not just difficult to newbies. Even for old hands, you have to check and re-check every line. After so many years of programming, I still can not feel being in my elements when coding in this satanic language.

This feeling has never been stronger than in the last few days, while I was guiding a programmer through the C++ rough sea. He has years of experiences in Java, Ruby, Go, Javascript, and can handle C pretty decently. And he is having a hard time with just initialization of objects. I sat back, and something hit me. Given the confusing mess of object initialization in C++, I would have the same problem of understanding if I were to learn it now.

Let's look at the many ways you can initialize an object. First of all, you can initialize an object with the "=" symbol, such as:

```
int x1 = 42;
```

Remember we all had to re-learn the meaning of the "=" sign when we first learned programming? No, it does not mean equal, we are assigning a value to the variable on the left hand side. To help us understand the concept, a lot of teachers even drew a box, and made a gesture to show that we actually put the value in that box.

So far so good. Then, in C++, we also can initialize an object with the parentheses, such as:

```
int x2(42);
```

So what's the difference? Well, there's not much difference here, at least, not this case. And if there is one, I can't tell. And I am pretty sure the huge majority of C++ programmers couldn't either. Ok, to be theoretically and semantically precise, the first one is a copy initialization, and the second one is a direct initialization. There might be differences when we deal with user-defined classes, however, they don't have any difference in this case.

And then, there's a third way, with curly braces, touted as the uniform way of object initialization in the latest C++ standards. A uniform way of object initialization which doesn't work everywhere, after all. We'll see why.

```
int x3{42};
```

And there is even a fourth way, by combining the curly braces with the equal sign:

```
int x4 = {42};
```

To be frank, I don't understand the purpose of having yet this fourth way. It's probably just because you can initialize object with equal sign and curly braces, then it just flows that combining the two should be legal as well.

It is confusing enough already to have four different ways just to initialize a freaking simple object, and so you think to yourself, they should at least work the same way everywhere, right? You can't be more wrong. Here's how this new C++ apprentice came to me crying, why is his C++ compiler balking at the following code?

```
class MyClass
{
public:
    MyClass() {};
    MyClass(int a, int b, int c): x1(a), x2(b), x3(c) {}

    int getX1() { return x1; }
    int getX2() { return x2; }
    int getX3() { return x3; }
    ...

private:
    int x1 = 0;
    int x2{0};
    int x3(0);
};
```

The compiler just throws a very subtle error message saying

```
error: expected identifier before numeric constant
```

on `x3`. Seriously, why? I had never found a satisfactory answer to this issue, I just have to remember it as an exception to the rules.

Now, let's look at the object initialization of our MyClass that we defined above, with two constructors. You can do:

```
MyClass c1(1, 2, 3);
```

And you can do:

```
MyClass c2{1, 2, 3};
```

You can even do:

```
MyClass c3 = {1, 2, 3};
```

And of course, you can certainly do:

```
MyClass c4;
```

Or you can do:

```
MyClass c4{};
```

But if you try to do this:

```
MyClass c5();
```

You will find yourself some surprises. The compiler would not complain about the initialization code above, but you don't exactly get what you think you'd get. You'd think you got an object, and when you try to invoke of the methods, such as:

```
int x1 = c5.getX1();
```

Then the compiler will throw this mystic error message in your face:

```
error: request for member ‘getX1’ in ‘c5’, which is of non-class type ‘MyClass()’
```

What are you talking about? I got an object of type `MyClass`, and I'm invoking a method of `MyClass`. Except, in this case, `c5` is not exactly what you'd think you got. You thought you've got an initialized object, but the compiler thought you have made a function declaration that returns an object of type `MyClass`.

Alright, so everyone calls this situation a vexing parse. For me personally, if you design a programming language, and you end up with a situation like this semantic confusion (not to say that this is the only exception to the parsing rules in C++!), that's a sign of failure. Right, right, you wouldn't call C++ a failure, it's being used by millions of programmers, running millions of devices. But semantically, it's still a failure.

And that's not all. Remember the uniform way of initialization with curly braces that we talked about? Let's look at the following situation.

```
float x1=1.0, x2=2.0, x3=3.0;
int sum1 = x1 + x2 + x3;
int sum2(x1 + x2 + x3);
```

We are just summing three floating point numbers. Everything is ok, although we lost some precision. But what if we do the following?

```
int sum3{x1 + x2 + x3};
```

Then the compiler just throws the error

```
error: narrowing conversion of ‘((x1 + x2) + x3)’ from ‘float’ to ‘int’ inside { }
```

Heh, why not? Since you can do narrowing on the above two cases, why not on this one? What's the difference? The only reason I can find from the C++ committee members was that, if we do not allow narrowing in the first two options, we would break too much legacy code, and therefore, they are allowed. But in the third one, the so-called uniform way of initialization, which was introduced in C++11, narrowing should not be allowed. If you think this is double standards, it really is. And it made things unnecessarily complicated. This is extremely annoying, especially when you deal with user-defined classes which have multiple contructors, it's getting convoluted.

However, it depends on which compiler you are using. If you are using the g++ compiler, it makes, to me, a more sensical decision. It only warns you about the narrowing issue, but will happily compile the code anyway. You might think that this is a bad decision on the part of g++, as it just hides away implicit narrowing conversions, which could cause some undefined system behavior later down the road. I agree, I'm not saying it's a good decision, I'm just saying it's a more sensical decision. If we wanted to be strict, let's apply the rules everywhere, and make the rules uniform (since we called it uniform anyway), and let's not have exceptions all over the place, as it just boggles the mind for no obvious gain.

And here, I haven't even talked about the issue with the "=" sign initialization of uncopyable objects. And then you have to read the code of every class you want to use, to see which one is defined as uncopyable and which one is copyable. This is demanding a lot. It took me quite a bit of time to explain to this C++ apprentice, and we haven't touched the concepts of copyable vs movable, and things like that.

And that's why I don't use C++ to develop large systems anymore, it's just not a productive language to work with. I still write codes in C++, but only limited to very specific areas where I want to focus on performance. And I really don't envy the job of those C++ compiler developers.
