# Pragmatic And Simple DI In Golang (updated)

## Table Of Contents

1. [What it is about](#what-it-is-about)
2. [Advantages over other approaches](#advantages-over-other-approaches)

## What it is about

Welcome to this... well - call it tutorial. So what do we want to achive? 
We want [Dependency Injection](https://de.wikipedia.org/wiki/Dependency_Injection). 

What else?

1. Testability (in parallel as well)
2. Have as little not testable code as possible
3. Easy access to dependencies
4. Elegance
5. Simplicity

Sure, 4 and 5 is in the eye of the beholder but read first and judge then :-) 

In fact this approach of using DI in Go is super simple. What do we need? 
First something that has a dependency to something else.

~~~Go
package samplePackage

type AnInterface interface {
    doSth()
}

type SampleService struct {
    Dep AnInterface
}

type SampleDependency struct {
    //...
}

func (dep *SampleDependency) doSth() {
    // Fullfills AnInterface
    // ...
}
~~~

Now the DI functionality. What does it contain? a package (let's call it "di")
and one function and one variable per dependency

~~~Go
package di

var sampleDependency samplePackage.AnInterface
var sampleService *samplePackage.SampleService

func SampleDependency() samplePackage.AnInterface {
    if sampleDepdendency != nil {
        return sampleDepdendency
    }
    
    sampleDependency = &samplePackage.SampleDependency{}
    
    return sampleDependency
}

func SampleService() *samplePackage.SampleService {
    if sampleService != nil {
        return sampleService
    }
    
    sampleService = &samplePackage.SampleService{
        Dep: SampleDependency(),
    }
    
    return sampleService
}
~~~

What are we doing here? We have one variable per service the instances 
are stored in. Wenn you call e. g. di.SampleService() you get an instance 
of the desired service.

But you are right. We have some boilerplate code for each service: One function and 
variable with the check if the service has already been defined. 

But to be honest - This is just a little cost. Let's assume you want to
be able to choose if you want to have a new instance:

~~~Go
package di

var aService *samplePackage.AService

func AService(newInstance bool) {
    if aService != nil && !newInstance {
        return aService
    }
    
    // ...
}
~~~
Isn't this simple? You are absolutely free to handle your dependencies. There is just one problem I met: 
What if the method is called twice at the same time? You will get two different instances of aService.
 
 So we add a mutex to aService and lock it on access (Do not be afraid - This is the last step):
 ~~~Go
 package di
 
 var aService struct {
    sync.Mutex
    instance *samplePackage.AService
 }
 
 func AService(newInstance bool) {
     aService.Lock()
     defer aService.Unlock()
     if aService.instance != nil && !newInstance {
         return aService.instance
     }
     
     // ...
 }
 ~~~
 Problem solved! One could maybe wrap this again but come on...

One more thing: I am coming from the PHP corner and since meanwhile it is mostly used
oop style I love encapsulation.

Let's have a look at our SampleService from the example above:

~~~Go
package samplePackage

type SampleService struct {
    Dep AnInterface
}
~~~
As you can see "Dep" is public. But for some good reasons we should have it private. 
How can we achive this with our DI approach? Nothing simpler then this: Let's create
a constructor first:
~~~Go
package samplePackage

func NewSampleService(dep AnInterface) *SampleService {
    return &SampleService{
        dep: dep,
    }
}

type SampleService struct {
    dep AnInterface
}
~~~
And suddenly our dependency is private - Awesome :-) And the di package:
~~~Go
package di

func SampleDependency() samplePackage.AnInterface {
    // ...
}

func SampleService() *samplePackage.SampleService {
    // ...
    sampleService = samplePackage.NewSampleService(SampleDependency())
    // ...
}
~~~

And how can we use this now? Here a real-world example using [Cobra](https://github.com/spf13/cobra):

~~~Go
package cmd

import (
    // ...
)

func CmdAddUser(userMgmt *auth.UserMgmt) *cobra.Command {
    var username string
    var password string
    command := &cobra.Command{
        Use:   "adduser",
        Short: "Adds a user",
        Long:  `Adds a user with username and password`,
        RunE: func(cmd *cobra.Command, args []string) error {
            if (username == "") {
                return errors.New("Username may not be empty")
            }
            if (password == "") {
                return errors.New("Password may not be empty")
            }
            err := userMgmt.AddUser(username, password)
            if (err != nil) {
                return err
            }

            fmt.Println("User has been created")
            return nil
        },
    }

    command.Flags().StringVarP(&username, "username", "u", "", "username")
    command.Flags().StringVarP(&password, "password", "p", "", "password")

    return command
}

func init() {
    RootCmd.AddCommand(CmdAddUser(di.UserMgmt()))
}
~~~
As you can see we have the absolute minimum of code within the di package (the UserMgmt method). 
The rest is all here and therefore easily testable.

At the end: It is 
* Just have a look at the points at the top of this page
* And: Everyone - even newbies - see immediately what the code is about - 
Neither any magic nor super sophisticated architecture necessary :-)

## Advantages over other approaches

What are the alternatives?
1. No DI at all - Quick and eezy peezy at the beginning but leading to dead end - 
Almost untestable and full of implicit dependencies
2. DI containers

There are some libraries around, e. g.
* [Godi](https://github.com/shawnburke/godi)
* [Go-IOC](https://github.com/shelakel/go-ioc)
* ...

The upside: It looks all so nice'ish and encapsulated but at the end you harly save lines of code
and it is full of type assertion magic. It is like working against the static typing of Go - Of course not completely
but I think you know what I mean.

3. Another approach is described here: https://www.youtube.com/watch?v=xlU_IhCBT84

If you do not want to watch the entire video here the essence:

~~~Go
var NewDependency func() *Dependency {
    return &Dependency{}
}

func DoSth() {
    NewDependency().DoSthElse()
}
~~~
So far so good. But what about testing DoSth? First: You need to know about the internal
structure of DoSth (evil implicit dependencies). Second: You need to mock NewDependency.
This could happen like this (We do not care about the packages now):
~~~Go
func TestDoSth() {
    defer func(original func() *Dependency) {
        NewDependency = original
    }(NewDependency)
    
    NewDependency = func() *Dependency {
        // return some mocked stuff
    }
    
    DoSth() // ... And expect the result in the mock
}
~~~

Can you see the problem? There are no parallel executed tests possible anymore.

## The End

I do not want to say that our approach is the best thing you could ever
use. There is a little repeating boilerplate code but from my point of view
the upsides are worth it. 

At [Loopline Systems](http://www.loopline-systems.com) we used it by now
and we will use it for our upcoming Go applications.

That is it so far. I hope this tutorial could help you. If you want to complain about something or
have something to add just let me know :-)

