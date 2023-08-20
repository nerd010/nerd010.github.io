# SwiftUI ScrollView


使用 SwiftUI  写到这个功能时候是完全不知道要怎么解决的，于是就查了从网上查了一下找到了解决方法。不过因为目前 Apple 提供的方案在很多情况下不能达到开发者想要的效果，所以网上的解决方案是要自定义 View 来实现对应的效果的。比如滚动距离的计算会在 `ZStack` 中使用 `ForEach` 放多个 `Spacer ()` 在最底层

```swift
struct testView: View {
	var body: some View {
		ScrollViewReader { reader in
			ScrollView {
				ZStack {
					ForEach(0...100) { i in
						Spacer()
							.id(i)
					}
					// some other views
				}
				reader.scrollTo(/*id*/)
			}
		}
	}
}
```

使用这种方式来实现 ScrollView 滚动到指定位置。

我要实现的是点击按钮后滚动一个屏幕宽度，在第二个屏幕宽度的位置显示其他的功能。这在 UIKit 中是比较好实现的。在使用 SwiftUI 时我使用与上面类似的方法实现了这个功能。区别在于我使用了与背景色相同的 View 来占位置。代码与上面的伪代码没有大的区别就不再写了。苹果官方也有[示例](https://developer.apple.com/documentation/swiftui/scrollviewreader)可以自己看一下。

## ScrollViewReader

ScrollViewReader 是在 iOS 14 发布时引入的。

ScrollViewReader 是一个提供编程滚动的视图。它创建一个 ScrollViewProxy 来滚动子视图。它让我们可以通过 `ScrollTo` 的函数来让一个视图可见，并滚动到它所在的位置。

它还有其他的一些使用方法，这里没有用到就先不写了。
