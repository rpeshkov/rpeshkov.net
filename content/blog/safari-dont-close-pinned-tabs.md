---
title: "Safari, don't close pinned tabs"
date: 2019-02-03T15:00:52Z
tags: ["safari", "macos"]
---

I love Safari. It's really fast, robust and great browser that works well for all my usecases. Though I hate how it handles pinned tabs by default.

I used to think that pinned tabs are tabs that can't be closed. That works well in Safari actually. If you try to close pinned tab, it will just switch you to another tab, without actually closing this tab. But things get bad when you close last not pinned tab with `Command+w` shortcut. Safari will just close the whole window and all your pinned tabs, even if some tab was playing music! This  happens because it looks like that Safari doesn't treat pinned tabs as real tabs and if you take a look to File menu when you have only 1 tab available, you'll see that `Command+w` binded to Close Window operation and not Close Tab. This annoying behavior can be fixed though.

In MacOS it's possible to define shortcut for menu entry of any particular application. It's done through Keyboard settings pane in System Preferences of MacOS, but it's easier to set it via command line.

Here's the commands:

```bash
defaults write com.apple.universalaccess com.apple.custommenu.apps -array-add '<string>com.apple.Safari</string>'
defaults write com.apple.Safari NSUserKeyEquivalents '{"Close Tab"="@w";}'
```

First line adds Safari to the list inside "App Shortcuts" section of shortcuts settings. Second line sets that `Command+w` shortcut should be used for closing tab.

After applying those commands it's required to restart Safari.

Once restarted, `Command+w` shortcut will be always closing tab and will never close the window, even if there's last tab left. This breaks MacOS HIG a bit, because now you can't close the window via `Command+w`, but still it's always possible to close the window via `Command+Shift+w` shortcut.
