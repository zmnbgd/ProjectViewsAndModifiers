DAY 23

Today you have 10 topics to work through, in which you’ll learn to build custom view modifiers and custom containers, as well as start to develop your understanding of how SwiftUI actually works internally.

Views and modifiers: Introduction

In this technique project we’re going to take a close look at views and view modifiers, and hopefully answer some of the most common questions folks have at this point – why does SwiftUI use structs for its views? Why does it use some View so much? How do modifiers really work? My hope is that by the end of this project you’ll have a thorough understanding of what makes SwiftUI tick.

Why does SwiftUI use structs for views?

If you ever programmed for UIKit or AppKit (Apple’s original user interface frameworks for iOS and macOS) you’ll know that they use classes for views rather than structs. SwiftUI does not: we prefer to use structs for views across the board, and there are a couple of reasons why.

First, there is an element of performance: structs are simpler and faster than classes. I say an element of performance because lots of people think this is the primary reason SwiftUI uses structs, when really it’s just one part of the bigger picture.

In UIKit, every view descended from a class called UIView that had many properties and methods – a background color, constraints that determined how it was positioned, a layer for rendering its contents into, and more. There were lots of these, and every UIView and UIView subclass had to have them, because that’s how inheritance works.

In SwiftUI, all our views are trivial structs and are almost free to create. Think about it: if you make a struct that holds a single integer, the entire size of your struct is… that one integer. Nothing else. No surprise extra values inherited from parent classes, or grandparent classes, or great-grandparent classes, etc – they contain exactly what you can see and nothing more.

However, even though performance is important there’s something much more important about views as structs: it forces us to think about isolating state in a clean way. You see, classes are able to change their values freely, which can lead to messier code – how would SwiftUI be able to know when a value changed in order to update the UI?

By producing views that don’t mutate over time, SwiftUI encourages us to move to a more functional design approach: our views become simple, inert things that convert data into UI, rather than intelligent things that can grow out of control.

You can see this in action when you look at the kinds of things that can be a view. We already used Color.red and LinearGradient as views – trivial types that hold very little data. In fact, you can’t get a great deal simpler than using Color.red as a view: it holds no information other than “fill my space with red”.

In comparison, Apple’s documentation for UIView lists about 200 properties and methods that UIView has, all of which get passed on to its subclasses whether they need them or not.


What is behind the main SwiftUI view?

When you’re just starting out with SwiftUI, you get this code:
struct ContentView: View {
    var body: some View {
        Text("Hello, world!")
            .padding()
    }
}
It’s common to then modify that text view with a background color and expect it to fill the screen:
struct ContentView: View {
    var body: some View {
        Text("Hello, world!")
            .padding()
            .background(.red)
    }
}

However, that doesn’t happen. Instead, we get a small red text view in the center of the screen, and a sea of white beyond it.
This confuses people, and usually leads to the question – “how do I make what’s behind the view turn red?”

Let me say this as clearly as I can: for SwiftUI developers, there is nothing behind our view. You shouldn’t try to make that white space turn red with weird hacks or workarounds, and you certainly shouldn’t try to reach outside of SwiftUI to do it.

Now, right now at least there is something behind our content view called a UIHostingController: it is the bridge between UIKit (Apple’s original iOS UI framework) and SwiftUI. However, if you start trying to modify that you’ll find that your code no longer works on Apple’s other platforms, and in fact might stop working entirely on iOS at some point in the future.

Instead, you should try to get into the mindset that there is nothing behind our view – that what you see is all we have.
Once you’re in that mindset, the correct solution is to make the text view take up more space; to allow it to fill the screen rather than being sized precisely around its content. We can do that by using the frame() modifier, passing in .infinity for both its maximum width and maximum height.

Text("Hello, world!")
    .frame(maxWidth: .infinity, maxHeight: .infinity)
    .background(.red)

Using maxWidth and maxHeight is different from using width and height – we’re not saying the text view must take up all that space, only that it can. If you have other views around, SwiftUI will make sure they all get enough space.


Why modifier order matters

Whenever we apply a modifier to a SwiftUI view, we actually create a new view with that change applied – we don’t just modify the existing view in place. If you think about it, this behavior makes sense: our views only hold the exact properties we give them, so if we set the background color or font size there is no place to store that data.
We’re going to look at why this happens shortly, but first I want to look at the practical implications of this behavior. Take a look at this code:

