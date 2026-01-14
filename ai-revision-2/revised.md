# The Anatomy of Context-Generic Programming: A Deep Dive into Fission-Driven Development

## Executive Summary

This report analyzes Context-Generic Programming (CGP) in Rust through the conceptual framework of **fusion versus fission-driven development**. Traditional Rust patterns—enums, feature flags, trait objects, and monolithic contexts—represent fusion approaches that merge variations into single unified types. CGP introduces fission patterns that split variations across multiple contexts, enabling code reuse through compile-time polymorphism and configurable static dispatch.

The central thesis is straightforward: CGP's perceived complexity stems not from technical implementation details, but from a fundamental philosophical departure that requires developers to manage multiple contexts simultaneously. This represents an inversion of Rust's deeply embedded cultural assumption that programs should maintain a single unified context.

CGP provides three distinct capabilities that justify this paradigm shift. **Dependency injection through getter traits** enables context-generic code to access values without coupling to concrete struct layouts, allowing the same implementation to work with contexts that store, compute, or delegate their values differently. **Type dictionaries through abstract types** solve the parameter pollution problem by encapsulating type information within contexts and referencing it through associated types rather than explicit generic parameters. **Configurable static dispatch through type-level lookup tables** enables multiple overlapping trait implementations to coexist and be selected per-context at compile time, working around Rust's coherence restrictions while maintaining zero-cost abstraction.

The primary source of adoption friction is unavoidable: CGP only provides value when multiple contexts exist, yet Rust developers are trained to maintain single contexts. This creates a chicken-and-egg problem where teams must commit to managing multiple contexts before experiencing CGP's benefits. However, secondary complexity sources—abstract types, configurable dispatch, and getter traits—are optional and mitigatable through selective adoption strategies.

The report demonstrates that fission and fusion patterns can coexist productively through hybrid approaches. Strategic application of classical patterns like enums and generic struct parameters prevents combinatorial explosion of contexts while leveraging CGP's benefits where variation genuinely exists. This **incremental fission** strategy offers a pragmatic adoption path: teams can start with blanket trait implementations using only standard Rust, introducing advanced CGP features selectively as multiple implementations emerge as genuine requirements.

Successful CGP adoption requires accepting the fission-driven mindset—that multiple contexts are a feature rather than a bug—before engaging with specific technical patterns. Teams must decide whether the benefits of multi-context code reuse justify the learning cost and cultural shift required. For scenarios involving genuine variation across deployment environments, testing contexts, or customer configurations, CGP provides capabilities that alternative approaches cannot match while maintaining Rust's performance guarantees.

---

## Table of Contents

1. **Foundations: What CGP Actually Is**
   - Context polymorphism through compile-time generics
   - How CGP differs from dynamic dispatch and other approaches

2. **The Spectrum: From Runtime to Compile-Time Polymorphism**
   - Dynamic dispatch: familiar but limited
   - The progression toward CGP components

3. **Why CGP Matters: Three Concrete Benefits**
   - Structural independence through getter traits
   - Escaping parameter pollution with abstract types
   - Multiple implementations via configurable dispatch

4. **Fusion vs Fission: The Core Framework**
   - One context or many: a fundamental choice
   - How this distinction shapes everything

5. **Rust's Fusion Toolkit**
   - Six patterns for maintaining context unity
   - When each pattern reaches its breaking point

6. **Fission Beyond Rust**
   - Duck typing, inheritance, and reflection
   - CGP's unique position: compile-time fission with safety

7. **Why Rust Resists Fission**
   - Language constraints and cultural values
   - The comfort of concrete reasoning

8. **The Adoption Challenge**
   - The chicken-and-egg problem
   - Overcoming the minimum context requirement

9. **Primary Complexity: Multiple Contexts Are Necessary**
   - Why this complexity cannot be eliminated
   - Accepting the necessary trade-off

10. **Managing Secondary Complexity**
    - Abstract types: when and how to use them
    - Configurable dispatch: selective adoption
    - Getter traits: balancing magic and explicitness

11. **Fusion-Fission Hybrids**
    - When to use classical patterns with CGP
    - Preventing context explosion

12. **Functional Patterns in CGP**
    - Higher-order providers and composition
    - The OOP vs FP tension

13. **Incremental Adoption Strategy**
    - Four steps to introducing CGP
    - Identifying candidates for fission

