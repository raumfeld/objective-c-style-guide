##Folder Structure


* Feature/
	* View/
	* ViewController/
	* Controller/
	* Model/
	
	
The folder structure reflects our view on the app's architecture. 

##App Architecture

The app is composed of four layers:

* **View** layer
* **ViewController** layer
* **Controller** layer
* **Model** layer 	

###View Layer

All classes that are subclasses of any class of **UIKit**. All of your UICollectionView, and custom view stuff goes here.

###ViewController Layer

Yeah, you got it: all subclasses of **UIViewController** go here. ViewController transform objects from the model layer into objects from the view layer:

NSObject -> UIView

###Controller Layer

This layer contains all of your efforts to store objects in databases, or communicate to other systems.

###Model Layer

This layer contains all of your business logic and the classes that represent your **domain model**. The names of classes in this layer refer to domain concepts like *zone configuration*, *zone*, or *room*:

```
@class RFZoneConfig
@class RFZone
@class RFRoom
@class RFRenderer
```

Model layer classes contain no code that uses third party libraries - only classes from **UIKit** and **Foundation** are allowed to be used. Responsibilities like communicating with a remote system, or storing objects persistently is delegated to classes in the controller layer.

As an example **RFZoneConfig**:

```
@interface RFZoneConfig



@end
```