Button("Hello, world!") {
    // do nothing
}    
.background(.red)
.frame(width: 200, height: 200)

What do you think that will look like when it runs?
Chances are you guessed wrong: you won’t see a 200x200 red button with "Hello, world!" in the middle. Instead, you’ll see a 200x200 empty square, with "Hello, world!" in the middle and with a red rectangle directly around "Hello, world!".

You can understand what’s happening here if you think about the way modifiers work: each one creates a new struct with that modifier applied, rather than just setting a property on the view.

You can peek into the underbelly of SwiftUI by asking for the type of our view’s body. Modify the button to this:

Button("Hello, world!") {
    print(type(of: self.body))
}    
.background(.red)
.frame(width: 200, height: 200)

Swift’s type(of:) method prints the exact type of a particular value, and in this instance it will print the following: ModifiedContent<ModifiedContent<Button<Text>, _BackgroundStyleModifier<Color>>, _FrameLayout>

To read what the type is, start from the innermost type and work your way out:
* The innermost type is ModifiedContent<Button<Text>, _BackgroundStyleModifier<Color>: our button has some text with a background color applied.
* Around that we have ModifiedContent<…, _FrameLayout>, which takes our first view (button + background color) and gives it a larger frame.

What this means is that the order of your modifiers matter.


Why does SwiftUI use “some View” for its view type?

SwiftUI relies very heavily on a Swift power feature called “opaque return types”, which you can see in action every time you write some View. This means “one object that conforms to the View protocol, but we don’t want to say what.”

Returning some View means even though we don’t know what view type is going back, the compiler does. That might sound small, but it has important implications.

First, using some View is important for performance: SwiftUI needs to be able to look at the views we are showing and understand how they change, so it can correctly update the user interface. If SwiftUI didn’t have this extra information, it would be really slow for SwiftUI to figure out exactly what changed – it would pretty much need to ditch everything and start again after every small change.


The second difference is important because of the way SwiftUI builds up its data using ModifiedContent. Previously I showed you this code:

Button("Hello World") {
    print(type(of: self.body))
}
.frame(width: 200, height: 200)
.background(.red)

The View protocol has an associated type attached to it, which is Swift’s way of saying that View by itself doesn’t mean anything – we need to say exactly what kind of view it is. It effectively has a hole in it, in a similar way to how Swift doesn’t let us say “this variable is an array” and instead requires that we say what’s in the array: “this variable is a string array.”

It is perfectly legal to write a view like this:

struct ContentView: View {
    var body: Text {
        Text("Hello World")
    }
}

Returning View makes no sense, because Swift wants to know what’s inside the view – it has a big hole that must be filled.

If we want to return one of those from our body property, what should we write? While you could try to figure out the exact combination of ModifiedContent structs to use, it’s hideously painful and the simple truth is that we don’t care because it’s all internal SwiftUI stuff.

What some View lets us do is say “this will a view, such as Button or Text, but I don’t want to say what.” So, the hole that View has will be filled by a real view object, but we aren’t required to write out the exact long type.

There are two places where it gets a bit more complicated:
1. How does VStack work – it conforms to the View protocol, but how does it fill the “what kind of content does it have?” hole if it can contain lots of different things inside it?
2. What happens if we send back two views directly from our body property, without wrapping them in a stack?
3. 
To answer the first question first, if you create a VStack with two text views inside, SwiftUI silently creates a TupleView to contain those two views – a special type of view that holds exactly two views inside it. So, the VStack fills the “what kind of view is this?” with the answer “it’s a TupleView containing two text views.”

And what if you have three text views inside the VStack? Then it’s a TupleView containing three views. Or four views. Or eight views, or even ten views – there is literally a version of TupleView that tracks ten different kinds of content:

TupleView<(C0, C1, C2, C3, C4, C5, C6, C7, C8, C9)>

And that’s why SwiftUI doesn’t allow more than 10 views inside a parent: they wrote versions of TupleView that handle 2 views through 10, but no more.

As for the second question, Swift silently applies a special attribute to the body property called @ViewBuilder. This has the effect of silently wrapping multiple views in one of those TupleView containers, so that even though it looks like we’re sending back multiple views they get combined into one TupleView.

This behavior isn’t magic: if you right-click on the View protocol and choose “Jump to Definition”, you’ll see the requirement for the body property and also see that it’s marked with the @ViewBuilder attribute:

