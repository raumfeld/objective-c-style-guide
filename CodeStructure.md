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

* **View** layer
* **ViewController** layer
* **Model** layer
* **Controller** layer 	

The purpose of this architecture is to clearly separate responsibilities and ease unit testing.

###View Layer

All of your `UICollectionView`,  `UITableView` and custom view classes and `XIB's` go here. 

###ViewController Layer

Yeah, you got it: all subclasses of `UIViewController` go here. ViewControllers should be as lightweight as possible. A viewController's [task](https://developer.apple.com/library/ios/featuredarticles/ViewControllerPGforiPhoneOS/Introduction/Introduction.html) is to do view-management: construct views from [domain model objects](#model-layer) and present those views to the user. A viewController should not download anything, not persist anything, or perform any other business logic. This is the task of classes in the [controller layer](#controller-layer) or [model layer](#model-layer).

###Model Layer

This layer contains the classes that represent your **domain model** and all of your **business logic**. The names of classes in this layer refer to domain concepts like *content item*, *content cache*, or *zone configuration*, *zone*, *room*:

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

The responsibilities of classes in this layer should be described by `@protocols`. A protocol should only contain domain concepts, no technical concepts.

As an example let's have a look at the `RFZoneController` protocol.