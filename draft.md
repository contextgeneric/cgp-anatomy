# The Anatomy of Context-Generic Programming

This document attempts to do a critical anatomical analysis of Context-Generic Programming (CGP), to analyze the source of complexities and workarounds.

## Defining Context-Generic Code

There are many ways to define what is Context-Generic Programming, some based on the libraries or patterns used, or some based on the look of CGP code. In this section, we will try to redefine CGP to be something that is more neutral, to help with our evaluation.

At its core, a context-generic code is any code that can be reused across multiple context (`Self`) types through the use of generics. There are two key components to this definition:

- Code reuse across context types (context polymorphism)
- Code reuse via generics

The first part of the definition, context polymorphism, is not unique to CGP, and has already been done widely in the software ecosystem. In particular with paradigms such as:

- Dynamic dispatch
- Dynamic typing
- Object-Oriented Programming

At its core, these alternative paradigms achieve code reuse largely in the form of type erasure and table lookups during runtime. The main differentiation of CGP with these other paradigms come from the second part, which is that it achieves context polymorphism through generics and table lookups at compile time.

## Example Context-Polymorphic Code

With the definition in place, let's look at a few example code that are context-polymorphic:

### Dyn Trait

The simplest way to write context-polymorphic code in Rust is through the use of dynamic dispatch via `dyn` traits:

```rust
pub fn rectangle_area(context: &dyn RectangleFields) -> f64 {
    context.width() * context.height()
}
```

The main simplicity here is that the syntax and semantics are familiar to developers coming from other language backgrounds, such as Java and Python.

## Impl Trait

In Rust, it is also common to use `impl` traits as a syntactic bridge to move programmers from using `dyn` traits toward generics:

```rust
pub fn rectangle_area(context: &impl RectangleFields) -> f64 {
    context.width() * context.height()
}
```

The use of `impl` traits preseve the same syntax as using `dyn` traits, but changes the underlying semantics to use generics. This enables a seamless transition for Rust developers from other language backgrounds to start using generics without realizing it.

## Generics

The basic form of polymorphism that is encouraged by CGP is to use generics instead of dynamic dispatch, and is written as follows:

```rust
pub fn rectangle_area<Context: RectangleFields>(context: &Context) -> f64 {
    context.width() * context.height()
}
```

This changes the **syntax**, but largely has the same **semantics** as using `impl` traits. However, even this syntactic transition is proven to be challenging to many Rust developers. This shows why syntax matters a lot on programming language design, and the ingenuity of Rust language designers to introduce `impl` traits to disguise the use of generics.

Under the definition of our document, the example code above is considered to be context-generic. This is because it can be reused by any `Context` type that implements `Context`. This is also the main starting point of the perceived complexity when using CGP: that it requires the use of generics.

## Blanket Traits

A more advanced form of context-generic code is in the form of blanket trait implementations:

```rust
pub trait RectangleArea {
    fn rectangle_area(&self) -> f64;
}

impl<Context> RectangleArea for Context
where
    Context: RectangleFields,
{
    fn rectangle_area(&self) -> f64 {
        self.width() * self.height()
    }
}
```

Blanket implementations are also known as extension traits, and its use is already very common in the Rust ecosystem today. Examples include `Itertools` and `StreamExt`. However, many Rust programmers only use these traits on concrete types, without being aware that it is a context-generic blanket implementation. As a result, it is often surprising for one to see this for the first time when reading context-generic code.

Blanket traits are the most idiomatic way to write context-generic code without relying on the `cgp` library. If a developer is interested in using CGP but is worried about being locked in to the `cgp` framework, it is sufficient for them to use only blanket traits for that purpose.

CGP also provides the blanket trait macro to simplify blanket trait definitions. So the same blanket implementation above can be shortened into:

```rust
#[blanket_trait]
pub trait RectangleArea: RectangleFields {
    fn rectangle_area(&self) -> f64 {
        self.width() * self.height()
    }
}
```

### CGP Components

CGP components are advanced form of blanket implementations that workaround the coherence restrictions in Rust, and allow overlapping implementations to be defined:

```rust
#[cgp_component(AreaCalculator)]
pub trait HasArea {
    fn area(&self) -> f64;
}

#[cgp_impl(RectangleArea)]
impl<Context> AreaCalculator for Context
where
    Context: RectangleFields,
{
    fn area(&self) -> f64 {
        self.width() * self.height()
    }
}
```

The look of a context-generic code using CGP component look very similar to blanket traits. As a result, there is little difference in cognitive requirement in understanding context-generic code written with CGP components as compared to blanket traits.

Because of the visual similarity, blanket traits become the defining boundary of what context-generic code looks like - it doesn't matter whether CGP components are used, they both look equally complex.

CGP components also introduce an additional wiring step that is required to be done on the concrete context, before the code could be reused:

```rust
#[derive(HasFields)]
pub struct Rectangle {
    pub width: f64,
    pub height: f64,
}

delegate_components! {
    Rectangle {
        AreaCalculatorComponent:
            RectangleArea,
    }
}
```

This visual noise is distinct enough that it is often perceived as a source of significant complexity by many readers. However, this is something that is avoidable, if there is only one context-generic implementation that is applicable to all contexts. In that case, there are only few lines of changes required to "downgrade" a context-generic code from using CGP component to become vanilla blanket traits.

### Plain Functions

Another approach of writing context-generic code is by simply writing plain functions:

```rust
pub fn rectangle_area(width: f64, height: f64) -> f64 {
    width * height
}
```

A plain function is context-generic, or more accurately, "context-free", because it does not depend on any context type at all. This means that it can always be called from any context, as long as there are explicit ways to pass the parameters around.

In fact, we can argue that functional programming is the purest form of context-generic programming. If everything is written as plain functions with explicit parameters that do not depend on some global contexts, they can easily be reusable by any concrete context regardless of how complex or simple the contexts are.

That said, in practice, there are many shortcomings of writing plain functions for everything, which we won't go into details here. But in many ways, CGP originates from the attempts to improve the ergonomics of functional programming to make them more reusable across concrete contexts.

This is also why many context-generic code that I have written follow the functional-style of organizing constructs - they are written as a form of "super functions" that can be called and composed more easily than plain functions. When a CGP component trait only has one method, there is a one-to-one correspondance between a CGP provider and a plain function.

This also means that if blanket traits are deemed too complex, and if functions over generic context are still too confusing, the best way to "downgrade" a context-generic code into its simplest form is by rewriting them as plain functions.

