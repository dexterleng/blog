+++
title = 'Introduction to Code Injection on macOS'
date = 2025-04-10T13:57:38+08:00
+++

While most animations in macOS are delightful, some are slow and/or disorientating. In my day-to-day, I find the `NSPopover` animation to annoy me the most. While you could enable `Accessibility -> Reduce Motion` in System Settings, it doesn't always work for all UI interactions, and instead of completely disabling animations, it replaces them with fades.

{{< rawhtml >}}
<video controls>
  <source src="/code-injection/NSPopover-before.mp4" type="video/mp4">
  Your browser does not support the video tag.  
</video>
{{< /rawhtml >}}

As part of learning how to reverse engineer, I wanted to explore different ways to disable animations in macOS apps by injecting code and swizzling methods.

Prerequisites:
- Disable System Integrity Protection ([instructions](https://developer.apple.com/documentation/security/disabling-and-enabling-system-integrity-protection?language=objc)). Be aware of the risks of disabling SIP before proceeding!
- Enable the arm64e ABI by running `sudo nvram boot-args=-arm64e_preview_abi` and rebooting

Create a new Swift Package in Xcode: `File -> New -> Package… -> Multiplatform -> Library`. I will name it "FixAnimationsTweak".

Configure `Package.swift` to build our dynamic library:

```swift
// swift-tools-version: 5.10
// The swift-tools-version declares the minimum version of Swift required to build this package.

import PackageDescription

let package = Package(
    name: "FixAnimationsTweak",
    platforms: [.macOS(.v14)],
    products: [
        .library(
            name: "FixAnimationsTweak",
            type: .dynamic,
            targets: ["FixAnimationsTweak"]),
    ],
    targets: [
        .target(
            name: "FixAnimationsTweak",
            swiftSettings: [
                .unsafeFlags(["-enable-experimental-feature", "SymbolLinkageMarkers"])
            ]
        ),
    ]
)
```

Few things to note:
1. I use Swift 5.10 because Swift 6 is unusable
2. `.dynamic` tells SPM to build a dynamic library (.dylib) which we need to inject into an application.
3. The `SymbolLinkageMarkers` language feature allows us to create an entrypoint in our library in Swift. Without it, we would have to write our entrypoint with Objective C's `+[NSObject load]`.

Update `FixAnimationsTweak.swift` to log some text when it is injected so we know it works:

```swift
import os

let logger = Logger(subsystem: "com.dexterleng.FixAnimationsTweak", category: "debug")

@_section("__DATA,__mod_init_func")
let initialize: @convention(c) () -> Void = {
    logger.info("Injected!")
}
```


Now lets build the dylib:

```bash
swift build --triple arm64e-apple-macosx14.0.0
```

The system applications that Apple ships with macOS 14.0 are all arm64e, which is an extended version of the arm64 architecture with pointer authentication. arm64e is not a backward compatible format - we cannot link an arm64 dynamic library with an arm64e program. Hence, we have to explicitly specify the architecture using the `—triple` flag. This has the unfortunate implication that you cannot use Xcode to build our library. I've tried specifying the architecture through `unsafeFlags` in `Package.swift` but it doesn't work. If anyone has found a workaround please let me know.

Once the build succeeds, you should find the library in `./.build/arm64e-apple-macosx/debug/libFixAnimationsTweak.dylib`.

To check that our library has been successfully injected, open Console and add these two search filters:
1. Process: Safari
2. Subsystem: com.dexterleng.FixAnimationsTweak

Now we can launch Safari with the library using the `DYLD_INSERT_LIBRARIES` environment variable:

```bash
DYLD_INSERT_LIBRARIES=./.build/arm64e-apple-macosx/debug/libFixAnimationsTweak.dylib /Applications/Safari.app/Contents/MacOS/Safari
```

Immediately, you should see `Injected!` in Console:

![](/code-injection/console.png)

We can now fix macOS animations.

## Disable NSPopover animations

Lucky for us, NSPopover is a public API, we can easily see its interface in Xcode by going to `File -> Open Quickly…` and searching for `NSPopover.h`. We can see the `animates` property is responsible for animation:

```objc
/*  Should the popover be animated when it shows, closes, or appears to transition to a detachable window.  This property also controls whether the popover animates when the content view or content size changes. AppKit does not guarantee which behaviors will be animated or that this property will be respected; it is regarded as a hint.  The default value is YES.
 */
@property BOOL animates;
```

To swizzle properties in Objective C, we must first realize that they are compiled into getters and setter instance methods, in our case `-[NSPopover animates]` and `-[NSPopover setAnimates:]`. We can confirm this using Hopper to inspect the disassembly of AppKit:

1. Open Hopper
2. Go to `File -> Read from DYLD Cache…`
3. Open `AppKit.framework`
4. Do a search for "nspopover animates"

You should see the getters and setters:

![](/code-injection/hopper.png)

Add this swizzling helper named `NSObject+exchange.swift` in `Sources/FixAnimationsTweak/`:

```swift
import Foundation

// taken from: https://gist.github.com/stephancasas/828f68b34d8f57a560c856fc0d12e55d#file-custommenubarextracornermask-swift-L40
extension NSObject {
    static func exchange(instanceMethod: String, in className: String, for newMethod: String) {
        exchange(instanceMethod: Selector(instanceMethod), in: className, for: Selector(newMethod))
    }
    
    static func exchange(instanceMethod: Selector, in className: String, for newMethod: Selector) {
        guard let classRef = objc_getClass(className) as? AnyClass,
              let original = class_getInstanceMethod(classRef, instanceMethod),
              let replacement = class_getInstanceMethod(self, newMethod)
        else {
            fatalError("Could not exchange instance method \(instanceMethod) on class \(className).");
        }
        
        method_exchangeImplementations(original, replacement);
    }
}
```

Next, update `FixAnimationsTweak.swift` to make `-[NSPopover animates]` always return false. I've also swizzled `-[NSPopover behavior]` to fix an issue where `animates = false` and `behavior = NSPopoverBehaviorTransient` don't work well together.

```swift
import os
import Foundation

let logger = Logger(subsystem: "com.dexterleng.FixAnimationsTweak", category: "debug")

@_section("__DATA,__mod_init_func")
let initialize: @convention(c) () -> Void = {
    logger.info("Injected!")
    
    NSObject.exchange(instanceMethod: Selector(("animates")),
                      in: "NSPopover",
                      for: #selector(NSObject.swizzled_NSPopover_animates))
    
    NSObject.exchange(instanceMethod: Selector(("behavior")),
                      in: "NSPopover",
                      for: #selector(NSObject.swizzled_NSPopover_behavior))
}

extension NSObject {
    @objc func swizzled_NSPopover_animates() -> Bool {
        let originalAnimates = self.swizzled_NSPopover_animates()
        logger.info("Swizzled -[NSPopover animates]: overriding original value \(originalAnimates) to false")
        return false
    }
    
    @objc func swizzled_NSPopover_behavior() -> Int {
        let originalBehavior = self.swizzled_NSPopover_behavior()
        logger.info("Swizzled -[NSPopover behavior]: original behavior \(originalBehavior)")
        // changes NSPopoverBehaviorTransient to NSPopoverBehaviorSemitransient
        // to fix click-button-to-close popover behavior when animates = false
        if originalBehavior == 1 {
            return 2
        }
        return originalBehavior
    }
}
```

Now let's build and launch Safari with DYLD_INSERT_LIBRARIES, you should see the Reader Mode popover is now super snappy:

{{< rawhtml >}}
<video controls>
  <source src="/code-injection/NSPopover-after.mp4" type="video/mp4">
  Your browser does not support the video tag.  
</video>
{{< /rawhtml >}}

## Bonus: Speed up everything else

We managed to disable animations for `NSPopover` but there are still AppKit components whose animations are still present. While we could swizzle all those components, what if we could swizzle a lower level animation API that all these AppKit components depend on? I found [Speedster](https://github.com/Hoangdus/Speedster), a iOS jailbreak tweak that disables animations by swizzling `-[CAAnimation setDuration:]` (see [code](https://github.com/Hoangdus/Speedster/blob/68c714318d98735f9459c433c253edc0238b1a7c/Speedster/Speedster.x#L325)).

Let's do that:

```swift
import os
import Foundation

let logger = Logger(subsystem: "com.dexterleng.FixAnimationsTweak", category: "debug")

@_section("__DATA,__mod_init_func")
let initialize: @convention(c) () -> Void = {
    logger.info("Injected!")
    
    NSObject.exchange(instanceMethod: Selector(("setDuration:")),
                      in: "CAAnimation",
                      for: #selector(NSObject.swizzled_CAAnimation_setDuration))
    
    NSObject.exchange(instanceMethod: Selector(("animates")),
                      in: "NSPopover",
                      for: #selector(NSObject.swizzled_NSPopover_animates))
    
    NSObject.exchange(instanceMethod: Selector(("behavior")),
                      in: "NSPopover",
                      for: #selector(NSObject.swizzled_NSPopover_behavior))
}

// CAAnimation swizzles
extension NSObject {
    @objc func swizzled_CAAnimation_setDuration(_ duration: Double) {
        // 0.0 causes infinite recursion between AppKit and QuartzCore
        swizzled_CAAnimation_setDuration(0.01)
    }
}
    
// NSPopover swizzles
extension NSObject {
    @objc func swizzled_NSPopover_animates() -> Bool {
        let originalAnimates = self.swizzled_NSPopover_animates()
        logger.info("Swizzled -[NSPopover animates]: overriding original value \(originalAnimates) to false")
        return false
    }
    
    @objc func swizzled_NSPopover_behavior() -> Int {
        let originalBehavior = self.swizzled_NSPopover_behavior()
        logger.info("Swizzled -[NSPopover behavior]: original behavior \(originalBehavior)")
        // changes NSPopoverBehaviorTransient to NSPopoverBehaviorSemitransient
        // to fix click-button-to-close popover behavior when animates = false
        if originalBehavior == 1 {
            return 2
        }
        return originalBehavior
    }
}
```

Check out how fast Safari is now:

{{< rawhtml >}}
<video controls>
  <source src="/code-injection/CAAnimation.mp4" type="video/mp4">
  Your browser does not support the video tag.  
</video>
{{< /rawhtml >}}

I don't think this is a great solution, because there are animations that you don't want to speed up. For example, the text cursor now blinks stupid quick. I wouldn't be surprised messing with `CAAnimation` leads to race conditions that the app programmer wasn't aware of / didn't test for.

## Conclusion

There you have it! I will try to write and share more as I dive deeper into the world of reverse engineering on macOS.

You can find the code on [GitHub](https://github.com/dexterleng/FixAnimationsTweak).