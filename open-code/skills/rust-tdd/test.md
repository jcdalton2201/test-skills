# Good and Bad Tests

## Good Tests

**Integration-style**: Test through real interfaces, not mocks of internal parts.

```rust
// GOOD: Tests observable behavior
#[test]
fn user_can_checkout_with_valid_cart() {
    let mut cart = create_cart();
    cart.add(product);
    let result = checkout(&cart, &payment_method).expect("checkout should succeed");
    assert_eq!(result.status, "confirmed");
}
```

Characteristics:

- Tests behavior users/callers care about
- Uses public API only
- Survives internal refactors
- Describes WHAT, not HOW
- One logical assertion per test

## Bad Tests

**Implementation-detail tests**: Coupled to internal structure.

```rust
// BAD: Tests implementation details
#[test]
fn checkout_calls_payment_service_process() {
    let mut mock_payment = MockPaymentService::new();
    checkout(&cart, &payment).expect("should checkout");
    assert!(mock_payment.process_was_called_with(&cart.total));
}
```

Red flags:

- Mocking internal collaborators
- Testing private methods
- Asserting on call counts/order
- Test breaks when refactoring without behavior change
- Test name describes HOW not WHAT
- Verifying through external means instead of interface

```rust
// BAD: Bypasses interface to verify
#[test]
fn create_user_saves_to_database() {
    create_user(&CreateUserRequest { name: "Alice".to_string() })
        .expect("should create user");
    let row = db.query("SELECT * FROM users WHERE name = ?", vec!["Alice"])
        .expect("should query");
    assert!(row.is_some());
}

// GOOD: Verifies through interface
#[test]
fn create_user_makes_user_retrievable() {
    let user = create_user(&CreateUserRequest { name: "Alice".to_string() })
        .expect("should create user");
    let retrieved = get_user(user.id).expect("should retrieve user");
    assert_eq!(retrieved.name, "Alice");
}
```
