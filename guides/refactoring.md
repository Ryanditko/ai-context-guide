# Refactoring

Guide for safely improving code structure without changing behavior.

## Fundamental Principles

### Definition

> Refactoring is a disciplined technique for restructuring code, altering its internal structure without changing its external behavior.
> — Martin Fowler

### The Two Hats

```
┌─────────────────────────────────────────────────────────────┐
│                    Adding Features                          │
│  - Add new tests for new behavior                          │
│  - Write code to make tests pass                           │
│  - Don't change existing tests                             │
└─────────────────────────────────────────────────────────────┘
                              ↕
┌─────────────────────────────────────────────────────────────┐
│                     Refactoring                             │
│  - Restructure existing code                               │
│  - All existing tests must still pass                      │
│  - Don't add new functionality                             │
└─────────────────────────────────────────────────────────────┘
```

**Rule**: Never wear both hats at the same time.

---

## When to Refactor

### The Rule of Three

```
1st time: Just do it
2nd time: Wince at the duplication, but do it anyway
3rd time: Refactor
```

### Refactoring Triggers

| Trigger | Symptoms | Action |
|---------|----------|--------|
| **Before adding feature** | Hard to understand where to add | Refactor for clarity first |
| **After bug fix** | Found confusing code | Clean up the area |
| **During code review** | Reviewer suggests improvements | Apply suggestions |
| **Boy Scout Rule** | Code slightly worse than needed | Leave it better than you found it |

### Code Smells Indicating Need

```
Bloaters:
├── Long Method (> 20 lines)
├── Large Class (> 200 lines)
├── Primitive Obsession (lots of primitives instead of objects)
├── Long Parameter List (> 3 parameters)
└── Data Clumps (same data groups appearing together)

Object-Orientation Abusers:
├── Switch Statements (repeated type checking)
├── Refused Bequest (subclass doesn't use inherited methods)
├── Alternative Classes with Different Interfaces
└── Temporary Field (only set in certain circumstances)

Change Preventers:
├── Divergent Change (one class changed for multiple reasons)
├── Shotgun Surgery (one change requires many small changes)
└── Parallel Inheritance Hierarchies

Dispensables:
├── Comments (explaining bad code)
├── Duplicate Code
├── Dead Code
├── Speculative Generality (unused abstractions)
└── Lazy Class (does too little)

Couplers:
├── Feature Envy (method uses another class more than its own)
├── Inappropriate Intimacy (classes too intertwined)
├── Message Chains (a.b.c.d.method())
└── Middle Man (class delegates everything)
```

---

## Safe Refactoring Process

### Prerequisites

```
Before Starting:
├── ✅ All tests passing
├── ✅ Code committed to version control
├── ✅ Sufficient test coverage for area being refactored
├── ✅ Clear understanding of current behavior
└── ✅ Time allocated (don't refactor under pressure)
```

### The Refactoring Cycle

```
┌─────────────────────────────────────────────┐
│           1. Run Tests (Green)              │
└─────────────────────┬───────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────┐
│           2. Make Small Change              │
└─────────────────────┬───────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────┐
│           3. Run Tests                      │
│      ┌──────────────┴──────────────┐       │
│      ▼                             ▼       │
│   Green?                         Red?      │
│      │                             │       │
│      ▼                             ▼       │
│   Commit                       Revert      │
│      │                             │       │
│      └─────────────┬───────────────┘       │
└────────────────────┴────────────────────────┘
                      │
                      ▼
                   Repeat
```

### Commit Strategy

```
# Small, frequent commits during refactoring

git commit -m "refactor: extract validateEmail to separate function"
git commit -m "refactor: rename userInfo to customerProfile"  
git commit -m "refactor: move discount logic to PricingService"
git commit -m "refactor: replace conditional with polymorphism"
```

---

## Common Refactoring Techniques

### Extract Method

**Before:**
```python
def print_owing(self):
    # Print banner
    print("*************************")
    print("***** Customer Owes *****")
    print("*************************")
    
    # Calculate outstanding
    outstanding = 0
    for order in self.orders:
        outstanding += order.amount
    
    # Print details
    print(f"name: {self.name}")
    print(f"amount: {outstanding}")
```

**After:**
```python
def print_owing(self):
    self._print_banner()
    outstanding = self._calculate_outstanding()
    self._print_details(outstanding)

def _print_banner(self):
    print("*************************")
    print("***** Customer Owes *****")
    print("*************************")

def _calculate_outstanding(self) -> float:
    return sum(order.amount for order in self.orders)

def _print_details(self, outstanding: float):
    print(f"name: {self.name}")
    print(f"amount: {outstanding}")
```

### Extract Class

**Before:**
```python
class Person:
    def __init__(self):
        self.name = ""
        self.office_area_code = ""
        self.office_number = ""
    
    def get_telephone_number(self):
        return f"({self.office_area_code}) {self.office_number}"
```

