# Pragmatic And Simple ID In Golang

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

type SampleService struct {
    Dep *SampleDependency
}

type SampleDependency struct {
    //...
}
~~~

Now the DI functionality. What does it contain? a package (let's call it "di")
and one function and one variable per dependency

~~~Go
package di

var sampleDependency *samplePackage.SampleDependency
var sampleService *samplePackage.SampleService

func SampleDependency() *samplePackage.SampleDependency {
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
Isn't this simple? You are absolutely free to handle your dependencies.

One more thing: I am coming from the PHP corner and since meanwhile it is mostly used
oop style I love encapsulation.

Let's have a look at our SampleService from the example above:

~~~Go
package samplePackage

type SampleService struct {
    Dep SampleDependency
}
~~~
As you can see "Dep" is public. But for some good reasons we should have it private. 
How can I achive this with this DI approach? Nothing simpler then this: Let's create
a constructor first:
~~~Go
package samplePackage

func NewSampleService(dep *SampleDependency) *SampleService {
    return &SampleService{
        dep: dep,
    }
}

type SampleService struct {
    dep SampleDependency
}
~~~
And suddenly our dependency is private - Awesome :-) And the di package:
~~~Go
package di

func SampleDependency() *samplePackage.SampleDependency {
    // ...
}

func SampleService() *samplePackage.SampleService {
    // ...
    sampleService = samplePackage.NewSampleService(SampleDependency())
    // ...
}
~~~

And how can we use this now? Let's assume we have a simple [Cobra](https://github.com/spf13/cobra) application:

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
As you can see we have the absolute minimum of code within the di package. 
The rest is all here and therefore easily testable.

At the end: It is 
* Just have a look at the points at the top of this page
* And: Everyone - even newbies - see immediately what the code is about - 
Neither any magic nor super sophisticated architecture necessary :-)

## Advantages over other approaches

What are the alternatives?
1. No dependency at all - Almost untestable and full of implicit dependencies
2. DI containers

There are some libraries around, e. g.
* [Godi](https://github.com/shawnburke/godi)
* [Go-IOC](https://github.com/shelakel/go-ioc)
* ...

The downside I see here is that they are working with type assertions. And when you
have a look at the usage you can see that you save not that many lines of code.

And: It is like working against the static typing of Go - Of course not completely
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
Fine so far but what about testing DoSth? First: You need to know about the internal
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

I do not want to say that this functional approach is the best thing you could ever
use. At [Loopline Systems](http://www.loopline-systems.com) we used it by now
and we will use it for our upcoming Go applications.

That is it so far. I hope I could help you. If you complain about something or
have something to add just let me know :-)
