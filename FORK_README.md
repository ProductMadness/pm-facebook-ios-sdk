# 🚀 Facebook iOS sdk swizzle fix

> This fork contains the changes neccessary for fixing the swizzle issue with the fb ios sdk, as well as building the `FBSDKCoreKit` library as an xcframework.

---

## 🔍 Overview

There is an open issue with regards to the swizzling done in the Facebook iOS sdk [more details here](https://github.com/facebook/facebook-ios-sdk/issues/2205).
This issue was causing major problems for us as it directly interferes with our AppsFlyer Deep Links. At best, the deep links would not work, and at worst, the app would crash when attempting to open via deep links.

The fix itself is fairly simple, there is a condition which determines what swizzling route the app will take. In our case, we need the proxy route. The issue lies upstream in which the bool for this conditional was 
being incorrectly set, resulting in us never getting the correct swizzle implementation. To make matters worse, the swizzle that we did get would completely change the method signature, which would break the delegate chain 
and results in other sdk's losing their chance to execute their code.

Before fix:
```
- (void)enableAutoSetup:(BOOL)proxyEnabled
{
  if (@available(iOS 14.0, *)) {
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
      @try  {
        if (proxyEnabled) {
          [self setupWithProxy];
        } else {
          [self setup];
        }
      } @catch (NSException *exception) {
        // Disable Auto Setup and log event if exception happens
        [self.featureChecker disableFeature:FBSDKFeatureAEMAutoSetup];
        [self.eventLogger logEvent:@"fb_mobile_auto_setup_exception" parameters:nil];
        fb_dispatch_on_default_thread(^{
          [self.crashHandler saveException:exception];
        });
      }
    });
  }
}
```

After fix:
```
- (void)enableAutoSetup:(BOOL)proxyEnabled
{
  if (@available(iOS 14.0, *)) {
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
      @try  {
        // removing the `proxyEnabled` conditional check here and always running the proxy setup
        // a bug was introduced on the sdk side which caused this condition to always return false
        // and that was not intended. see https://github.com/facebook/facebook-ios-sdk/issues/2205
        
        // the removal of the condition here seems to be the least disruptive solution
        [self setupWithProxy];
      } @catch (NSException *exception) {
        // Disable Auto Setup and log event if exception happens
        [self.featureChecker disableFeature:FBSDKFeatureAEMAutoSetup];
        [self.eventLogger logEvent:@"fb_mobile_auto_setup_exception" parameters:nil];
        fb_dispatch_on_default_thread(^{
          [self.crashHandler saveException:exception];
        });
      }
    });
  }
}
```

Additionally, when forking the Facebook iOS sdk and attempting to implement the above fix, we ran into many issues building the project. To solve that we have implemented an action that will 
build the `FBSDKCoreKit` library and package it as an xcframework, to be released here in the fork. We then pipe the reference to this release back into the `FBSDKCoreKit.podspec` so that when 
build, it will properly fetch the source containing the swizzle fix.

---

## 🔧 Usage

> Below is a quick summary on how to use this fix, please see the Client - SDK Collision (AppsFlyer, Facebook) confluence page for a more detailed breakdown.

1. If you need to modify the swizzle patch, or any objective-c code, do so and push your commits to the [swizzle patch branch](https://github.com/ProductMadness/pm-facebook-ios-sdk/tree/swizzle-patch-v17-0-1).
2. The [Build-CoreKit-Dynamic](https://github.com/ProductMadness/pm-facebook-ios-sdk/actions) action will then run. If there is already a release ([here](https://github.com/ProductMadness/pm-facebook-ios-sdk/releases/tag/vv17.0.1-patched)), you will need to delete it and re-run the action.
3. After the release has been built, modify the `FBSDKCoreKit.podspecs` ([here](https://github.com/ProductMadness/pm-facebook-ios-sdk/blob/main/FBSDKCoreKit.podspec)) to point to the new sha256 of your release, the url should not need to be updated.
4. At this point you should be ok to build the wildcat project, but you can also check the `PodFilePostProcessor` class in the phx facebook package to make sure that it is injecting the correct branch.
5. If you had to modify the phx package, continue with the usual workflow when it comes to handling packages in wildcat.
6. Launch your builds and enjoy! 🍾
