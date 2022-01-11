Managing data and state in SwiftUI is a very pleasant experience. In this short guide, we will cover the building blocks of the various attributes that allow us to manage data flow in SwiftUI applications. Most of the below is copied from Apple’s official documentation on State and Data Flow.
Part 1: Introduction
There are two adjacent goals to keep in mind when managing data within SwiftUI:
Managing User Interface State: We need to have a way to present data in our SwiftUI views and update them upon user interaction. This is achieved through state variables and bindings.
Managing Model Data in Your App: We need to be able to connect our data model to our SwiftUI Apps. This is achieved through observable objects.
We cover both of these concepts below in Parts 2 and 3 of this guide respectively. As an appendix, Part 4 contains a brief introduction to the concept of property wrappers, which are heavily used as an abstraction mechanism in SwiftUI (an understanding of property wrappers is not required for this guide).
Part 2: Managing User Interface State
You can provide immutable data to a view by declaring a standard Swift property.
struct PlayerView: View {
    let episode: Episode // immutable value
    
    ...
}
However, to store mutable data within views, you need to declare variables with the State property wrapper. This tells SwiftUI to manage the underlying storage.
struct PlayerView: View {
    let episode: Episode // immutable value
    @State private var isPlaying: Bool = false 
     
    ...
}
Note that views in SwiftUI are thrown away and recreated regularly. When this happens, the entire view struct is initialized all over again. Because of this, any values that you create in a SwiftUI view are reset to their default values unless you’ve marked these values using @State.
Sharing Access to Values to Children Views
You can use the Binding property wrapper to share control of state with a child view (note the binding wrapper is written on the property in the child view, not the parent view). You can then pass the state variable to the child view by prefixing it with the dollar sign ($):
struct PlayButton: View {
    @Binding var isPlaying: Bool
    
    ...
}
struct PlayerView: View {
    let episode: Episode // immutable value
    @State private var isPlaying: Bool = false 
    
    var body: some View {
        PlayButton(isPlaying: $isPlaying) // <-- Pass a binding
    }
}
Bindings can access properties of a state variable (e.g. $episode.isFavourite, where isFavourite is a property of Episode). You can also get a binding from other bindings (i.e. variables with a @Binding wrapper), again by using the $ prefix when instantiating the descendant views. This allows you to pass a binding through an arbitrary number of levels of view hierarchy.
Part 3: Managing Model Data in Your App
We can make our data classes visible to SwiftUI by adopting the ObservableObject protocol. Any properties published with the @Published property wrapper will then be hooked into SwiftUI:
class Book: ObservableObject {
    @Published var title = "Great Expectations"
    
    let identifier = UUID() // A property not published
}
To avoid the overhead of a published property when you do not need it, only publish properties that both can change and that matter to the user interface.
Monitoring Changes in Observable Objects
You can tell SwiftUI to monitor an observable object by adding the ObservedObject attribute to the property’s declaration. In the below example, when the title of the book changes, SwiftUI will update all the affected views.
struct BookView: View {
    @ObservedObject var book: Book
    var body: some View {
        Text(book.title)
    }
}
You can also pass entire observable objects to children views and share model objects across levels of a view hierarchy:
struct BookView: View {
    @ObservedObject var book: Book
    var body: some View {
        BookEditView(book: book)
    }
}
struct BookEditView: View {
    @ObservedObject var book: Book
    ...
}
Instantiate a Model Object in a View
When a view creates its own @ObservedObject instance, the instance is recreated every time the view is discarded and redrawn. Since SwiftUI might create or recreate a view at any time, it’s unsafe to create an observed object inside a view.
Instead, SwiftUI provides the StateObject attribute, which is a combination of @ObservedObject and @State. State objects behave like observed objects, except that SwiftUI knows to create and manage a single object instance for a given view instance, regardless of how many times it recreates the view. You can use the object locally, or pass the state object into another view’s observed object property, as shown in the below example.
struct LibraryView: View {
    @StateObject var book = Book()
    var body: some View {
        BookView(book: book)
    }
}
Sometimes it is useful to create state objects within the top level App object to allow them to be shared across the entire app, such as in the following.
@main
struct BookReader: App {
    @StateObject var library = Library ()
   ...
}
@ObservedObject remains useful when you want to recreate an observable property in a view every time it is redrawn. For more information on the differences between @ObservedObject and @StateObject, refer to the Stack Overflow discussion on the topic.
Sharing an Object Throughout an App
If you have a data model that you want to use throughout your app, but without having to pass it through many layers of hierarchy, you can use the environmentObject(_:) view modifier to put the object into the environment instead:
@main
struct BookReader: App {
    @StateObject var library = Library ()
    var body: some Scene {
        WindowGroup {
            LibraryView()
            .environmentObject(Library)
        }
    }
}
Any descendant view of the view to which you apply the modifier can then access the data model instance by declaring a property with the EnvironmentObject attribute:
struct LibraryView: View {
    @EnvironmentObject var Library: Library
    
    ...
}
This allows us to avoid having to pass the shared object directly in the initialisation of the descendant view (e.g. (LibraryView(library: Library)).
Allow Users to Modify Data through Bindings
You can create a two-way connection for observed objects, state objects or enviroment objects to allow users to change the data and flow updates back into the data model automatically. This is achieved by creating a binding by prefixing the name of the object with the dollar sign ($), as in the following example.
struct BookEditView: View {
    @ObservedObject var book: Book
    var body: some View {
        TextField("Title", text: $book.title)
    }
}
And that’s all there is to it! The combination of state variables, bindings, observable, state and environment objects work harmoniously together to connect data models to SwiftUI and to manage state within SwiftUI views.
Part 4: Appendix: Property Wrappers
A property wrapper adds a layer of separation between code that manages how a property is stored and the code that defines a property. For example, if would like to store certain data in a database, you could either write the functionality to do so for every data object that you wish to store, or write it once as a property wrapper and use that property wrapper on each relevant data object.
Below is a simple example where we define a property wrapper TwelveOrLess that sets a mximum value for the property as 12:
@propertyWrapper
struct TwelverOrLess {
    private var number = 0;
    var wrappedValue: Int {
        get { return number }
        set { number = min(newValue, 12) }
    }
}
struct SmallRectangle {
    @TwelveOrLess var height: Int
    @TwelveOrLess var width: Int
}
var rectangle = SmallRectangle()
print(rectangle.height)
// Prints "0"
rectangle.height = 10
print(rectangle.height)
// Prints "10"
rectangle.height = 24
print(rectangle.height)
// Prints "12"
When you apply a wrapper to a property, the compiler synthesizes code that provides storage for the wrapper and code that provides access to the property through the wrapper. For example, below is an equivalent version of SmallRectangle that explicitly wraps its properties in the TwelveOrLess structure:
struct SmallRectangle {
    private var _height = TwelveOrLess()
    private var _width = TwelveOrLess()
    
    var height: Int {
        get { return _height.wrappedValue }
        set { _height.wrappedValue = newValue }
    }
    var width: Int {
        get { return _width.wrappedValue }
        set { _width.wrappedValue = newValue }
    }
}
SwiftUI makes frequent use of property wrappers. You will not need to have a deep understanding of how they work, but it is useful to at least know what they are, which we covered in the above. More details on Swift’s property wrappers can be found within the official documentation for the language.
