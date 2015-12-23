---
title: SOLID Swift: Dependency Inversion Principle
layout: posts
comments: true

---

This is he first article in a series about applying the SOLID principles in Swift,
mainly for the iOS and Mac platforms. I will not start with the *S*, but with the *D*
— The Dependency Inversion Principle, DIP
<!--break-->

What does the DIP state?

> High-level modules should not depend on low-level modules. Both should depend on abstractions.  
>
> Abstractions should not depend on details. Details should depend on abstractions.

In context of Swift we could say

> High-level types should not depend on low-level types, both should depend on protocols.  
>  
> Protocols should not depend on details. Details should depend on protocols.

## Evolution of a struct

lets say we have a struct to keep track of the hours we worked each day.

{% highlight objc %}
struct TrackedHours {

    let date: NSDate
    let duration: Double

    init(date: NSDate, duration: Double) {
        self.date = date
        self.duration = duration
    }
}
{% endhighlight %}

Now this struct does want we want, but it might be a little tedious to use, as
we first must create a NSDate instance to pass it in.  

{% highlight objc %}
TrackedHours(date: { let c = NSDateComponents(); c.day = 29; c.month = 11; c.year = 2015;  return NSCalendar.currentCalendar().dateFromComponents(c)!}(), duration: 7),
{% endhighlight %}



We could add an init that takes a date string and creates a date formatter to use
`dateFromString()` to create the date.  
As `dateFromString()` can return `nil`, we introduce some error handling


{% highlight objc %}

enum HourError: ErrorType {
    case InvalidDate
}

struct TrackedHours {

    let date: NSDate
    let duration: Double

    init(date: NSDate, duration: Double) {
        self.date = date
        self.duration = duration
    }

    init (dateString:String, duration:Double) throws {
        let dateFormatter = NSDateFormatter()
        dateFormatter.dateFormat = "MMM d, y"
        let date = dateFormatter.dateFromString(dateString)
        guard date != nil else { throw HourError.InvalidDate}
        self.init(date: date!, duration:duration)
    }
}
{% endhighlight %}

Now the struct is much easier to use

{% highlight objc %}
try? TrackedHours(dateString: "Nov 29, 2015", time: 7)
{% endhighlight %}

But we have a hidden dependency, as it depends on NSDateFormatter now. Further-more
we are tied to a single date format. If the user of this struct needs to deal with
other formats, the convenient init is useless.

Instead we should inject any dependency.

Dependency *Injection*? Wasn't this article suppose to cover Dependency *Inversion*?

Well, yes: Dependency Injection (DI) is one way of conform to the Dependency Inversion Principle (DIP).

Often your hear about Dependency Injection Frameworks, Reflection,… , but seriously: It is much simple:

> «Dependency injection means giving an object its instance variables. Really. That's it.» — [James Shore](http://www.jamesshore.com/Blog/Dependency-Injection-Demystified.html)

So this means we could just add a date formatter object to our struct, but actually we don't need to write it to a property,
but that is an implementation detail. Only the convenient init needs to use the
formatter, we don't need to keep it.

{% highlight objc %}
enum HourError: ErrorType {
    case InvalidDate
}

struct TrackedHours {

    let date: NSDate
    let duration: Double

    init(date: NSDate, duration: Double) {
        self.date = date
        self.duration = duration
    }

    init (dateFormatter:NSDateFormatter, dateString:String, duration:Double) throws {
        let date = dateFormatter.dateFromString(dateString)
        guard date != nil else { throw HourError.InvalidDate}
        self.init(date: date!, duration:duration)
    }
}
{% endhighlight %}

Now we can pas in a date formatter with the format we need.

{% highlight objc %}
let dateFormatter = NSDateFormatter()
dateFormatter.dateFormat = "MMM d, y"
try? TrackedHours(dateFormatter: dateFormatter, dateString: "Nov 29, 2015", duration: 7),
{% endhighlight %}

Now our struct complies to Dependency Injection, and in most cases this is just as good as perfect —
but in regards to Dependency *Injection* it isn't perfect. Our struct depends on a concrete
NSDateFormatter, rather than on a protocol or abstraction. Sou we should type the `dateFormatter`
parameter in the `init` with an protocol that `NSDateFormatter` implements. But there
isn't any.

Well, let us create one using an `extension`

For now e will give the protocol the only method we need from `NSDateFormatter`

{% highlight objc %}
protocol DateFormatting {
    func dateFromString(string: String) -> NSDate?
}
{% endhighlight %}

and create an empty extension to indicate that NSDateFormatter implements the
protocol.

{% highlight objc %}
extension NSDateFormatter : DateFormatting {
}
{% endhighlight %}

thats it! Now we can type the `dateFormatter` parameter as `DateFormatting`

{% highlight objc %}
struct TrackedHours {

    let date: NSDate
    let duration: Double

    init(date: NSDate, duration: Double) {
        self.date = date
        self.duration = duration
    }

    init (dateFormatter:DateFormatting, dateString:String, duration:Double) throws {
        let date = dateFormatter.dateFromString(dateString)
        guard date != nil else { throw HourError.InvalidDate}
        self.init(date: date!, duration:duration)
    }
}
{% endhighlight %}

But why is it better to type it as the protocol and not the concrete type `NSDateFormatter`?

Imagine your data about the tracked hours comes from different sources in different formats.
Now it is easy to create an instance that conforms to `DateFormatting` by keeping
a list of date formatter and ask each of it to create a date from a string, it
would be a light weight proxy.  
