# Dependency Injection

# Constructor Injection

You pass all the class dependencies as constructor parameter. 

```swift
protocol EngineProtocol {
  func start()
  func stop()
}

protocol TransmissionProtocol {
  func changeGear(gear: Gear)
}

final class Car {
  private let engine: EngineProtocol
  private let transmission: TransmissionProtocol

  init(engine: EngineProtocol, transmission: TransmissionProtocol) {
    self.engine = engine
    self.transmission = transmission
  }
}
```

`EngineProtocol` and `TransmissionProtocol` are services and `Car` is the client. You can create an instance of `Car` whit any dependencies that conform to expected protocols. 

# Setter Injection

```swift
final class Car {
  private var engine: EngineProtocol?
  private var transmission: TransmissionProtocol?

  func setEngine(engine: EngineProtocol) {
    self.engine = engine
  }

  func setTransmission(transmission: TransmissionProtocol) {
    self.transmission = transmission
  }
}
```

This is a good approach when you only have a few dependencies and some are optional.

# Interface Injection

Required the client conforms to protocols used to inject dependencies

```swift
protocol EngineMountable {
  func mountEngine(engine: EngineProtocol)
}

protocol TransmissionMountable {
  func mountTransmission(transmission: TransmissionProtocol)
}

final class Car: EngineMountable, TransmissionMountable {
  private var engine: EngineProtocol?
  private var transmission: TransmissionProtocol?

  func mountEngine(engine: EngineProtocol) {
    self.engine = engine
  }

  func mountTransmission(transmission: TransmissionProtocol) {
    self.transmission = transmission
  }
}
```

Your code gets even more decoupled. In addition, an injector can be completely unaware of the client's actual implementation. 

# Dependency Injection Container

The container is responsible for registering and resolving all dependencies in your project 

```swift
import SwiftUI

protocol ProfileContentProviderProtocol {
  var privacyLevel: PrivacyLevel { get }
  var canSendMessage: Bool { get }
  var canStartVideoChat: Bool { get }
  var photosView: AnyView { get }
  var feedView: AnyView { get }
  var friendsView: AnyView { get }
  
}

final class ProfileContentProvider: ProfileContentProviderProtocol {
  let privacyLevel: PrivacyLevel
  private let user: User
  
  init(privacyLevel: PrivacyLevel = DIContainer.shared.resolve(type: PrivacyLevel.self)!, user: User = DIContainer.shared.resolve(type: User.self)!) {
    self.privacyLevel = privacyLevel
    self.user = user
  }
  
  var canSendMessage: Bool {
    privacyLevel > .everyone
  }
  
  var canStartVideoChat: Bool {
    privacyLevel > .everyone
  }
  
  var photosView: AnyView {
    privacyLevel > .everyone ?
      AnyView(PhotosView(photos: user.photos)) : AnyView(EmptyView())
  }
  
  var feedView: AnyView {
    privacyLevel > .everyone ?
      AnyView(HistoryFeedView(posts: user.historyFeed)) : AnyView(RestrictedAccessView())
  }
  
  var friendsView: AnyView {
    privacyLevel > .everyone ?
      AnyView(UsersView(title: "Friend", users: user.friends)) : AnyView(EmptyView())
  }
  
}
```