14. **Conclusions: When CGP Makes Sense**
    - Decision criteria for adoption
    - Future opportunities

---

# Chapter 1: Foundations: What CGP Actually Is

Context-Generic Programming is code that achieves reusability across multiple context types through compile-time generics. This definition emphasizes two interconnected properties: **context polymorphism** (the ability to operate across different context types where the context is the primary subject of operations) and **compile-time mechanism** (using generics and monomorphization rather than runtime dispatch). Understanding this definition requires first recognizing that context polymorphism itself is not novel—dynamic dispatch, dynamic typing, and object-oriented inheritance all provide ways to write code that works across varying types. What distinguishes CGP is achieving this polymorphism entirely at compile time through Rust's generic system.

## Why Generics Matter for Context Polymorphism

The compile-time nature of CGP's polymorphism provides two critical advantages over runtime approaches. First, monomorphization eliminates dispatch overhead by generating specialized implementations for each concrete type at compile time. When context-generic code calls a trait method, the Rust compiler knows exactly which implementation will be invoked and can inline function bodies, propagate constants, and eliminate dead code—optimizations impossible with vtable indirection. Second, compile-time type resolution enables stronger correctness guarantees through trait bound verification, surfacing errors during development rather than as runtime exceptions.

Consider a simple example calculating rectangle area:

````rust
pub trait RectangleFields {
    fn width(&self) -> f64;
    fn height(&self) -> f64;
}

pub fn rectangle_area<Context>(context: &Context) -> f64
where
    Context: RectangleFields,
{
    context.width() * context.height()
}
````

This generic function works with any type implementing `RectangleFields`, generating specialized implementations through monomorphization. Compare this to dynamic dispatch using `&dyn RectangleFields`, which would require vtable lookups preventing compiler optimization, or to dynamic typing where method presence is verified only at runtime.

The use of generics also leverages Rust's trait system for expressing requirements. Traits specify not just method signatures but associated types, constants, and type relationships. Trait bounds compose naturally, building complex requirements from simpler components, with the compiler verifying satisfaction at zero runtime cost.

## How CGP Differs from Generic Programming

Generic programming has existed for decades across statically typed languages—C++ templates, Java generics, ML parametric polymorphism. CGP's distinctiveness lies in making the **context itself** the primary type parameter rather than auxiliary types. In many generic scenarios, generics parameterize over input or output types while the receiver type remains concrete. A generic method on a specific struct type keeps the struct concrete while the method parameters are generic. CGP inverts this by making the context the primary type parameter, allowing the same code to work across fundamentally different context types.

This distinction becomes clearer when we examine CGP's position between trait objects and plain generic functions. Trait objects (`dyn Trait`) provide context polymorphism through type erasure and runtime dispatch—different concrete types can be used interchangeably where `dyn Trait` is expected, but at the cost of vtable indirection and optimization barriers. CGP provides the same flexibility to work with different types through a common interface, but achieves this entirely at compile time. The generic code is parameterized by type variables with trait bounds, and the compiler generates specialized implementations for each concrete instantiation. By runtime, no polymorphism remains—only concrete, individually optimized implementations exist.

This compile-time approach enables capabilities impossible with runtime dispatch. Associated types let context-generic code reference types determined by concrete implementations without explicit generic parameters. Const generics enable polymorphism over compile-time constants. Higher-ranked trait bounds express multi-level polymorphism requirements. These emerge naturally from compile-time resolution and represent expressiveness beyond runtime dispatch mechanisms.

Moreover, CGP's relationship with Rust's ownership system differs fundamentally from trait objects. Because the compiler knows concrete types during monomorphization, it can precisely track ownership, borrowing, and lifetimes through entire call chains of generic code. Context-generic code takes full advantage of Rust's memory safety guarantees without runtime cost, whereas trait objects impose limitations on expressible lifetime relationships due to type erasure.

Context-Generic Programming thus occupies a specific design space: achieving context polymorphism through compile-time generics and monomorphization rather than runtime dispatch or type erasure. Having established this foundation, we now examine the spectrum of context-polymorphic patterns in Rust, from familiar runtime techniques to advanced compile-time approaches, revealing where CGP fits within this landscape and why its position generates both enthusiasm and resistance.

---

# Chapter 2: The Spectrum: From Runtime to Compile-Time Polymorphism

