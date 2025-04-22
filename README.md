class: center, middle
# Book Club - Comisc Python - 04/23/2025
### Chapter 8: Events and the Message Bus
### Agustin Alexander
---
## Chapter 8: Events and the Message Bus

*   We've built core features, but real-world systems have lots of "goop around the edge": reporting, permissions, workflows.
*   Example Requirement: Notify the buying team via email when an allocation fails due to being out of stock.
*   Let's see how our architecture handles this kind of side-effect.
*   We'll explore common pitfalls and introduce Domain Events and a Message Bus.

---

### The Goal: Avoid Making a Mess

Adding features like notifications can easily clutter our codebase if not done carefully.

Where *not* to put the email sending logic?

1.  Web Controllers/Endpoints?
2.  Domain Model?
3.  Service Layer?

Let's look at why these are often poor choices.

---

### Pitfall 1: Cluttering Web Controllers

```python
# src/allocation/entrypoints/flask_app.py
@app.route("/allocate", methods=['POST'])
def allocate_endpoint():
    line = model.OrderLine(
        request.json['orderid'], request.json['sku'], request.json['qty'],
    )
    try:
        uow = unit_of_work.SqlAlchemyUnitOfWork()
        batchref = services.allocate(line, uow)
    except (model.OutOfStock, services.InvalidSku) as e:
        # Yikes! Direct email sending in the endpoint
        send_mail(
            'out of stock',
            'stock_admin@made.com',
            f'{line.orderid} - {line.sku}'
        )
        return jsonify({'message': str(e)}), 400

    return jsonify({'batchref': batchref}), 201
```

*   **Problem:** Mixes HTTP concerns with notification logic. Harder to unit test notifications. Violates separation of concerns.

---

### Pitfall 2: Polluting the Domain Model

```python
# src/allocation/domain/model.py
class Product:
    # ... (init omitted) ...
    def allocate(self, line: OrderLine) -> str:
        try:
            batch = next(
                b for b in sorted(self.batches) if b.can_allocate(line)
            )
            # ... (allocation logic) ...
        except StopIteration:
            # Even worse! Infrastructure dependency in the model!
            email.send_mail('stock@made.com', f'Out of stock for {line.sku}')
            raise OutOfStock(f'Out of stock for sku {line.sku}')
```

*   **Problem:** Domain model should be pure and free of infrastructure (like email). Violates DIP. Model should focus on domain rules, not side effects.

---

### Pitfall 3: Tangling the Service Layer

```python
# src/allocation/service_layer/services.py
def allocate(
        orderid: str, sku: str, qty: int,
        uow: unit_of_work.AbstractUnitOfWork
) -> str:
    line = OrderLine(orderid, sku, qty)
    with uow:
        product = uow.products.get(sku=line.sku)
        if product is None:
            raise InvalidSku(f'Invalid sku {line.sku}')
        try:
            batchref = product.allocate(line)
            uow.commit()
            return batchref
        except model.OutOfStock:
            # Better than model/endpoint, but still awkward.
            email.send_mail('stock@made.com', f'Out of stock for {line.sku}')
            raise # Re-raising feels clunky
```

*   **Problem:** Still mixes allocation orchestration with notification implementation details. Catching and re-raising specifically to send an email feels like a workaround.

---

### The Root Cause: Single Responsibility Principle (SRP)

*   Our `allocate` functions (endpoint, service, model) are doing more than just allocation. They are doing "allocate *and* send email if out of stock".
*   **SRP:** A class/function should have only one reason to change.
    *   Changing notification method (email -> SMS) shouldn't force changes in allocation logic.
*   **Goal:** Separate the "out of stock" _fact_ from the _reaction_ (sending notification).
*   Apply Dependency Inversion Principle (DIP) to notifications.

---
class: center, middle
### Solution: Domain Events and a Message Bus

1.  **Domain Events:** Objects representing significant occurrences in the domain.
2.  **Message Bus:** A mechanism to route events to interested handlers (subscribers).

