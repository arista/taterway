# TypeScript Dependency Provisioning - Analysis

## Summary of the Pattern

The document describes a dependency injection pattern where:
1. Components declare their dependencies via a typed `Deps` (or `Ctx`) object
2. A top-level provisioner (App class) assembles and wires dependencies
3. When components call other components, they receive "curried" versions with deps pre-applied
4. Props (caller-provided) and Deps (provisioner-provided) can be merged into a single Context

## Strengths

**Explicit Dependencies**
- All dependencies are visible in the type signature
- No hidden coupling to globals, singletons, or service locators
- Makes code easier to reason about

**Testability**
- Mocking is trivial - just pass different implementations in the Deps object
- No need for test frameworks to intercept constructors or mock modules
- Unit tests are truly isolated

**Type Safety**
- TypeScript enforces that all required dependencies are provided
- Refactoring is safer - changing a dependency interface causes compile errors at all usage sites

**Flexibility**
- Different instances of the same component can receive different dependency configurations
- The consumer decides what to provide vs what the provisioner supplies

## Concerns and Trade-offs

**Boilerplate**
- Every component needs Props/Deps/Ctx interfaces defined
- Every callable component needs a curried interface type (IUploadDirectory, CreatePeriodicUploader)
- The App class grows with every component, wiring each one manually

**Cognitive Load**
- Developers must understand the props/deps/ctx distinction
- The currying pattern adds indirection

**Scaling the Provisioner**
- The document acknowledges this: "Much of the complexity...concentrates into the top-level provisioning class"
- For large applications, the App constructor becomes a massive wiring blob
- Could be mitigated by splitting into sub-provisioners, but that's not addressed

## Open Question: Where Should Curried Types Live?

The document presents three options:

1. **With the defining component** - PeriodicUploader defines PeriodicUploaderProps
2. **With the App** - Provisioner defines the split
3. **With the consumer** - Consumer declares what it will provide

The author leans toward option 3. My thoughts:

**Pros of consumer-defined:**
- Maximum flexibility - different consumers can use different subsets
- No need to guess the "right" props/deps split upfront
- Follows the "consumer knows best" principle

**Cons of consumer-defined:**
- Duplication if many consumers want the same split
- The component author has no way to suggest a "default" split
- Interfaces are scattered across the codebase

**Alternative: Hybrid approach**
- Component defines a *suggested* Props type as documentation
- Consumers can use it, extend it, or define their own
- The provisioner adapts to whatever the consumer declared

## The Ctx Unification

Merging Props and Deps into a single Ctx is pragmatic. The distinction often *is* arbitrary:
- Is `period` a parameter or a configuration dependency?
- Is `dirname` passed by the caller or injected from config?

However, some value exists in the distinction:
- Props tend to vary per-call; Deps tend to be stable across calls
- Props are domain-specific; Deps are infrastructure
- The distinction helps newcomers understand the component's contract

**Suggestion:** Keep both available. Components can define:
```typescript
interface UploadDirectoryCtx {
  // Everything the component needs
}

// Optional: suggested split for common use cases
type UploadDirectoryProps = Pick<UploadDirectoryCtx, 'dirname' | 'dest'>
type UploadDirectoryDeps = Omit<UploadDirectoryCtx, keyof UploadDirectoryProps>
```

## Circular Dependencies and Lazy Instantiation

The document's solution (getter-based lazy instantiation) works but has drawbacks:
- Every property access becomes a function call
- The `_scheduler ||= ...` pattern is error-prone (can't be readonly)
- Harder to reason about initialization order

**Alternative considerations:**
- Could circular dependencies indicate a design smell to be refactored out?
- If truly needed, a two-phase init (construct, then wire) is more explicit
- Consider whether the circular dep is really needed at construction time, or just at runtime

## The Startable Pattern

The start/stop lifecycle pattern is clean. Observations:

- Returning the stop function from start() is elegant (keeps related logic together)
- The `onStops.unshift` ensures reverse-order shutdown
- Missing: error handling during start (what if one service fails to start?)
- Missing: async start/stop (many services need async initialization)

**Suggested enhancement:**
```typescript
interface Startable {
  start(): Promise<OnStop | void>
}

type OnStop = () => Promise<void> | void
```

## Questions for Further Exploration

1. **How does this compare to existing DI frameworks?** (tsyringe, inversify, etc.)
   - Those use decorators and reflection
   - This pattern is more explicit but more verbose

2. **How to handle optional dependencies?**
   - Deps properties could be optional (`scheduler?: IScheduler`)
   - But then every usage site needs null checks

3. **How to handle configuration?**
   - Is config just another dependency?
   - Or does it get special treatment (passed to provisioner, distributed to components)?

4. **How to organize a large provisioner?**
   - Split by domain? (AuthProvisioner, SchedulingProvisioner)
   - How do sub-provisioners access each other's components?

5. **Runtime vs compile-time guarantees?**
   - The lazy getter pattern defers wiring to runtime
   - Circular dependencies might cause runtime errors on first access
   - Is there a way to verify the dependency graph at compile time?

## Overall Assessment

This is a thoughtful, principled approach to dependency injection that prioritizes explicitness and testability over convenience. It's well-suited for:
- Applications where testability is paramount
- Teams that prefer explicit wiring over "magic"
- Codebases where the dependency graph is relatively simple

It may struggle with:
- Very large applications (boilerplate and provisioner complexity)
- Teams unfamiliar with functional patterns (currying)
- Rapid prototyping (the ceremony can slow iteration)

The pattern could benefit from:
- Helper utilities to reduce boilerplate
- Guidelines for organizing large provisioners
- A clear story for async initialization and error handling
