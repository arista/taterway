# TypeScript Dependency Injection

The dependency injection pattern described here is fully typesafe, requires no external framework, and can be achieved with minimal support code.  The goal is for application components to be written with the expectation that all their dependencies will be passed to them.  Components don't care how the dependencies are provided, and shouldn't need to "reach out" to global or external values to satisfy those dependencies.

For most applications, those "components" will fall into two categories:

* Class instances (sometimes called "services"), which can expect all of their dependencies to be provided as constructor args
* Function calls, which can expect all of their dependencies to be passed as arguments to the function call

## Contexts

For class instances and function calls, the pattern expects that the component receives a single object, which we call the "Context".  The Context contains everything the component needs - those values that we would normally think of as "arguments" or "props", along with those values that we would consider "dependencies".

For example, consider an `uploadDirectory` function:

```
type UploadDirectoryCtx = {
  dirname: string
  dest: string
  directoryService: IDirectoryService
  uploadService: IUploadService
}

function uploadDirectory(ctx: UploadDirectoryCtx): void {
  ctx.directoryService.readDir(ctx.dirname)
  ...
  ctx.uploadService.writeFile(...)
}
```
This is an example of a function written to work with dependency injection.  It takes a couple arguments that would naturally be expected, like `dirname` and `dest`.  But it also declares that it needs interfaces to an `IDirectoryService` and an `IUploadService`, which it uses rather than making direct calls to some global service or library.

Similarly, one of those dependencies might itself be a component designed to work with dependency injection.  Consider, for example, an S3 implementation of the `IUploadService`:

```
interface IUploadService {
  writeFile(filename: string, bytes: ArrayBuffer)
}

type S3UploadServiceCtx = {
  bucket: string
  prefix: string
  client: S3Client
}

class S3UploadService implements IUploadService {
  constructor(public ctx: S3UploadServiceCtx) {}
  
  writeFile(filename: string, bytes: ArrayBuffer) {
    const {bucket, prefix, client} = this.ctx
    client...
  }
}
```

Here, the component's Context is passed in to its constructor and stored for use by its methods, such as `writeFile`.  Again, this component takes a couple properties that would typically be expected, such as `bucket` and `prefix`, but also expects an `S3Client` to be passed in.

Once instantiated, this component could then be passed to the `uploadDirectory` function to satisfy its `uploadService` requirement.

## Composition Root

Components should therefore be written with the expectation that all of their needs will be provided.  This means that *somewhere* in the program, there's something that's meeting those needs, providing the components with their declared dependencies.  This role is sometimes called the **Composition Root**.  This is the part of the program that constructs concrete instances of components, "wiring" up their dependencies to point to other instances.  It is typically a central class created very close to the program's entry point - such as an `App` class.

A simple `App` class might look like this:

```
class App {
  s3Client = new aws.S3Client({...})
  uploadService = new S3UploadService({
    bucket: "our bucket",
    prefix: "our prefix",
    client: this.s3Client,
  })
  directoryService = new ...
  
  constructor(public ctx: AppCtx) {}
}

type AppCtx = {
  config: ...
}
```
On construction, the App class will create the required instances, wiring them up to point to each other as needed.  The program can then access those instances as needed.

Note that the `App` class here has its own Context, which for the root level might be provided by the programs inputs, such as command line arguments, environment, etc.

The Composition Root is the only area of the program that has knowledge of all the concrete implementations, and how they are wired together.  For smaller programs, this might be a single `App` class.  For more complex programs, this responsibility might end up delegated to multiple classes.  Each of those delegated classes would likely have its own Context connecting to central services kept in the root class.

## Factories and Assisted Injection

While the Composition Root (the `App`) can provide pre-created and pre-wired instances, it can also expose functions and factory methods that "fill in" required dependencies.

For example, consider the `IUploadService`, whose `S3UploadService` implementation requires a `bucket`, `prefix`, and `client`.  It's possible that an application might wish to create several such `IUploadService` instances, supplying only `bucket`, and `prefix`, and having the `client` be filled in automatically.  In other words, it might want to call something like the `S3UploadServiceFactory` shown here:

```
type S3UploadServiceFactory = (props: S3UploadServiceProps)=>IUploadService
type S3UploadServiceProps = {
  bucket: string
  prefix: string
}
```

The `App` can do this with an "Assisted Injection" pattern, filling in the dependencies not provided by the caller:

```
class App {
  ...
  s3UploadServiceFactory: S3UploadServiceFactory = (props) => new S3UploadService({
    ...props,
    client: this.s3Client,
  })
}
```

