+++
title = 'Test'
date = 2025-04-10T13:57:38+08:00
+++

## Heading
### Subheading
#### subsubheading

hi `hitest`

```
import Settings

class AppDelegate: NSObject, NSApplicationDelegate {
    static var main: AppDelegate!
    static let logger = Logger(subsystem: Bundle.main.bundleIdentifier!, category: String(describing: AppDelegate.self))
    
    private var settingsWindowController: SettingsWindowController?
}
```

> quote
>
> another one
>
> another

This is a body with ***bold+italics***, bold+underline, bold+strikethrough, bold+link, italics+underline, italics+strikethrough, italics+link, underline+strikethrough, underline+link, strikethrough+link.

- [ ] hi

- Dash 1
  - Dash 2
  - [ ] check
* Bullet 1
  - Bullet 2
    - [x] check
1. Number 1
- [ ] checklist

## 2x2 Table

| (1,1) | (1,2) |
| --- | --- |
| (2,1) | (2,2) |

# 3x3 Table w/ Rich Text

| (1,1)**bo****ld** | (1,2)_italic_ | (1,3)link | asdjflkasdjfka sjdfklasjflkasjdflks |
| --- | --- | --- | --- |
| (2,1)***bo******ld******italic*** | (2,2)**bol****d w****ith*****it******alic*****nest****ed** | (2,3)**b****old****+lin****k** | adfasdfasdf |
| (3,1)~~str~~~~iketh~~~~rough~~ |  |  | |
The End.

