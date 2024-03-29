# Enable Target Activities for Audience Manager Audiences via Target Server-side Delivery API on Mobile Apps

Nowadays, Adobe Target server-side implementation is more and more popular, as the experience can display in non-browser-based device. Because your server sits between the client and Target, this type of implementation is also ideal if you need greater control and security or have complex backend processes that you want to run on your server. 

However, when you implement Target server-side on a mobile app, you'll find that your Target activities can target audiences created on Target, but they can't target your audiences from Adudience Manager in real time, as you can't provide Target server-side API required information to enable the real-time integration.

This guide enables you to compose meaningful server-side Target delivery requests with correct values of Audience Manager blob and locationHint. It describes steps to retrieve AAM blob and locationHint values from the AEP SDK and how to compose the Target server-side delivery API call with the information. 

With the AEP SDK, every service is wrapped into an extension, no matter it's provided by Adobe or built by clients. Extensions don't communicate with each other directly. Instead, the `ACPCore` component manages an `extension hub` to facilitate the communications. An extension can distpatch events to the `event hub` and can subscribe to events fired by other extensions. The values of AAM blob and locationHint are held by the Identity extension. To be able to get the values, we'll need to implement a custom extension and subscribe to certain event fired by the Identity extension. 

> The [Appendix](#appendix---how-to-get-aam-blob-and-locationhint-in-sdk4) below describes how to get the same values in SDK4.

## 1 - Build an extension  

As the first step, we'll need to following instructions at [Building an extension](https://aep-sdks.gitbook.io/docs/resources/building-mobile-extensions/building-an-extension) to build a custom extension.

**Sample MyExtension.h**
```objectivec
#import "ACPExtension.h"
#import "ACPExtensionEvent.h"

@interface MyExtension : ACPExtension
- (nullable NSString*) name;
- (nullable NSString*) version;
- (void) unexpectedError: (nonnull NSError*) error;
- (void) onUnregister;
- (void) handleEvent: (ACPExtensionEvent*) event;
@end
```

**Sample MyExtension.m**
> Xcode might complain about the missing of MyExtensionListener. Don't worry, it will be added in the next step.

```objectivec
#import "MyExtension.h"
#import "MyExtensionListener.h"
#import <ACPExtension.h>

@implementation MyExtension

- (instancetype) init {
    if (self = [super init]) {
        NSError *error = nil;

        // register a listener for configuration events
        if ([self.api registerListener:[MyExtensionListener class]
                               eventType:@"com.adobe.eventType.hub"
                             eventSource:@"com.adobe.eventSource.sharedState"
                                 error:&error]) {
            NSLog(@"MyExtensionListener successfully registered for Event Hub Shared State events");
        } else if (error) {
            NSLog(@"An error occured while registering MyExtensionListener, error code: %ld", [error code]);
        }
    }

    return self;
}

- (nullable NSString *) name {
    return @"my.company";
}

- (NSString *) version {
    return @"1.0.0";
}

- (void) onUnregister {
    [super onUnregister];
    // extension unregistered successfully - perform cleanup
}

- (void) handleEvent: (ACPExtensionEvent*) event {
    // process event
}
@end

```

**Sample code to register the extension**

```objectivec
 ...
    [ACPLifecycle registerExtension];
    [ACPSignal registerExtension];
    [ACPUserProfile registerExtension];

    NSError *error = nil;
    if ([ACPCore registerExtension:[MyExtension class] error:&error]) {
        NSLog(@"MyExtension was registered");
    } else {
        NSLog(@"Error registering MyExtension: %@ %d", [error domain], (int)[error code]);
    }

    [ACPCore start:^{
        [ACPCore lifecycleStart:nil];
    }];
 ...
```

## 2 - Create an event listener

Follow instructions [here](https://aep-sdks.gitbook.io/docs/resources/building-mobile-extensions/event-listeners#creating-your-event-listener) to create an event listener. 

**Sample MyExtensionListener.h**

```objectivec
#import <ACPCore/ACPCore.h>
#import "ACPExtensionListener.h"
#import "ACPExtensionEvent.h"

@interface MyExtensionListener : ACPExtensionListener
    - (void) hear:(ACPExtensionEvent *)event;
@end
```

**Sample MyExtensionListener.m**

```objectivec
@implementation MyExtensionListener

/**
 * Returns the parent extension that owns this listener
 */
- (MyExtension*) getParentExtension {
    MyExtension* parentExtension = nil;
    if ([[self extension] isKindOfClass:MyExtension.class]) {
        parentExtension = (MyExtension*) [self extension];
    }

    return parentExtension;
}


    - (void) hear:(ACPExtensionEvent *)event {
        
        MyExtension* parentExtension = [self getParentExtension];
        if (parentExtension == nil) {
            NSLog(@"The parent extension was nil, skipping event");
            return;
        }

        [parentExtension handleEvent:event];
        
        NSDictionary* data = event.eventData;
        NSString* eventOwner = [data objectForKey:@"stateowner"];
        if ([eventOwner isEqualToString:@"com.adobe.module.identity"]) {
            
            NSError* error = nil;
            NSDictionary* identitySharedState = [[[self extension] api] getSharedEventState:@"com.adobe.module.identity" event:event error:&error];
            if (identitySharedState) {
                NSString* blob = [identitySharedState objectForKey:@"blob"];
                NSString* locationhint = [identitySharedState objectForKey:@"locationhint"];
                //NSLog(@"The configuration when event \"%@\" was sent was:\n%@", event.eventName, identitySharedState);
            }
            
        }
    }


@end
```

## 3 - Register for shared state and handle it in the event listener

As the Identity extension will always share its state early in the app lifecycle, the most reliable way is to listen for shared state changes.

As we're only interested in Identity shared states, we'll need to filter out which events we process in the "hear" method of the `ACPExtensionListener`.  We will know a new shared state is available from Identity when the EventData has a value of "com.adobe.module.identity" in the "stateowner" key.

**Sample code from MyExtension.m**

```objectivec
- (instancetype) init {
    if (self = [super init]) {
        NSError *error = nil;

        // register a listener for configuration events
        if ([self.api registerListener:[MyExtensionListener class]
                               eventType:@"com.adobe.eventType.hub"
                             eventSource:@"com.adobe.eventSource.sharedState"
                                 error:&error]) {
            NSLog(@"MyExtensionListener successfully registered for Event Hub Shared State events");
        } else if (error) {
            NSLog(@"An error occured while registering MyExtensionListener, error code: %ld", [error code]);
        }
    }

    return self;
}
```

From the `hear` method in the `MyExtensionListener`, we can get AAM blob and locationhint by processing the shared state.

**Sample code from MyExtensionListener.m**

```objectivec
    - (void) hear:(ACPExtensionEvent *)event {
        
        MyExtension* parentExtension = [self getParentExtension];
        if (parentExtension == nil) {
            NSLog(@"The parent extension was nil, skipping event");
            return;
        }

        [parentExtension handleEvent:event];
        
        NSDictionary* data = event.eventData;
        NSString* eventOwner = [data objectForKey:@"stateowner"];
        if ([eventOwner isEqualToString:@"com.adobe.module.identity"]) {
            
            NSError* error = nil;
            NSDictionary* identitySharedState = [[[self extension] api] getSharedEventState:@"com.adobe.module.identity" event:event error:&error];
            if (identitySharedState) {
                NSString* blob = [identitySharedState objectForKey:@"blob"];
                NSString* locationhint = [identitySharedState objectForKey:@"locationhint"];
                //NSLog(@"The configuration when event \"%@\" was sent was:\n%@", event.eventName, identitySharedState);
            }
            
        }
    }
```

## 4 - Compose Target server-side API requests

At this point we have already retrieved Audience Manager blob and locationHint from the SDK. Once they are sent to your server side process, your server side process needs to compose the Target server-side delivery API request to provide the information to Target server, and Target server will query Audience Manager in real time for segment qualification. Details on Target server-side delivery API can be found [here](https://developers.adobetarget.com/api/delivery-api/#section/Integration-with-Experience-Cloud/Adobe-Audience-Manager).
```
curl -X POST \
  'https://demo.tt.omtrdc.net/rest/v1/delivery?client=demo&sessionId=d359234570e04f14e1faeeba02d6ab9914e' \
  -H 'Content-Type: application/json' \
  -H 'cache-control: no-cache' \
  -d '{
      ...
      "experienceCloud": {
        "audienceManager": {
          "locationHint": 9,
          "blob": "32fdghkjh34kj5h43"
        }
      },
      ...
    }'
```

## Appendix - How to get AAM blob and locationHint in SDK4.


**For iOS (assuming customer is not using App Groups):**
```objectivec
NSUserDefaults *defaults = [NSUserDefaults standardUserDefaults];
NSString *locationHint = [defaults   objectForKey:@"ADBMOBILE_PERSISTED_MID_HINT"];
NSString *locationBlob = [defaults objectForKey:@"ADBMOBILE_PERSISTED_MID_BLOB"];
```

**For Android:**
```java
// where "this" is an Activity
SharedPreferences preferences = this.getSharedPreferences("APP_MEASUREMENT_CACHE", 0);
String locationHint = preferences.getString("ADBMOBILE_PERSISTED_MID_HINT", null);
String locationBlob = preferences.getString("ADBMOBILE_PERSISTED_MID_BLOB", null);
```

---

[Go Back](../README.md)