**After:**
```python
class Person:
    def __init__(self):
        self.name = ""
        self.office_telephone = TelephoneNumber()
    
    def get_telephone_number(self):
        return self.office_telephone.get_number()


class TelephoneNumber:
    def __init__(self):
        self.area_code = ""
        self.number = ""
    
    def get_number(self):
        return f"({self.area_code}) {self.number}"
```

### Replace Conditional with Polymorphism

**Before:**
```python
def calculate_shipping(order):
    if order.shipping_type == "standard":
        return 5.00
    elif order.shipping_type == "express":
        return 15.00
    elif order.shipping_type == "overnight":
        return 25.00
    else:
        raise ValueError(f"Unknown shipping type: {order.shipping_type}")
```

**After:**
```python
class ShippingStrategy(ABC):
    @abstractmethod
    def calculate(self, order) -> float:
        pass


class StandardShipping(ShippingStrategy):
    def calculate(self, order) -> float:
        return 5.00


class ExpressShipping(ShippingStrategy):
    def calculate(self, order) -> float:
        return 15.00


class OvernightShipping(ShippingStrategy):
    def calculate(self, order) -> float:
        return 25.00


# Usage
shipping_strategies = {
    "standard": StandardShipping(),
    "express": ExpressShipping(),
    "overnight": OvernightShipping(),
}

def calculate_shipping(order):
    strategy = shipping_strategies.get(order.shipping_type)
    if not strategy:
        raise ValueError(f"Unknown shipping type: {order.shipping_type}")
    return strategy.calculate(order)
```

### Replace Magic Numbers with Constants

**Before:**
```python
def calculate_potential_energy(mass, height):
    return mass * 9.81 * height

def is_adult(age):
    return age >= 18

def apply_discount(price):
    if price > 100:
        return price * 0.9
    return price
```

**After:**
```python
GRAVITATIONAL_CONSTANT = 9.81
ADULT_AGE_THRESHOLD = 18
DISCOUNT_THRESHOLD = 100
DISCOUNT_PERCENTAGE = 0.10

def calculate_potential_energy(mass, height):
    return mass * GRAVITATIONAL_CONSTANT * height

def is_adult(age):
    return age >= ADULT_AGE_THRESHOLD

def apply_discount(price):
    if price > DISCOUNT_THRESHOLD:
        return price * (1 - DISCOUNT_PERCENTAGE)
    return price
```

### Introduce Parameter Object

**Before:**
```python
def amount_invoiced(start_date, end_date):
    pass

def amount_received(start_date, end_date):
    pass

def amount_overdue(start_date, end_date):
    pass
```

**After:**
```python
@dataclass(frozen=True)
class DateRange:
    start: date
    end: date
    
    def contains(self, d: date) -> bool:
        return self.start <= d <= self.end
    
    @property
    def days(self) -> int:
        return (self.end - self.start).days


def amount_invoiced(period: DateRange):
    pass

def amount_received(period: DateRange):
    pass

def amount_overdue(period: DateRange):
    pass
```

### Replace Temp with Query

**Before:**
```python
def calculate_total(order):
    base_price = order.quantity * order.item_price
    
    if base_price > 1000:
        discount_factor = 0.95
    else:
        discount_factor = 0.98
    
    return base_price * discount_factor
```

**After:**
```python
def calculate_total(order):
    return self._base_price(order) * self._discount_factor(order)

def _base_price(self, order):
    return order.quantity * order.item_price

def _discount_factor(self, order):
    if self._base_price(order) > 1000:
        return 0.95
    return 0.98
```

### Move Method

**Before:**
```python
class Account:
    def __init__(self, account_type):
        self.account_type = account_type
    
    def overdraft_charge(self):
        if self.account_type.is_premium():
            result = 10
            if self.days_overdrawn > 7:
                result += (self.days_overdrawn - 7) * 0.85
            return result
        else:
            return self.days_overdrawn * 1.75
```

**After:**
```python
class Account:
    def __init__(self, account_type):
        self.account_type = account_type
    
    def overdraft_charge(self):
        return self.account_type.overdraft_charge(self.days_overdrawn)


class AccountType:
    def overdraft_charge(self, days_overdrawn):
        if self.is_premium():
            result = 10
            if days_overdrawn > 7:
                result += (days_overdrawn - 7) * 0.85
            return result
        else:
            return days_overdrawn * 1.75
```

---

## Refactoring Patterns by Smell

### Long Method → Extract Method

```
Signs:
- Method > 20 lines
- Comments explaining sections
- Multiple levels of nesting

Actions:
1. Identify cohesive blocks
2. Extract to well-named methods
3. Use meaningful parameter names
4. Keep methods at same abstraction level
```

### Large Class → Extract Class / Extract Superclass

```
Signs:
- Class > 200 lines
- Too many instance variables
- Methods clustered around subsets of variables

Actions:
1. Identify data clumps (fields used together)
2. Extract to new class
3. Use composition or inheritance
4. Delegate methods to new class
```