Context-polymorphic patterns in Rust exist along a spectrum that moves from familiar runtime techniques toward increasingly sophisticated compile-time approaches. This spectrum represents a gradual transition where each step trades some accessibility for additional power, with CGP occupying the compile-time end. Understanding this progression reveals where CGP fits within Rust's broader landscape and why developers perceive distinct complexity thresholds as they move from one pattern to the next.

## Dynamic Dispatch: The Familiar Starting Point

Trait objects provide the most accessible entry to context polymorphism for developers arriving from languages with runtime polymorphism. When code accepts `&dyn Trait`, it works with any concrete type implementing that trait, determined at runtime through vtable indirection:

````rust
pub trait RectangleFields {
    fn width(&self) -> f64;
    fn height(&self) -> f64;
}

pub fn rectangle_area(context: &dyn RectangleFields) -> f64 {
    context.width() * context.height()
}
````

This pattern mirrors interfaces in Java, protocols in Python, or virtual methods in C++, making it immediately comprehensible without understanding generics or monomorphization. The function signature clearly indicates acceptance of any `RectangleFields` implementer, and the body accesses methods exactly as on any value.

**Limitations driving toward compile-time alternatives:**

The `&dyn RectangleFields` parameter contains two pointers—one to data, one to a vtable—requiring runtime indirection for each method call. This prevents compiler optimizations that require concrete type knowledge: inlining, constant propagation, and cross-function analysis all become impossible at trait object boundaries. For performance-critical paths, this overhead accumulates measurably.

Type erasure also imposes object-safety restrictions: traits cannot have generic methods, methods returning `Self` by value, or associated types beyond those with concrete bounds. Many useful traits including `Clone` cannot be used as trait objects, limiting this pattern's applicability precisely where advanced type system features would provide value.

## Impl Trait: The Syntactic Bridge

The `impl Trait` syntax provides nearly identical appearance while fundamentally changing semantics from runtime dispatch to compile-time monomorphization:

````rust
pub fn rectangle_area_impl(context: &impl RectangleFields) -> f64 {
    context.width() * context.height()
}
````

To readers, this looks almost identical to the trait object version—the function still clearly accepts any `RectangleFields` implementer, and the body remains unchanged. Yet the compiler treats this as syntactic sugar for a generic function, generating specialized implementations for each concrete type at call sites.

This transformation happens invisibly, requiring no caller changes and no explicit generic understanding. Developers gain compile-time dispatch performance benefits without engaging with generic type systems. The genius lies in this scaffolded introduction: when errors about unsatisfied trait bounds occur, they connect directly to existing mental models of "this function works with anything implementing this interface."

**Why move beyond:** While `impl Trait` successfully hides generic machinery, it cannot express relationships between multiple parameters, return types depending on generic types, or complex trait bounds involving associated types. The anonymous nature that makes it accessible becomes a limitation when these capabilities become necessary.

## Generic Functions: Making Abstraction Visible

Explicit generic functions expose the machinery hidden by `impl Trait`, requiring direct engagement with type parameters and trait bounds:

````rust
pub fn rectangle_area_generic<Context>(context: &Context) -> f64
where
    Context: RectangleFields,
{
    context.width() * context.height()
}
````

The `<Context>` parameter in angle brackets and separated `where` clause make the generic nature immediately visible. This represents a psychological hurdle despite semantic equivalence to `impl Trait`—the code no longer reads like simple function signatures but requires recognizing `Context` as a type variable instantiated differently at each call site.

**Benefits justifying explicit syntax:**

Named type parameters enable referring to the same generic type multiple times within signatures—impossible with anonymous `impl Trait`. The `where` clause provides space for expressing complex trait bounds involving multiple traits, associated types, or higher-ranked trait bounds that parameter lists cannot accommodate.

The explicit signal also aids comprehension for experienced developers. Seeing `<Context>` immediately triggers mental models about generic code behavior, whereas `impl Trait` can obscure whether a function is truly generic or just using trait objects.

## Blanket Implementations: Entering CGP Territory

Blanket trait implementations reorganize generic machinery from function-level to trait-level abstraction, fundamentally reshaping capability composition:

````rust
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
````

This pattern extends types with new capabilities without modifying their definitions. Any type implementing `RectangleFields` automatically gains `RectangleArea` through the blanket implementation. This automatic acquisition differs from generic functions—where callers must explicitly choose to use the function, blanket implementations confer capabilities that become part of types themselves.

