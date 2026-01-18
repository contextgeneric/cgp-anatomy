Revise the section `Special Topic: Functional Programming Patterns` in `Chapter 9: Managing CGP Complexity` with the following changes.

Include the example of how `ScaledArea` would need to be implemented with a monolithic `ShapeProvider` trait:

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

    fn scale(&mut self, factor: f64) {
        InnerProvider::scale(self, factor)
    }

    fn rotate(&mut self, angle: f64) {
        InnerProvider::rotate(self, angle)
    }
}
```

Use the same writing style as the other sections. Use full sentences and avoid point form.

Avoid repeating the same content that have already been mentioned in the other sections. Ensure that the revised section flows well with the other sections.

Before you start writing the revised chapter, first write down a detailed outline of the changes or additions that you are going to introduce.