### Feature Envy → Move Method

```
Signs:
- Method uses more features of another class
- Lots of getters from other objects
- Calculations that belong elsewhere

Actions:
1. Move method to class it uses most
2. Consider if data should move too
3. Update callers to use new location
```

### Primitive Obsession → Replace with Value Object

```
Signs:
- Primitives with special meaning
- Validation repeated in multiple places
- Comments explaining what primitive represents

Actions:
1. Create value object class
2. Move validation to constructor
3. Add behavior methods
4. Replace primitives with value objects
```

### Data Clumps → Introduce Parameter Object

```
Signs:
- Same parameters appear together
- Same fields appear in multiple classes
- Methods that only use subset of parameters

Actions:
1. Create class for the data clump
2. Move methods that use only this data
3. Replace parameters with object
```

---

## Refactoring Catalog Quick Reference

| Smell | Refactoring | Description |
|-------|-------------|-------------|
| Long Method | Extract Method | Break into smaller methods |
| Large Class | Extract Class | Split responsibilities |
| Long Parameter List | Parameter Object | Group related parameters |
| Duplicated Code | Extract Method/Class | Single source of truth |
| Divergent Change | Extract Class | Separate concerns |
| Shotgun Surgery | Move Method/Field | Consolidate changes |
| Feature Envy | Move Method | Put behavior with data |
| Data Clumps | Extract Class | Group related data |
| Primitive Obsession | Replace with Object | Use value objects |
| Switch Statements | Replace with Polymorphism | Use inheritance |
| Parallel Hierarchies | Move Method/Field | Consolidate hierarchies |
| Lazy Class | Inline Class | Remove unnecessary abstraction |
| Speculative Generality | Remove | Delete unused code |
| Temporary Field | Extract Class | Move to proper home |
| Message Chains | Hide Delegate | Reduce coupling |
| Middle Man | Remove Middle Man | Eliminate delegation |
| Inappropriate Intimacy | Move Method/Field | Separate classes |
| Comments | Extract Method | Self-documenting code |

---

## Testing During Refactoring

### Characterization Tests

When refactoring legacy code without tests:

```python
# Step 1: Write tests that describe current behavior
def test_calculate_price_current_behavior():
    """Characterization test - documents current behavior."""
    # Call the function and capture what it returns
    result = calculate_price(100, "premium", True)
    
    # Assert the current behavior (even if it's wrong)
    assert result == 85.5  # Whatever it currently returns
```

### Coverage Before Refactoring

```bash
# Ensure high coverage of code to refactor
pytest --cov=src/module_to_refactor --cov-report=term-missing

# Look for:
# - Lines not covered
# - Branches not tested
# - Edge cases missed
```

### Mutation Testing

```bash
# Verify tests catch bugs
mutmut run --paths-to-mutate=src/module.py

# Good: Most mutants are killed (tests caught the change)
# Bad: Many mutants survived (tests are weak)
```

---

## IDE Refactoring Tools

### Common Automated Refactorings

| Refactoring | Shortcut (VS Code) | Description |
|-------------|-------------------|-------------|
| Rename | F2 | Rename symbol across codebase |
| Extract Method | Cmd+Shift+R | Extract selection to method |
| Extract Variable | Cmd+Shift+L | Extract expression to variable |
| Inline | Cmd+Shift+I | Inline variable/method |
| Move | - | Move to different file/class |

### Safe Refactoring Steps with IDE

```
1. Select code to refactor
2. Use IDE refactoring (not manual)
3. Review changes in preview
4. Run tests before accepting
5. Commit immediately after
```

---

## Anti-Patterns

### Big Bang Refactoring

```
❌ Wrong:
"Let's rewrite this entire module over the weekend"

✅ Right:
Make small, incremental changes with tests passing after each
```

### Refactoring Without Tests

```
❌ Wrong:
Refactoring code without test coverage

✅ Right:
Write characterization tests first, then refactor
```

### Mixing Refactoring with Features

```
❌ Wrong:
"While I'm here, let me also add this new feature"

✅ Right:
Separate commits: refactor first, then add feature
```

### Gold Plating

```
❌ Wrong:
Refactoring code that works and nobody needs to change

✅ Right:
Refactor code you're actively working with
```

---

## Refactoring Checklist

### Before Starting

- [ ] All tests passing
- [ ] Code committed
- [ ] Understand current behavior
- [ ] Identified specific smell to address

### During Refactoring

- [ ] Making small changes
- [ ] Running tests after each change
- [ ] Committing frequently
- [ ] Not adding new features
- [ ] Not fixing bugs (unless that's the goal)

### After Refactoring

- [ ] All original tests still pass
- [ ] No new functionality added
- [ ] Code is clearer/simpler
- [ ] Technical debt documented if remaining
- [ ] Team informed of significant changes
