Revise Chapter 6: Fission-Driven Patterns Across Languages with the following changes:

In the section Inheritance and Subtyping in Object-Oriented Programming, change the example Java code to use a `Rectangle` base class and a `RectangleIn2dSpace` that inherits from `Rectangle`.

After that section, introduce a new section that talks about interfaces and mixins with the following points:

- Interfaces in languages like Java and Go shares a lot of similarity and weakness as `dyn` traits in Rust.
- Provide an example `rectangle_area` function that works with any object that implement a `RectangleFields` interface that provide `width` and `height` getter methods.
- Similar to `dyn` traits, interfaces typically need to be implemented on concrete types. This means that object-oriented classes cannot easily reuse interface implementation aside from through inheritance.
- Dynamic-typed languages can employ mixins to reuse interface implementations. This leverages duck typing.
- On the other hand, it is much harder to provide mixin-like feature in static typed languages. (If there is any such example, mention them)

In the section Runtime Reflection and Type Erasure, update the example Bevy code to implement a `rectangle_area` function that works with any entity that contains a `width` and a `height` field.

Before you start writing the revised chapter, first write down a detailed outline of the changes or additions that you are going to introduce.
