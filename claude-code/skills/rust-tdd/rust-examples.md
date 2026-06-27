# Rust TDD Examples - Caveman Mode

## Example 1: Simple Validation

**Requirement**: Create a function that validates email addresses.

### RED - Write test first
```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn valid_email_passes() {
        assert!(is_valid_email("user@example.com"));
    }

    #[test]
    fn invalid_email_fails() {
        assert!(!is_valid_email("not-an-email"));
    }
}
```

### GREEN - Minimal code
```rust
fn is_valid_email(email: &str) -> bool {
    email.contains('@')
}
```

### Observation
Tests pass. Good enough for tracer bullet. Real validation would come in next cycle if needed.

---

## Example 2: Returning Results

**Requirement**: Parse a number from string, handle errors gracefully.

### RED
```rust
#[test]
fn valid_number_parses() {
    let result = parse_number("42");
    assert_eq!(result, Ok(42));
}

#[test]
fn invalid_number_fails() {
    let result = parse_number("not-a-number");
    assert!(result.is_err());
}
```

### GREEN
```rust
fn parse_number(s: &str) -> Result<i32, String> {
    s.parse::<i32>().map_err(|e| e.to_string())
}
```

### Key Pattern
- Always return `Result<T, E>` for fallible operations
- Tests use `.is_err()`, `.is_ok()`, `.expect()`, or pattern match
- Don't use `.unwrap()` in production, but fine in tests for tracer bullets

---

## Example 3: Dependency Injection (Testability)

**Requirement**: Process an order with payment. Must be testable without hitting real payment API.

### RED
```rust
#[test]
fn order_processes_with_payment() {
    let mut mock_payment = MockPaymentGateway::new();
    mock_payment.set_charge_result(Ok(100));
    
    let order = Order::new(100);
    let result = process_order(&order, &mock_payment);
    
    assert!(result.is_ok());
}

trait PaymentGateway {
    fn charge(&self, amount: u64) -> Result<u64, String>;
}

struct MockPaymentGateway {
    charge_result: Result<u64, String>,
}

impl MockPaymentGateway {
    fn new() -> Self {
        Self { charge_result: Ok(0) }
    }
    fn set_charge_result(&mut self, result: Result<u64, String>) {
        self.charge_result = result;
    }
}

impl PaymentGateway for MockPaymentGateway {
    fn charge(&self, amount: u64) -> Result<u64, String> {
        self.charge_result.clone()
    }
}
```

### GREEN
```rust
fn process_order(order: &Order, gateway: &dyn PaymentGateway) -> Result<(), String> {
    let total = order.amount();
    gateway.charge(total)?;
    Ok(())
}
```

### Key Pattern
- Accept `&dyn Trait` for mockability, not concrete types
- Mock implements same trait
- Tests never create real dependencies
- Use `?` operator for error propagation in tests

---

## Example 4: Mutable State (Borrow Checker Aware)

**Requirement**: Add items to a shopping cart and calculate total.

### RED
```rust
#[test]
fn add_item_increases_total() {
    let mut cart = Cart::new();
    cart.add_item(Item::new("Widget", 10));
    
    assert_eq!(cart.total(), 10);
}

#[test]
fn add_multiple_items() {
    let mut cart = Cart::new();
    cart.add_item(Item::new("Widget", 10));
    cart.add_item(Item::new("Gadget", 20));
    
    assert_eq!(cart.total(), 30);
}
```

### GREEN
```rust
#[derive(Clone)]
struct Item {
    name: String,
    price: u64,
}

impl Item {
    fn new(name: &str, price: u64) -> Self {
        Self { name: name.to_string(), price }
    }
}

struct Cart {
    items: Vec<Item>,
}

impl Cart {
    fn new() -> Self {
        Self { items: Vec::new() }
    }
    
    fn add_item(&mut self, item: Item) {
        self.items.push(item);
    }
    
    fn total(&self) -> u64 {
        self.items.iter().map(|i| i.price).sum()
    }
}
```

### Key Pattern
- `&mut self` for methods that change state
- `&self` for read-only operations
- Tests use `let mut` for mutable bindings
- Borrow checker prevents many bugs automatically

---

## Example 5: Error Handling in Tests

**Requirement**: Validate configuration, fail fast if invalid.

### RED
```rust
#[test]
#[should_panic(expected = "port must be positive")]
fn negative_port_panics() {
    Config::new(-1, "localhost");
}

#[test]
fn valid_config_works() {
    let config = Config::new(8080, "localhost");
    assert_eq!(config.port(), 8080);
}
```

### GREEN
```rust
struct Config {
    port: u16,
    host: String,
}

impl Config {
    fn new(port: i32, host: &str) -> Self {
        assert!(port > 0, "port must be positive");
        Self {
            port: port as u16,
            host: host.to_string(),
        }
    }
    
    fn port(&self) -> u16 {
        self.port
    }
}
```

### Alternative: Return Result Instead
```rust
#[test]
fn invalid_port_returns_error() {
    let result = Config::try_new(-1, "localhost");
    assert!(result.is_err());
}

// Implementation
impl Config {
    fn try_new(port: i32, host: &str) -> Result<Self, String> {
        if port <= 0 {
            return Err("port must be positive".to_string());
        }
        Ok(Self {
            port: port as u16,
            host: host.to_string(),
        })
    }
}
```

### Key Pattern
- Use `#[should_panic]` for constructor validation
- Prefer `Result` types for recoverable errors
- Match on `Result` in tests: `result.map_err(|e| println!("{}", e))`

---

## Rust-Specific TDD Tips

### Borrow Checker & Tests
- Tests force you to think about ownership early (good!)
- If test needs `&mut`, implementation likely does too
- Pattern: immutable input, return new value (functional style) is easier to test

### Error Handling
- Always use `Result<T, E>` for fallible operations
- In tests: `.expect("msg")` for failures, `.unwrap()` acceptable for tracer bullets
- Custom error types can wait until refactor phase

### Mocking Strategy
1. Define trait for external dependency
2. Create mock struct that implements trait
3. Pass mock to function via `&dyn Trait`
4. NO monkey-patching like JavaScript - design must allow injection

### Common Test Patterns
```rust
// Assert equality
assert_eq!(actual, expected);

// Assert boolean
assert!(condition);

// Assert not
assert!(!condition);

// Check error
assert!(result.is_err());
assert_eq!(result.unwrap_err(), expected_error);

// Check success
assert!(result.is_ok());
assert_eq!(result.unwrap(), expected_value);
```

### Test Module Structure
```rust
#[cfg(test)]
mod tests {
    use super::*;  // Import from parent module
    
    #[test]
    fn test_name() {
        // test code
    }
}
```

Always put tests in same file as implementation, in a `#[cfg(test)]` module.