## Benefits provided by CGP

There are many ways context-generic code can be written, but without clear benefits, one would question why we should write context-generic code instead of context-specific code. Fundamentally, there are 3 core benefits provided by CGP, that are otherwise challenging to be provided by alternative patterns:

### Getter Methods (Dependency injection of values)

The first benefit that CGP brings is that it enables a form of structural typing through getter methods. This is also a form of dependency injection that is most familiar to developers from other background like Spring.

In the earlier example, we already seen getter traits in action through the example `RectangleFields` getter trait. But with the help of `#[cgp_auto_getter]`, the implementation of `RectangleFields` can be further streamlined, requiring the context to only derive `HasField` to implement the trait automatically.

```rust
#[cgp_auto_getter]
pub trait RectangleFields {
    fn width(&self) -> f64;

    fn height(&self) -> f64;
}

pub trait RectangleArea {
    fn rectangle_area(&self) -> f64;
}

impl<Context> RectangleArea for Context
where
    Context: RectangleFields,
{
    fn rectangle_area(&self) -> f64 {
        self.width() * self.height()
    }
}
```

### Abstract Types (Dependency injection of types)

The second benefit is that the generic context in CGP can act as a type dictionary, to be passed as a single parameter instead of multiple generic parameters.

For example, we might want to generalize the area function earlier to work with any scalar type. Without abstract types, this would need to be passed as an additional generic parameter such as follows:

```rust
#[cgp_auto_getter]
pub trait RectangleFields<S: Scalar> {
    fn width(&self) -> S;

    fn height(&self) -> S;
}

pub trait RectangleArea<S: Scalar> {
    fn rectangle_area(&self) -> S;
}

impl<Context, S: Scalar> RectangleArea<S> for Context
where
    Context: RectangleFields<S>,
{
    fn rectangle_area(&self) -> S {
        self.width() * self.height()
    }
}
```

The passing of this generic `S` type can quickly become tedious, if the type is used everywhere. But with abstract types, the code can be simplified into:

```rust
#[cgp_type]
pub trait HasScalarType {
    type Scalar: Scalar
}

#[cgp_auto_getter]
pub trait RectangleFields: HasScalarType {
    fn width(&self) -> Self::Scalar;

    fn height(&self) -> Self::Scalar;
}

pub trait RectangleArea: HasScalarType {
    fn rectangle_area(&self) -> Self::Scalar;
}

impl<Context> RectangleArea for Context
where
    Context: RectangleFields,
{
    fn rectangle_area(&self) -> Self::Scalar {
        self.width() * self.height()
    }
}
```

The benefits of abstract types becomes more apparent as the complexity of the code base increases and more generic parameters are involved. Abstract types significantly improve the ergonomic and scalability of the use of generic types, so that there is no problem even when there are a dozens of them.

### Configurable Static Dispatch

The most distinguish feature that CGP offers is configurable static dispatch in the form of type-level lookup tables. It allows multiple overlapping provider implementation of a trait to co-exist, and be chosed by a concrete context using `delegate_components!` to setup the wiring.

## Fusion- vs Fission-Driven Development

In this document, we introduce a new axis of categorizing design patterns, which we would call fusion- vs fission-driven development.

There are many CGP patterns available, each solving programming problems differently with different levels of complexities. However, there is one defining feature that exist across all CGP patterns, that distinguish them from classical programming patterns:

**CGP solves programming problems by introducing different contexts for different purposes.**

Basically, every programming challenge is ultimately answered the same way by CGP, regardless of the specific programming techniques used:

- How do I customize a program downstream? - define a new context.
- How do I add a new variant to the program? - define a new context.
- How do I configure the program to run differently on different platforms? - define a new context for each platform.

On the other hand, classical design patterns are designed around the constraints of there being a globally unique context. For example:

- How do I customize a program downstream? - fork or use a dynamic plugin system.
- How do I add a new variant to the program? - use enum or dyn traits.
- How do I configure the program to run differently on different platforms? - use feature flags.

To better describe the two approaches, we can say that classical Rust design patterns have a **fusion-driven approach**, while CGP design patterns have a **fission-driven approach** toward solving programming problems - One wants to *fuse* the need of multiple contexts into one, while another wants to *split* one context into multiple contexts.

This fundamental difference in philosophy is the main tension that separates CGP from classical programming. CGP only really brings benefits when there are at least two concrete contexts, and works better as the number of contexts increases. On the other hand, classical programming patterns work best when there can only be one concrete context, and introduces strong friction to prevent alternative contexts from being defined.

### Correspondance to Initial vs Final Encoding

The juxtaposition between fusion-driven development and fission-driven development has a deep correspondance to the theoretical foundation of initial vs final encoding.

Fusion-driven development makes use of initial encoding techniques to group distinct types together in the form of enums or type-erased values such as `Any`, `dyn` traits, or existential types. On the other hand, fision-driven development make use of final encoding techniques to reuse code across distinct types in the form of generics and typeclass constraints.

### Fusion-Driven Patterns

Here are some example fusion-driven patterns in Rust:

#### Enum

```rust
pub enum AnyDatabase {
    Postgres(Postgres),
    Sqlite(Sqlite),
}

pub struct App {
    pub database: AnyDatabase,
    pub api_client: ApiClient,
}
```

#### Feature flags

```rust
pub struct App {
    #[cfg(feature = "postgres")]
    pub database: Postgres,

    #[cfg(feature = "sqlite")]
    pub database: Sqlite,

    pub api_client: ApiClient,
}
```

#### `Box<dyn Trait>` objects


```rust
pub struct App {
    pub database: Box<dyn Database>,
    pub api_client: ApiClient,
}
```

#### Generic structs


```rust
pub struct App<Database> {
    pub database: Database,
    pub api_client: ApiClient,
}
```

#### Context-Specific Code

```rust
fn query_user(app: &App, user_id: &UserId) -> Result<User> {
    ...
}
```

#### Trait impl on concrete types

The Rust language community already gives plenty of guidelines to programmers on how to write reusable code that can work with multiple types that implement the same trait, either with generics or dyn traits. Although these programming patterns can be considered context-generic, one notable lack of such use is when writing trait impls.

Due to the coherence restrictions, many application-level Rust traits are defined with the assumption that they will be implemented on concrete types. This also means that the trait impl body of one context cannot be easily reused by the trait impl body of another context, even if they share some common behavior.

For example:

