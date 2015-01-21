#Code structure

In this document we describe how our code base is organized. Since we introduced this guide to an existing code base it is very likely that you will encounter structures that do not comply with rules presented here. If you find code that does not correspond to these rules, please refactor it!

## Table of Contents

* [Folder Structure](#folder-structure)
* [App Architecture](#app-architecture)
	* [View Layer](#view-layer) 
	* [ViewController Layer](#viewcontroller-layer)
	* [Model Layer](#model-layer)
	* [Controller Layer](#controller-layer)
	    * [Dependency Injection](#dependency-injection)

##Folder Structure

The code for the iOS ***Raumfeld Control*** is organized by topics such as *Zones*, *Content*,  *Playback*, or *UPnP*:

* RaumfeldControl/
	* Zones/
	* Content/
	* Playback/
	* UPnP/

Each topic is organized by this folder structure:

* Topic/
	* View/
	* ViewController/
	* Model/
	* Controller/
	
This folder structure reflects our view on the app's architecture. 

##App Architecture

The app is composed of four layers:

* [**View** layer](#view-layer)
* [**ViewController** layer](#viewcontroller-layer)
* [**Model** layer](#model-layer)
* [**Controller** layer](#controller-layer) 	

The purpose of this architecture is to clearly separate responsibilities and ease unit testing.

###View Layer

All of your `UICollectionView`,  `UITableView` and custom view classes and `XIB's` go here. 

###ViewController Layer

Yeah, you got it: all subclasses of `UIViewController` go here. ViewControllers should be as lightweight as possible. A viewController's [task](https://developer.apple.com/library/ios/featuredarticles/ViewControllerPGforiPhoneOS/Introduction/Introduction.html) is to do view-management: construct views from [domain model objects](#model-layer) and present those views to the user. A viewController should not download anything, not persist anything, or perform any other business logic. This is the task of classes in the [controller layer](#controller-layer) or [model layer](#model-layer).

###Model Layer

> Classes in this layer represent your **domain model** and all of your **business logic**. 

The names of classes in this layer refer to domain concepts like *content item*, *content cache*, or *zone configuration*, *zone*, *room*:

```objc
@class ContentItem
@class ContentCache

@class RFZoneConfig
@class RFZone
@class RFRoom
```

Model layer classes should not download anything, not persist anything, communicate with other systems, or contain third party code. Classes in this layer are only allowed to reference classes from the [controller layer](#controller-layer). The reason for this restriction is to keep this layer as independent as possible from any changes in iOS versions, or third party libraries. Our underlying assumption is:

> We control the application domain. Others control the technology. 

Responsibilities like communicating with a remote system, or storing objects persistently is delegated to classes in the [controller layer](#controller-layer).

###Controller Layer

This layer contains all of your efforts to store objects in databases, do encryption, or communicate to other systems. The integration of third libraries like [AFNetworking](https://github.com/AFNetworking/AFNetworking), [FMDB](https://github.com/ccgus/fmdb), or [CocoaSecurity](https://github.com/kelp404/CocoaSecurity) happens here.

> Classes in this layer translate technical concepts to domain concepts. 

As an example let's have a look at the `RFZoneController` protocol and the class `RFZoneControllerImpl` that conforms to this protocol and translates an XML snippet into an instance of `RFZoneConfig`. 
The protocol defines how to access the app's zone configuration - by accessing the property `zoneConfig`:

```objc
@protocol RFZoneController <NSObject>

@property (nonatomic, readonly) RFZoneConfig *zoneConfig;

@end
```

This protocol does not reveal how a zone configuration is retrieved. And this does not matter for users of this protocol. It is irrelevant to users wether the zone configuration is downloaded as an XML file via HTTP or loaded via `NSKeyedUnarchiver` from the device's local storage.

It is the class `RFZoneControllerImpl` that knows how to get the zone configuration. Nobody else. It knows that the zone configuration is downloaded as an XML snippet via an instance of `URLPoll` and parsed by `RFZoneConfigParser`:

```objc
//RFZoneControllerImpl.h

@interface RFZoneControllerImpl : NSObject <RFZoneController>
@end
```

```objc
//RFZoneControllerImpl.m

#pragma mark - private

@interface RFZoneControllerImpl () <URLDownloadDelegate>

@property (nonatomic, strong) URLPoll *zoneDownload;
@property (nonatomic, strong) RFZoneConfigParser *xmlParser;

@end

#pragma mark -

@implementation

//....

@end
```

By defining the responsibility of the class `RFZoneControllerImpl` via the protocol `RFZoneController` we have achieved two things:

1. Loose coupling of the [model layer](#model-layer) and the [controller layer](#controller-layer)
2. Better testability of the [model layer](#model-layer)

###Loose coupling

The [model layer](#model-layer) only uses the protocol `RFZoneController`. It does not need to know how this protocol is implemented. If we decide to switch to JSON as a transport format between host and app, the [model layer](#model-layer) is not affected. The [model layer](#model-layer) is not only loosely coupled with the implementation of this protocol, it is independent of the technology that we use. This eases maintenance of this layer significantly.

###Better testability

Because the [model layer](#model-layer) is independent of the technology that we use, it is easier to test. Let's consider a class from the [model layer](#model-layer) that needs a `zoneController`:

```objc
@interface AModelLayerClass

- (instancetype)initWithZoneController:(id<RFZoneController>)zoneController NS_DESIGNATED_INITIALIZER;

@end
```

In order to use and test this class we need to initialize it with a `zoneController`. Or expressed in another way: we need to *inject the dependency* to an object conforming to `RFZoneController`.

###Dependency Injection

Let's have a look at the class `AModelLayerClass` and how it is used. 

This might be our first try to instantiate it in a method:

```objc
@implementation SomeClass

- (NSString *) computeString
{
	id<RFZoneController> zoneController = [[RFZoneControllerImpl alloc] init];
	AModelLayerClass *instance          = [[AModelLayerClass alloc] initWithZoneController:zoneController];
	
	NSString *result = nil;
	//do something with instance and compute result
	
	return result;
}

@end
```

This couples the method `computeString` tightly to the class `RFZoneControllerImpl`. 
If we want to test `computeString`, we are going to instantiate `RFZoneControllerImpl` and probably trigger side effects like starting an `URLPoll`:

```objc
- (void)testComputeString
{
	SomeClass *sut   = [[SomeClass alloc] init];
	NSString *result = [sut computeString];
	
	XCTAssertTrue([result isEqualToString:@"dependency hell"]);
}
```

This test is not isolated. This might work. But as your code base grows, you will have plenty of unit tests full of side effects that will not only run slow but influence each other.


One of your thoughts might be "Let's mock the zone controller to isolate the test!". You are right. This is a possible way and there are powerful mocking libraries like [OCMock](http://ocmock.org) and [OCMockito](https://github.com/jonreid/OCMockito) at hand. Instead of mocking the code as it is, we propose a different way of isolating unit tests and avoiding side effects.

Let's have a look at a new implementation of `computeString`:


```objc
@implementation SomeClass

- (NSString *) computeString
{
	id<RFZoneController> zoneController = [RFFactory zoneController];
	AModelLayerClass *instance          = [[AModelLayerClass alloc] initWithZoneController:zoneController];
	
	//...
	
	return result;
}

@end
```

`computeString` just asks the `RFFactory` for an instance of `RFZoneController`. The `RFFactory` returns an instance - there is no need to create a specific instance of `RFZoneController`. The method is independent of the implementation of the `RFZoneProtocol`. The unit test for `computeString` does not change. The `RFFactory` is configured to return a mock for `zoneController` when a unit test is run. Thus, the test is isolated.

To illustrate the possibilities of dependency injection a bit more, we refactor `computeString` one more time:

```objc
@implementation SomeClass

- (NSString *) computeString
{
	AModelLayerClass *instance = [RFFactory aModelLayerClassInstance];
	
	//...
	
	return result;
}

@end
```

`RFFactory` gives us a fully configured instance of `AModelLayerClass`. It takes care of creating the appropriate instance of `RFZoneController` and injects it into `instance`.

The last refactoring shows that by applying dependency injection we get rid of instantiation logic and object graph wiring. This is done by our `RFFactory` which can be called a *"dependency injection container" (DIC)*. 

By delegating object instantiation and object graph wiring to a dependency injection container we get a huge benefit with regard to testability of our app: dependencies can be controlled at a central location. This location is the configuration of a DIC. In case of our app, this is the `RFAssembly`:

```objc
@interface RFAssembly

- (WebService *)webService;
- (ControlContext *)controlContext;
- (id<RFZoneController>)zoneController;
//...

@end
```

`RFAssembly` lists the app's components that are instantiated and wired by our `RFFactory`. `RFFactory` is configured with an `RFAssembly`.

To get an instance of `ControlContext`, use the `RFFactory`:

```objc
ControlContext *controlContext = [RFFactory controlContext]; 
```

####No Singletons

The use of singletons in the app is deprecated. Don't do this in your code:

```objc
+ (id)sharedInstance {
    static MyClass *sharedInstance = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        sharedInstance = [[self alloc] init];
    });
    return sharedInstance;
}
```
If your class has singleton semantics - only one instance is required in production - let its lifecycle be manged by `RFFactory` (see [RFFactoryGuide](RFFactoryGuide.md)).
The above code creates a single instance that will live throughout ***all*** unit tests. It will aggregate state and hinder you from writing isolated unit tests. If the class' lifecycle is managed by `RFFactory` a new instance will be created for every unit test which gives you fully isolated unit tests.