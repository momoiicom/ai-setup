# The MVVM-Class Architecture

## What This Is

This project uses a **Strict MVVM (Model-View-ViewModel)** architecture. This is the same pattern used in modern iOS (SwiftUI) and Android (Jetpack Compose) development.

**You MUST follow these rules for ALL code you generate. No exceptions.**

---

## The 3 Layers

There are exactly 3 layers. Every file you create belongs to one of them.

### Layer 1: Models (Data Only — No Logic)

**What:** Plain TypeScript `interface` or `type` definitions.
**Purpose:** Describe the shape of data (from APIs, databases, etc.).
**Rules:**

- Models are ONLY data shapes. They contain NO methods, NO logic, NO side effects.
- One model per concept (e.g., `Product`, `Order`, `User`).

**Example:**

```typescript
// models/Product.ts
interface Product {
  id: string;
  title: string;
  price: number;
  inventory: number;
}
```

### Layer 2: ViewModels (All Logic — No UI)

**What:** TypeScript **classes** that contain ALL business logic.
**Purpose:** Handle state, validation, API calls, data transformation, and side effects.
**Rules:**

- Every major feature or screen gets its own ViewModel class.
- Internal state uses `private` properties. The outside world interacts through `public` methods and `public` getters only.
- **CRITICAL: A ViewModel must NEVER import or reference React, DOM, CLI, or any View-layer code.** It must work in a plain Node.js / Vitest test with zero UI dependencies.
- All logic lives here: validation, fetching, filtering, sorting, error handling, etc.

**Example:**

```typescript
// viewmodels/ProductListViewModel.ts
import { BaseViewModel } from "./BaseViewModel";

// Dependency interface (defines what the ViewModel needs from the outside world)
interface ProductService {
  fetchProducts(): Promise<Product[]>;
}

class ProductListViewModel extends BaseViewModel {
  // --- Private state (never accessed directly from the View) ---
  private _products: Product[] = [];
  private _isLoading = false;
  private _error: string | null = null;

  // --- Constructor (dependencies are injected here) ---
  constructor(private productService: ProductService) {
    super();
  }

  // --- Public getters (read-only access for the View) ---
  get products(): Product[] { return this._products; }
  get isLoading(): boolean { return this._isLoading; }
  get error(): string | null { return this._error; }

  // --- Public methods (the View calls these to trigger actions) ---
  async loadProducts(): Promise<void> {
    this._isLoading = true;
    this._error = null;
    this.notify();
    try {
      const data = await this.productService.fetchProducts();
      this._products = this.sortByPrice(data);
    } catch (e) {
      this._error = e instanceof Error ? e.message : "Failed to load";
    } finally {
      this._isLoading = false;
      this.notify();
    }
  }

  getAffordableProducts(maxPrice: number): Product[] {
    return this._products.filter(p => p.price <= maxPrice);
  }

  // --- Private methods (internal helpers — not visible to the View) ---
  private sortByPrice(items: Product[]): Product[] {
    return [...items].sort((a, b) => a.price - b.price);
  }
}
```

### Layer 3: Views (Display Only — No Logic)

**What:** React components OR CLI scripts that display data and forward user actions.
**Purpose:** Render what the ViewModel exposes. Call ViewModel methods when the user acts.
**Rules:**

- Views are "dumb." They contain NO business logic, NO data transformation, NO validation.
- A View reads state from its ViewModel and calls ViewModel methods. That is ALL it does.
- In **React**: use a thin binding (e.g., MobX, Zustand, or a custom hook) to connect the ViewModel class to the component.
- In **CLI**: the entry script instantiates the ViewModel, calls its methods, and prints the results.

**Example (React):**

```tsx
// views/ProductList.tsx
import { useViewModel } from "@/hooks/useViewModel";
import { ProductListViewModel } from "@/viewmodels/ProductListViewModel";
import { productService } from "@/services/productService";

function ProductList() {
  const vm = useViewModel(() => new ProductListViewModel(productService));

  useEffect(() => { vm.loadProducts(); }, []);

  if (vm.isLoading) return <Spinner />;
  return (
    <ul>
      {vm.products.map(p => <li key={p.id}>{p.title}</li>)}
    </ul>
  );
}
```

