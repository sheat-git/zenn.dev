---
title: "【SwiftUI】NavigationStackの階層が意図しないものになる原因と対処法"
published: true
type: "tech"
topics: ["swiftui", "navigationstack"]
emoji: 🔧
---

## 意図しない挙動

`navigationDestination(for:destination:)`を設定しているViewから`NavigationLink(title:destination:)`などで遷移した先のViewにて、設定した型の`NavigationLink(title:value:)`などで遷移しようとすると、遷移の階層がタップの順と異なるというもの。

### 環境

- Macbook Air (M2 2022)
- MacOS 13.3.1
- Xcode 14.3

### 手順

`View1`から`View2`に遷移したのち、`value2`に遷移しようとすると、階層が`View1/View2/value2`ではなく、`View1/value2/View2`になってしまう。

![意図しない挙動](/images/swiftui-navigationstack-unexpected-path/unexpected.gif =300x)

### コード

```swift:Views.swift
import SwiftUI

struct View1: View {
    var body: some View {
        NavigationStack {
            VStack {
                NavigationLink("value1", value: "value1")
                NavigationLink("View2", destination: View2())
            }
            .navigationTitle("View1")
            .navigationDestination(for: String.self) { value in
                Text(value)
                    .navigationTitle(value)
            }
        }
    }
}

struct View2: View {
    var body: some View {
        NavigationLink("value2", value: "value2")
            .navigationTitle("View2")
    }
}
```

## 原因

`NavigationLink(title:destination:)`や`navigationDestination(isPresented:destination:)`などでは`NavigationStack`の`path`が更新されず、これらの遷移が`path`の遷移の最後に追加される仕様になってるっぽい。

## 対処法

`destination`を`value`と同様に扱うことで解決する。
しかし、`View`や`AnyView`は`Hashable`に準拠していないので、以下のようにラッパーを作成し、それを用いて`NavigationLink`と`navigationDestination`を拡張する。

```swift:Navigation+.swift
import SwiftUI

extension NavigationLink {
    init<D: View>(_ titleKey: LocalizedStringKey, @ViewBuilder view: @escaping () -> D) where Destination == Never, Label == Text {
        self.init(titleKey, value: ViewWrapper(view))
    }
    
    init<S: StringProtocol, D: View>(_ title: S, @ViewBuilder view: @escaping () -> D) where Destination == Never, Label == Text {
        self.init(title, value: ViewWrapper(view))
    }
    
    init<D: View>(@ViewBuilder view: @escaping () -> D, @ViewBuilder label: () -> Label) where Destination == Never {
        self.init(value: ViewWrapper(view), label: label)
    }
}

extension View {
    func navigationDestinationForView() -> some View {
        self.navigationDestination(for: ViewWrapper.self) { wrapper in
            wrapper.view()
        }
    }
}

private struct ViewWrapper: Hashable {
    static func == (lhs: ViewWrapper, rhs: ViewWrapper) -> Bool {
        lhs.id == rhs.id
    }
    
    func hash(into hasher: inout Hasher) {
        hasher.combine(id)
    }
    
    let id = UUID()
    let view: () -> AnyView
    
    init<D: View>(_ view: @escaping () -> D) {
        self.view = { AnyView(view()) }
    }
}
```

そして、`View1`を以下のように更新すると、遷移の順とタップの順が同じと言う挙動が得られる。

```swift:View1
struct View1: View {
    var body: some View {
        NavigationStack {
            VStack {
                NavigationLink("value1", value: "value1")
                NavigationLink("View2", view: { View2() })
            }
            .navigationTitle("View1")
            .navigationDestination(for: String.self) { value in
                Text(value)
                    .navigationTitle(value)
            }
            .navigationDestinationForView()
        }
    }
}
```

上記を用いると、下のような挙動が得られる。

![意図した挙動](/images/swiftui-navigationstack-unexpected-path/expected.gif =300x)
