Revise Chapter 9: Managing CGP Complexity of the report in report.md, following review suggestions in review.md to improve the given chapter.

Your main target audience are Rust developers who just learned about CGP and want to consider strategies to adopt CGP.

You writings should be highly engaging, detailed, and comprehensive. Your explanation should be in full sentences and avoid point forms.

You should still include all content that have been written in the original report, just rearrange them differently or remove duplication according to the review. Include all code examples in the original report, unless they are duplication that are mentioned earlier already.

The revised table of content is in summary.md, and you should continue starting with chapter 9. Additionally, also include the following changes:

The term "type family" should be replaced with "type collection" when there are multiple abstract types in one trait. The term "type family" has different meaning when used in Haskell.

For the section on abstract types, use examples like `UserId` and `User` with traits like `CanGetUser`.

When talking about trait granularity, change the example code to grouping user-related CRUD operations. For example, separate traits like `CanGetUser`, `CanUpdateUser`, `CanCreateUser` vs a single trait `HasUserServices`.

When applicable, instead of using the example in the original report, write your example code based on the example below.

For the section on functional programming, include content in the section below.

Before writing the chapter and each major section, write down detailed action items that list all the changes that you are going to make for that chapter based on the suggestions in review.md. If the review suggestion tells you to include content moved from other chapters, be sure to include that in your action list.

After that, also write down a detailed outline of each sub-section in the chapter, before you start writing the section.

After writing each major section, stop and wait for further instruction.

## Example

There is a possible workaround to tame the complexity of configurable static dispatch, by implementing the component as regular Rust trait. For example, given:

```rust
pub struct ProductionApp {
    pub database: Database,
    pub api_client: AwsApiClient,
}

pub struct TestApp {
    pub database: Database,
    pub api_client: MockApiClient,
}
```

Suppose we want to implement a generic `get_user` function that can work with both `ProductionApp` and `TestApp` using `Database`, but with a `send_email` function that works differently for both contexts, we can write something like follows:

```rust
pub trait CanGetUser {
    fn get_user(&self, user_id: &UserId) -> Result<User>;
}

impl<Context> CanGetUser for Context
where
    Context: HasDatabase,
{
    fn query_user(&self, user_id: &UserId) -> Result<User> {
        // perform SQL query
        ...
    }
}

#[cgp_component(EmailSender)]
pub trait CanSendEmail {
    fn send_email(&self, recipient: &EmailAddress, message: &str) -> Result<()>;
}
```

Essentially, we define a `EmailSender` CGP component with configurable static dispatch, so that it can be implemented differently for `ProductionApp` and `TestApp`. On the other hand, the implementation of `CanGetUser` is applicable for both contexts, which shares the same `Database` type. So it can be defined as a blanket trait without using CGP component.

The typical CGP way to implement `EmailSender` differently for each context is to use configurable static dispatch, such as:

```rust
#[cgp_impl(SendEmailWithAws)]
impl<Context> EmailSender for Context
where
    Context: HasAwsApiClient,
{
    fn send_email(&self, recipient: &EmailAddress, message: &str) -> Result<()> {
        // send email with AWS client
    }
}

#[cgp_impl(MockSendEmail)]
impl<Context> EmailSender for Context
where
    Context: HasMockApiClient,
{
    fn send_email(&self, recipient: &EmailAddress, message: &str) -> Result<()> {
        // record sent email for inspection later
    }
}

delegate_components! {
    ProductionApp {
        EmailSenderComponent:
            SendEmailWithAws,
    }
}

delegate_components! {
    TestApp {
        EmailSenderComponent:
            MockSendEmail,
    }
}
```

But if `SendEmailWithAws` and `MockSendEmail` are only used once, they can technically be instead implemented on `ProductionApp` and `TestApp` directly:

```rust
impl CanSendEmail for ProductionApp {
    fn send_email(&self, recipient: &EmailAddress, message: &str) -> Result<()> {
        ...
    }
}

impl CanSendEmail for TestApp {
    fn send_email(&self, recipient: &EmailAddress, message: &str) -> Result<()> {
        ...
    }
}
```

This way, we would be able to not use any configurable static dispatch, but still keep the code for `CanQueryUser` context-generic and be able to maintain two contexts.


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