```rust
pub trait Application {
    type Database;

    fn database(&self) -> &Self::Database;

    fn api_client(&self) -> ApiClient;

    fn query_user(&self, user_id: &UserId) -> User;

    fn fetch_file(&self, file_id: &FileId) -> Result<Vec<u8>>;

    fn send_email(&self, recipient: &EmailAddress, message: &str) -> Result<()>;
}
```

The trait impl on concrete types can be considered a fusion pattern, since if the target context is split into two, the trait impl body that share the same implementation has to be fully duplicated. This makes it difficult to apply fission on contexts that contain concrete trait impls, and thus incentivize the developer to prevent the context from splitting.

When fission becomes inevitable, one common way to workaround it is by having light weight implementation of each method of the trait to call free standing functions, which can be context-generic. This way, even if two similar contexts need to implement the same trait, their impl body would only consist of light weight calls to plain functions.

Although this approach works, the amount of boilerplate and the need to define helper plain functions makes if far from usable for every trait in the code base. As a result, many trait impls retain the fusion property and this contributes to the resistance of fission being applied to the corresponding contexts.

It is worth noting that even though a trait can be implemented for multiple concrete contexts, the implementations as a group are considered neither fusion-driven or fission-driven. This is because each of the trait impl are fully independent, with no code shared between the different trait impl blocks.

In contrast, the code that be reused on any type that implements a trait, such as generic functions, or even generic methods from another trait, are considered fission-driven. This is because it is those fission property that enables the trait to be implemented by multiple concrete contexts in order to use those functions.


### Other Fission-Driven Patterns

Desipite the intimidating name, fission-driven development is already widely used in the wild in other programming languages, particularly in dynamic-typed languages and object-oriented languages.

The lack of fission-driven techniques is more apparent in languages like Rust, which lack dynamic-typed or object-oriented features.

#### Duck Typing

It is worth noting that aside from CGP, there are other fission-driven design patterns already available in the wild. Most notably, fission techniques are already widely used in dynamic-typed languages in the form of duck typing.

Without the constraint of static types, fission is trivially achieved in dynamic-typed languages by simply creating objects with the required fields.

That said, fission-driven development in a dynamic-typed environment isn't without its own serious limitation: any unsatisfied dependency would only surface during runtime when the field is accessed. Worse yet, unintended type coercion can make certain mistakes incredibly difficult to debug.

Due to the lack of compile-time checks, developers have been cautious to employ fission-driven development techniques in dynamic-typed languages. In contrast, CGP brings the novelty of achieving the same level of expressiveness as how fission-driven development could have been done in dynamic-typed languages, if there were ways to catch the configuration errors at compile time.

#### Inheritance and Subtyping

In static-typed Object-oriented programming languages, fission-driven code can easily be written by targetting a base class. That same code can be reused for any class that inherits from the base class.

However, fission-driven development in OOP is more restricted than in dynamic-typed languages, due to challenges in multiple inheritance. Because a subclass can typically only inherit from one base class, this significantly limits how that subclass from being reused on fission code that depend on other base classes.

Due to this limitation, even in OOP, fission-driven development is not often done through inheritance and subtyping alone. Instead, OOP developers tend to resort to other techniques like dynamic typing and reflection to enable fission-driven development.

#### Runtime Reflection

Runtime reflection is typically designed the escape hatch to work around the limitations of fission-driven development in static-typed languages. It enables a fission code to query for arbitrary capabilities from a type-erased value, and then perform action based on whether the requested capability is found.

Reflection is already widely used in Rust, particularly through reflection-based frameworks like Bevy. Most natably, Bevy functions are fission-driven code that can be used on any context that contains specific property. More recently, reflection have been proven useful and popular enough that it is being considered to be supported as native language feature in Rust. This shows that Rust programmers are not totally against fission-driven development.

While reflection is very powerful, it requires sacrificing a large part of static typing and return to an environment that is much closer to dynamic-typed programming. When a reflection-based fission code fails due to missing dependency, it result in runtime errors that may be difficult to debug. Type erasure also prevents the compiler from fully optimizing reflection-based fission-driven code. This can result in slow down of programs that make heavy use of reflection.

Although reflection may have less cognitive overhead than the generic approach, it nevertheless has some viral effect that affect the code base in similar ways as CGP. It is just that instead of working with a generic context, reflection-based fission code need to work with a type-erased context. This would still require a large part of the code base to be designed to work with this type-erased context. But this brings less anxiety to developers, because this style of code look similar to dynamic-typed code that is already familiar to many developers.

#### Algebraic Effects

Even functional programming has well-supported fission-driven patterns. Although functional programming tend to lack features such as dynamic typing and reflection, they are compensated by equally powerful features like existential types, GADTs, and initial encoding.

Most notably, advanced functional applications tend to use fee monad and algebraic effects to abstractly represent the effects and operations that a code has performed.

Although there is no explicit context type in algebraic effects, we can think of the canonical context type being the environment type that is typically found in a reader monad, or a reader effect. Application-specific effect handlers tend to use the reader effect to retrieve parameters from this implicit context, such as the database config, to handle an effectful operation such as database query.

The fission-property of algebraic effects can also be achieved through getter effects, which bear similarity to getter methods in CGP. For example, instead of using the reader effect to get a concrete context that contains a database config field, an effect handler can depend on a getter effect like `get_database_config` to treat the act of getting the database config field as an effectful operation.

Algebraic effects offer a more advanced form of fission-driven development, because it allows the code to work not only with any context, but also with any effect.

In comparison, CGP is less powerful than algebraic effects, but also simpler. Without an effect system, one can't use CGP to emulate advanced effect manipulation of the continuation, such as calling the continuation zero or multiple times. However, when it comes to simple effects like reader or getter effects, or for application-specific effects that always handled by calling the continuation once, CGP arguably provide simpler semantics and ergonomics in handling them.

Moreover, since algebraic effects are based on initial encoding, existential types and GADTs, they are arguably still a form of type erasure that require runtime dispatch. On the other hand, since CGP is based on final encoding, it brings more advantage in terms of performance and simplicity of reasoning.

#### Comparing CGP with other fission-driven patterns

When comparing with other fission-driven development approaches, we can see that CGP is unique in how it enables fission-driven patterns through generics and final encoding.

### Why Rust is fusion-driven

As we can see, the concept of fission-driven development is not entirely unqiue. Instead, if we look across the broader programming language design space, it look like Rust is more like an outlier as a fusion-centric language, as compared to many other languages that already naturally support fission-centric code.