@ViewBuilder var body: Self.Body { get }

Of course, how SwiftUI interprets multiple views going back without a stack around them isn’t specifically defined anywhere, but as you’ll learn later on that’s actually helpful.


Conditional modifiers


It’s common to want modifiers that apply only when a certain condition is met, and in SwiftUI the easiest way to do that is with the ternary conditional operator.

As a reminder, to use the ternary operator you write your condition first, then a question mark and what should be used if the condition is true, then a colon followed by what should be used if the condition is false. If you forget this order a lot, remember Scott Michaud’s helpful mnemonic: What do you want to check, True, False, or “WTF” for short.

For example, if you had a property that could be either true or false, you could use that to control the foreground color of a button like this:

struct ContentView: View {
    @State private var useRedText = false

    var body: some View {
        Button("Hello World") {
            // flip the Boolean between true and false
            useRedText.toggle()            
        }
        .foregroundColor(useRedText ? .red : .blue)
    }
}


So, when useRedText is true the modifier effectively reads .foregroundColor(.red), and when it’s false the modifier becomes .foregroundColor(.blue). Because SwiftUI watches for changes in our @State properties and re-invokes our body property, whenever that property changes the color will immediately update.

You can often use regular if conditions to return different views based on some state, but this actually creates more work for SwiftUI – rather than seeing one Button being used with different colors, it now sees two different Button views, and when we flip the Boolean condition it will destroy one to create the other rather than just recolor what it has.

So, this kind of code might look the same, but it’s actually less efficient:

var body: some View {
    if useRedText {
        Button("Hello World") {
            useRedText.toggle()
        }
        .foregroundColor(.red)
    } else {
        Button("Hello World") {
            useRedText.toggle()
        }
        .foregroundColor(.blue)
    }
}

Sometimes using if statements are unavoidable, but where possible prefer to use the ternary operator instead.


Environment modifiers


Many modifiers can be applied to containers, which allows us to apply the same modifier to many views at the same time.
For example, if we have four text views in a VStack and want to give them all the same font modifier, we could apply the modifier to the VStack directly and have that change apply to all four text views:

VStack {
    Text("Gryffindor")
    Text("Hufflepuff")
    Text("Ravenclaw")
    Text("Slytherin")
}
.font(.title)

This is called an environment modifier, and is different from a regular modifier that is applied to a view.
From a coding perspective these modifiers are used exactly the same way as regular modifiers. However, they behave subtly differently because if any of those child views override the same modifier, the child’s version takes priority.

VStack {
    Text("Gryffindor")
        .font(.largeTitle)
    Text("Hufflepuff")
    Text("Ravenclaw")
    Text("Slytherin")
}
.font(.title)


There, font() is an environment modifier, which means the Gryffindor text view can override it with a custom font.

VStack {
    Text("Gryffindor")
        .blur(radius: 0)
    Text("Hufflepuff")
    Text("Ravenclaw")
    Text("Slytherin")
}
.blur(radius: 5)


That won’t work the same way: blur() is a regular modifier, so any blurs applied to child views are added to the VStack blur rather than replacing it.
To the best of my knowledge there is no way of knowing ahead of time which modifiers are environment modifiers and which are regular modifiers other than reading the individual documentation for each modifier and hope it’s mentioned. Still, I’d rather have them than not: being able to apply one modifier everywhere is much better than copying and pasting the same thing into multiple places.


Views as properties

There are lots of ways to make it easier to use complex view hierarchies in SwiftUI, and one option is to use properties – to create a view as a property of your own view, then use that property inside your layouts.
For example, we could create two text views like this as properties, then use them inside a VStack:

struct ContentView: View {
    let motto1 = Text("Draco dormiens")
    let motto2 = Text("nunquam titillandus")

    var body: some View {
        VStack {
            motto1
            motto2
        }
    }
}

You can even apply modifiers directly to those properties as they are being used, like this:

VStack {
    motto1
        .foregroundColor(.red)
    motto2
        .foregroundColor(.blue)
}

Creating views as properties can be helpful to keep your body code clearer – not only does it help avoid repetition, but it can also get more complex code out of the body property.
Swift doesn’t let us create one stored property that refers to other stored properties, because it would cause problems when the object is created. This means trying to create a TextField bound to a local property will cause problems.
However, you can create computed properties if you want, like this:

var motto1: some View {
    Text("Draco dormiens")
}