![Events flowing through the system](https://github.com/pcah/cosmic-python-book/blob/master/images/apwp_0801.png?raw=true)

---

### Step 1: Define Domain Events

*   Events are simple data carriers (Value Objects), often `dataclasses`.
*   Named using domain language, past tense.
*   Part of the domain model layer.

```python
# src/allocation/domain/events.py
from dataclasses import dataclass

class Event:  # <1> Base class for type hinting and common attrs
    pass

@dataclass
class OutOfStock(Event): # <2> Specific event for OOS
    sku: str
```

---

### Step 2: Raise Events from the Domain Model

*   The model records *what happened* by appending events to a list.
*   It doesn't know *who* cares or *what* they will do.

**Test:**

```python
# tests/unit/test_product.py
def test_records_out_of_stock_event_if_cannot_allocate():
    # ... (setup product and batch) ...
    product.allocate(OrderLine('order1', 'SMALL-FORK', 10))

    allocation = product.allocate(OrderLine('order2', 'SMALL-FORK', 1))
    assert product.events[-1] == events.OutOfStock(sku="SMALL-FORK") # <1> Check event list
    assert allocation is None
```

---

### Step 2: Raise Events (Implementation)

```python
# src/allocation/domain/model.py
class Product:
    def __init__(self, sku: str, batches: List[Batch], version_number: int = 0):
        # ... (other attrs) ...
        self.events = []  # type: List[events.Event]  # <1> Initialize event list

    def allocate(self, line: OrderLine) -> Optional[str]: # Return optional str
        try:
            batch = next(
                b for b in sorted(self.batches) if b.can_allocate(line)
            )
            batch.allocate(line)
            self.version_number += 1
            return batch.reference
        except StopIteration:
            self.events.append(events.OutOfStock(sku=line.sku)) # <2> Record event
            return None # <3> Return None instead of raising OutOfStock exception
```

**Note:** Avoid using exceptions for control flow when domain events can represent the same outcome.

---

### Step 3: Create a Message Bus

*   A simple Publish/Subscribe mechanism.
*   Maps event types to handler functions.
*   Often just a dictionary.
**Note:** This basic bus is synchronous. Error handling added for robustness.
Code in next slide... 
---
Step 3: Create a Message Bus
```python
# src/allocation/service_layer/messagebus.py
def handle(event: events.Event):
    # Need to get handlers for the specific event type
    handlers = HANDLERS.get(type(event), [])
    for handler in handlers:
        try:
            # Could add error handling, dependency injection here
            handler(event)
        except Exception:
            # Log error, decide how to proceed
            print(f"Exception handling event {event}:")
            # traceback.print_exc() # Or proper logging
            continue # Or re-raise, or put on dead-letter queue

def send_out_of_stock_notification(event: events.OutOfStock):
    email.send_mail(
        'stock@made.com',
        f'Out of stock for {event.sku}',
    )

# Map event types to lists of handler functions
HANDLERS: Dict[Type[events.Event], List[Callable]] = {
    events.OutOfStock: [send_out_of_stock_notification],
}
```


---

### Aside: Is this like Celery?

*   **No.** This message bus is primarily for *in-process*, synchronous decoupling and choreography of tasks within a single unit of work (initially).
*   It's more like a UI event loop or actor framework conceptually.
*   **Celery:** For distributing *self-contained* tasks *asynchronously* across processes/machines, often via brokers (Redis, RabbitMQ).
*   If async/distribution is needed, consider *external events* persisted and consumed by separate workers/services (See Chapter 11). Our internal events *could* be published externally.

---

### Connecting Model Events to the Bus

We need something to take events recorded on the model (`product.events`) and pass them to `messagebus.handle()`.

Three Options:

1.  Service Layer explicitly gets events from model & calls `handle()`.
2.  Service Layer creates and raises its *own* events, calling `handle()`.
3.  Unit of Work automatically collects events from tracked aggregates & calls `handle()`.

---

### Option 1: Service Layer Publishes Model Events
*   **Pros:** Explicit, relatively simple.
*   **Cons:** Service layer still has some event-handling responsibility. Repetitive code. Careful placement relative to `commit()` needed.
* Code in next slide...
---
### Option 1: Service Layer Publishes Model Events
```python
def allocate(orderid: str, sku: str, qty: int, uow: unit_of_work.AbstractUnitOfWork) -> str:
    line = model.OrderLine(orderid, sku, qty)
    with uow:
        product = uow.products.get(sku=line.sku)
        if product is None:
            raise model.InvalidSku(f'Invalid sku {line.sku}')
        try: # <1> Still need try/finally (or similar)
            batchref = product.allocate(line) # May append events
        finally: # <1> Or maybe better after commit?
            # Collect events *before* commit in case commit fails? No, domain events mean something *happened*.
            # Collect events *after* commit? Yes, usually safer.
            pass # Decide where event handling goes relative to commit

        uow.commit()
        # <2> Explicitly handle events from the aggregate *after* successful commit
        # (Need a way to access product events post-commit if needed, or handle before commit but only if commit succeeds)
        # Let's assume product is still accessible or events collected before commit:
        events_to_publish = list(product.events) # Make a copy
        product.events.clear() # Clear original list
    for event in events_to_publish:
         messagebus.handle(event)
    return batchref # allocate now returns Optional[str] or raises if needed
```

---

### Option 2: Service Layer Raises Its Own Events

```python
# src/allocation/service_layer/services.py
# (Model does NOT raise events in this option)
def allocate(orderid: str, sku: str, qty: int, uow: unit_of_work.AbstractUnitOfWork) -> str:
    line = model.OrderLine(orderid, sku, qty)
    with uow:
        product = uow.products.get(sku=line.sku)
        if product is None:
            raise model.InvalidSku(f'Invalid sku {line.sku}')
        # Assume product.allocate returns None on failure, no exception/event
        batchref = product.allocate(line)
        uow.commit() # Commit the state change (or lack thereof)

    # After commit, check the outcome and publish *service level* event
    if batchref is None: # Service layer interprets outcome
         messagebus.handle(events.OutOfStock(line.sku)) # and raises event

    return batchref
```
*   **Pros:** Keeps domain model purely focused on state changes.
*   **Cons:** Service layer becomes responsible for *knowing when* to raise domain events, blurring boundaries. Logic for "what constitutes an OutOfStock event" is now outside the model.

---

### Option 3 (Preferred): Unit of Work Publishes Events

*   The UoW manages the transaction boundary (`commit`, `rollback`).
*   It interacts with the Repository, which loads aggregates.
*   It's a natural place to collect and publish events *after* a successful commit.

**Requires:**
1.  Repository tracks aggregates it has seen (`.seen` attribute).
2.  UoW iterates over seen aggregates after `_commit()` and calls `messagebus.handle()`.

---

### Option 3: UoW Implementation - ABC

```python
# src/allocation/service_layer/unit_of_work.py
class AbstractUnitOfWork(abc.ABC):
    products: repository.AbstractRepository

    def __enter__(self):
        return self

    def __exit__(self, *args):
        self.rollback() # Default action on exit unless committed

    # Collect events from all tracked aggregates
    def collect_new_events(self):
        for product in self.products.seen:
            while product.events:
                yield product.events.pop(0)

    def commit(self):
        self._commit()          # <1> Delegate actual commit
        # <2> Publish events *after* successful commit
        for event in self.collect_new_events():
            messagebus.handle(event)

    @abc.abstractmethod
    def _commit(self):          # <1> Subclasses implement this
        raise NotImplementedError

    @abc.abstractmethod
    def rollback(self):
        raise NotImplementedError
```
---
### Option 3: UoW Implementation - Example SQLAlchemy
```python
class SqlAlchemyUnitOfWork(AbstractUnitOfWork):
    def __init__(self, session_factory):
        self.session_factory = session_factory

    def __enter__(self):
        self.session = self.session_factory()
        self.products = repository.SqlAlchemyRepository(self.session)
        return super().__enter__()

    def __exit__(self, *args):
        super().__exit__(*args)
        self.session.close()

    def _commit(self):          # <1> Implements actual commit
        self.session.commit()

    def rollback(self):
        self.session.rollback()
```
---

### Option 3: Repository Tracking - ABC

```python
# src/allocation/adapters/repository.py
class AbstractRepository(abc.ABC):
    def __init__(self):
        self.seen = set() # type: Set[model.Product] # <1> Track seen aggregates

    def add(self, product: model.Product): # <2> Track on add
        self._add(product)
        self.seen.add(product)

    def get(self, sku) -> model.Product | None: # <3> Track on get (allow None)
        product = self._get(sku)
        if product:
            self.seen.add(product)
        return product

    # Optional: Add get_by_reference if needed, also tracks
    def get_by_reference(self, reference) -> model.Product | None:
         product = self._get_by_reference(reference)
         if product:
             self.seen.add(product)
         return product

    @abc.abstractmethod
    def _add(self, product: model.Product): # <2> Subclass implements actual add
        raise NotImplementedError

    @abc.abstractmethod
    def _get(self, sku) -> model.Product | None: # <3> Subclass implements actual get
        raise NotImplementedError
  ```
---
###  Option 3: Repository Tracking - Example SQLAlchemy Implementation
```python
class SqlAlchemyRepository(AbstractRepository):
    def __init__(self, session):
        super().__init__() # <1> Remember super().__init__()
        self.session = session

    def _add(self, product): # <2>
        self.session.add(product)

    def _get(self, sku): # <3>
        return self.session.query(model.Product).filter_by(sku=sku).first()

```
**Note:** Using `_private()` methods and subclassing is one way; composition (wrappers) is another (see exercise).

---

### Option 3: Result - Clean Service Layer!

With the UoW handling event publishing, the service layer is back to being simple orchestration:

```python
# src/allocation/service_layer/services.py
from allocation.domain import model, events
from allocation.service_layer import unit_of_work
from typing import Optional # Import Optional

def allocate(
        orderid: str, sku: str, qty: int,
        uow: unit_of_work.AbstractUnitOfWork
) -> Optional[str]: # Returns Optional[str] as allocate can return None
    line = model.OrderLine(orderid, sku, qty)
    with uow: # UoW context manager handles begin/commit/rollback/publish
        product = uow.products.get(sku=line.sku)
        if product is None:
            raise model.InvalidSku(f'Invalid sku {line.sku}')
        batchref = product.allocate(line) # Model raises event internally
        uow.commit() # UoW._commit() runs, then UoW publishes events
    return batchref # Return the result from allocate
```
*   No explicit `messagebus.handle` calls here!

---

### Option 3: Updating Fakes

Remember to update fakes to match the new `AbstractRepository` and `AbstractUnitOfWork` interfaces (implement `_add`, `_get`, `_commit`, call `super().__init__()`).

```python
# tests/unit/test_services.py
# Assume FakeRepository is defined elsewhere or inline
class FakeRepository(repository.AbstractRepository):
    def __init__(self, products: List[model.Product]):
        super().__init__() # Important!
        self._products = set(products)

    def _add(self, product: model.Product):
        self._products.add(product)

    def _get(self, sku: str) -> model.Product | None:
        return next((p for p in self._products if p.sku == sku), None)

    def list(self) -> List[model.Product]: # If list is needed
        return list(self._products)
```
*   Maintaining fakes is work, but usually manageable as core abstractions stabilize. ABCs/Protocols help.
---

### Option 3: Updating Fakes - UOW
```python
class FakeUnitOfWork(unit_of_work.AbstractUnitOfWork):
    def __init__(self):
        self.products = FakeRepository([])
        self.committed = False
        # Track collected events for assertions if needed
        self.published_events = []

    def __enter__(self):
        self.published_events = [] # Reset on enter
        return super().__enter__()

    def _commit(self):
        self.committed = True

    def rollback(self):
        pass # Fake rollback does nothing
```
---

### Wrap-Up

*   **Domain Events** help model real-world workflows ("When X happens, then Y").
*   They decouple primary actions (allocation) from side-effects (notifications).
*   Improves testability, observability, and separates concerns (SRP).
*   **Message Bus** routes events to handlers.
*   Events are key to managing consistency across aggregate boundaries (more next chapter).

---

### Recap: Domain Events and the Message Bus

*   **Events & SRP:** Separate core logic from side-effects/reactions. Communicate between aggregates for eventual consistency.
*   **Message Bus:** Simple Pub/Sub (often a `dict`) mapping events to handlers. Infrastructure, not domain logic.
*   **Option 1 (Service Publishes):** Service calls `bus.handle(event)` explicitly after commit. Simple but repetitive & needs careful placement.
*   **Option 2 (Model Publishes, Service Relays):** Model raises events (`agg.events.append(...)`), Service collects & calls `bus.handle(agg.events)`. Better SRP, still service layer involvement.
*   **Option 3 (UoW Publishes):** Repository tracks seen aggregates; UoW collects events from `repo.seen` and calls `bus.handle(event)` after `commit`. Cleanest service layer, but more complex/magic setup. Recommended.

---

### Domain Events: The Trade-Offs - Cons

### Cons:
*   ❌ Adds complexity (Message Bus, event tracking/publishing mechanism).
*   ❌ UoW publishing hides event handling (magic); `commit()` implicitly triggers handlers.
*   ❌ Default bus is synchronous; handlers run within the `commit()` call, potentially blocking the caller (e.g., web request) longer than expected. Async adds more complexity.
*   ❌ Event chains can obscure overall workflow (harder to trace request flow end-to-end).
*   ❌ Risk of circular dependencies or infinite loops between handlers if not carefully designed.
*   ❌ Error handling within handlers needs consideration (retry? dead-letter queue?).

---
### Domain Events: The Trade-Offs - Pros
### Pros:
*   ✅ Nice separation of responsibilities (SRP).
*   ✅ Handlers decoupled from core logic; easy to change/add reactions.
*   ✅ Events model real-world concepts & improve communication.
*   ✅ Enables eventual consistency across aggregates.

---
class: center, middle
# Thank You!

### Questions ???