The main reason Rust programmers put more focus in fusion-driven patterns also lies in the limited support of the language for other fission-driven patterns. As a static-typed language, it is awkward to use dynamic-typing to achieve fission in Rust. And since Rust strongly rejects the classical notion of OOP inheritance and subtyping, it is also not possible to achieve fission through that route.

The lack of fission-driven patterns is also a reason why many beginner developers coming from other language background like to ask for support for language features like inheritance or dynamic typing. What they are really looking for is how to avoid fusion and continue writing fission-driven code like in other languages that they are used to.

This also brings a unique perspective on the fusion-driven culture in Rust. The lack of fission-driven features have drive Rust developers to become strong proponent of fusion-driven patterns. And because fusion-driven development is relatively unique to Rust, they take pride in this uniqueness that differentiate them from other languages.

Consequently, this creates a divide between the fusion-driven mindset of Rust developers as compared to the fission-driven mindset of other languages. Due to how fission-driven features are typically supported, Rust developers tend to strongly associate fission-driven patterns with specific programming paradigms like OOP or dynamic typing, and consider them as anti-pattern. As a result, when they first encounter CGP and see fission-driven code in Rust, this can evoke a strong negative reaction that makes them think that CGP achieves fission the same way as these other languages, which are not desirable in Rust.

Despite that, the most prominant use of fission patterns in Rust is through the use of runtime reflection in frameworks like Bevy. This also demonstrates that Rust users still have strong desire to have some ways to support fission-driven development in Rust.

### The tension of adopting CGP

This tension of multi-context vs single-context implementations is arguably the biggest obstacle in convincing Rust developers to adopt CGP. Since CGP can only bring value with multiple concrete contexts, the very first action to adopt it is to write all code to be generic and fully decoupled from the concrete context, and then define multiple contexts that make use of the code.

But this action is the complete opposite of how Rust developers are used to writing code. The conventional wisdom and default assumption is that splitting monolithic contexts and reusing code between them is very challenging and often not worth the effort. So developers try very hard to keep only one context. As a result, the first question that developers will inevitably ask is: why would I want to have two contexts instead of keeping just one context, and why would I want to write context-generic code when I can write them as methods on just one concrete context?

This results in the ultimate dilemma to get CGP adopted: if there is only one concrete context, CGP cannot bring meaningful value since everything can instead be written concretely. But if there is more than one concrete contexts, developers are not used to that level of complexity and are forced to write everything as context-generic code.

The main challenge is that there is no simple middleground that can help a developer to transition from 1 to N contexts. Technically, we could still write context-generic code when there is only one context, but that does not help in motivating the benefits CGP can bring. And although developer can still make use of classical patterns with multiple contexts, they are forced to use these patterns on top of CGP. In other words, CGP becomes viral and unavoidable once there are two contexts that need to reuse the same code.

### Use of CGP in mono-contexts

It is worth clarifying that even though CGP cannot deliver its primary benefits to code bases with monolithic contexts, it is totally possible and still useful to use CGP even for monolithic contexts.

In particular, the use of CGP in mono-contexts bring forward-compatibility for ease transition to multiple contexts in the future, when the need arise.

CGP can also be used to narrow the capabilities given to a specific method for a mono-context. If the mono-context contains many fields, it is not always clear which part of the context is accessed by a given method by just looking at its method signature. But it the method is written as a context-generic function, then the exact capability needs to be specified in its trait bound, and thus become more explicit.

The main challenge is that these secondary benefits of CGP are often not sufficient to justify their use in a mono-context code base. Skeptics would argue to that the complexity of CGP are not worth the benefits CGP provide, and they would be right since there is only one mono-context available.

Even when code bases transition to use CGP with fission-driven development, they always have the option to switch back to mono-contexts if deemed necessary. There is no need to rewrite all context-generic code to target the mono-context, since they can work with any context that satisfy the requirement, whether there is one or many of them.

The main challenge of transitioning back from multi-context to mono-context lies more on the need to apply fusion-driven patterns to merge different contexts back to one again. During this process, one may encounter difficulty in applying fusion and discover that the fusion-correspondance of a CGP pattern may not be as powerful as they expect. After all, the fission-driven approach of CGP is not just philosophical, but arguably more powerful than the corresponding fission-driven approaches to justify the switch from fusion to fission.

## The primary unresolvable complexity of CGP

In summary, I'd argue that the true source of complexity that developers perceive about CGP comes from the fact that they need to manage two or more contexts in their code base. This can feel very uncomfortable because it is a significant departure from the comfort zone of writing code for only one monolithic context.

It is also very difficult to have a conversation about how to tame this complexity, because the most obvious way to do so is to merge everything back to one context again. But at that point, it is hard to use CGP to solve problems because it can only solve them by introducing new contexts.

Furthermore, the complexity of reusing code between two or more contexts cannot be easily tamed using classical patterns. Even when classical patterns are used, they would only help if that result in merging all contexts back into one. This gives a feeling of lock in, because CGP becomes the only idiomatic way of reusing code between two or more contexts.

This fundamental incompatibility is why adopting CGP can feel so radical and disruptive. It also means that if one cannot accept the complexity of introducing two or more contexts to their code base, then there would be no room for accepting CGP regardless of the theoretical benefits it can bring.

Nevertheless, it is important to note that this is all based on the subjective perception of whether having two or more contexts is really a source of complexity. Objectively speaking, the whole point of CGP is that it elegantly tames the complexity of sharing code between multiple contexts, and thus making it trivial to create as many contexts as one needs. But in order to prove it convincingly, one must first try it out by introducing multiple contexts into their code base. This requires a very high initial cost that one might not necessarily willing to pay.

This means that regardless of the perceived or actual complexity, one must first accept the use of more than one contexts as the necessary complexity that is required to use CGP.

## Taming secondary sources of complexity

There are other secondary sources of complexities, but they are selective features of CGP can be opted out, while still benefit from other parts of CGP.

### Abstract Types

The most significant secondary source of complexity is the use of abstract types. This is mainly because generics alone are already complicated, and abstract types implies even heavier use of generics.

However, the use of abstract types are optional, and one can completely opt out of it and still gain the additional benefits provided by CGP.

### Configurable Static Dispatch

The most complained source of complexity in CGP is the use of configurable static dispatch. This is because the use of `delegate_components!` together with type-level lookup tables is an unfamiliar concept to most Rust developers. As a result, it can feel challenging to write the first CGP wiring, even if it is effectively just specifying keys and values.

