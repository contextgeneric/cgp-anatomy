Revise Chapter 9: Managing CGP Complexity with the following changes:

The term "type family" should be replaced with "type collection" when there are multiple abstract types in one trait. The term "type family" has different meaning when used in Haskell.

In the section Balancing Trait Granularity, change the example code to grouping user-related CRUD operations. For example, separate traits like `CanGetUser`, `CanUpdateUser`, `CanCreateUser` vs a single trait `HasUserServices`.

When implementing database-dependent methods like `query_user`, the implementation should run SQL query on the database instead of calling application-specific methods like `database.query_user()`, which should not present on the database.

Include any missing content from the original draft in draft.md, unless they are already mentioned in other chapters. In particular, expand the section Functional Context-Generic Programming to include the content in the original draft.

Before you start writing the revised chapter, first write down a detailed outline of the changes or additions that you are going to introduce.
