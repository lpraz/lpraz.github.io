---
layout: post
title: "Why can't I use WebDriverWait with IWebElement in Selenium/C#?"
date: 2019-09-02 09:15:00 -0500
categories: c-sharp selenium quality-assurance
---
In Selenium (or at least its C# bindings), `WebDriverWait` is a class that sees a lot of use. In fact, the pattern of explicit waits, [which is recommended over the alternative of implicit waits][stackoverflow-reijmersdal], relies on this class. However, it has a limitation that becomes glaring as your [page objects][fowler] become more deeply nested. In fact, it's all there in the name: `WebDriverWait` can only be used with `IWebDriver`.

Let me provide an example to illustrate how this might become a problem. Say you have a class representing a [Bootstrap modal][bootstrap-modal], that starts out something like this:

```csharp
public class Modal
{
    private readonly IWebElement _root;

    public Modal(IWebElement root)
    {
        _root = root;
    }
    
    // other modal contents (subclass?)
}
```

In fact, this is how I start out a lot of page objects (I'll save any further detail for another article).

Nearly every Bootstrap modal has a "x" button in the top-right corner for closing the modal. To be able to close modals from this page object, we'll add a `Close()` method that clicks on that button and waits for the modal to be invisible. We want it to look something like this:

```csharp
public void Close()
{
    _root.FindElement(By.ClassName("close")).Click();
    // uh-oh...
    new WebDriverWait(_root, TimeSpan.FromSeconds(1)).Until(e => !e.Visible);
}
```

But, of course, we can't use `WebDriverWait` with anything other than an `IWebDriver`, and to make the `IWebDriver` accessible by the `Modal` class isn't the greatest ever for separation of concerns under this way of organizing page objects. hence our problem. In addition, `IWebDriver` and `IWebElement` both implement a common `ISearchContext` interface which provides, among other things, the `FindElement(By)` and `FindElements(By)` methods that can be used with both interfaces, and [the code for `WebDriverWait`][github-webdriverwait] doesn't appear to use anything `IWebDriver`-specific, either. So why can't we use `WebDriverWait`, or even a generally-applicable `SearchContextWait`?

A bit of Google-fu brought me to [this Github issue][github] opened by NickAb in 2015 which asked the same question. The issue was ultimately closed in 2016 by Jim Evans, due to the amount of "dependent code in the ecosystem" to change `WebElementWait`'s signature to take `ISearchContext`s instead of `IWebDriver`s. However, it's easy to implement one's own `WebElementWait`, and Evans even provides a code snippet (reproduced here, without comments, for convenience) detailing this:

```csharp
public class WebElementWait : DefaultWait<IWebElement>
{
    public WebDriverWait(IWebElement element, TimeSpan timeout)
        : base(element, new SystemClock())
    {
        this.Timeout = timeout;
        this.IgnoreExceptionTypes(typeof(NotFoundException));
    }
}
```

To make a `SearchContextWait` instead of a `WebElementWait`, simply change `IWebElement` to `ISearchContext` wherever it appears.

However, this isn't exactly idiomatic Selenium. Although it barely adds any extra code on top of the framework, it's still something that isn't included by default with the framework itself. If you want to use something that already exists in Selenium, you may want to look into `FluentWait`s as well, which use more of a fluent API approach to waiting for things in Selenium. For the curious, StackExchange SQA user LittlePanda details the differences differences between implicit, explicit and fluent waits [here][stackoverflow-littlepanda], and Github user up1 has some [code snippets][github-gist] if you just want to see what they look like in practice.

[stackoverflow-reijmersdal]: https://sqa.stackexchange.com/questions/12583/is-it-a-bad-practice-to-use-implicit-wait-in-selenium-webdriver-should-one-use
[github]: https://github.com/SeleniumHQ/selenium/issues/851
[fowler]: https://martinfowler.com/bliki/PageObject.html
[bootstrap-modal]: https://getbootstrap.com/docs/4.0/components/modal/
[stackoverflow-littlepanda]: https://sqa.stackexchange.com/questions/12866/how-fluentwait-is-different-from-webdriverwait
[github-gist]: https://gist.github.com/up1/d925783ea8e5f706f3bb58c3ce1ef7eb
[github-webdriverwait]: https://github.com/SeleniumHQ/selenium/blob/07a18746ff756e90fd79ef253a328bd7dfa9e6dc/dotnet/src/webdriver/Support/WebDriverWait.cs