> **Note on `useEffect` for initial data loading:** The `useEffect(() => { vm.loadProducts(); }, [])` pattern is a valid use of Effects — it synchronizes with an external system (the API) because the component was displayed. However, in **Next.js App Router** projects, this Effect is often unnecessary: the Server Component fetches data and passes it as props, and the ViewModel is seeded in the factory (see the [Next.js App Router](#nextjs-app-router) section). Prefer server-side fetching when available — it eliminates the loading spinner entirely for the initial render.

---

## Error Handling & Loading State Patterns

Errors and loading states are the most common ViewModel state besides the data itself. Every ViewModel that does async work MUST follow these patterns.

### The Standard State Properties

Every async ViewModel exposes at minimum these three state groups through private properties and public getters:


| Property                    | Type                | Purpose                                              |
| --------------------------- | ------------------- | ---------------------------------------------------- |
| `_isLoading`                | `boolean`           | `true` while the initial data load is in progress    |
| `_error`                    | `string | null`     | Human-readable error message, or `null` when healthy |
| `_data` (e.g., `_products`) | `T[]` or `T | null` | The actual data                                      |


### Error Flow: ViewModel to View

The ViewModel catches all errors internally and exposes them as **plain strings** via a public getter. The View never sees exceptions — it reads `vm.error` and decides how to render it.

**ViewModel (catches and stores):**

```typescript
async loadProducts(): Promise<void> {
  this._isLoading = true;
  this._error = null;
  this.notify();
  try {
    const data = await this.productService.fetchProducts();
    this._products = this.sortByPrice(data);
  } catch (e) {
    this._error = e instanceof Error ? e.message : "An unexpected error occurred";
    this._products = [];
  } finally {
    this._isLoading = false;
    this.notify();
  }
}
```

**View (reads and renders):**

```tsx
function ProductList() {
  const vm = useViewModel(() => new ProductListViewModel(productService));

  useEffect(() => { vm.loadProducts(); }, []);

  if (vm.isLoading) return <Spinner />;
  if (vm.error) return <ErrorBanner message={vm.error} onRetry={() => vm.loadProducts()} />;
  if (vm.products.length === 0) return <EmptyState />;
  return <ul>{vm.products.map(p => <li key={p.id}>{p.title}</li>)}</ul>;
}
```

The View renders exactly four states in a predictable order: **loading → error → empty → data**. This order should be consistent across all Views.

### Rules for Error Handling

- **The ViewModel NEVER throws to the View.** All exceptions are caught inside the ViewModel and stored in `_error`.
- **Errors are plain strings.** The View does not need to know about error types, HTTP status codes, or stack traces. If the ViewModel needs to distinguish error types internally (e.g., retry on network failure, don't retry on 404), it can — but the View only sees a message.
- **Provide a retry mechanism.** If an operation can fail, the ViewModel should expose the same public method again (e.g., `loadProducts()`). The View can wire this to a "Try Again" button.
- **Clear the error on retry.** When the user retries, set `_error = null` before starting the new request.

### Multi-State Loading

For ViewModels that handle multiple async operations (e.g., loading data AND submitting a form), use **named loading states** instead of a single boolean:

```typescript
class OrderViewModel extends BaseViewModel {
  private _isLoadingOrder = false;
  private _isSubmitting = false;
  private _error: string | null = null;

  get isLoadingOrder(): boolean { return this._isLoadingOrder; }
  get isSubmitting(): boolean { return this._isSubmitting; }
  get error(): string | null { return this._error; }

  async loadOrder(id: string): Promise<void> {
    this._isLoadingOrder = true;
    this._error = null;
    this.notify();
    try {
      this._order = await this.orderService.getOrder(id);
    } catch (e) {
      this._error = e instanceof Error ? e.message : "Failed to load order";
    } finally {
      this._isLoadingOrder = false;
      this.notify();
    }
  }

  async submitOrder(): Promise<boolean> {
    this._isSubmitting = true;
    this._error = null;
    this.notify();
    try {
      await this.orderService.submit(this._order!);
      return true;
    } catch (e) {
      this._error = e instanceof Error ? e.message : "Failed to submit order";
      return false;
    } finally {
      this._isSubmitting = false;
      this.notify();
    }
  }
}
```

The View can now independently show a spinner for the page load and a disabled/loading state on the submit button:

```tsx
function OrderPage() {
  const vm = useViewModel(() => new OrderViewModel(orderService));

  if (vm.isLoadingOrder) return <Spinner />;
  if (vm.error) return <ErrorBanner message={vm.error} />;

  return (
    <form onSubmit={async (e) => { e.preventDefault(); await vm.submitOrder(); }}>
      {/* order fields */}
      <button type="submit" disabled={vm.isSubmitting}>
        {vm.isSubmitting ? "Submitting..." : "Place Order"}
      </button>
    </form>
  );
}
```

### When to Use Which Pattern


| Situation                                            | Pattern                                                  |
| ---------------------------------------------------- | -------------------------------------------------------- |
| ViewModel does one async operation (load data)       | Single `_isLoading` boolean                              |
| ViewModel does multiple independent async operations | Named booleans: `_isLoadingX`, `_isSubmitting`, etc.     |
| Operation can be retried by the user                 | Expose the same public method; View wires it to a button |
| Operation returns success/failure to the View        | Return `Promise<boolean>` from the public method         |


---

## Optimistic Updates

Optimistic updates improve perceived performance by updating the UI immediately before the server confirms the operation. If the server fails, the change is rolled back. This pattern is essential for good UX in interactive applications.

### The Core Pattern

1. **Snapshot** — Store the current state so you can roll back.
2. **Optimistic Apply** — Update the state immediately and call `notify()`.
3. **Server Call** — Send the request to the server.
4. **On Success** — Clear the pending snapshot.
5. **On Failure** — Roll back to the snapshot and set an error.

### Example: Adding to Cart

```typescript
// viewmodels/CartViewModel.ts
interface CartService {
  addItem(item: { productId: string; quantity: number }): Promise<void>;
  removeItem(productId: string): Promise<void>;
  updateQuantity(productId: string, quantity: number): Promise<void>;
  getItems(): Promise<CartItem[]>;
}

class CartViewModel extends BaseViewModel {
  private _items: CartItem[] = [];
  private _isSyncing = false;
  private _error: string | null = null;

  constructor(private cartService: CartService) {
    super();
  }

  get items(): CartItem[] { return this._items; }
  get isSyncing(): boolean { return this._isSyncing; }
  get error(): string | null { return this._error; }
  get itemCount(): number { return this._items.length; }

  async addItem(product: Product): Promise<void> {
    const previousItems = [...this._items];
    const item: CartItem = { productId: product.id, quantity: 1, product };

    this._items = [...this._items, item];
    this._isSyncing = true;
    this._error = null;
    this.notify();

    try {
      await this.cartService.addItem({ productId: product.id, quantity: 1 });
    } catch (e) {
      this._items = previousItems;
      this._error = e instanceof Error ? e.message : "Failed to add item to cart";
    } finally {
      this._isSyncing = false;
      this.notify();
    }
  }

  async removeItem(productId: string): Promise<void> {
    const previousItems = [...this._items];

    this._items = this._items.filter(i => i.productId !== productId);
    this._isSyncing = true;
    this._error = null;
    this.notify();

    try {
      await this.cartService.removeItem(productId);
    } catch (e) {
      this._items = previousItems;
      this._error = e instanceof Error ? e.message : "Failed to remove item";
    } finally {
      this._isSyncing = false;
      this.notify();
    }
  }

  async updateQuantity(productId: string, quantity: number): Promise<void> {
    const previousItems = [...this._items];

    this._items = this._items.map(i =>
      i.productId === productId ? { ...i, quantity } : i
    );
    this._isSyncing = true;
    this._error = null;
    this.notify();

    try {
      await this.cartService.updateQuantity(productId, quantity);
    } catch (e) {
      this._items = previousItems;
      this._error = e instanceof Error ? e.message : "Failed to update quantity";
    } finally {
      this._isSyncing = false;
      this.notify();
    }
  }
}
```

### View: Showing Sync State

The View can optionally show a "syncing" indicator for pending operations:

```tsx
function Cart() {
  const vm = useViewModel(() => new CartViewModel(cartService));

  return (
    <div>
      {vm.items.map(item => (
        <div key={item.productId}>
          {item.product.title} × {item.quantity}
          {vm.isSyncing && <Spinner size="small" />}
        </div>
      ))}
      {vm.error && <Toast message={vm.error} />}
    </div>
  );
}
```

### Conflict Resolution

When the server data may have changed during the optimistic operation, handle it based on business requirements:


| Strategy                 | When to Use                            | How                                              |
| ------------------------ | -------------------------------------- | ------------------------------------------------ |
| **Last-write-wins**      | Non-critical data (likes, preferences) | Optimistic update stays; server response ignored |
| **Server-authoritative** | Critical data (inventory, balances)    | After success, re-fetch server data to confirm   |
| **Merge**                | Collaborative data (documents)         | Apply server delta to local state                |


**Example: Server-authoritative with re-fetch**

```typescript
async addItem(product: Product): Promise<void> {
  const previousItems = [...this._items];
  this._items = [...this._items, { productId: product.id, quantity: 1, product }];
  this._isSyncing = true;
  this._error = null;
  this.notify();

  try {
    await this.cartService.addItem({ productId: product.id, quantity: 1 });
    this._items = await this.cartService.getItems();
  } catch (e) {
    this._items = previousItems;
    this._error = e instanceof Error ? e.message : "Failed to add item";
  } finally {
    this._isSyncing = false;
    this.notify();
  }
}
```

### Rules for Optimistic Updates

- **Always store a rollback snapshot** before the optimistic update.
- **Call `notify()` immediately after the optimistic update** so the UI reflects the change.
- **Roll back on any failure** — network error, validation error, or server error.
- **Clear errors before retrying** — set `_error = null` when the user attempts the action again.
- **Test rollback scenarios** — ensure the state is exactly restored after a simulated failure.

### Testing Optimistic Updates

```typescript
// Helper to create a mock CartService
function createMockCartService(overrides: Partial<CartService> = {}): CartService {
  return {
    addItem: async () => {},
    removeItem: async () => {},
    updateQuantity: async () => {},
    getItems: async () => [],
    ...overrides,
  };
}

describe("CartViewModel optimistic updates", () => {
  it("adds item optimistically and persists on success", async () => {
    const addItem = vi.fn().mockResolvedValue(undefined);
    const vm = new CartViewModel(createMockCartService({ addItem }));
    const product = { id: "1", title: "Widget", price: 10, inventory: 5 };

    await vm.addItem(product);

    expect(vm.items).toHaveLength(1);
    expect(vm.items[0].productId).toBe("1");
    expect(addItem).toHaveBeenCalled();
    expect(vm.isSyncing).toBe(false);
  });

  it("rolls back on failure", async () => {
    const vm = new CartViewModel(createMockCartService({
      addItem: async () => { throw new Error("Network error"); },
    }));
    const product = { id: "1", title: "Widget", price: 10, inventory: 5 };

    await vm.addItem(product);

    expect(vm.items).toHaveLength(0);
    expect(vm.error).toBe("Network error");
    expect(vm.isSyncing).toBe(false);
  });

  it("shows syncing state during operation", async () => {
    let resolveAdd!: () => void;
    const vm = new CartViewModel(createMockCartService({
      addItem: () => new Promise<void>(r => { resolveAdd = r; }),
    }));
    const product = { id: "1", title: "Widget", price: 10, inventory: 5 };

    const promise = vm.addItem(product);
    expect(vm.isSyncing).toBe(true);
    expect(vm.items).toHaveLength(1);

    resolveAdd();
    await promise;
    expect(vm.isSyncing).toBe(false);
  });
});
```

---

## Testing Strategy

### Core Principle

**Test the ViewModels, not the Views.** ViewModels contain ALL the logic, so that is where tests provide the most value. Because ViewModels have zero UI dependencies, they run in plain Vitest/Node — no browser, no DOM, no React test renderer needed.

### How to Write a ViewModel Test

Every test follows the same 3-step pattern:

1. **Arrange** — Create an instance of the ViewModel (and mock any external dependencies like API calls).
2. **Act** — Call one or more public methods on the ViewModel.
3. **Assert** — Check the ViewModel's public getters/properties for the expected state.

**Example:**

```typescript
// __tests__/ProductListViewModel.test.ts
import { describe, it, expect, beforeEach } from "vitest";
import { ProductListViewModel } from "../viewmodels/ProductListViewModel";
import type { ProductService } from "../viewmodels/ProductListViewModel";

// A reusable mock that implements the same interface the ViewModel expects
function createMockService(overrides: Partial<ProductService> = {}): ProductService {
  return {
    fetchProducts: async () => [],
    ...overrides,
  };
}

describe("ProductListViewModel", () => {
  it("starts in an empty, idle state", () => {
    const vm = new ProductListViewModel(createMockService());

    expect(vm.products).toEqual([]);
    expect(vm.isLoading).toBe(false);
    expect(vm.error).toBeNull();
  });

  it("loads products and sorts them by price", async () => {
    const vm = new ProductListViewModel(createMockService({
      fetchProducts: async () => [
        { id: "1", title: "Expensive", price: 99, inventory: 5 },
        { id: "2", title: "Cheap", price: 9, inventory: 3 },
      ],
    }));

    await vm.loadProducts();

    expect(vm.isLoading).toBe(false);
    expect(vm.products[0].title).toBe("Cheap");
    expect(vm.products[1].title).toBe("Expensive");
  });

  it("exposes an error message when the API fails", async () => {
    const vm = new ProductListViewModel(createMockService({
      fetchProducts: async () => { throw new Error("Network down"); },
    }));

    await vm.loadProducts();

    expect(vm.isLoading).toBe(false);
    expect(vm.error).toBe("Network down");
    expect(vm.products).toEqual([]);
  });

  it("filters products by max price", async () => {
    const vm = new ProductListViewModel(createMockService({
      fetchProducts: async () => [
        { id: "1", title: "A", price: 10, inventory: 1 },
        { id: "2", title: "B", price: 50, inventory: 1 },
        { id: "3", title: "C", price: 25, inventory: 1 },
      ],
    }));

    await vm.loadProducts();
    const affordable = vm.getAffordableProducts(30);

    expect(affordable).toHaveLength(2);
    expect(affordable.map(p => p.title)).toEqual(["A", "C"]);
  });
});
```

### What to Test (and What NOT to Test)


| DO Test                                  | DO NOT Test                                                 |
| ---------------------------------------- | ----------------------------------------------------------- |
| Initial/default state of the ViewModel   | How a React component renders                               |
| State after calling a public method      | CSS or styling                                              |
| Error states (API failure, validation)   | That a button exists in the DOM                             |
| Edge cases (empty lists, null values)    | Private methods directly (test them through public methods) |
| Computed/derived values (filters, sorts) | Third-party library internals                               |


### Rules for the AI

- **When you create a ViewModel, you MUST also create a test file for it.** No ViewModel ships without tests.
- **Mock external dependencies** (API calls, storage, timers) — never make real network requests in tests.
- **Test public surface only.** If you need to test a private method, that is a sign it should be extracted into its own utility or that you should test it indirectly through the public method that uses it.
- **One `describe` block per ViewModel, one `it` block per behavior.** Keep tests focused and readable.

### Test Coverage Targets

Aim for **≥90% line coverage on ViewModels.** ViewModels contain all business logic and are trivially testable — there is no reason for low coverage. Views should only be snapshot-tested or manually tested; do not chase coverage metrics on View components.

---

## File & Folder Structure

Every project using this architecture MUST use this folder layout. Do NOT invent your own structure.

```
src/
├── models/              # Layer 1: Data shapes only
│   ├── Product.ts
│   ├── Order.ts
│   └── User.ts
├── viewmodels/          # Layer 2: All business logic
│   ├── ProductListViewModel.ts
│   ├── OrderViewModel.ts
│   └── CartViewModel.ts
├── views/               # Layer 3: UI only (React components or CLI scripts)
│   ├── ProductList.tsx
│   ├── OrderPage.tsx
│   └── CartDrawer.tsx
├── services/            # External integrations (API clients, storage wrappers)
│   ├── api.ts
│   └── storage.ts
└── __tests__/           # Tests mirror the viewmodels/ folder
    ├── ProductListViewModel.test.ts
    ├── OrderViewModel.test.ts
    └── CartViewModel.test.ts
```

### Naming Rules


| Item            | Convention                             | Example                             |
| --------------- | -------------------------------------- | ----------------------------------- |
| Model file      | `PascalCase`, singular noun            | `Product.ts`, `User.ts`             |
| ViewModel file  | `PascalCase` + `ViewModel` suffix      | `ProductListViewModel.ts`           |
| ViewModel class | Same as file name                      | `class ProductListViewModel`        |
| View file       | `PascalCase`, matches what it renders  | `ProductList.tsx`, `CartDrawer.tsx` |
| Test file       | ViewModel name + `.test.ts`            | `ProductListViewModel.test.ts`      |
| Service file    | `camelCase`, describes the integration | `api.ts`, `storage.ts`              |


### What Goes in `services/`?

The `services/` folder is for thin wrappers around external systems (REST APIs, GraphQL, localStorage, etc.). A service file:

- Handles the raw HTTP call or storage access.
- Returns plain data (matching a Model type).
- Contains NO business logic, NO state, NO UI.

ViewModels call services. Views NEVER call services directly.

---

## Dependency Injection

ViewModels should receive their external dependencies (API services, storage, etc.) through the **constructor**, not by importing them directly. This makes testing simple and keeps ViewModels decoupled from specific implementations.

### Why This Matters

If a ViewModel imports `fetchProducts` directly at the top of the file, tests must use `vi.spyOn` or module mocking to intercept it. This is fragile. Instead, pass the dependency in:

**BAD — hard-coded import:**

```typescript
import { fetchProducts } from "../services/api";

class ProductListViewModel {
  async loadProducts() {
    this._products = await fetchProducts(); // tightly coupled
  }
}
```

**GOOD — injected dependency:**

```typescript
interface ProductService {
  fetchProducts(): Promise<Product[]>;
}

class ProductListViewModel {
  constructor(private productService: ProductService) {}

  async loadProducts() {
    this._isLoading = true;
    this._error = null;
    try {
      const data = await this.productService.fetchProducts();
      this._products = this.sortByPrice(data);
    } catch (e) {
      this._error = e instanceof Error ? e.message : "Failed to load";
    } finally {
      this._isLoading = false;
    }
  }
}
```

### How This Simplifies Tests

```typescript
it("loads products", async () => {
  const mockService: ProductService = {
    fetchProducts: async () => [
      { id: "1", title: "Widget", price: 10, inventory: 5 },
    ],
  };

  const vm = new ProductListViewModel(mockService);
  await vm.loadProducts();

  expect(vm.products).toHaveLength(1);
});
```

No `vi.spyOn`, no module mocking, no magic. The test controls the dependency explicitly.

### Rules

- Define an `interface` for each external dependency a ViewModel needs.
- Pass the concrete implementation through the constructor.
- In production, the View (or a factory/hook) creates the ViewModel with the real service.
- In tests, pass a mock/stub that implements the same interface.

---

## Real-Time / Reactive Data (e.g., Convex, Firebase, Supabase)

Real-time backends like Convex push live data updates to the client. This creates a question: **who subscribes?**

### Core Rule: The ViewModel NEVER subscribes directly.

The ViewModel does not know about Convex, Firebase, WebSockets, or any subscription mechanism. It remains a plain class that receives data and processes it.

### How It Works

The subscription lives in a thin **reactive adapter** — either a custom hook or a small wrapper component in the View layer. This adapter:

1. Subscribes to the real-time data source (e.g., Convex `useQuery`).
2. Pushes new data into the ViewModel whenever it changes.
3. The ViewModel processes the data (filtering, sorting, deriving state) and exposes results through its public getters.

### The Pattern

**Step 1: The ViewModel accepts data via a public method**

```typescript
// viewmodels/ProductListViewModel.ts
class ProductListViewModel {
  private _products: Product[] = [];
  private _error: string | null = null;

  get products(): Product[] { return this._products; }
  get affordableProducts(): Product[] {
    return this._products.filter(p => p.price < 50);
  }
  get error(): string | null { return this._error; }

  updateProducts(products: Product[]): void {
    this._products = this.sortByPrice(products);
    this._error = null;
  }

  setError(message: string): void {
    this._error = message;
  }

  private sortByPrice(items: Product[]): Product[] {
    return [...items].sort((a, b) => a.price - b.price);
  }
}
```

The ViewModel has no `loadProducts()` that fetches. Instead it has `updateProducts()` — a method that says "here is new data, process it."

**Step 2: The View subscribes and pushes data into the ViewModel**

```tsx
// views/ProductList.tsx
import { useQuery } from "convex/react";
import { api } from "../convex/_generated/api";

function ProductList() {
  const vm = useViewModel(ProductListViewModel);
  const products = useQuery(api.products.list);

  useEffect(() => {
    if (products) vm.updateProducts(products);
  }, [products]);

  if (vm.error) return <ErrorBanner message={vm.error} />;
  return (
    <ul>
      {vm.products.map(p => <li key={p.id}>{p.title}</li>)}
    </ul>
  );
}
```

Convex's `useQuery` re-runs automatically when backend data changes. Each time it does, the View pushes the fresh data into the ViewModel. The ViewModel sorts, filters, and derives — the View just renders.

> **Trade-off: extra render cycle.** This `useEffect` falls into the ["adjusting state when a prop changes"](https://react.dev/learn/you-might-not-need-an-effect#adjusting-some-state-when-a-prop-changes) category that the React docs generally discourage. When `products` changes, the component renders once with stale ViewModel state, then the Effect fires `vm.updateProducts()` → `notify()` → a second render with the correct state. This is a **deliberate trade-off** to keep the ViewModel as the single owner of processed state. If the extra render is unacceptable for your use case, you can call `vm.updateProducts()` directly during rendering (before the return statement) instead of in an Effect — but be aware this blurs the line between "View does no logic" and "View feeds data to the ViewModel."

### Why Not Subscribe Inside the ViewModel?


| If the ViewModel subscribes directly...                 | Problem                                  |
| ------------------------------------------------------- | ---------------------------------------- |
| It must import `ConvexClient` or similar                | Coupled to a specific real-time provider |
| It needs React hooks or event listeners                 | No longer a plain testable class         |
| Swapping Convex for Firebase means rewriting ViewModels | Violates dependency inversion            |


By keeping the subscription in the View (or a thin adapter hook), the ViewModel stays provider-agnostic. In tests, you just call `vm.updateProducts(mockData)` — no subscriptions to mock at all.

### Testing Real-Time ViewModels

```typescript
it("processes incoming real-time data", () => {
  const vm = new ProductListViewModel();

  vm.updateProducts([
    { id: "1", title: "Expensive", price: 99, inventory: 5 },
    { id: "2", title: "Cheap", price: 9, inventory: 3 },
  ]);

  expect(vm.products[0].title).toBe("Cheap");
  expect(vm.affordableProducts).toHaveLength(1);
});

it("handles multiple updates (simulating real-time pushes)", () => {
  const vm = new ProductListViewModel();

  vm.updateProducts([{ id: "1", title: "A", price: 10, inventory: 1 }]);
  expect(vm.products).toHaveLength(1);

  vm.updateProducts([
    { id: "1", title: "A", price: 10, inventory: 1 },
    { id: "2", title: "B", price: 20, inventory: 1 },
  ]);
  expect(vm.products).toHaveLength(2);
});
```

No Convex mock, no subscription setup, no async. The test just calls `updateProducts()` multiple times to simulate real-time pushes. This is the payoff of keeping subscriptions out of the ViewModel.

**Testing disposal mid-subscription:** Ensure the ViewModel behaves correctly if it receives updates after the View has unmounted (i.e., after all listeners have unsubscribed).

```typescript
it("handles updates after all listeners unsubscribe (disposal mid-subscription)", () => {
  const vm = new ProductListViewModel();
  const listener = vi.fn();

  const unsubscribe = vm.subscribe(listener);
  vm.updateProducts([{ id: "1", title: "A", price: 10, inventory: 1 }]);
  expect(listener).toHaveBeenCalledTimes(1);

  unsubscribe();

  // Simulate a late real-time push arriving after the View unmounted
  vm.updateProducts([{ id: "2", title: "B", price: 20, inventory: 1 }]);
  expect(listener).toHaveBeenCalledTimes(1); // no additional call
  expect(vm.products).toHaveLength(2); // state still updates internally
});
```

---

## React Binding: `BaseViewModel` and `useViewModel`

This section defines how ViewModel classes connect to React. This applies to **all** React usage (Next.js, Vite, etc.), not just one framework.

> **Prerequisite:** Before creating any ViewModel, ensure `BaseViewModel.ts` exists in `src/viewmodels/`. Every ViewModel extends this class. If you are starting a new project, create `BaseViewModel.ts` **first** — no other ViewModel will compile without it.

### The `BaseViewModel` Class

Every ViewModel extends `BaseViewModel`. This base class provides a simple subscription mechanism so React knows when to re-render.

```typescript
// viewmodels/BaseViewModel.ts
type Listener = () => void;

export abstract class BaseViewModel {
  private _listeners: Set<Listener> = new Set();
  private _version = 0;

  subscribe(listener: Listener): () => void {
    this._listeners.add(listener);
    return () => this._listeners.delete(listener);
  }

  getSnapshot(): number {
    return this._version;
  }

  protected notify(): void {
    this._version++;
    this._listeners.forEach(listener => listener());
  }
}
```

**How it works:**

- `subscribe` / `getSnapshot` — These are the two methods React's `useSyncExternalStore` requires. React calls `subscribe` to listen for changes and `getSnapshot` to detect if state has changed (via the version number).
- `notify()` — Every ViewModel calls `this.notify()` after mutating its own state. This bumps the version and triggers a React re-render.

**Rule:** In **async methods**, call `this.notify()` immediately after each state mutation (e.g., once after setting loading/clearing errors at the start, and once in the `finally` block). In **synchronous methods**, call `this.notify()` once at the end, after all mutations are complete. This avoids stale intermediate states in async flows while keeping sync methods simple.

**Updated ViewModel example:**

```typescript
// viewmodels/ProductListViewModel.ts
import { BaseViewModel } from "./BaseViewModel";

class ProductListViewModel extends BaseViewModel {
  private _products: Product[] = [];
  private _isLoading = false;
  private _error: string | null = null;

  constructor(private productService: ProductService) {
    super();
  }

  get products(): Product[] { return this._products; }
  get isLoading(): boolean { return this._isLoading; }
  get error(): string | null { return this._error; }

  async loadProducts(): Promise<void> {
    this._isLoading = true;
    this.notify();
    try {
      const data = await this.productService.fetchProducts();
      this._products = this.sortByPrice(data);
      this._error = null;
    } catch (e) {
      this._error = e instanceof Error ? e.message : "Failed to load";
    } finally {
      this._isLoading = false;
      this.notify();
    }
  }

  getAffordableProducts(maxPrice: number): Product[] {
    return this._products.filter(p => p.price <= maxPrice);
  }

  private sortByPrice(items: Product[]): Product[] {
    return [...items].sort((a, b) => a.price - b.price);
  }
}
```

### The `useViewModel` Hook

This lightweight hook connects any `BaseViewModel` subclass to a React component. It replaces heavy libraries like MobX or Zustand.

```typescript
// hooks/useViewModel.ts
import { useSyncExternalStore, useRef } from "react";
import type { BaseViewModel } from "@/viewmodels/BaseViewModel";

export function useViewModel<T extends BaseViewModel>(factory: () => T): T {
  const vmRef = useRef<T | null>(null);
  if (!vmRef.current) {
    vmRef.current = factory();
  }

  const vm = vmRef.current;

  useSyncExternalStore(
    (callback) => vm.subscribe(callback),
    () => vm.getSnapshot()
  );

  return vm;
}
```

**How to use it in a View:**

```tsx
const vm = useViewModel(() => new ProductListViewModel(realProductService));
```

The factory function runs once (on mount). React re-renders the component whenever the ViewModel calls `this.notify()`.

> **Alternative state-management libraries:** The `useViewModel` hook above is intentionally lightweight. If your project already uses MobX, Zustand, or another reactive library, only the **binding** changes — replace `useViewModel` with the equivalent `observer()` wrapper or Zustand store hook. The ViewModel class itself stays exactly the same because it has no dependency on the binding mechanism.

---

## Next.js App Router

Next.js introduces Server Components, which changes where data fetching happens. This section explains how the MVVM layers map to the App Router.

### How the Layers Map to Next.js


| MVVM Layer        | Next.js Equivalent                                | Location                               |
| ----------------- | ------------------------------------------------- | -------------------------------------- |
| **Model**         | Same — TypeScript interfaces                      | `src/models/`                          |
| **ViewModel**     | Same — plain classes (client-side only)           | `src/viewmodels/`                      |
| **View (server)** | Server Component — fetches data, passes props     | `app/**/page.tsx`, `app/**/layout.tsx` |
| **View (client)** | Client Component — binds to ViewModel             | `app/**/*Client.tsx` or `src/views/`   |
| **Service**       | Same — API wrappers, can also wrap Server Actions | `src/services/`                        |


### Next.js Folder Structure

```
app/                          # Next.js App Router (routing + Server Components)
├── products/
│   ├── page.tsx              # Server Component: fetches data, renders Client Component
│   └── ProductListClient.tsx # Client Component: binds to ViewModel
├── orders/
│   ├── page.tsx
│   └── OrderPageClient.tsx
└── layout.tsx
src/
├── models/                   # Layer 1: Data shapes
├── viewmodels/               # Layer 2: All business logic (includes BaseViewModel.ts)
├── services/                 # External integrations + Server Action wrappers
├── hooks/                    # useViewModel.ts and other shared hooks
└── __tests__/                # Tests mirror viewmodels/
```

### Server Components vs. Client Components

- **Server Components** (default in App Router) fetch data on the server and pass it as props. They are part of Layer 3 (View) but they do **NOT** instantiate ViewModels. They have no state, no interactivity.
- **Client Components** (marked with `"use client"`) instantiate the ViewModel, bind to it via `useViewModel`, and handle all interactivity.

**Note:** Server Components can import services directly (e.g., `import { fetchProducts } from "@/services/api"`). The dependency injection rule applies to **ViewModels**, not to Server Components. Server Components run on the server, are never unit-tested the same way, and don't need swappable dependencies.

**Example:**

```tsx
// app/products/page.tsx (Server Component)
import { ProductListClient } from "./ProductListClient";
import { fetchProducts } from "@/services/api";

export default async function ProductsPage() {
  const initialProducts = await fetchProducts();
  return <ProductListClient initialProducts={initialProducts} />;
}
```

```tsx
// app/products/ProductListClient.tsx (Client Component)
"use client";
import { useViewModel } from "@/hooks/useViewModel";
import { ProductListViewModel } from "@/viewmodels/ProductListViewModel";
import { productService } from "@/services/productService";

export function ProductListClient({ initialProducts }: { initialProducts: Product[] }) {
  const vm = useViewModel(() => {
    const instance = new ProductListViewModel(productService);
    instance.updateProducts(initialProducts);
    return instance;
  });

  return (
    <ul>
      {vm.products.map(p => <li key={p.id}>{p.title}</li>)}
    </ul>
  );
}
```

The Server Component fetches data (no spinner needed — it's server-rendered). The Client Component creates the ViewModel with an injected service and seeds it with the server-fetched data. From this point, the ViewModel owns all state and logic.

### Server Actions

Next.js Server Actions handle mutations (form submissions, database writes).

**Rule:** Wrap Server Actions behind a **Service interface**, just like any other external dependency.

```typescript
// services/productService.ts
"use server";
import type { ProductService } from "@/viewmodels/ProductListViewModel";

async function createProductAction(data: NewProduct): Promise<Product> {
  // Server Action: runs on the server
  return await db.products.create(data);
}

export const productService: ProductService = {
  fetchProducts: async () => { /* ... */ },
  createProduct: createProductAction,
};
```

The ViewModel calls `this.productService.createProduct(data)`. It has no idea a Server Action is involved. In tests, you pass a mock service instead.

---

## ViewModel-to-ViewModel Communication

Sometimes one ViewModel needs data or state from another (e.g., `CartViewModel` needs the product list from `ProductListViewModel`). There are strict rules for how this works.

### Rule: ViewModels NEVER import other ViewModels directly.

A ViewModel must not instantiate or hold a reference to another ViewModel. This creates tight coupling and makes both harder to test.

### Allowed Patterns (pick one per situation)

**Pattern 1: Shared Service**
Both ViewModels depend on the same service. The service holds shared state or data.

```typescript
interface ProductService {
  fetchProducts(): Promise<Product[]>;
  getProductById(id: string): Product | undefined;
}

class ProductListViewModel {
  constructor(private productService: ProductService) {}
  // uses productService.fetchProducts()
}

class CartViewModel {
  constructor(private productService: ProductService) {}
  // uses productService.getProductById()
}
```

Both ViewModels receive the same `ProductService` instance. They share data through the service without knowing about each other.

**Pattern 2: Pass Data Down via the View**
The View reads from one ViewModel and passes the result as a method argument to another.

```tsx
function CheckoutPage() {
  const productVm = useViewModel(ProductListViewModel);
  const cartVm = useViewModel(CartViewModel);

  const handleAddToCart = (productId: string) => {
    const product = productVm.getProductById(productId);
    if (product) cartVm.addItem(product);
  };

  return <ProductGrid onAdd={handleAddToCart} />;
}
```

This keeps both ViewModels independent. The View acts as the "glue."

**Pattern 3: Event Bus (for complex cases only)**
If many ViewModels need to react to the same event, use a simple typed event emitter. This is the most decoupled option but adds indirection — only use it when Patterns 1 and 2 are insufficient.

### What is NEVER allowed


| Violation                                                                        | Why it's wrong                                  |
| -------------------------------------------------------------------------------- | ----------------------------------------------- |
| `import { CartViewModel } from "../viewmodels/CartViewModel"` inside a ViewModel | Direct coupling between ViewModels              |
| One ViewModel calling another ViewModel's methods                                | Creates hidden dependencies, breaks testability |
| Storing a ViewModel instance inside another ViewModel                            | Ownership/lifecycle confusion                   |


### Avoiding Circular Service Dependencies

The same principle applies to services. If Service A depends on Service B and Service B depends on Service A, you have a circular dependency that will cause import failures or tangled logic. **Refactor into a third service that both depend on.** Extract the shared concern into a new service (e.g., `SharedProductCacheService`) and have both original services depend on it instead of each other.

---

## Common Anti-Patterns

These are mistakes that violate the architecture. If you (the AI) catch yourself doing any of these, **STOP and refactor immediately.**

### Anti-Pattern 1: Logic in the View

**BAD:**

```tsx
function ProductList({ products }) {
  const sorted = products.sort((a, b) => a.price - b.price);
  const affordable = sorted.filter(p => p.price < 50);
  const total = affordable.reduce((sum, p) => sum + p.price, 0);

  return <div>Total: {total}</div>;
}
```

**WHY:** Sorting, filtering, and calculating belong in the ViewModel. The View should receive the final result.

**FIX:** Move all three operations into the ViewModel. The View just renders `vm.affordableTotal`.

---

### Anti-Pattern 2: UI Code in the ViewModel

**BAD:**

```typescript
class ProductListViewModel {
  getProductLabel(product: Product): JSX.Element {
    return <span className="label">{product.title}</span>;
  }
}
```

**WHY:** The ViewModel must NEVER return JSX, HTML, or any UI-specific type. It returns plain data; the View decides how to render it.

**FIX:** Return a plain string or object. Let the View wrap it in JSX.

---

### Anti-Pattern 3: Direct API Calls in the View

**BAD:**

```tsx
function ProductList() {
  const [products, setProducts] = useState([]);

  useEffect(() => {
    fetch("/api/products")
      .then(res => res.json())
      .then(data => setProducts(data));
  }, []);

  return <ul>{products.map(p => <li>{p.title}</li>)}</ul>;
}
```

**WHY:** The View is now doing fetching, state management, and rendering. It should only render.

**FIX:** Move `fetch` and `useState` logic into a ViewModel. The View calls `vm.loadProducts()` and reads `vm.products`.

---

### Anti-Pattern 4: God ViewModel

**BAD:**

```typescript
class AppViewModel {
  // handles products, cart, user auth, orders, settings...
  private _products: Product[] = [];
  private _cartItems: CartItem[] = [];
  private _user: User | null = null;
  private _orders: Order[] = [];
  private _settings: Settings = defaultSettings;
  // ... 500 lines of mixed concerns
}
```

**WHY:** One ViewModel per feature/screen. A single ViewModel handling everything defeats the purpose of separation.

**FIX:** Split into `ProductListViewModel`, `CartViewModel`, `AuthViewModel`, `OrderViewModel`, `SettingsViewModel`.

---

### Anti-Pattern 5: Skipping the Interface for Dependencies

**BAD:**

```typescript
import { fetchProducts } from "../services/api";

class ProductListViewModel {
  async load() { this._products = await fetchProducts(); }
}
```

**WHY:** Tight coupling to a concrete implementation. Tests require module-level mocking hacks.

**FIX:** Define a `ProductService` interface, inject it via the constructor (see Dependency Injection section above).

---

## Performance: Memoizing Expensive Computed Properties

Public getters on a ViewModel run every time React re-renders. For simple filters or lookups, this is fine. For expensive computations (large lists, complex aggregations), cache the result and only recompute when the underlying data changes.

### The Pattern

```typescript
class ProductListViewModel extends BaseViewModel {
  private _products: Product[] = [];
  private _affordableCache: Product[] | null = null;

  get products(): Product[] { return this._products; }

  get affordableProducts(): Product[] {
    if (this._affordableCache === null) {
      this._affordableCache = this._products.filter(p => p.price < 50);
    }
    return this._affordableCache;
  }

  updateProducts(products: Product[]): void {
    this._products = this.sortByPrice(products);
    this._affordableCache = null; // invalidate cache
    this.notify();
  }

  private sortByPrice(items: Product[]): Product[] {
    return [...items].sort((a, b) => a.price - b.price);
  }
}
```

### Rules for Memoization

- **Only memoize when profiling shows a real cost.** Don't prematurely cache trivial getters.
- **Invalidate the cache in every method that changes the source data** — set the cache to `null` before calling `notify()`.
- **Keep the cache private.** The public getter is the only way to access the result; consumers don't know about the cache.
- **Prefer this manual invalidation over third-party memoization libraries** to keep ViewModels dependency-free.

---

## Quick Decision Checklist

When writing code, ask yourself:


| Question                                         | Answer                                                                   |
| ------------------------------------------------ | ------------------------------------------------------------------------ |
| "Where does this `interface` / `type` go?"       | **Layer 1 — Model** in `src/models/`                                     |
| "Where does this logic / fetch / validation go?" | **Layer 2 — ViewModel class** in `src/viewmodels/`                       |
| "Where does this JSX / console.log go?"          | **Layer 3 — View** in `src/views/`                                       |
| "Where does this HTTP call / storage access go?" | **Service** in `src/services/`                                           |
| "My React component is getting complex logic."   | **STOP. Move that logic into the ViewModel.**                            |
| "My ViewModel imports React or returns JSX."     | **STOP. That is a violation. Remove it.**                                |
| "My ViewModel imports another ViewModel."        | **STOP. Use a shared service or pass data via the View.**                |
| "My ViewModel calls `fetch` directly."           | **STOP. Inject a service interface via the constructor.**                |
| "My View calls an API directly."                 | **STOP. The View calls the ViewModel, the ViewModel calls the service.** |
| "I'm putting everything in one big ViewModel."   | **STOP. One ViewModel per feature/screen.**                              |


---

## Linter / ESLint Rules

Architecture rules are only effective if they are enforced. Use ESLint to catch violations at compile time, not code review. This section provides copy-paste ready configuration.

### Core Rules

These rules prevent the most common architecture violations:


| Rule                    | What It Enforces           | Violation Example                                 |
| ----------------------- | -------------------------- | ------------------------------------------------- |
| `no-restricted-imports` | No React in ViewModels     | `import { useState } from "react"` in a ViewModel |
| `no-restricted-imports` | No DOM APIs in ViewModels  | `document.getElementById()` in a ViewModel        |
| `no-restricted-imports` | No services in Views       | `import { api } from "@/services/api"` in a View  |
| `no-restricted-imports` | No cross-ViewModel imports | `import { CartViewModel }` inside a ViewModel     |
| `no-restricted-imports` | No direct HTTP clients     | `import axios from "axios"` in a View             |
| Custom rule or naming   | Models contain only types  | `class Product {}` in `models/Product.ts`         |


### Complete ESLint Configuration

Add this to your `.eslintrc.js` or `eslint.config.js`:

```js
// .eslintrc.js (legacy format)
module.exports = {
  overrides: [
    // --- ViewModels: No UI dependencies, no concrete service imports ---
    // NOTE: ViewModels define their own dependency interfaces (e.g., ProductService).
    // This ban prevents importing concrete implementations from services/.
    // Interface definitions should live in the ViewModel file or in models/.
    {
      files: ["src/viewmodels/**/*.ts"],
      rules: {
        "no-restricted-imports": ["error", {
          patterns: [
            "react", "react-dom", "react-*", "@react-*",
            "**/views/**",
            "**/services/**",
          ],
          paths: [
            { name: "react", message: "ViewModels must not import React. Move UI logic to the View." },
            { name: "react-dom", message: "ViewModels must not import ReactDOM." },
          ]
        }]
      }
    },

    // --- Views: No direct HTTP clients ---
    // NOTE: Views CAN import from services/ to pass concrete service instances
    // to ViewModels via the factory (e.g., `new ProductListViewModel(productService)`).
    // But Views must NOT use HTTP clients directly — all data access goes through ViewModels.
    {
      files: ["src/views/**/*.tsx", "src/views/**/*.ts", "app/**/*.tsx", "app/**/*.ts"],
      rules: {
        "no-restricted-imports": ["error", {
          patterns: ["axios", "node-fetch", "cross-fetch", "got", "ky", "superagent"],
          paths: [
            { name: "axios", message: "Views must not call APIs directly. Use a ViewModel." },
            { name: "node-fetch", message: "Views must not call APIs directly. Use a ViewModel." },
          ]
        }]
      }
    },

    // --- Models: Only type definitions ---
    {
      files: ["src/models/**/*.ts"],
      rules: {
        "no-restricted-syntax": ["error",
          { selector: "ClassDeclaration", message: "Model files must contain only `interface` or `type` definitions. No classes, no logic." },
          { selector: "FunctionDeclaration", message: "Model files must not contain functions. Logic belongs in ViewModels." }
        ],
        "no-var": "error",
      }
    },

    // --- ViewModels must not import other ViewModels ---
    // Scoped to viewmodels/ only. Views and other layers CAN import ViewModels.
    {
      files: ["src/viewmodels/**/*.ts"],
      rules: {
        "no-restricted-imports": ["error", {
          patterns: [
            {
              group: ["./[A-Z]*ViewModel*", "../viewmodels/*"],
              message: "ViewModels must not import other ViewModels. Use a shared service or pass data via the View."
            }
          ]
        }]
      }
    }
  ]
};
```

### Flat Config Format (ESLint 9+)

If using the new flat config format:

```js
// eslint.config.js
import js from "@eslint/js";
import ts from "typescript-eslint";

export default ts.config(
  js.configs.recommended,
  ...ts.configs.recommended,
  // ViewModels: no React, no Views, no concrete services
  {
    files: ["src/viewmodels/**/*.ts"],
    rules: {
      "no-restricted-imports": ["error", {
        patterns: [
          "react", "react-dom", "react-*", "@react-*",
          "**/views/**", "**/services/**",
          { group: ["./[A-Z]*ViewModel*", "../viewmodels/*"], message: "ViewModels must not import other ViewModels." }
        ],
      }]
    }
  },
  // Views: no direct HTTP clients (but CAN import services to pass to ViewModels)
  {
    files: ["src/views/**/*.tsx", "src/views/**/*.ts", "app/**/*.tsx", "app/**/*.ts"],
    rules: {
      "no-restricted-imports": ["error", {
        patterns: ["axios", "node-fetch", "cross-fetch", "got", "ky", "superagent"]
      }]
    }
  },
  // Models: only interfaces and types
  {
    files: ["src/models/**/*.ts"],
    rules: {
      "no-restricted-syntax": ["error",
        { selector: "ClassDeclaration", message: "Model files must contain only `interface` or `type`." },
        { selector: "FunctionDeclaration", message: "Model files must not contain functions." }
      ]
    }
  }
);
```

### Additional Recommended Rules

Beyond architecture enforcement, these rules improve code quality:


| Rule                                               | Purpose                                           |
| -------------------------------------------------- | ------------------------------------------------- |
| `@typescript-eslint/explicit-function-return-type` | Enforces return types on public ViewModel methods |
| `@typescript-eslint/no-explicit-any`               | Prevents `any` in Models and ViewModels           |
| `@typescript-eslint/no-floating-promises`          | Catches unhandled async calls in ViewModels       |
| `prefer-const`                                     | Enforces immutability patterns                    |


```js
// Add to overrides for viewmodels/
{
  files: ["src/viewmodels/**/*.ts"],
  rules: {
    "@typescript-eslint/explicit-function-return-type": ["warn", {
      allowExpressions: true,
      allowTypedFunctionExpressions: true,
    }],
    "@typescript-eslint/no-floating-promises": "error",
  }
}
```

### Pre-Commit Hook

To ensure linting runs before every commit, add a husky hook:

```json
// package.json
{
  "scripts": {
    "lint": "eslint src --ext .ts,.tsx",
    "lint:fix": "eslint src --ext .ts,.tsx --fix"
  },
  "lint-staged": {
    "*.{ts,tsx}": ["eslint --fix", "prettier --write"]
  }
}
```

```bash
# .husky/pre-commit
npm run lint
```

### CI Enforcement

In your CI pipeline (GitHub Actions, GitLab CI, etc.), fail the build on lint errors:

```yaml
# .github/workflows/ci.yml
- name: Lint
  run: npm run lint
```

### Rules for the AI

- **When creating a new project, ALWAYS include the ESLint configuration above.**
- **When adding a ViewModel, verify no banned imports are used.**
- **When adding a Model, ensure it contains only `interface` or `type` definitions.**
- **If a lint error occurs, fix it immediately — do not disable the rule.**

---

## Why We Do This

The goal is **separation of concerns**, and this architecture is specifically designed to work well with AI-assisted development.

### For Code Quality

The ViewModel (the "brain") holds all logic and is easy to test. The View (the "screen") only displays and forwards actions. We can test the logic without needing a browser, and swap the entire UI without touching business logic.

### For AI Context Efficiency

Each ViewModel is a **self-contained unit**. When you (the AI) need to work on a feature, you only need to load:

- The ViewModel class for that feature
- The Model interfaces it uses
- The Service interfaces it depends on

You do NOT need the entire codebase in context. A single ViewModel file contains everything needed to understand, modify, and test one feature. This means:

- **Smaller context window** — Less code loaded, fewer tokens used, more room for reasoning.
- **Focused changes** — A task like "add filtering to products" touches exactly one file: `ProductListViewModel.ts`.
- **Isolated testing** — You can write and run tests for one ViewModel without knowing anything about the rest of the app.
- **Parallel work** — Two separate AI calls can work on `CartViewModel` and `OrderViewModel` simultaneously without conflicts, because ViewModels never import each other.

This is the core reason every rule in this document exists: **keep each unit small, independent, and fully testable in isolation.**