# iOS:如何控制App内屏幕常亮

跟UIApplication的中的idleTimerDisabled有关，官方描述如下：

```swift
var idleTimerDisabled: Bool
. 是一个布尔值，用来控制这个App在空闲的时候是否禁用
. 这个属性的默认值是false。大多数应用程序在用户长时间内没有触动时，系统将设备放置到一个“休眠”的状态,屏幕变暗。这样做是为了节约资源。这个属性设置为true时，禁用“idle timer”，避免系统进入休眠。
. 在大多数情况时我们应该将它设置为false，包括音频应用程序，但是有些比如游戏等应用程序需要将它设置为true
```

因此只要在didFinishLaunchingWithOptions中加上如下代码即可：

```swift
    application.idleTimerDisabled=true
```