**Distinctive properties:**

The dependency injection property represents conceptual advancement: `RectangleArea`'s interface mentions nothing about `RectangleFields`—dependencies hide in the `impl` block's `where` clause. Trait interfaces stay focused on provided capabilities while implementation details remain hidden.

This visual pattern—`impl<Context> Trait for Context`—marks where developers first perceive code as "CGP-like," even without CGP-specific constructs. The appearance signals context-polymorphic architectural organization.

**When this becomes necessary:**

Blanket implementations become essential when capabilities should compose automatically rather than requiring explicit function calls. Standard library patterns like `Iterator` methods and `StreamExt` from `futures` demonstrate this value. They represent the furthest context-polymorphic pattern achievable using only standard Rust, making them natural stopping points for incremental adoption before committing to CGP frameworks.

## CGP Components: Configurable Dispatch

CGP components add one capability impossible with standard Rust: multiple overlapping implementations selected per-context at compile time. This works around coherence restrictions while maintaining zero-cost abstraction:

````rust
use cgp::prelude::*;

#[cgp_component(AreaCalculator)]
pub trait HasArea {
    fn area(&self) -> f64;
}

#[cgp_impl(new RectangleArea)]
impl<Context> AreaCalculator for Context
where
    Context: RectangleFields,
{
    fn area(&self) -> f64 {
        self.width() * self.height()
    }
}

#[cgp_impl(new CircleArea)]
impl<Context> AreaCalculator for Context
where
    Context: CircleFields,
{
    fn area(&self) -> f64 {
        use std::f64::consts::PI;
        let r = self.radius();
        PI * r * r
    }
}
````

Both `RectangleArea` and `CircleArea` implement `AreaCalculator`, which coherence rules would ordinarily prohibit. CGP works around this by making each implementation target a unique provider type rather than directly implementing for contexts. Contexts then explicitly select providers through wiring:

````rust
use cgp::prelude::*;

#[derive(HasField)]
pub struct Rectangle {
    pub width: f64,
    pub height: f64,
}

delegate_components! {
    Rectangle {
        AreaCalculatorComponent: RectangleArea,
    }
}

#[derive(HasField)]
pub struct Circle {
    pub radius: f64,
}

delegate_components! {
    Circle {
        AreaCalculatorComponent: CircleArea,
    }
}
````

This wiring establishes type-level lookup tables mapping component types to provider types. When `area()` is called on `Rectangle`, framework-generated blanket implementations consult the lookup table, find delegation to `RectangleArea`, and dispatch accordingly. This entire lookup happens at compile time through type system machinery—generated code contains no runtime indirection.

**When simpler patterns become insufficient:**

This complexity only provides value when multiple implementations must coexist. If one implementation suffices for all contexts, the component "downgrades" to a simple blanket implementation with minimal code changes. The configurable dispatch machinery becomes necessary precisely when different contexts genuinely need different implementations of the same capability—when test contexts need mock implementations, when different deployments require different backends, or when performance-critical paths need specialized implementations.

The progression from trait objects through CGP components represents increasing compile-time sophistication while decreasing runtime overhead. Each step trades accessibility for capability, with CGP occupying the position where maximum flexibility meets zero-cost abstraction at the expense of requiring explicit type-level wiring. Understanding this spectrum reveals CGP not as foreign complexity but as the logical endpoint of Rust's compile-time polymorphism philosophy.

---

# Chapter 3: Why CGP Matters: Three Concrete Benefits

Having explored the spectrum of context-polymorphic patterns and established how CGP achieves compile-time polymorphism through generics, we now examine the concrete value proposition that justifies CGP's adoption despite its learning curve. This chapter analyzes three distinct capabilities that CGP provides, each addressing a specific limitation of standard Rust patterns: structural independence through getter traits, type organization through abstract types, and implementation selection through configurable dispatch.

## Structural Independence Through Getter Traits

**The Problem:** Direct field access couples code to specific struct layouts. When implementation logic accesses `self.width` and `self.height` directly, it can only work with contexts that store width and height as fields with exactly those names. This prevents reusing the same logic with contexts that compute dimensions, delegate to nested structures, or use different field names.

**The Solution:** Getter traits abstract field access through methods, enabling context-generic code to request values without knowing how contexts provide them. Using the `#[cgp_auto_getter]` macro, we define requirements through method signatures:

