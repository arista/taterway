# TypeScript Dependency Injection - Analysis

## Overview

This document presents a well-structured, framework-free approach to DI in TypeScript. It uses standard terminology (Composition Root, Assisted Injection) and builds progressively from simple patterns to more sophisticated techniques.

## What Works Well

**Clear Mental Model**
The Context pattern (combining props + deps into a single typed object) is clean. Components don't care where values come from - they just declare what they need. The distinction between "what callers provide" vs "what the system provides" is handled at the wiring layer, not in the component itself.

**Standard Terminology**
Using "Composition Root" and "Assisted Injection" makes this document accessible to developers familiar with DI literature. The earlier "provisioning" framing was less recognizable.

**Progressive Complexity**
The document builds from simple direct instantiation → Create on Demand → Reference on Demand → Service Lifecycles. Each layer addresses a real problem that emerges as applications grow.

**The Assisted Injection Section**
This is the heart of the document. The pattern of splitting `Ctx = Props & Deps` and exposing a function that only requires `Props` is elegant. The example progression from `uploadDirectory` to `IUploadDirectory` is clear.

## Dependencies Shortcut with Props (Resolved)

The document now includes the `withProps` helper using `Object.create()` - this is the clean solution that doesn't mutate and preserves lazy getters through the prototype chain.

## Observations on Create on Demand

The `createOnDemand` implementation is solid, including circular dependency detection. One note: the `creating` flag should probably be thread-local in environments with async initialization, but for synchronous-only construction this works.

The pattern:
```typescript
somethingFn = createOnDemand(() => new Something({...}))
get something() { return this.somethingFn() }
```

is verbose. Consider a decorator or helper that combines both:

```typescript
@lazy get something() { return new Something({...}) }
```

Though this requires decorator support, which may conflict with the "no framework" goal.

## Reference on Demand - The `self` Pattern

The document uses:
```typescript
class App {
  self = this
  uploadDirectory: IUploadDirectory = (props) => uploadDirectory({
    ...props,
    get directoryService() { return self.directoryService },
  })
}
```

This works but is easy to forget. The `self = this` at the class level is unusual and may confuse readers.

**Alternative: Arrow function getters aren't possible, but you could use a factory:**

```typescript
const lazyRef = <T>(getter: () => T) => ({ get value() { return getter() } })

// Then:
uploadDirectory: IUploadDirectory = (props) => uploadDirectory({
  ...props,
  ...lazyRef(() => this.directoryService),  // Hmm, doesn't quite work
})
```

Actually, the `self = this` pattern is probably the simplest. Just document it clearly.

## Service Lifecycles

The start/stop pattern is good. Returning the stop function from start() keeps related logic together.

Missing pieces worth considering:
- **Async start/stop** - Many services need async initialization (database connections, etc.)
- **Error handling** - What happens if one service fails to start?
- **Dependencies between startables** - Service A needs Service B to be started first

A more complete interface:
```typescript
interface Startable {
  start(): Promise<StopFn | void>
}
type StopFn = () => Promise<void> | void

// In App:
async startAll() {
  for (const service of this.startables) {
    const stop = await service.start()
    if (stop) this.stops.unshift(stop)
  }
}

async stopAll() {
  for (const stop of this.stops) {
    await stop()
  }
}
```

## Minor Issues

1. ~~**ArrayBuffer typo**~~ - Fixed

2. **Missing: Optional dependencies** - What if a component can work with or without a logger? The pattern doesn't address optional deps.

## Questions This Raises

1. ~~**How does this scale?**~~ - Now addressed in "Composition Root delegation" section.

2. **Testing the App itself** - The App is hard to unit test since it wires everything. Is integration testing the only option?

3. **Dynamic/runtime dependency selection** - What if the choice of implementation depends on runtime config? The current pattern assumes static wiring.

## Comparison to Frameworks

| Aspect | This Pattern | InversifyJS/tsyringe |
|--------|-------------|---------------------|
| Boilerplate | Higher (manual wiring) | Lower (decorators) |
| Type safety | Full | Partial (runtime tokens) |
| Transparency | High (all wiring visible) | Lower (container magic) |
| Learning curve | DI concepts only | DI + framework API |
| Bundle size | Zero | Framework overhead |

## Overall

This is a mature, well-reasoned document. The pattern is sound and the terminology is correct.

**Resolved:**
- ~~FIXME with `withProps` helper~~ - Now included
- ~~ArrayBuffer typo~~ - Fixed
- ~~Scaling the Composition Root~~ - New "Composition Root delegation" section addresses this

**Remaining:**
- Async initialization - The Service Lifecycles section could benefit from showing async start/stop and error handling

The pattern is best suited for small-to-medium applications where the team values explicitness over convenience.
