---
title: Contract Based Development in Swift, or: Yet Another Singleton Rant
layout: posts
comments: true

---

Recently I started working an existing Swift project. The original author
isn't able to work on it due to illness — I wish him a quick recovery and
time to relax with your family.

One task I was asked was to gain a higher decoupling.

The project had a high coupling mainly through the excessive use of
singletons — they were everywhere:

* Model layer: `User.current()`
* Networking: `APIManager.sharedManager()`
* extra methods in APPDelgate
* …

And the usual suspects:

* `NSNotificationCenter.defaultCenter()`
* `UIApplication.sharedApplication()`

## Why are singletons creating coupling? Apple uses them too!

Well, we know that cocoa contains quite a few singletons in the
`sharedSomething()` fashion, but not the existence of singletons are
problematic, but how we use them — and I don't know how Apple's engineers
uses them.

But I know that singletons seduce us to take short circuits — those of the
dangerous kind: We use singletons to fulfill dependencies from within a
class. We might be tempted to write this:

{% highlight swift %}
class ActivityViewController: UIViewController{

    //....

    func buttonTappedInActivityCell(cell: ActivityTableCell) {
        if let friendOrUserId = cell.activity?.actorReferenceId {
            User.current()!.befriend(friendOrUserId, success: nil, failure: nil);
        }
    }
    //....

}
{% endhighlight %}

This is problematic as this hides the dependencies of `ActivityViewController`,
You'll have to scan the whole code to find them.  
Again: it is not the singleton itself, that is problematic. `User.current()` is
quite useful if you don't want to write CoreData query code over and over again.  
The problem is that we use the singleton as a singleton from with-in a class.

But we could use it from outside the class:  
We define a property that we write our singleton to. This will increase the
visability of the dependency immensly.


{% highlight swift %}
class ActivityViewController: UIViewController{
    var currentUser: User?
    //....

    func buttonTappedInActivityCell(cell: ActivityTableCell) {
        if let friendOrUserId = cell.activity?.actorReferenceId {
            currentUser!.befriend(friendOrUserId, success: nil, failure: nil);
        }
    }
    //....

}
{% endhighlight %}

We just have to pass in an user object before you bring the view controller
on the screen. This user object could be the singleton.

{% highlight swift %}
activityViewController.currentUser = User.current()
{% endhighlight %}

This is simple, but you will gain so much if you stick to it:
* higher decoupling
* better testability, as it become dead simple to pass a mock object instead

## Contracts

Now I want to take it a step further:

In one of the view controllers I found 7(!) different singletons and I realized
after the refactoring described above that the visibility of the dependencies
wasn't high enough, as they where somehow lost in the long list of properties.  
Also I needed a way of setting the deoendencies for different view controllers
from within a custom UITabBarController subclass. I didn't want to check for each
selected viewcontroller if it was a certain subclass and set the properties
accordingly, as this would had introduced new coupling, the tabbr controller
must now the name and the properties of each its viewcontrollers.

I introduced contracts: A viewcontroller that needs to know the current user
would implement a `Current User Contract` and share this information, the tabBar
controller would access this and set the current user accordenly.

## implementation of Contracts

We are using contracts in cocoa ever since — we just use the name very often:
A delegate promises to fulfill a certain protocol. This is a contract.

In Swift we need class protocols for that:

{% highlight swift %}
protocol CurrentUserAccessor : class{
    var currentUser : User? {set get}
}
{% endhighlight %}

That's it. That is our contract.  
As  we see our current `ActivityViewController` does already fulfill it — it
just doens't give this information to the tabBar controller.  
To change that, we just add the contract's name to the protocol list


{% highlight swift %}
class ActivityViewController: UIViewController, CurrentUserAccessor {
  var currentUser: User?
{% endhighlight %}


Now before my tab bar controller brings a view controller to the screen, it will
run this method to fulfill the viewcontrollers contracts, without even knowing
them. It just knows the contracts:

{% highlight swift %}

func configureTargetViewController(viewController: UIViewController?){
    if let tvc = viewController {

        if let apiManagerAccessor = tvc as? APIManagerAccessor{
            apiManagerAccessor.apiManager = apiManager
        }

        if let readerOpeningViewController = tvc as? ReaderOpener{
            readerOpeningViewController.readerManager = readerManager
        }

        if let applicationAccessor = tvc as? ApplicationAccessor{
            applicationAccessor.application = UIApplication.sharedApplication()
        }

        if let currentUserAcceesor = tvc as? CurrentUserAccessor{
            currentUserAcceesor.currentUser = User.current()
        }

        if let notificationCenterAccessor = tvc as? NotificationCenterAccessor{
            notificationCenterAccessor.notificationCenter = NSNotificationCenter.defaultCenter()
        }
    }
}
{% endhighlight %}


I think, this is pretty cool and I apply it for other parts of the code as well.



{% highlight swift %}
override func prepareForSegue(segue: UIStoryboardSegue, sender: AnyObject?) {
    if let currentUserAcceesor = tvc as? CurrentUserAccessor{
        currentUserAcceesor.currentUser = User.current()
    }


    if let apiManagerAccessor = segue.destinationViewController  as? APIManagerAccessor{
        apiManagerAccessor.apiManager = self.apiManager
    }
}
{% endhighlight %}