````rust
use cgp::prelude::*;

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
````

Now the same `RectangleArea` implementation works with contexts that store dimensions directly, compute them dynamically, or delegate to nested structures:

````rust
use cgp::prelude::*;

// Direct storage
#[derive(HasField)]
pub struct Rectangle {
    pub width: f64,
    pub height: f64,
}

// Computed dimensions
pub struct Square {
    pub side: f64,
}

impl RectangleFields for Square {
    fn width(&self) -> f64 {
        self.side
    }

    fn height(&self) -> f64 {
        self.side
    }
}
````

**Benefits:** The `RectangleArea` implementation never changes despite these structural differences. Code reuses naturally across contexts with incompatible layouts, without requiring refactoring or wrapper types. Testing becomes simpler as mock contexts can provide values through computation rather than maintaining complex data structures.

## Escaping Parameter Pollution With Abstract Types

**The Problem:** Generic parameters proliferate as systems grow. Consider generalizing our rectangle to work with any numeric type:

````rust
use num_traits::Num;

pub trait RectangleFields<Scalar: Num + Copy> {
    fn width(&self) -> Scalar;
    fn height(&self) -> Scalar;
}

pub trait RectangleArea<Scalar: Num + Copy, Error> {
    fn rectangle_area(&self) -> Result<Scalar, Error>;
}
````

Adding error handling and coordinates creates cascading pollution:

````rust
use num_traits::Num;

pub trait PositionedRectangle<Scalar: Num + Copy, Coordinate: Num + Copy> {
    fn x(&self) -> Coordinate;
    fn y(&self) -> Coordinate;
}

impl<Context, Scalar, Error, Coordinate> RectangleArea<Scalar, Error> for Context
where
    Scalar: Num + Copy,
    Coordinate: Num + Copy,
    Context: RectangleFields<Scalar> + PositionedRectangle<Scalar, Coordinate>,
{
    fn rectangle_area(&self) -> Result<Scalar, Error> {
        // Coordinate appears in bounds but isn't used here
        Ok(self.width() * self.height())
    }
}
````

**The Solution:** Abstract types move generic parameters into the context, referencing them through associated types:

````rust
use cgp::prelude::*;
use num_traits::Num;

#[cgp_type]
pub trait HasScalarType {
    type Scalar: Num + Copy;
}

#[cgp_type]
pub trait HasErrorType {
    type Error;
}

#[cgp_type]
pub trait HasCoordinateType {
    type Coordinate: Num + Copy;
}

#[cgp_auto_getter]
pub trait RectangleFields: HasScalarType {
    fn width(&self) -> Self::Scalar;
    fn height(&self) -> Self::Scalar;
}

pub trait RectangleArea: HasScalarType + HasErrorType {
    fn rectangle_area(&self) -> Result<Self::Scalar, Self::Error>;
}

impl<Context> RectangleArea for Context
where
    Context: RectangleFields,
{
    fn rectangle_area(&self) -> Result<Self::Scalar, Self::Error> {
        Ok(self.width() * self.height())
    }
}
````

**Benefits:** Implementation blocks no longer enumerate type parameters. Traits declare only the abstract types they directly use through supertrait bounds. Adding new types to contexts doesn't require updating every trait throughout the codebase. The `RectangleArea` implementation above references `Self::Scalar` and `Self::Error` without receiving them as generic parameters.

## Multiple Implementations Via Configurable Dispatch

**The Problem:** Rust's coherence rules prevent multiple overlapping trait implementations. We cannot define both rectangle and circle area calculations for contexts that might satisfy both requirements:

````rust
pub trait HasArea {
    fn area(&self) -> f64;
}

// These implementations would conflict:
// impl<Context: RectangleFields> HasArea for Context { ... }
// impl<Context: CircleFields> HasArea for Context { ... }
````

Without configurable dispatch, we must either fragment the interface into separate traits or use enum-based dispatching that prevents downstream extensibility.

**The Solution:** CGP components enable multiple implementations through provider indirection and explicit wiring:

````rust
use cgp::prelude::*;

#[cgp_component(AreaCalculator)]
pub trait HasArea {
    fn area(&self) -> f64;
}

#[cgp_impl(new RectangleArea)]
impl<Context> AreaCalculator for Context
where
    Context: RectangleFields,
{
    fn area(&self) -> f64 {
        self.width() * self.height()
    }
}

