+++
title = "Replicating macOS Dictation's glowing text cursor effect"
date = 2025-04-13T19:09:57+08:00
+++

{{< rawhtml >}}
<video controls>
  <source src="/glowing-cursor/dictation-cursor.mp4" type="video/mp4">
</video>
{{< /rawhtml >}}

As part of learning to reverse engineer private APIs, I wanted to replicate the glowing text cursor effect that shows when you press the dictation key on macOS. To spare your time I shall give you the code now, and you can decide if you want to read more about my process in discovering how to do this.

## TLDR

```
// Enable glowing effect
let indicator = textView.perform(Selector(("_insertionIndicator"))).takeUnretainedValue()
let notification = NSNotification(name: .init("_NSTextInputContextDictationDidStartNotification"), object: nil)
_ = indicator.perform(Selector(("dictationStateDidChange:")), with: notification)

// Disable glowing effect
let indicator = textView.perform(Selector(("_insertionIndicator"))).takeUnretainedValue()
let notification = NSNotification(name: .init("_NSTextInputContextDictationDidEndNotification"), object: nil)
_ = indicator.perform(Selector(("dictationStateDidChange:")), with: notification)
```

## Set up
To understand how Dictation triggers this effect on the text cursor, let's create a SwiftUI project with a `NSTextView`. You could use `TextField` or `TextEditor` but you will need to do some SwiftUI-foo to get the backing `NSTextView/NSTextField`.

### BetterTextField.swift
This simply wraps a `NSScrollView` and `NSTextView` in a SwiftUI component, and passes the `NSTextView` to a `configure` method from the parent.

```
import SwiftUI

struct BetterTextView: NSViewRepresentable {
    @Binding var text: String
    let configure: (_ textView: NSTextView) -> Void

    func makeNSView(context: Context) -> NSScrollView {
        let textView = NSTextView()
        textView.isEditable = true
        textView.isSelectable = true
        textView.isRichText = false
        textView.drawsBackground = false
        textView.delegate = context.coordinator
        textView.font = NSFont.systemFont(ofSize: 14)

        textView.isVerticallyResizable = true
        textView.isHorizontallyResizable = true
        textView.autoresizingMask = [.width, .height]
        textView.textContainer?.widthTracksTextView = true
        textView.textContainer?.heightTracksTextView = false
        textView.textContainerInset = NSSize(width: 8, height: 8)
        textView.textContainer?.lineFragmentPadding = 4

        let scrollView = NSScrollView()
        scrollView.hasVerticalScroller = true
        scrollView.hasHorizontalScroller = false
        scrollView.borderType = .noBorder
        scrollView.autohidesScrollers = true
        scrollView.translatesAutoresizingMaskIntoConstraints = false
        scrollView.documentView = textView

        textView.translatesAutoresizingMaskIntoConstraints = false
        NSLayoutConstraint.activate([
            textView.leadingAnchor.constraint(equalTo: scrollView.contentView.leadingAnchor),
            textView.trailingAnchor.constraint(equalTo: scrollView.contentView.trailingAnchor),
            textView.topAnchor.constraint(equalTo: scrollView.contentView.topAnchor),
            textView.bottomAnchor.constraint(equalTo: scrollView.contentView.bottomAnchor)
        ])

        configure(textView)
        return scrollView
    }

    func updateNSView(_ nsView: NSScrollView, context: Context) {
        if let textView = nsView.documentView as? NSTextView, textView.string != text {
            textView.string = text
        }
    }

    func makeCoordinator() -> Coordinator {
        Coordinator(self)
    }

    class Coordinator: NSObject, NSTextViewDelegate {
        var parent: BetterTextView

        init(_ parent: BetterTextView) {
            self.parent = parent
        }

        func textDidChange(_ notification: Notification) {
            if let textView = notification.object as? NSTextView {
                parent.text = textView.string
            }
        }
    }
}
```

### ContentView.swift