The best we can tame this complexity is by minimizing the number of traits that make use of configurable static dispatch. But this has to be worked backward based on the differences between the contexts available. For example, if there are two contexts in the code base, each context has different behavior for one component, then that component is most likely needs to be implemented and reused through static dispatch.

However, even for that, there is a possible workaround to tame this source of complexity, by implementing the component as regular Rust trait. For example, given:

```rust
pub struct Config { ... } // complex config

#[derive(HasField)]
pub struct FooApp {
    pub config: Config,
    pub foo: Foo,
}

#[derive(HasField)]
pub struct BarApp {
    pub config: Config,
    pub bar: Bar
}
```

Suppose we want to implement a generic `process_stuff` function that can work with both `FooApp` and `BarApp` using a `Config`, but with some custom logic that depends on either `Foo` or `Bar`, we can write something like follows:

```rust
#[cgp_component(FooOrBarProcessor)]
pub trait CanProcessFooOrBar {
    fn process_foo_or_bar(&self);
}

pub trait CanProcessStuff {
    fn process_stuff(&self);
}

impl<Context> CanProcessStuff for Context
where
    Context: HasConfig,
{
    fn process_stuff(&self) {
        let config = self.config();
        // do something
        self.process_foo_or_bar();
        // do something else
    }
}
```

Essentially, we define a `CanProcessFooOrBar` CGP component with configurable static dispatch, so that it can be implemented differently for `FooApp` and `BarApp`. On the other hand, the implementation of `CanProcessStuff` is applicable for all contexts, so it can be defined as a blanket trait without using CGP component.

Now, the typical CGP way to implement `CanProcessFooOrBar` differently for each context is to use configurable static dispatch, such as:

```rust
#[cgp_impl(ProcessWithFoo)]
impl<Context> FooOrBarProcessor for Context
where
    Context: HasFoo,
{
    fn process_foo_or_bar(&self) { ... }
}

#[cgp_impl(ProcessWithBar)]
impl<Context> FooOrBarProcessor for Context
where
    Context: HasBar,
{
    fn process_foo_or_bar(&self) { ... }
}

delegate_components! {
    FooApp {
        FooOrBarProcessorComponent:
            ProcessWithFoo,
    }
}

delegate_components! {
    BarApp {
        FooOrBarProcessorComponent:
            ProcessWithBar,
    }
}
```

But if `ProcessWithFoo` and `ProcessWithBar` are only used once, they can technically be instead implemented on `FooApp` and `BarApp` directly:

```rust
impl CanProcessFooOrBar for FooApp {
    fn process_foo_or_bar(&self) { ... }
}

impl CanProcessFooOrBar for BarApp {
    fn process_foo_or_bar(&self) { ... }
}
```

This way, we would be able to not use any configurable static dispatch, but still keep the code for `CanProcessStuff` context-generic and be able to maintain two contexts.

### Use of Fusion-Driven techniques with CGP

In a way, the direct implementation of `CanProcessFooOrBar` on `FooApp` and `BarApp` can be considered a *fusion* technique. That is, the classical approach of Rust trait `impl` on concrete types are fusion techniques.

This is because when a trait is implemented directly on a concrete context, it tightly couples the trait implementation with that time. This is a fusion property, as it prevents the context to be split by fission techniques later on, since it would then require code duplication of the trait implementation if it needs to be reused by the second context.

For the example case, the trait impl introduces a partial fusion effect on `FooApp` and `BarApp`, and prevent them from being split further. When the time comes, if a context like `FooApp` needs to be split, a fission technique need to be re-applied to make it splittable again. That is, we can use CGP providers and configurable static dispatch as as the fission counterpart of vanilla trait impl to make the implementation of `CanProcessFooOrBar` to be shared across two split contexts like `FooAppA` and `FooAppB`.

It is also worth noting that the classical fusion techniques are not necessarily incompatible with the fission techniques from CGP. In fact, they can be used together when appropriate to prevent a nuclear explosion of too many contexts being created due to the combinatory explosive nature of configuration permutations.

For example, one can re-introduce fusion techniques like `dyn` traits or enums for a specific context, to allow it to be configurable at runtime instead of compile time. On the other hand, fission techniques like CGP can be used again to replace a `dyn` trait or enum during appropriate time, such as when dyn-compatibility cannot be satisfied or when extensibilty is required.

#### Generic Structs vs Multiple contexts

One notable fusion-driven pattern that shares many similarities with CGP is the use of context structs that contain generic parameters. For example:

```rust
pub struct ClassicApp<Database, ApiClient> {
    pub database: Database,
    pub api_client: ApiClient,
    ...
}
```

The example `ClassicApp` struct makes use of generic parameters to enable its field members to contain different types, yet still maintain a single definition of the struct.

Although `ClassicApp` could preserve the mono-context property of fusion-driven applications, it achieves that in exchange for verbose implementations that require specifying all generic parameters for every method implementation. For example:

```rust
impl<Database, ApiClient> ClassicApp<Database, ApiClient>
where
    Database: sqlx::Database,
{
    fn query_user(&self, user_id: &UserId) -> anyhow::Result<User> {
        ...
    }
}
```

In the above example, when implementing the `query_user` method, the extra `ApiClient` generic parameter still needs to be specified even though it is not really used by the particular impl.

This shows a significant limitation of the fusion-driven generic structs pattern: as the number of generic fields grow, it quickly becomes tedious and unmanageable to add all generic parameters to each impl side. Worse yet, each generic parameter is position-based, and all impl definitions need to be updated when a new generic parameter is added or removed, even if they are not used for a particular impl.

In contrast, the fission-driven approach provided by CGP significantly improves the ergonomic of generic implementation, for example:

```rust
pub trait HasDatabase {
    type Database: sqlx::Database;

    fn database(&self) -> &Self::Database;
}

pub trait CanQueryUser {
    fn query_user(&self, user_id: &UserId) -> anyhow::Result<User>;
}

impl<Context> CanQueryUser for Context
where
    Context: HasDatabase,
{
    fn query_user(&self, user_id: &UserId) -> anyhow::Result<User> {
        ...
    }
}
```

With CGP's approach, the concrete struct is replaced by a generic context, and `Database` becomes an abstract type. The blanket trait `CanQueryUser` can now make use of dependency injection to get a generic `Database` from the context, while also ignore other generic or abstract types such as `ApiClient`.