#[cgp_impl(new CircleArea)]
impl<Context> AreaCalculator for Context
where
    Context: CircleFields,
{
    fn area(&self) -> f64 {
        use std::f64::consts::PI;
        let r = self.radius();
        PI * r * r
    }
}
````

Contexts explicitly select their provider:

````rust
use cgp::prelude::*;

#[derive(HasField)]
pub struct Rectangle {
    pub width: f64,
    pub height: f64,
}

delegate_components! {
    Rectangle {
        AreaCalculatorComponent: RectangleArea,
    }
}

#[derive(HasField)]
pub struct Circle {
    pub radius: f64,
}

delegate_components! {
    Circle {
        AreaCalculatorComponent: CircleArea,
    }
}
````

**Benefits:** Both implementations coexist without conflict. Downstream code can add new implementations without modifying upstream definitions. Different contexts select appropriate providers through wiring. The entire dispatch mechanism compiles away—generated code contains no runtime indirection, matching hand-written implementations in performance.

This capability becomes essential when variation genuinely exists: test contexts need mock implementations, different deployments require different backends, performance-critical paths need specialized implementations. For components with only one implementation across all contexts, the pattern naturally degrades to simple blanket traits, making complexity opt-in rather than mandatory.

---

These three pillars—structural independence, type organization, and implementation selection—address limitations that emerge at scale when managing multiple contexts. Getter traits prevent tight coupling to struct layouts. Abstract types prevent generic parameter proliferation. Configurable dispatch enables extensibility while maintaining compile-time resolution. Together, they justify CGP's learning cost for systems where multi-context code reuse provides genuine value.

---

# Chapter 4: Fusion vs Fission: The Core Framework

The conceptual framework of fusion versus fission provides the lens for understanding CGP's fundamental nature and why it feels radically different from conventional Rust programming. This chapter establishes the core distinction between these approaches and demonstrates how this single axis explains most of the adoption friction developers experience.

## One Context or Many: A Fundamental Choice

Software development involves a foundational architectural decision that developers rarely recognize as a choice: when variation exists in a system, should it be accommodated within a single unified type, or split across multiple distinct types? This question represents the fusion-fission axis.

**Fusion-driven development** solves problems by merging variations into single unified contexts. When encountering requirements for different configurations, environments, or implementations, fusion developers ask: "How can I incorporate this variation into my existing type without splitting it?" This manifests as enums encompassing variants, feature flags controlling compilation, trait objects enabling runtime selection, or fields added to structs to support new capabilities.

**Fission-driven development** solves problems by splitting variations across multiple contexts. When encountering the same requirements, fission developers ask: "What new context type represents this variation, and how can I ensure my code works with all relevant contexts?" This manifests as distinct types for different scenarios, with shared behavior implemented through generic code that operates across contexts.

### Quick Reference: Fusion vs Fission

| Aspect | Fusion | Fission |
|--------|--------|---------|
| **Context Count** | One unified | Multiple specialized |
| **Variation Handling** | Enums, flags, runtime dispatch | Separate types, compile-time polymorphism |
| **Code Location** | Methods on concrete types | Generic implementations with trait bounds |
| **Extensibility** | Modify existing type | Add new type, wire to providers |
| **Mental Model** | Single concrete reality | Family of compatible abstractions |
| **Evolution** | Accumulate fields/variants | Add contexts, reuse generic code |
| **Testing** | Mock through configuration | Separate test context types |
| **Coupling** | High - direct field access | Low - dependency injection via traits |

## Fusion in Practice

Consider an application needing both production and test configurations. The fusion approach maintains a single `Application` type:

````rust
pub enum DatabaseBackend {
    Production(PostgresDatabase),
    Test(MockDatabase),
}

pub struct Application {
    pub database: DatabaseBackend,
    pub config: Config,
}

impl Application {
    pub fn query_user(&self, user_id: &UserId) -> Result<User> {
        match &self.database {
            DatabaseBackend::Production(db) => {
                let row = db.query("SELECT * FROM users WHERE id = $1", &[user_id])?;
                Ok(User::from_row(row))
            }
            DatabaseBackend::Test(mock) => {
                Ok(mock.get_test_user(user_id))
            }
        }
    }
}
````