```swift

import SwiftUI

struct ProfileView: View {
  private var user: User = Mock.user()
  private let provider: ProfileContentProviderProtocol
  
  init(
    provider: ProfileContentProviderProtocol =
      DIContainer.shared.resolve(type: ProfileContentProviderProtocol.self)!,
    user: User = DIContainer.shared.resolve(type: User.self)!
  ) {
    self.provider = provider
    self.user = user
  }

  var body: some View {
    NavigationView {
      ScrollView(.vertical, showsIndicators: true) {
        VStack {
          ProfileHeaderView(
            user: user,
            canSendMessage: provider.canSendMessage,
            canStartVideoChat: provider.canStartVideoChat
          )
          provider.friendsView
          provider.photosView
          provider.feedView
        }
      }
      .navigationTitle("Profile")
    }
  }
}

struct ProfileView_Previews: PreviewProvider {
  static let user = Mock.user()
  static var previews: some View {
    ProfileView(provider: ProfileContentProvider(privacyLevel: .friend, user: user), user: user)
  }
}

// MARK: - Profile views

struct PhotosView: View {
  private let photos: [String]

  init(photos: [String]) {
    self.photos = photos
  }

  var body: some View {
    VStack {
      Text("Recent photos").font(.title2)
      ScrollView(.horizontal, showsIndicators: false) {
        HStack {
          ForEach(photos, id: \.self) { url in
            ImageView(withURL: url).frame(width: 200, height: 200).clipped()
          }
        }
      }
    }
  }
}

struct UsersView: View {
  private let users: [User]
  private let title: String

  init(title: String, users: [User]) {
    self.users = users
    self.title = title
  }

  var body: some View {
    VStack {
      Text(title).font(.title2)
      ScrollView(.horizontal, showsIndicators: false) {
        HStack {
          ForEach(users, id: \.self) { user in
            VStack {
              ImageView(withURL: user.imageURL).frame(width: 80, height: 80).clipped()
              Text(user.name)
            }
          }
        }
      }
    }
  }
}

struct ProfileHeaderView: View {
  private let user: User
  private let canSendMessage: Bool
  private let canStartVideoChat: Bool

  init(user: User, canSendMessage: Bool, canStartVideoChat: Bool) {
    self.user = user
    self.canSendMessage = canSendMessage
    self.canStartVideoChat = canStartVideoChat
  }

  var body: some View {
    VStack {
      HStack(alignment: .center, spacing: 16) {
        Spacer()
        if canStartVideoChat {
          Button(action: {}) {
            Image(systemName: "video")
          }
        }
        if canSendMessage {
          Button(action: {}) {
            Image(systemName: "message")
          }
        }
      }.padding(.trailing)
      ImageView(withURL: user.imageURL).clipShape(Circle()).frame(width: 100, height: 100).clipped()
      Text(user.name).font(.largeTitle)
      HStack {
        Image(systemName: "location")
        Text(user.area).font(.subheadline)
      }.padding(2)
      Text(user.bio).font(.body).padding()
    }
  }
}

// MARK: - History feed views

struct HistoryFeedView: View {
  private let posts: [Post]

  init(posts: [Post]) {
    self.posts = posts
  }

  var body: some View {
    ScrollView(.vertical, showsIndicators: true) {
      VStack {
        Text("Recent posts").font(.title2)
        ForEach(posts, id: \.self) { post in
          PostView(post: post)
        }
      }
    }
  }
}

struct PostView: View {
  private let post: Post

  init(post: Post) {
    self.post = post
  }

  var body: some View {
    VStack {
      ImageView(withURL: post.pictureURL).frame(height: 200).clipped()
      HStack {
        Text(post.message)
        Spacer()
        HStack {
          Image(systemName: "hand.thumbsup")
          Text(String(post.likesCount))
        }
        HStack {
          Image(systemName: "bubble.right")
          Text(String(post.commentsCount))
        }
      }.padding()
    }
  }
}

struct RestrictedAccessView: View {
  var body: some View {
    VStack {
      Image(systemName: "eye.slash").padding()
      Text("The access to the full profile info is restricted")
    }
  }
}
```

```swift
protocol DIContainerProtocol {
  func register<Component>(type: Component.Type, component: Any)
  func resolve<Component>(type: Component.Type) -> Component?
}

final class DIContainer: DIContainerProtocol {
  static let shared = DIContainer()
  
  private init() {}
  
  var components: [String:Any] = [:]
  
  func register<Component>(type: Component.Type, component: Any) {
    components["\(type)"] = component
  }
  
  func resolve<Component>(type: Component.Type) -> Component? {
    return components["\(type)"] as? Component
  }

}
```

## How to use

```swift
let container = DIContainer.shared
    container.register(type: PrivacyLevel.self, component: PrivacyLevel.friend)
    container.register(type: User.self, component: Mock.user())
    container.register(
      type: ProfileContentProviderProtocol.self,
      component: ProfileContentProvider())
```