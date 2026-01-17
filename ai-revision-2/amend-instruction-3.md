## Instruction

Revise the section Secondary Complexity: Configurable Static Dispatch in Chapter 9: Managing CGP Complexity to use the code example below.

Before you start writing the revised section, first write down a detailed outline of the changes or additions that you are going to introduce.

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