This is often a great way to carve up your complex views into smaller chunks, but be careful: unlike the body property, Swift won’t automatically apply the @ViewBuilder attribute here, so if you want to send multiple views back you have three options.

First, you can place them in a stack, like this:

var spells: some View {
    VStack {
        Text("Lumos")
        Text("Obliviate")
    }
}

If you don’t specifically want to organize them in a stack, you can also send back a Group. When this happens, the arrangement of your views is determined by how you use them elsewhere in your code:

var spells: some View {
    Group {
        Text("Lumos")
        Text("Obliviate")
    }
}

The third option is to add the @ViewBuilder attribute yourself, like this:

@ViewBuilder var spells: some View {
    Text("Lumos")
    Text("Obliviate")
}

Of them all, I prefer to use @ViewBuilder because it mimics the way body works, however I’m also wary when I see folks cram lots of functionality into their properties – it’s usually a sign that their views are getting a bit too complex, and need to be broken up. Speaking of which, let’s tackle that next…

View composition


SwiftUI lets us break complex views down into smaller views without incurring much if any performance impact. This means that we can split up one large view into multiple smaller views, and SwiftUI takes care of reassembling them for us.

For example, in this view we have a particular way of styling text views – they have a large font, some padding, foreground and background colors, plus a capsule shape:

struct ContentView: View {
    var body: some View {
        VStack(spacing: 10) {
            Text("First")
                .font(.largeTitle)
                .padding()
                .foregroundColor(.white)
                .background(.blue)
                .clipShape(Capsule())

            Text("Second")
                .font(.largeTitle)
                .padding()
                .foregroundColor(.white)
                .background(.blue)
                .clipShape(Capsule())
        }
    }
}

Because those two text views are identical apart from their text, we can wrap them up in a new custom view, like this:

struct CapsuleText: View {
    var text: String

    var body: some View {
        Text(text)
            .font(.largeTitle)
            .padding()
            .foregroundColor(.white)
            .background(.blue)
            .clipShape(Capsule())
    }
}

We can then use that CapsuleText view inside our original view, like this:

struct ContentView: View {
    var body: some View {
        VStack(spacing: 10) {
            CapsuleText(text: "First")
            CapsuleText(text: "Second")
        }
    }
}

Of course, we can also store some modifiers in the view and customize others when we use them. For example, if we removed foregroundColor from CapsuleText, we could then apply custom colors when creating instances of that view like this:

VStack(spacing: 10) {
    CapsuleText(text: "First")
        .foregroundColor(.white)
    CapsuleText(text: "Second")
        .foregroundColor(.yellow)
}

Don’t worry about performance issues here – it’s extremely efficient to break up SwiftUI views in this way.


Custom modifiers

SwiftUI gives us a range of built-in modifiers, such as font(), background(), and clipShape(). However, it’s also possible to create custom modifiers that do something specific.

To create a custom modifier, create a new struct that conforms to the ViewModifier protocol. This has only one requirement, which is a method called body that accepts whatever content it’s being given to work with, and must return some View.

For example, we might say that all titles in our app should have a particular style, so first we need to create a custom ViewModifier struct that does what we want:


struct Title: ViewModifier {
    func body(content: Content) -> some View {
        content
            .font(.largeTitle)
            .foregroundColor(.white)
            .padding()
            .background(.blue)
            .clipShape(RoundedRectangle(cornerRadius: 10))
    }
}

We can now use that with the modifier() modifier – yes, it’s a modifier called “modifier”, but it lets us apply any sort of modifier to a view, like this:

Text("Hello World")
    .modifier(Title())


When working with custom modifiers, it’s usually a smart idea to create extensions on View that make them easier to use. For example, we might wrap the Title modifier in an extension such as this:

extension View {
    func titleStyle() -> some View {
        modifier(Title())
    }
}

We can now use the modifier like this:

Text("Hello World")
    .titleStyle()

Custom modifiers can do much more than just apply other existing modifiers – they can also create new view structure, as needed. Remember, modifiers return new objects rather than modifying existing ones, so we could create one that embeds the view in a stack and adds another view:

Tip: Often folks wonder when it’s better to add a custom view modifier versus just adding a new method to View, and really it comes down to one main reason: custom view modifiers can have their own stored properties, whereas extensions to View cannot.

DAY 24

Views and modifiers: Wrap up

Review for Project 3: Views and Modifiers
  