```
import SwiftUI

struct ContentView: View {
    @State var text = ""
    @State var _textView: NSTextView?
    
    var body: some View {
        ZStack(alignment: .bottomTrailing) {
            BetterTextView(text: $text) { nsTextView in
                DispatchQueue.main.async {
                    self._textView = nsTextView
                }
            }
            .frame(maxWidth: .infinity, maxHeight: .infinity)

        }
    }
}
```

## Inspecting AppKit.framework with Hopper

Let's look for sus symbols in AppKit:
1. Open Hopper
2. Go to `File -> Read from DYLD Cache…`
3. Open `AppKit.framework`
4. Do a search for "glow"

![](/glowing-cursor/hopper-glow.png)

Immediately you should see a few methods that might be relevant. What caught my eye was `-[NSTextInsertionIndicator setShowsGlow:]`. The Apple [docs](https://developer.apple.com/documentation/appkit/nstextinsertionindicator) revealed that `NSTextInsertionIndicator` is indeed what `NSTextView` and `NSTextField` use to display the text cursor. Searching on Hopper also reveals that you can get a `NSTextView`'s indicator with `-[NSTextView _insertionIndicator]`. Let's try to make our insertion indicator glow:

```
struct ContentView: View {
    @State var text = ""
    @State var _textView: NSTextView?
    
    var body: some View {
        ZStack(alignment: .bottomTrailing) {
            BetterTextView(text: $text) { nsTextView in
                DispatchQueue.main.async {
                    self._textView = nsTextView
                }
            }
            .frame(maxWidth: .infinity, maxHeight: .infinity)

            Button("Enable Glow") {
                enableGlow()
            }
            .padding()
        }
    }
    
    func enableGlow() {
        guard let textView = _textView else { return }
        let indicator = textView.perform(Selector(("_insertionIndicator"))).takeUnretainedValue()
        _ = indicator.perform(Selector(("setShowsGlow:")), with: true)
    }
}
```

Build and launch the app and click on Enable Glow and you'll realise it doesn't work at all! What's going on? I suspect it's one of the following:

1. We passed the wrong argument to `-[NSTextInsertionIndicator setShowsGlow:]`
2. It is never called when Dictation is activated.
3. Calling it is a necessary but not a sufficient condition.

Rather than spending more time in Hopper reading incomprehensible ARM assembly and speculating what the issue is, let's inspect the runtime behaviour using the LLDB debugger. Add a symbolic breakpoint for the method:

![](/glowing-cursor/symbolic-breakpoint.png)

Then, activate Dictation with our `NSTextView` focused. Our breakpoint is triggered! We can cross off (2) knowing it is definitely called.

Let's inspect our registers with a few print commands:

```
(lldb) po $x0
<NSTextInsertionIndicator: 0x12f762210>
(lldb) po (SEL) $x1
"setShowsGlow:"
(lldb) po $x2
1
```

As you can see Dictation calls the method and passes `1` as argument, which is what `true` in Swift will also do (you can verify this). We can cross off (1). If you continue execution, the debugger will be paused again because `setShowsGlow:` is called with `<nil>` when Dictation is deactivated.

This leaves us with the fact that maybe we need to do more than calling `setShowsGlow:`. Run `bt`to inspect the backtrace:

```
(lldb) bt
* thread #1, queue = 'com.apple.main-thread', stop reason = breakpoint 1.1
  * frame #0: 0x000000019ef345ec AppKit`-[NSTextInsertionIndicator setShowsGlow:]
    frame #1: 0x000000019ef32f70 AppKit`-[NSTextInsertionIndicator updateSubstate] + 580
    frame #2: 0x000000019ef328d4 AppKit`-[NSTextInsertionIndicator dictationStateDidChange:] + 308
    frame #3: 0x000000019aa31370 CoreFoundation`__CFNOTIFICATIONCENTER_IS_CALLING_OUT_TO_AN_OBSERVER__ + 148
    frame #4: 0x000000019aac220c CoreFoundation`___CFXRegistrationPost_block_invoke + 88
    frame #5: 0x000000019aac2154 CoreFoundation`_CFXRegistrationPost + 436
    frame #6: 0x000000019a9fffac CoreFoundation`_CFXNotificationPost + 732
    frame #7: 0x000000019bbba6b8 Foundation`-[NSNotificationCenter postNotificationName:object:userInfo:] + 88
    frame #8: 0x000000019f3b7724 AppKit`-[NSDictationManager updateDictationState:] + 260
    frame #9: 0x00000001a60f8c30 HIToolbox`__68-[IMKInputSession_Modern imkxpc_setApplicationProperty:value:reply:]_block_invoke + 124
    frame #10: 0x00000001a60d7d10 HIToolbox`__57+[IMKInputSession_Modern IMKXPCPerformBlockOnMainThread:]_block_invoke + 36
    frame #11: 0x00000001a13d8460 HIServices`invocation function for block in wrapBlockWithVoucher(void () block_pointer) + 56
    frame #12: 0x00000001a13d7fe8 HIServices`_ZL24deferredBlockOpportunity_block_invoke_2 + 496
    frame #13: 0x000000019aa3c430 CoreFoundation`__CFRUNLOOP_IS_CALLING_OUT_TO_A_BLOCK__ + 28
    frame #14: 0x000000019aa3c340 CoreFoundation`__CFRunLoopDoBlocks + 356
    frame #15: 0x000000019aa3b770 CoreFoundation`__CFRunLoopRun + 2432
    frame #16: 0x000000019aa3a734 CoreFoundation`CFRunLoopRunSpecific + 588
    frame #17: 0x00000001a5fa9530 HIToolbox`RunCurrentEventLoopInMode + 292
    frame #18: 0x00000001a5faf348 HIToolbox`ReceiveNextEventCommon + 676
    frame #19: 0x00000001a5faf508 HIToolbox`_BlockUntilNextEventMatchingListInModeWithFilter + 76
    frame #20: 0x000000019e5b2848 AppKit`_DPSNextEvent + 660
    frame #21: 0x000000019ef18c24 AppKit`-[NSApplication(NSEventRouting) _nextEventMatchingEventMask:untilDate:inMode:dequeue:] + 688
    frame #22: 0x000000019e5a5874 AppKit`-[NSApplication run] + 480
    frame #23: 0x000000019e57c068 AppKit`NSApplicationMain + 888
    frame #24: 0x00000001c96da70c SwiftUI`merged generic specialization <SwiftUI.TestingAppDelegate> of function signature specialization <Arg[0] = Existential To Protocol Constrained Generic> of SwiftUI.runApp(__C.NSResponder & __C.NSApplicationDelegate) -> Swift.Never + 160
    frame #25: 0x00000001c9b509a0 SwiftUI`SwiftUI.runApp<τ_0_0 where τ_0_0: SwiftUI.App>(τ_0_0) -> Swift.Never + 140
    frame #26: 0x00000001c9ecce68 SwiftUI`static SwiftUI.App.main() -> () + 224
    frame #27: 0x0000000104a5c4bc GlowingCursor.debug.dylib`static GlowingCursorApp.$main() at <compiler-generated>:0
    frame #28: 0x0000000104a5c56c GlowingCursor.debug.dylib`main at GlowingCursorApp.swift:4:8
    frame #29: 0x000000019a5d4274 dyld`start + 2840
```

Looking at the assembly for `-[NSTextInsertionIndicator updateSubstate]:` reveals that after `setShowsGlow:` is called with `1`, there also calls to methods like `setCurrentAnimation:`, `setOpacity:`, `addTrailingGlow`, and `animateIn`. I think these are the actual methods that are responsible for the glow effect, and `setShowsGlow:` does very little more than setting a boolean flag. However, `-[NSTextInsertionIndicator updateSubstate]:` does not have any parameters. This leaves us to either:

1. Manually call the subroutines that `updateSubstate` calls to set the effect
2. Call the parent method `-[NSTextInsertionIndicator dictationStateDidChange:]`

While (1) is going to be more powerful and gives us control over how exactly we want the glow effect to look, it is also going to take more time compared to (2), which seems like a very simple API.

Let's set a breakpoint for `-[NSTextInsertionIndicator dictationStateDidChange:]` and run `po $x2`, you should see `NSConcreteNotification 0x600003421980 {name = _NSTextInputContextDictationDidStartNotification}` and `NSConcreteNotification 0x60000345d400 {name = _NSTextInputContextDictationDidEndNotification}` being passed when Dictation is activated and deactivated respectively.

A few LLDB commands reveal that `NSConcreteNotification` belongs to `Foundation.framework`:

```
(lldb) po $x2
NSConcreteNotification 0x60000345d400 {name = _NSTextInputContextDictationDidEndNotification}
(lldb) expr -- (const char *)object_getClassName((id)0x60000345d400)
(const char *) $0 = 0x000000019c9813dc "NSConcreteNotification"
(lldb) expr -- (void *)object_getClass((id)0x60000345d400)
(void *) $1 = 0x00000002042e25a0
(lldb) image lookup -a 0x00000002042e25a0
      Address: Foundation[0x00000001e9dda5a0] (Foundation.__DATA_DIRTY.__objc_data + 4480)
      Summary: (void *)0x00000002042e25c8: NSConcreteNotification
```

Exporting `Foundation.framework` in Hopper (`File -> Export Objective-C Header File…`) reveals `NSConcreteNotification` is a subclass of `NSNotification`:

```
@interface NSConcreteNotification : NSNotification {
    NSString * name;
    id object;
    NSDictionary * userInfo;
}
- (void)dealloc;
- (id)name;
- (id)object;
- (id)userInfo;
- (id)initWithName:(id)v1 object:(id)v2 userInfo:(id)v3;
@end
```

This looks awfully just like the interface of `NSNotification` in Swift, so lets try that:

```
struct ContentView: View {
    @State var text = ""
    @State var isGlowEnabled = false
    @State var _textView: NSTextView?
    
    var body: some View {
        ZStack(alignment: .bottomTrailing) {
            BetterTextView(text: $text) { nsTextView in
                DispatchQueue.main.async {
                    self._textView = nsTextView
                }
            }
            .frame(maxWidth: .infinity, maxHeight: .infinity)

            Button(isGlowEnabled ? "Disable Glow" : "Enable Glow") {
                if isGlowEnabled {
                    disableGlow()
                } else {
                    enableGlow()
                }
            }
            .padding()
        }
    }
    
    func enableGlow() {
        guard let textView = _textView else { return }
        isGlowEnabled = true
        let indicator = textView.perform(Selector(("_insertionIndicator"))).takeUnretainedValue()
        let notification = NSNotification(name: .init("_NSTextInputContextDictationDidStartNotification"), object: nil)
        _ = indicator.perform(Selector(("dictationStateDidChange:")), with: notification)
    }
    
    func disableGlow() {
        guard let textView = _textView else { return }
        isGlowEnabled = false
        let indicator = textView.perform(Selector(("_insertionIndicator"))).takeUnretainedValue()
        let notification = NSNotification(name: .init("_NSTextInputContextDictationDidEndNotification"), object: nil)
        _ = indicator.perform(Selector(("dictationStateDidChange:")), with: notification)
    }
}
```

It works!

{{< rawhtml >}}
<video controls>
  <source src="/glowing-cursor/glowing-cursor.mp4" type="video/mp4">
</video>
{{< /rawhtml >}}

## Conclusion
I hope you learned a thing or two about reverse engineeering private APIs! You can find the code on [GitHub](https://github.com/dexterleng/GlowingCursor).