Variation exists through the enum—there is still exactly one `Application` type. Every function accepting `Application` works with both production and test configurations. Adding fields, changing implementations, or accessing data requires modifying the single unified definition. This creates a monolithic context that grows to accommodate all variations.

**Benefits:** Complete explicitness about what configurations exist. Pattern matching ensures exhaustive handling. No generic complexity—just concrete types and control flow. Compiler can optimize entire program knowing concrete structure.

**Costs:** Adding new database backends requires modifying the enum and updating every match site. Third-party extensions cannot introduce new variants. Different backends must share memory layout. Testing requires constructing production-like structures even for mock scenarios.

## Fission in Practice

The fission approach creates distinct context types:

````rust
use cgp::prelude::*;

#[cgp_auto_getter]
pub trait HasDatabase {
    type Database: DatabaseOps;
    fn database(&self) -> &Self::Database;
}

pub trait CanQueryUser: HasDatabase {
    fn query_user(&self, user_id: &UserId) -> Result<User>;
}

impl<Context> CanQueryUser for Context
where
    Context: HasDatabase,
{
    fn query_user(&self, user_id: &UserId) -> Result<User> {
        let row = self.database().query("SELECT * FROM users WHERE id = $1", &[user_id])?;
        Ok(User::from_row(row))
    }
}

// Separate production context
#[derive(HasField)]
pub struct ProductionApp {
    pub database: PostgresDatabase,
    pub config: Config,
}

// Separate test context
#[derive(HasField)]
pub struct TestApp {
    pub database: MockDatabase,
}
````

The `query_user` implementation works with **any** context providing database access through `HasDatabase`. `ProductionApp` and `TestApp` are distinct types with different fields and layouts. Generic code reuses automatically—no duplication, no manual forwarding.

**Benefits:** New contexts introduced without modifying existing code. Test contexts need only relevant fields. Different contexts can have incompatible layouts. Third-party code can define new contexts. Implementation written once, reused everywhere.

**Costs:** Must think abstractly about "any context with database access" rather than concretely about `Application`. Requires understanding trait bounds and generic implementations. Multiple context types proliferate—need discipline to manage them. Unfamiliar to developers trained in fusion patterns.

## How This Distinction Shapes Everything

The fusion-fission choice cascades through every architectural decision:

**Organization:** Fusion codebases organize around concrete types—`Application` is the organizing principle, with methods implemented directly on it. Fission codebases organize around capabilities—`CanQueryUser` is the organizing principle, with contexts as implementation details.

**Testing:** Fusion testing configures the single `Application` type to use mocks, often requiring complex setup to satisfy all fields even when irrelevant to the test. Fission testing creates minimal test contexts containing only dependencies relevant to the tested capability.

**Evolution:** Fusion evolves by extending existing types—adding enum variants, fields, or methods to `Application`. Fission evolves by adding new contexts that satisfy existing trait bounds, leaving both implementations and existing contexts unchanged.

**Team Boundaries:** Fusion creates contention around shared type definitions—all teams modify the same `Application` struct. Fission enables parallel development—teams define their own contexts that satisfy shared trait interfaces.

**Third-Party Extensions:** Fusion requires either wrapper types or forking to add capabilities—cannot modify library-defined enums or structs. Fission enables direct extension—define new contexts implementing required traits.

The philosophical divergence runs deeper than technical patterns. Fusion developers believe programs represent single concrete systems that should be reflected directly in types. The types describe what **exists**. Fission developers believe programs represent families of related systems sharing structure while differing in details. The types specify what is **required**.

This explains why CGP adoption feels so disruptive—it requires inverting the fundamental assumption that Rust codebases should maintain single unified contexts. Teams must accept that managing multiple contexts is not a failure of design but the necessary condition for achieving multi-context code reuse through compile-time polymorphism.

Neither approach is universally superior. Fusion excels when variation is limited and concrete reasoning's benefits outweigh accommodation costs. Fission excels when variation is extensive and code reuse through abstraction outweighs abstract reasoning costs. The tension between them represents a fundamental trade-off, not a problem with a single solution.

For CGP adoption, the critical insight is recognizing this as a paradigm choice rather than a technical complexity issue. Teams must decide whether their system's variation pattern justifies the fission-driven mindset shift. If the answer is yes, the technical patterns follow naturally. If no, forcing CGP onto inherently fusion-appropriate problems creates unnecessary complexity.

---