This pattern works best when it's fairly clear to users what would be considered "props" that are their responsibility, vs. "dependencies" that are the responsibility of the Composition Root.  In many cases this separation can be defined right with the component if it has a pretty good idea how it will be used:

```
type S3UploadServiceProps = {
  bucket: string
  prefix: string
}

type S3UploadServiceCtx = S3UploadServiceProps & {
  client: S3Client
}

class S3UploadService implements IUploadService {
  constructor(public ctx: S3UploadServiceCtx) {}
}
```
The "Assisted Injection" pattern can also be applied to functions that require dependencies, such as the `uploadDirectory` function.  First, we need to rewrite the definitions around `uploadDirectory` to split out the "props":

```
type UploadDirectoryProps = {
  dirname: string
  dest: string
}
type UploadDirectoryCtx = UploadDirectoryProps & {
  directoryService: IDirectoryService
  uploadService: IUploadService
}

type IUploadDirectory = (props: UploadDirectoryProps) => void

function uploadDirectory(ctx: UploadDirectoryCtx): void {
  ctx.directoryService.readDir(ctx.dirname)
  ...
  ctx.uploadService.writeFile(...)
}
```

The `uploadDirectory` function remains the same - we've just split out the props from the other dependencies.  That allows us to expose `uploadDirectory` on `App`:

```
class App {
  ...
  uploadDirectory: IUploadDirectory = (props) => uploadDirectory({
    ...props,
    directoryService: this.directoryService,
    uploadService: this.uploadService,
  })
}
```
## Assisted Injection and dependencies

The "Assisted Injection" pattern above can be used by components to invoke functions or create instances that require dependencies that they can't supply.  For example, consider a component that wants to call `uploadDirectory`:

```
function uploadDirectories(ctx: UploadDirectoriesCtx): void {
  ctx.uploadDirectory({dirname: "dir1", dest: "/dest/dir1"})
  ctx.uploadDirectory({dirname: "dir2", dest: "/dest/dir2"})
  ctx.uploadDirectory({dirname: "dir3", dest: "/dest/dir3"})
}
```
Here the `uploadDirectories` function is calling `uploadDirectory`, but isn't passing the dependencies that the function requires.  Instead, it's expecting a form of `uploadDirectory` that doesn't require those dependencies:
    
```
type UploadDirectoriesCtx = {
  uploadDirectory: IUploadDirectory
}
```
and the `App` can fill those in:

```
class App {
  uploadDirectories = ()=> {
    uploadDirectories({
      uploadDirectory: this.uploadDirectory
    })
  }
}
```
A similar pattern can be followed for components that want to create new instances of some class, such as the `S3UploadServiceFactory`.  A component can declare an `S3UploadServiceFactory` as a dependency, and expect the `App` to fill it in.

## Composition Root implementations

As described previously the simplest form of a Composition Root just instantiates all components when constructed, wiring them up as appropriate.  This can work for smaller programs, but becomes more expensive and brittle as programs become more complex.  The order of initialization needs to be precise, and references between components won't work.

### Create on Demand

A more robust approach is to avoid creating any instances up front, and instead declare the instances to be created on demand.  Imagine a `createOnDemand` function that takes a constructor function and runs it once the first time it's needed, storing its result and returning it on subsequent calls.  The `App` class might be rewritten to use it like this:

```
class App {
  s3ClientFn = createOnDemand(()=>new aws.S3Client({...}))
  get s3Client() { return this.s3ClientFn() }

  uploadServiceFn = createOnDemand(()=>new S3UploadService({
    bucket: "our bucket",
    prefix: "our prefix",
    client: this.s3Client,
  }))
  get uploadService() { return this.uploadServiceFn() }

  ...
}

type AppCtx = {
  config: ...
}
```

It's a little more verbose, in that each "service" needs a `createOnDemand` wrapper, and a getter that calls it.  But using this pattern removes the need to initialize the services in a particular order, and also opens the door (although not quite) to enabling services to reference each other.

Here's a possible implementation of `createOnDemand`:

```
function createOnDemand<T>(f: ()=>T): ()=>T {
  var created = false
  var creating = false
  var val!:T

  return ()=>{
    if (!created) {
      if (creating) {
        throw new Error(`Circular initialization`)
      }
      try {
        creating = true
        val = f()
        created = true
      }
      finally {
        creating = false
      }
    }
    return val
  }
}
```

### Reference on Demand