Now when it comes to a concrete context, a fully fission-driven approach may require four contexts to be defined when there are 2 different ways to configure the database, and 2 different ways to configure the API client, such as:

```rust
pub struct PostgresProductionApp {
    pub database: Postgres,
    pub api_client: AwsApiClient,
}

pub struct SqliteProductionApp {
    pub database: Sqlite,
    pub api_client: AwsApiClient,
}

pub struct PostgresMockApp {
    pub database: Postgres,
    pub api_client: MockApiClient,
}

pub struct SqliteMockApp {
    pub database: Sqlite,
    pub api_client: MockApiClient,
}
```

As we can see, the fission-driven approach can quickly go out of hand, if there is an actual need to make use of all possible configurations. However, it is totally fine to use fusion-driven patterns to tame this nuclear explosion, so that we can keep a small number of contexts in the end, such as:

```rust
pub struct ProductionApp<Database> {
    pub database: Database,
    pub api_client: AwsApiClient,
}

pub struct MockApp<Database> {
    pub database: Database,
    pub api_client: MockApiClient,
}
```

In the above example, we make use of fusion to fuse the `database` field with generic parameters, but use fission to separate the `api_client` field into two contexts. With this configuration, we can use fission to statically eliminate the possibility of instantiating a mock API client in production, but use fusion to reuse the same context for different database types.

It is also worth noting that even when generic structs are used, CGP can still be used to improve the ergonomics by hiding all the other generic parameters. For example, both `ProductionApp` and `ClassicApp` can implement `HasDatabase` to reuse `CanQueryUser`:

```rust
impl<Database> HasDatabase for ProductionApp<Database>
where
    Database: sqlx::Database,
{
    type Database = Database;

    fn database(&self) -> &Self::Database {
        &self.database
    }
}

impl<Database, ApiClient> HasDatabase for ClassicApp<Database, ApiClient>
where
    Database: sqlx::Database,
{
    type Database = Database;

    fn database(&self) -> &Self::Database {
        &self.database
    }
}
```

This also demonstrates that even in mono-context settings, CGP techniques can be used to hide unrelated generic parameters like `ApiClient` from a concrete struct like `ClassicApp`. And when it becomes necessary, the existing use of CGP would allow fission to be applied to `ClassicApp` and split it into multiple contexts like `ProductionApp` and `MockApp`.

It is also worth noting that other fusion techniques can similarly be applied to reduce the number of contexts. For example:

```rust
pub enum AnyDatabase {
    Postgres(Postgres),
    Sqlite(Sqlite),
}

pub struct ProductionApp {
    pub database: AnyDatabase,
    pub api_client: AwsApiClient,
}

pub struct MockApp {
    pub database: AnyDatabase,
    pub api_client: MockApiClient,
}
```

In the above example, we make use of enums instead of generic structs to allow both `ProductionApp` and `MockApp` to be instantiated with either `Postgres` or `Sqlite`. But as long as the same constraints are satisfied, i.e. `sqlx::Database` is implemented for `AnyDatabase`, then existing context-generic code like `CanQueryUser` can still be reused seamlessly on it.

This also demonstrates another secondary benefit that CGP brings: we can make use of fission to not only create specialized structs, but also allow seamless transition to use different fusion patterns on existing code without requiring significant refactoring of the code base.

Furthermore, the fission property of the application means that in case if external users want to support new databases in the future, such as `MySql`, they can still use fission to create new third party contexts such as:

```rust
pub enum ExtendedAnyDatabase {
    Postgres(Postgres),
    Sqlite(Sqlite),
    MySql(MySql),
}

pub struct ThirdPartyApp {
    pub database: ExtendedAnyDatabase,
    pub api_client: AwsApiClient,
}
```

In other words, when fusion techniques like enums are used together with fission, it removes the bottlneck of an enum becoming the bottleneck of extensibility. Third party users no longer need to request the project a new `MySql` variant to `AnyDatabase` or threaten to fork the project, since they can define their own enums to do so with minimal effort.

### Getter Traits

Finally, despite it being the simplest of all patterns, getter traits are often perceived as main source of complexity, especially for developers with experience with dependency injection frameworks like Spring.

Technically, the use of getter traits in CGP is mainly to emulate structural typing and row polymophism. It is mainly a fission technique to allow a context-generic code to access the fields in a context after it has been split from one into many. Without it, it wouldn't be possible to write any context-generic code at all, because the only other way to read the fields is to have access to the concrete type.

One possible explanation is that auto getter traits feel too magical when Rust developers are first exposed to it. Since the trait is implemented automatically, there lacks an explicit trace of how the field of a context is accessed.

Originally, the auto getter traits were introduced by CGP to reduce the boilerplate of needing to implement them manually for each context. But if the main goal is to make getter traits feel less magical, the simplest way is to let the developers implement them the same way as any vanilla Rust trait. For example:

```rust
#[cgp_auto_getter]
pub trait HasConfig {
    fn config(&self) -> &Config;
}

pub struct FooApp {
    pub config: Config,
    pub foo: Foo,
}

impl HasConfig for FooApp {
    fn config(&self) -> &Config {
        &self.config
    }
}

pub struct BarApp {
    pub config: Config,
    pub bar: Bar
}

impl HasConfig for BarApp {
    fn config(&self) -> &Config {
        &self.config
    }
}
```

The key to understanding is that an auto getter trait like `HasConfig` is still just a Rust trait that happen to have a special blanket implementation. But if a context like `FooApp` don't derive `HasField`, it can just implement `HasConfig` manually because it don't overlap with the blanket implementation.

This can hopefully reduce the perception that getter traits are too magical, and thus too complex. At the very least, this requires a delicate trade off between accepting the complexity caused by the boilerplate of getter trait impls, or the complexity caused by the automagical implementation of auto getter traits.

### Functional Context-Generic Programming

A final secondary source of complexity comes when functional programming techniques are applied together with CGP. When a CGP component trait contains only one method, its provider can be treated like a plain function. With that, one can apply functional programming techniques such as higher order functions, in the form of higher order providers in CGP.

Suppose that we want to always apply a scale factor when calculating area, a naive way to do so is to define multiple providers like `ScaledRectangleArea`, `ScaledCircleArea`, etc, that correspond to each non-scaled providers. However, this requires duplication of the core area calculation logic, which can be error prone. An functional way to solve this in CGP is to define a higher order provider like `ScaledArea` as follows:

