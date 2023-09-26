---
layout: post
title: "Only Log When Things Go Wrong"
date: 2019-09-09 21:00:00 -0500
categories: software-engineering best-practices
---
Logging can be a highly beneficial practice when used appropriately. Being able to look through a tidy, well-organized set of logs is massively helpful in uncovering problems with an application. However, logging can also become a problem in and of itself. For example, imagine looking through a set of logs, trying to get information about a certain user interaction in which a bug showed up. However, you have plenty of irrelevant logs which you must sift through to get to the relevant logs. This can be described as your logs' [signal-to-noise-ratio (SNR)][wikipedia-snr]: the desired information is the "signal", and everything else is the "noise".

Logging activity can be split up into two broad categories, and therefore, two ways to reduce the amount of irrelevant logging (and increase your logs' SNR). One of these is logging produced when things go wrong, and reducing the amount of this logging is easier said than done: stop things from going wrong (i.e. fix bugs). The other category, meanwhile, is logging produced when something goes right. This can include logging requests sent to a web application, or even something as simple as [trace logging][wikipedia-tracing]. When this type of logging is done, it tends to get done a lot, which has a massive impact on your SNR. If enough logging is done, it may even prevent some useful logs from being seen at all, if you use something like Microsoft's [Application Insights][appinsights-sampling] that only takes a portion of all telemetry sent to it once a threshold is reached.

Therefore, when an application is running in production, **only log when things go wrong.** This serves to increase your logs' SNR, but you also gain benefits from having simpler, more straightforward code which is easier to read, debug, fix, and add on to. You may also gain the benefit of having better separation of concerns (although, at least in object-oriented programming, logging is probably best done with a [decorator][wikipedia-decorator] class).

What about silent failures, however? Logging when things go right can still have some utility here, at least in the outset. Silent failures, however, are produced by code that pretends everything is okay, even when it's not. Here's a toy example (in C#):

```csharp
public int Factorial(int n)
{
    if (n < 0)
        return -1;
    
    if (n == 0)
        return 1;
    
    return n * Factorial(n - 1);
}
```

This sort of error handling lends itself well to C, which doesn't include any of its own error-handling facilities. However, the trouble happens when any calling code doesn't check for abnormal, out-of-range values, always accepting the result as valid and continuing right along. This can result in strange behaviour that makes debugging harder. There are other ways of error handling in C, but if you're using a language with a concept of exceptions, learn to use them. Exceptions are great because the programmer doesn't have to think about them if there's no meaningful way they can be handled. For example, the `return -1` in the above toy example could be replaced by `throw InvalidOperationException()`, and any code that passed a negative value to `Factorial` would stop before anything else might happen, even if that code wasn't explicitly listening for it.

Exceptions, however, can also be non-meaningfully handled. For example, they can be caught, silently thrown away, and have an error value returned in their place:

```csharp
public double? Divide(double numerator, double denominator)
{
    try
    {
        return numerator / denominator;
    }
    catch (DivideByZeroException)
    {
        return null;
    }
}
```

Or, worse yet for our logging SNR, they can get logged before they get thrown away as well:

```csharp
public double Divide(double numerator, double denominator)
{
    try
    {
        return numerator / denominator;
    }
    catch (DivideByZeroException e)
    {
        Console.WriteLine(e);
        return -1;
    }
}
```

Re-throwing the exception would be better for making any possible bugs more explicit. We would, after all, be logging when something goes wrong. However, if several methods in the call stack use the same pattern, the same exception is logged several times, resulting in redundant logs and an even worse SNR. The solution to this is to only catch the exception where you can meaningfully handle it. Some exceptions can be more easily handled closer to their source, but this often can't be done except at a very high level in the code, or even not at all, in which case the application would fail entirely. For example, an ASP.NET MVC web app would let exceptions propagate to the framework's exception-handling code (and perhaps use its own exception filter), while a desktop application might cancel whatever it was trying to do and show an error dialog. The key principle still stands, however: **only log when things go wrong** (and they can't be made right).

None of this is meant to discredit logging for other purposes (eg: analytics), which have their own best practices. However, appropriately handling things that go wrong, and only logging when things go wrong, makes your logs much more useful, and you may also benefit from having cleaner code to boot.

[wikipedia-snr]: https://en.wikipedia.org/wiki/Signal-to-noise_ratio
[wikipedia-tracing]: https://en.wikipedia.org/wiki/Tracing_(software)
[appinsights-sampling]: https://docs.microsoft.com/en-us/azure/azure-monitor/app/sampling
[wikipedia-decorator]: https://en.wikipedia.org/wiki/Decorator_pattern