Even if we use the "Create on Demand" pattern above, constructing a component will still cause all of its dependencies to be created first.  This prevents components from referencing each other, and can be wasteful if a component doesn't ever need to use that dependency.

One way to address this is for the `App` class to supply dependencies through getters.  For example, instead of:

```
class App {
  ...
  uploadDirectory: IUploadDirectory = (props) => uploadDirectory({
    ...props,
    directoryService: this.directoryService,
    uploadService: this.uploadService,
  })
}
```

wrap getters around those dependencies:

```
class App {
  self = this
  ...
  uploadDirectory: IUploadDirectory = (props) => uploadDirectory({
    ...props,
    get directoryService() { return self.directoryService },
    get uploadService() { return self.uploadService },
  })
  ...
}
```

With this in place, calling `App.uploadDirectory` no longer immediately retrieves (and potentially constructs) the `directoryService` and `uploadService`.  Instead, those dependencies are only filled when requested by the function implementation.

(Note the need for `self`, since for getters used in this way, `this` refers to the context object, not `App`)

The same pattern can be applied to both functions and class instances.

This pattern now unlocks the ability for services to reference each other, since the services can now be constructed without actually constructing all their dependencies.  However, this will only work if those services don't reference each other in their constructors.  This is addressed later in `Service Lifecycles`.

### Dependencies Shortcut

It might be observed that a lot of components end up looking for the same dependencies, which are already being supplied by the `App` with the same names.  For example:

```
class App {
  self = this

  get scheduler() {...}
  get log() {...}
  get s3() {...}
  ...
  component1 = createOnDemand(()=>new Component1({
    get scheduler() { return self.scheduler },
    get log() { return self.log },
    get s3() { return self.s3 },
  }))

  component2 = createOnDemand(()=>new Component2({
    get scheduler() { return self.scheduler },
    get log() { return self.log },
    get navigation() { return self.navigation },
  }))
  ...
}
```
For these cases, we can take a shortcut of simply supplying the `App` as the context:

```
class App {
  self = this

  get scheduler() {...}
  get log() {...}
  get s3() {...}
  ...
  component1 = createOnDemand(()=>new Component1(this))
  component2 = createOnDemand(()=>new Component2(this))
  ...
}
```
This gets more complicated if props need to be merged in.  The `uploadDirectory` is an example:

```
class App {
  self = this
  ...
  uploadDirectory: IUploadDirectory = (props) => uploadDirectory({
    ...props,
    get directoryService() { return self.directoryService },
    get uploadService() { return self.uploadService },
  })
  ...
}
```

We can't just pass in the `App` for the context, since that won't include the props.  Instead, we need a helper that will merge the `App` context with those props:

```
class App {
  self = this
  ...
  uploadDirectory: IUploadDirectory = (props) => uploadDirectory(withProps(this, props))
  ...
}
```
A possible implementation of `withProps`:

```typescript
function withProps<T extends object, P extends object>(base: T, props: P): T & P {
  return Object.create(base, Object.getOwnPropertyDescriptors(props)) as T & P
}

// Usage in App:
uploadDirectory: IUploadDirectory = (props) =>
  uploadDirectory(withProps(this, props))
```

### Composition Root delegation

As described earlier, a single `App` class can become unwieldy for a large complex app, and splitting it up might be beneficial.  When doing so, it's good to remember that the Composition Root is just another service with its own Context - there isn't really anything special about it.

So when decomposing, first identify a suitable domain that makes sense to collect a set of components out of the main `App`.  Then create a class to act as the "local" Composition Root for those components, and implement it using the same patterns for the main `App`.  If those components require dependencies that aren't in the local Composition Root, then they can be called out in that local's own Context.  Then the `App` just treats that local root as its own service, with its own dependencies.

## Service Lifecycles

Earlier it was mentioned that components with mutual references to each other would only be possible if those components didn't reference each other in their constructors.  This implies some sort of lifecycle, in which components do nothing in their constructors except store the context.

If a component needs to initiate some process, then it should place that initiation outside the constructor, such as in a `start()` method.  The App can even have a shutdown cycle, in which components that return a function from `start()` will expect those functions to be called on shutdown.

The App can then manually call `start()` on those services that require it.  Or to be a little fancier, a component that needs to be started can request a `serviceManager` dependency, then add itself to that serviceManager in its constructor (the only thing it's allowed to do in the constructor).  Somewhere in the App's operation (perhaps setImmediate) it will start any pending registered services, recording their shutdown functions (if any), to be executed in reverse order on shutdown.