```rust
#[cgp_impl(new ScaledArea<InnerCalculator>)]
impl<Context, InnerCalculator> AreaCalculator for Context
where
    Context: HasScaleFactor,
    InnerCalculator: AreaCalculator<Context>,
{
    fn area(&self) -> f64 {
        InnerCalculator::area(self) * self.scale_factor()
    }
}
```

With higher order providers, we can now define a scaled rectangle area provider simply as a type alias:

```rust
type ScaledRectangleArea = ScaledArea<RectangleArea>;
```

Higher order functions are very common in functional programming languages like Haskell. However, higher order functions have not been traditionally be used together with generic context that contains additional constraint. Furthermore, the Rust syntax for defining higher order functions is not very ergonomic, making it awkward to make use of higher order functions in Rust.

For example, the `ScaledArea` provider is roughly equivalent to the following higher order function:

```rust
fn scaled_area<Context: ScaleFactorField>(
    inner_calculator: impl Fn(&Context) -> f64,
) -> impl Fn(&Context) -> f64 {
    move |context| inner_calculator(context) * context.scale_factor()
}
```

As we can see, the `scaled_area` function needs to accept an `impl Fn` closure and return a new `impl Fn` closure. Furthermore, the body needs to call `move` to capture the `inner_calculator` closure from the function parameter to use it in the returned closure.

The verbose syntax together with the need to relate closures with `Fn` traits make it significantly harder to define and use higher order functions in Rust, as compared to languages like Haskell. Worse, even with the higher order function defined, there is no simple way to define a function equivalent to `ScaledRectangleArea` through composition at the top level. Conceptually, we want to be able to write something like:

```rust
fn scaled_rectangle_area = scaled_area(rectangle_area);
```

But something like that is not possible in Rust today. Even in languages like Haskell, if the generic function contains additional constraints, those constraints must be propagated explicitly in the type signature of the composed function. So we would need to write something like:

```rust
fn scaled_rectangle_area<Context: RectangleFields + ScaleFactorField>(context: &Context) -> f64
    = scaled_area(rectangle_area);
```

which is cosiderably more verbose than they could be. As for today's Rust, the best we can write to perform such function composition is as follows:

```rust
fn scaled_rectangle_area<Context: RectangleFields + ScaleFactorField>(context: &Context) -> f64 {
    scaled_area(rectangle_area)(context)
}
```

which does not really leverage the property of higher order functions, since the same closure needs to be reconstructed every time the function is called.

Because of this, Rust functions are rarely composed using higher order functions. Idiomatically, the functions `scaled_area` and `scaled_rectangle_area` should instead be written as follows:

```rust
fn scaled_area<Context: ScaleFactorField>(
    context: &Context,
    inner_calculator: impl Fn(&Context) -> f64,
) -> f64 {
    inner_calculator(context) * context.scale_factor()
}

fn scaled_rectangle_area<Context: RectangleFields + ScaleFactorField>(context: &Context) -> f64 {
    scaled_area(context, rectangle_area)
}
```

The exploration shows that CGP provides a more concise syntax for composition of generic functions containing constraints. The definition of a composed provider like `ScaledRectangleArea` is even shorter than how they would need to be written in Haskell, even when Haskell has better support on higher order functions.

However, despite the significant advantage of using functional programming with CGP, this presents a significant learning curve to many Rust developers, who are also unfamiliar with functional programming in addition to CGP. The need to learn not only one but two new paradigms at once and using it together is enough to put of even the most dedicated learner.

Instead of functional programming, Rust developers tend to come from OOP background, which has a different philosophy in designing interfaces. Instead of defining a trait based on a behavior like `AreaCalculator`, many Rust developers prefer to define traits based on a conceptual entity like `Shape`, for example:

```rust
#[cgp_component(ShapeProvider)]
pub trait Shape {
    fn area(&self) -> f64;

    fn scale_factor(&self) -> f64;

    fn perimeter(&self) -> f64;

    // ...
}
```

The Rust developer would prefer an upfront design of all possible methods a shape could have, and put them all in a single trait. Although this makes sense from an object-oriented perspective, this makes it more challenging to define higher-order providers like `ScaleArea`, which would have to implement all other methods as passthrough:

```rust
#[cgp_impl(new ScaledArea<InnerProvider>)]
impl<Context, InnerProvider> ShapeProvider for Context
where
    InnerProvider: ShapeProvider<Context>,
{
    fn area(&self) -> f64 {
        InnerProvider::scale_factor(self) * InnerProvider::area(self)
    }

    fn scale_factor(&self) -> f64 {
        InnerProvider::scale_factor(self)
    }

    fn perimeter(&self) -> f64 {
        InnerProvider::perimeter(self)
    }
}
```

This not only makes the composition awkward, but also introduce tight coupling between methods that may be unrelated. For example, even if one wants to define a `Rectangle` context without a scale factor, it would still need to implement the `scale_factor` method by returning `1.0`. On the other hand, with every method split into its own trait, the `Rectangle` context would not need to worry about scale factor at all, if it does not use providers like `ScaledArea`. And while the example scale factor here might be simple, it would be much harder to skip if when complex real world methods are involved.

The design of monolithic traits could become even more complicated when abstract types are involved. For example, instead of having separate type traits, a Rust developer would prefer to introduce abstract types into the same `Shape` trait, together with all possible trait bounds:

```rust
#[cgp_component(ShapeProvider)]
pub trait Shape {
    type Scalar: Copy + Add + Mul;

    type Angle: Copy + Add + Mul;

    fn area(&self) -> Self::Scalar;

    fn scale_factor(&self) -> Self::Scalar;

    fn perimeter(&self) -> Self::Scalar;

    fn scale(&mut self, factor: Self::Scalar);

    fn rotate(&mut self, angle: Self::Angle);
}
```

While it is possible to define CGP traits in this way, the design can make it significantly harder to write reusable provider implementations that can be reused across multiple contexts.

The grouping of abstract types also means that even if a feature is not used by a context, such as `Angle` and `rotate`, the only way the context to ignore the constructs is to assign them with dummy types and `unimplemented!()` stubs. This can in fact be commonly seen in real world code bases, where certain types or methods are skipped when implemented on mock contexts for testing.

At the same time, it is also understandable that monolithic traits like `Shape` can bring significant comfort to many Rust developers from OOP background, while functional traits like `AreaCalculator` brings discomfort and anxiety.

