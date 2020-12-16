---
title:  Registering View and ViewModel as Counterparts in Xcode
date:   2020-12-15 12:00:00 -0100
category: Programming
---

`Ctrl+Cmd+Up` and `Ctrl+Cmd+Down` are Xcode shortcuts I use a lot, since the Objective-C days. On the old days, these shortcuts were used to switch between the header and implementation files. In Swift, the shortcuts switch between the implementation and the generated interface (sort of a header file).

But with MVVM knocking on the door with SwiftUI, it would be nice to switch between the View and the ViewModel. Luckily this is actually possible by adding additional counterparts to the Xcode configuration (Xcode needs to be restarted after setting the new counterparts):

```cli
$ defaults write com.apple.dt.Xcode IDEAdditionalCounterpartSuffixes \
           -array-add "ViewModel" "View"
```

To revert the step, delete the additional counterparts. The default counterparts from Xcode won't be removed.

```cli
$ defaults delete com.apple.dt.Xcode IDEAdditionalCounterpartSuffixes
```

Happy hacking.
