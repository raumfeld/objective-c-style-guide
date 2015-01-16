#Code structure

In this document we describe how our code base is organized. Since we introduced this guide to an existing code base it is very likely that you will encounter structures that do not comply with rules presented here. If you find code that does not correspond to these rules, please refactor it!

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

The [model layer](#model-layer) only uses the protocol `RFZoneController`. It does not need to know how this protocol is implemented. If we decide to switch to JSON as a transport format between host and app, the [model layer](#model-layer) is not affected. The [model layer](#model-layer) is not only loosely coupled with the implementation of this protocol, it is independent of the technology that we use.

###Better testability

Because the [model layer](#model-layer) is independent of the technology that we use, it is easier to test. Let's consider a class from the [model layer](#model-layer) that needs a `zoneController`:

```objc
@interface AModelLayerClass

@property(nonatomic, strong) id<RFZoneController> zoneController;

@end
```

In order to isolate unit tests for this class we just need to inject a mock into the `zoneController` property.

###Dependency Injection

You have asked yourself how an instance of `AModelLayerClass` will get its `zoneController`? It will be injected with the help of a *dependency injection container (DIC)*. 

As of now, we have not integrated a *DIC* into the app. We are planning to integrate the [Typhoon DIC](https://github.com/appsquickly/Typhoon).

In order to get the implementation for the `RFZoneController` protocol we have to call the static `instance`method on `RFZoneControllerImpl`:

```objc
@implementation AModelLayerClass

- (instancetype)init
{
	self = [super init];
	if(self)
	{
		_zoneController = [RFZoneControllerImpl instance];
	}
	return self;
}

@end
```

This couples `AModelLayerClass` to `RFZoneControllerImpl`. With the help of an DIC this coupling is removed.

To still be able to isolate `AModelLayerClass` in your unit tests you simply have to mock the `instance` method of `RFZoneControllerImpl` to return a mock (with the help of [OCMock](http://ocmock.org)):

```objc
id classMock = OCMClassMock([RFZoneControllerImpl class]);

OCMStub([classMock instance]).
	andReturn(OCMProtocolMock(@protocol(RFZoneController)));
```

Now, all calls to `instance`will return a mock object.