Nevertheless, the biggest challenge of using monolithic traits with CGP is whether there is any advantage of implementing such traits in a context-generic way. When a Rust developer is designing a monolithic trait like `Shape`, they are not really thinking about writing context-generic impl on it. Rather, developers tend to think in terms of concrete mindset, where the trait would be implemented directly by a concrete context like `Rectangle`.

If the developer tries to implement the monolithic trait in a context-generic way, they would typically also need monolithic getter traits such as follows:

```rust
trait RectangleFields {
    type Scalar: Copy + Add + Mul;

    fn width(&self) -> Self::Scalar;

    fn height(&self) -> Self::Scalar;

    fn scale_factor(&self) -> Self::Scalar;

    // ...
}

#[cgp_impl(RectangleProvider)]
impl<Context> ShapeProvider for Context
where
    Context: RectangleFields,
{
    type Scalar = Context::Scalar;

    fn area(&self) -> Self::Scalar {
        self.width() * self.height()
    }

    fn scale_factor(&self) -> Self::Scalar {
        self.scale_factor()
    }
}
```

With such degree of inflexibility, one could reasonably argue that it might have been better off if the trait is better off being implemented on a concrete `Rectangle` type without using CGP at all.

As we can see, the main issue with monolithic traits is that it subtly increases the fusion property of the code, even though the code is written in a context-generic style. This is because the developer is still in a fusion-driven mindset when writing fission-driven code.

#### Incremental Fission

It is very challenging to steer Rust developers away from monolithic trait designs, and if we try to push for functional styles all at once, it would only result in further backslash against adoption. So instead, we need to find a middleground so that developers can still write monolithic traits, at least initially, even in a fission-driven environment.

The first strategy is to acknowledge that as the size of the monolithic trait increases, it becomes less likely to benefit from the fission-driven programming style provided by CGP. If a monolithic trait is so large that it can only be implemented by one concrete context, then there is no benefit that CGP can provide.

The second strategy is that instead of focusing on the design of the monolithic trait, one should decide how to split up the monolithic trait based on the end goal of what kind of contexts does the developer wants to split into. For example, if the code base has an application context with a monolithic trait impl like:

```rust
pub struct ProductionApp {
    database: Database,
    api_client: ProductionApiClient,
    email_sender: ProductionEmailSender,
}

pub trait AppServices {
    fn create_user(&self, email: &EmailAddress) -> Result<User>;

    fn get_user(&self, user_id: &UserId) -> Result<User>;

    fn fetch_file(&self, file_id: &FileId) -> Result<Vec<u8>>;

    fn send_email(&self, recipient: &EmailAddress, message: &str) -> Result<()>;
}

impl AppServices for ProductionApp {
    ...
}
```

and if the goal is to introduce a secondary context like:

```rust
pub struct MockApp {
    database: Database,
    api_client: MockApiClient,
    email_sender: MockEmailSender,
}
```

Since the `database` field remains the same, and the `api_client` and `email_sender` fields are different, the best course of action is to spin off the methods that depend on the database into a separate trait. The trait that contains the database methods can be implemented in a context-generic way, while the trait that contains the other methods can be implemented in the classical way:

```rust
pub trait UserServices {
    fn create_user(&self, email: &EmailAddress) -> Result<User>;

    fn get_user(&self, user_id: &UserId) -> Result<User>;
}

impl<Context> UserServices for Context
where
    Context: HasDatabase,
{
    ...
}

pub trait ExternalServices {
    fn fetch_file(&self, file_id: &FileId) -> Result<Vec<u8>>;

    fn send_email(&self, recipient: &EmailAddress, message: &str) -> Result<()>;
}

impl ExternalServices for ProductionApp {
    ...
}

impl ExternalServices for MockApp {
    ...
}
```

With this approach, fission can be applied incrementally like a precision strike to a code base, depending on the exact contexts that the developer want to split into. For the remaining trait methods that do not require reuse, we should encourage developers to continue using the classic method of implementing the trait directly on the concrete type.

Through this, we can hopefully provide a gradual transition to use CGP, with the goal of teaching developers a fission-driven mindset rather than any specific CGP technique.

That said, this approach of incremental fission is without its downside. Every time the developer wants to apply fission, they must also apply significant refactoring to the code base to update both the use and implementation of the monolithic traits.

Depending on the development culture, even something as simple as splitting a `AppServices` into two smaller `UserServices` and `ExternalServices` traits could be considered too expensive to spare any time in the development budget. Furthermore, if the majority of the code base make use of the common field like database, then this is still essentially asking the developer to transition a large part of their code base to become context-generic in one large go.

Nevertheless, this is a necessary trade off that the developer must accept, if they want to continue using monolithic traits with CGP. In a way, fission-driven development has an inevitable nature of not only pushing the tendency for concrete contexts to split, but also for monolithic traits to break down into smaller traits.

As a result, the minimal a developer needs to accept is that they must first learn about the fission-driven mindset, and be prepared to split not only their contexts, but also their monolithic traits. Without this minimal agreement, there is no room for convincing a developer to adopt CGP.

Ironically, the ultimate conclusion when managing traits in a fission-driven environment is that traits are most stable when they contain only one item. And when they contain more than one, developers must be ready to break them down when a fission-driven action is applied. It is an inevitable and delicate trade off that one can only choose between the stability of trait definitions, or the comfort of using monolithic traits in a fission-driven environment.


## Conclusion

This document has given an anatomical analysis of CGP, identify the different sources of complexity that arise from using CGP, and come up with some recommendations on how to mitigate some of the complexities.

We first identify CGP as fundamentally being a new form of fission-driven development, that solves programming patterns by splitting one context into multiple slightly different contexts. We then contrast this with the classical Rust design patterns, which are fusion-driven development that focus on merging multiple features into the one monolithic context.

We then identify the need for two or more contexts as the primary source of complexity that has to be accepted, before one can move forward to adopt and benefit from CGP.

We analyzed the three main pillar of benefits that CGP can provide, and suggest approaches that can be used to simplify the patterns while still retain the benefits that CGP provide. First, the use of abstract types have to be reduced to minimal, as they introduce the most significant cognitive overhead to developers. Secondly, the use of configurable static dispatch can be replaced with explicit trait impl on concrete type, if a provider implementation is used only by one context. Thirdly, the view of auto getter traits being too complex and magical can be mitigated by having developers to not derive `HasField` and instead implement the getter traits explicitly.

With these recommendations in place, we can hopefully reduce the complexity of CGP to a minimum level to make it easier for developers to onboard to the new paradigm of fission-driven development.
