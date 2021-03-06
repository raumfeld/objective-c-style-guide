This document describes how to use our dependency injection container a.k.a. `RFFactory`. Furthermore, it describes how to [add new components to `RFFactory`](#adding-new-components) and how to [make your test cases aware of `RFFactory`](#rftestcase).

#RFFactory

`RFFactory` is a handy macro that is defined as follows:

```objc
#define RFFactory (RFAssembly *)[TyphoonComponentFactory defaultFactory]
```

As you can see, we are using the [Typhoon framework](https://github.com/appsquickly/Typhoon).

`RFFactory` allows you to easily get core components of the app. For instance, to get a reference to the `ControlContext` you do this:

```objc
ControlContext *controlContext = [RFFactory controlContext];
```

Which components can be retrieved with `RFFactory` is described by the class `RFAssembly`:

```objc
@interface RFAssembly: TyphoonAssembly

- (WebService *)webService;

- (ControlContext *)controlContext;

- (id<RFZoneController>)zoneController;

//...

@end
```

The header of `RFAssembly` describes which components are managed by  `RFFactory`. "Managed" means that the `TyphoonComponentFactory` instantiates those objects and wires them together (injects their dependencies).

For example, the `ControlContext` must be instantiated with an instance of `WebService` and `RFZoneController`:

```objc
@interface ControlContext

- (instancetype) initWithWebService: (WebService *) webService
                     zoneController: (id<RFZoneController>)zoneController NS_DESIGNATED_INITIALIZER;
                     
@end
```
The implementation of `RFAssembly` defines how an instance of `ControlContext` is instantiated and wired with other components:

```objc
@implementation RFAssembly

- (ControlContext *)controlContext
{
    return [TyphoonDefinition withClass:[ControlContext class] configuration:^(TyphoonDefinition *definition) {

        definition.scope = TyphoonScopeSingleton;
        
        [definition useInitializer:@selector(initWithWebService:zoneController:)
                        parameters:^(TyphoonMethod *initializer) {
                            
            [initializer injectParameterWith:[self webService]];
            [initializer injectParameterWith:[self zoneController]];
        }];
    }];
}

@end
```

`definition.scope = TyphoonScopeSingleton` means that onyl one instance through out the app lifecycle is created. Furthermote, the `definition` states that the instance must be created using the selector `initWithWebService:zoneController:` with two parameters.

`RFAssembly` is used to create the app's default factory in `RaumfeldControlAppDelegate`:

```objc
TyphoonComponentFactory *factory = [[TyphoonBlockComponentFactory alloc] 
                                       initWithAssemblies:@[[RFAssembly assembly]]];
[factory makeDefault];
```

##Using RFFactory

As shown above, `RFFactory` is very handy to inject dependencies into your code:

```objc
ControlContext *controlContext = [RFFactory controlContext];
```

However, it is preferred to declare dependencies in your class' designated initializer and add your class to `RFAssembly` (see [Adding New Components](#adding-new-components) for a detailed example) instead of using `RFFactory`.

Avoid doing this:

```objc
@implementation YourClass

- (instancetype)init
{
	//...
		
	self.controlContext = [RFFactory controlContext];
		
	return self;
}

@end
```

Instead, do this:

```objc
@implementation YourClass

- (instancetype)initWithControlContext:(ControlContext*)controlContext 
{
	//...
		
	self.controlContext = controlContext;
		
	return self;
}

@end
```

This has the benefit that your class' dependencies are clearly communicated and do not disappear somewhere in its implementation. Another benefit is that your class can be instantiated and wired to other objects by `RFFactory`.

`RFFactory` should be used as little as possible. Ideally, we want to get the root of an object graph from `RFFactory` and let it instantiate and wire the rest of the object graph. But since we are introducing it to an existing code base, we have to be pragmatic.

##Adding New Components

There are two steps that are required in order to make a new component available through `RFFactory`:

1. Add a new method to `RFAssembly`
2. Mock the component in `RFTestCase` `injectMocks`

If you have done this, `RFFactrory` will provide other components with the production implementation that you specified in `RFAssembly` and with a mock in test cases that inherit from `RFTestCase`.

###Example

1. Declare a dependency in your class' designated initializer and add your class to the `RFAssembly`:
	
	```objc
	@interface YourClass
	- (instancetype)initWithControlContext:(ControlContext*)controlContext NS_DESIGNATED_INITIALIZER;
	@end
	```
	
	The declaration in the desiganted initializer is the prefered way to inject dependencies. This allows us to clearly communicate a class' dependencies.
	
	```objc
	@interface RFAssembly
	- (YourClass *)yourClass;
	@end
	
	@implementation RFAssembly
	
	- (YourClass *)yourClass
	{
    	return [TyphoonDefinition withClass:[YourClass class] configuration:^(TyphoonDefinition *definition) {
        	
        	definition.scope = TyphoonScopeSingleton;
        
        	[definition useInitializer:@selector(initWithControlContext:)
                            parameters:^(TyphoonMethod *initializer) {
                            
				[initializer injectParameterWith:[self controlContext]];
                            
			}];
    	}];
	}
	
	@end
	```
	
2. Mock your class in `RFTestCase` `injectMocks`

	```objc
	- (void)injectMocks:(TyphoonComponentFactory *)assembly
	{
		[self     mockClass:[YourClass class]
                 inAssembly:assembly
              usingSelector:@selector(yourClass)];
	}
	```

#RFTestCase

`RFTestCase` must be the base class of every new test case. It provides a `factory` that provides mocks for all components specified by `RFAssembly`.

Let's have a look at `ControlContext_Tests`:

```objc
@interface ControlContext_Tests : RFTestCase

@property (nonatomic, strong) ControlContext *sut;

@end
```

```objc
@implementation ControlContext_Tests

- (void)setUp {
    [super setUp];
    
    self.sut = [[ControlContext alloc] initWithWebService:[self.factory webService]
                                           zoneController:[self.factory zoneController]];
}

@end
```

As shown above, you create the instance of your SUT (system under test) with its designated initializer. 
This allows us to communicate on which components the SUT depends upon. In the example we clearly see that a `ControlContext` requires a webService and a zoneController. Sicne we are in a unit test, `[self.factory webService]` and `[self.factory zoneController]` return mocks (instances of `OCMockObject`). Thus, all tests in this test case are (should be ;) ) isolated. If you need to stub these mocks or setup expectations, just get the mocks from the factory and stub/expect as you are used to:

```objc
id webServiceMock = [self.factory webService];
[[webServiceMock expect] resume];
```