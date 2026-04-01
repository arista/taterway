# TypeScript Dependency Provisioning

Each program component, such as a function or a class, is explicitly passed any parameters and dependencies that it requires.  Components do not "reach out" to global entities or static methods, but instead rely solely on what's been passed in.

## Component Design

When a component is designed, it not only defines what parameters it expects, but also any dependencies it might need to do its work.  All of these values are collected into a **Context** object that must be assembled and passed to the component:

```
function uploadDirectory(ctx: UploadDirectoryCtx): void {
  ctx.directoryService.readDir(ctx.dirname)
  ...
  ctx.uploadService.writeFile(...)
}

type UploadDirectoryProps = {
  dirname: string
  dest: string
}

type IUploadDirectory = (props: UploadDirectoryProps): void

type UploadDirectoryCtx = UploadDirectoryProps & {
  directoryService: IDirectoryService
  uploadService: IUploadService
}
```
In this case, the `uploadDirectory` function depends solely on the values passed in through the `UploadDirectoryCtx`.   This includes both values that one would normally think of as "parameters" (`dirname` and `dest`), as well as values that one might think of as "dependencies" (`directoryService` and `uploadService`).  All of those are combined into a single Context object that the function uses.

In some cases, the distinction between "parameters" and "dependencies" is clear enough that a component might want to formalize it.  That's the case above, where the Context is actually a combination of `UploadDirectoryProps`, representing the parameters, and the additional values representing the dependencies.  The function also defines `IUploadDirectory` as a form of the function that just requires the parameters.  This is essentially the component anticipating how it will be used by consumers, who are just going to want to provide the props and not worry about the other dependencies.  But again, this is just anticipating usage - the component's implementation just requires the whole Context, and the application can provide that Context however it chooses.

The same principles can be applied to a class, in which a class is constructed with a "Context" that contains all of the dependencies that might be used by any method in the class:

```
class Scheduler implements IScheduler {
  constructor(public ctx: SchedulerCtx) {
    this.jobs = []
  }
  
  pollNextJob() {
    if (this.ctx.timeService.currentTime >= this.nextJob.time) {
      ...
    }
  }

  ...
}

type SchedulerCtx = {
  timeService: ITimeService
}

interface IScheduler {
  addJob(job: ...)
  addPeriodicJob(period: ..., job: ...)
  ...
}
```

Here the class is constructed with a `SchedulerCtx`, which contains all of the dependencies that might be used by any method in the class.  This particular class doesn't have any parameters, so there isn't a need to define a separate **Props**  type.

## Provisioning Service

A "provisioning service" takes on the responsibility of providing each component with the full Context object that component requires.  This is typically the job of a top-level "App" class, which ends up having knowledge of all components in the system.  For large systems, this can become unwieldy, and such systems may architect that "App" class to delegate some of that provisioning responsibility among various subsystems.

In its simplest form, the App class can construct each component, filling in the required Context structures with pointers to other components:

```
class App {
  timeService = new TimeService(...)
  scheduler = new Scheduler({
    timeService: this.timeService
  })
  directoryService = ...
  uploadService = ...
  uploadDirectory: IUploadDirectory = (props) => {
    uploadDirectory({
       ...props,
       directoryService: this.directoryService
       uploadService: this.uploadService
    })
  }
  
  constructor(public ctx: AppCtx) {}
}

type AppCtx = {
  ...
}
```

Note that even the App class can declare its own Context that it requires, perhaps filled in from command line arguments or environment variables.

### Create On Demand

The above approach can work for simpler cases, but can become fragile and unwieldy as component dependencies become more complex.  Creating components on demand can alleviate a lot of this.  Rather than creating components immediately on constructing the App, the App instead creates a set of memoized constructor functions that create the appropriate component when first called.  That might look like this:

```
class App {
  timeServiceFn = createOnDemand(()=>new TimeService(...))
  get timeService() { return this.timeServiceFn() }

  schedulerFn = createOnDemand(()=>new Scheduler({
    timeService: this.timeService
  })
  get scheduler() { return this.schedulerFn() }

  directoryServiceFn = ...
  get directoryService() { return this.directoryServiceFn() }

  uploadServiceFn = ...
  get uploadService() { return this.uploadServiceFn() }

  uploadDirectory:IUploadDirectory = (props) => {
    uploadDirectory({
       ...props,
       directoryService: this.directoryService
       uploadService: this.uploadService
    })
  }
}
```
The `createOnDemand` function returns a function that, when called for the first time, calls the passed-in construction function and returns the result, and when called subsequently, returns that same instance.  A sample implementation:

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
So now when the App is constructed, no actual components are created.  Only once a component is accessed will it, and all its dependencies, be constructed.

## Call Provisioning

When a component is doing its work, it may rely on other components that require their own ctx objects.  For example, a `PeriodicUploader` might need to periodically call the `uploadDirectory` function above, which requires an `UploadDirectoryCtx` to be passed in.

The `PeriodicUploader` is not expected to assemble that Ctx on its own - in fact, it shouldn't even "see" that Ctx.  Instead, the `PeriodicUploader` declares one of its dependencies to be a curried form of the `uploadDirectory` function that only requires what the consumer is expected to provide:

```
function uploadDirectory(ctx: UploadDirectoryCtx): void {...}

type IUploadDirectory = (props: UploadDirectoryProps): void

type UploadDirectoryProps = {
  dirname: string
  dest: string
}


class PeriodicUploader {
  constructor(public ctx: PeriodicUploaderCtx) {...}
  
  start() {
    const {dirname, dest, period, scheduler, uploadDirectory} = this.ctx
    scheduler.addPeriodicJob(period, ()=>{
      uploadDirectory({dirname, dest})
    })
  }
}

type PeriodicUploaderProps = {
  dirname: string
  dest: string
  period: number
}

type PeriodicUploaderCtx = PeriodicUploaderProps & {
  scheduler: IScheduler
  uploadDirectory: IUploadDirectory
}
```

This allows the `PeriodicUploader` to make use of the `uploadDirectory` function, without knowing anything about its `UploadDirectoryCtx` requirement.  Instead, that requirement would be fulfilled by the top-level App service:

```
class App {
  ...
  uploadDirectoryFn = createOnDemand<IUploadDirectory>(()=>(props) => uploadDirectory({
    ...props,
    directoryService: this.directoryService,
    uploadService: this.uploadService,
  }))
  
  get uploadDirectory() { return this.uploadDirectoryFn() }
}
```

Now the `App` is able to provide a form of the `uploadDirectory` function to any component that needs it.

Similarly, a component might need to create an instance of a class, but find that the class declares a Ctx with dependencies.  For example, consider a `startPeriodicUploads` function that needs to create a new `PeriodicUploader` class.  In this case, the `PeriodicUploader` needs to define a "curried" constructor that doesn't require the Ctx.  This can be done by declaring a factory method:

```
class PeriodicUploader {...}

type CreatePeriodicUploader = (props: PeriodicUploaderProps): IPeriodicUploader
```

The `startPeriodicUploads` can then use this:

```
function startPeriodicUploads(ctx: StartPeriodicUploadsCtx) {
  const hourlyUploader = this.ctx.createPeriodicUploader({
    dirname: "/uploads/hourly",
    dest: "/backups/hourly",
    period: 60,
  })
  hourlyUploader.start()

  const dailyUploader = this.ctx.createPeriodicUploader({
    dirname: "/uploads/daily",
    dest: "/backups/daily",
    period: 60 * 24,
  })
  dailyUploader.start()
}

interface StartPeriodicUploadsCtx {
  createPeriodicUploader: CreatePeriodicUploader
}

type IStartPeriodicUploader = ()=>void
```
And again, all of this gets assembled in the App:

```
class App {
  ...
  createPeriodicUploader: CreatePeriodicUploader
  startPeriodicUploads: IStartPeriodicUploads
  
  constructor() {
    ...
    this.createPeriodicUploader = (props) => new PeriodicUploader(props, {
      scheduler: this.scheduler,
      uploadDirectory: thisUploadDirectory,
    })

    this.startPeriodicUploads = (props) => startPeriodicUploads({
      createPeriodicUploader: this.createPeriodicUploader
    })
  }
}
```

## Combining Props and Ctx

The above examples assume that the component is responsible for distinguishing between what are "parameters", and what are "dependencies".  Sometimes, however, this distinction isn't clear.  For example, the `period` for the `PeriodicUploader` might sometimes be passed in directly by the caller, or it might be provisioned by the App from some configuration variable.

From the component's perspective, it just cares about getting the `period`.  It's not particularly interested in declaring that the period be considered a "parameter" or a "dependency".  From the component's perspective, the parameters and the ctx are really just one bag of **Context** that the function needs:

```
class PeriodicUploader {
  constructor(public ctx: PeriodicUploaderCtx) {...}
  ...
}

interface PeriodicUploaderCtx {
  dirname: string
  dest: string
  period: number
  scheduler: IScheduler
  uploadDirectory: IUploadDirectory
}
```

The same is true for function calls - they would generally expect a single "ctx" parameter that contains everything the function needs to do its work, without trying to distinguish between a "parameter" and a "dependency".  That distinction really is a decision for the component user, not necessarily the component builder.

Making that distinction *somewhere*, though, is useful since it's required for defining the "curreid" functions and constructors.  So this remains an open question - where should the definitions of the "curried" functions live?

* With the defining component - `PeriodicUploader`, for example, defines its own best guess at `PeriodicUploaderProps`
* With the App - the App takes its best guess at `PeriodicUploaderProps`
* With the consuming component - the consumer defines what it's going to provide, and expects the provisioner to supply the rest:

```
function startPeriodicUploads(ctx: StartPeriodicUploadsCtx) {...}

type StartPeriodicUploadsCtx = {
  createPeriodicUploader: (props: {
    dirname: string
    dest: string
    period: number
  }) => IPeriodicUploader
}
```

I am leaning toward the last approach.  That lets multiple components in the same app use a component different ways.  And if multiple components *do* end up using the component the same way with the same set of Props, then that interface can be factored out to one of the other above choices.  But it doesn't need to be factored out right away.

## Considerations for the Provisioner

Much of the complexity of the approach concentrates into the top-level provisioning class.  In the most general case, the provisioning class constructs a separate, tailor-made context for every component.  This provides the most flexibility.

For more trivial cases, the provisioning class might be able to get away with just passing itself as the context.  This would work if components all named their dependencies consistently (e.g., dependencies all refer to `scheduler` and the provisioning class exposes it as `scheduler`).  A system might actually get pretty far with this.  When it needs to switch to something more general, it can do so without affecting the components.

The above examples also assume that components don't reference each other.  This means that the provisioning system can create all the components up front, as long as components are only created after their dependencies.  However, if components reference each other, forming a reference graph rather than a tree, then different techniques need to be used:

* the provisioning class should create components on demand, rather than attempting to create them all in the constructor:

```
class App {
  _scheduler: Scheduler|null = null
  get scheduler() {
    return this._scheduler ||= new Scheduler({...})
  }
}

```

* the contexts provided to components should define getters to access dependencies, instead of supplying the dependencies directly to the context:

```
class App {
  _scheduler: Scheduler|null = null
  get scheduler() {
    const self = this
    return this._scheduler ||= new Scheduler({
      get timeService() {return self.timeService}
    })
  }
}

```

* component classes should not perform any actions in their constructor, except to store their context and initialize state.  They shouldn't, for example, access anything in the context:

```
class Scheduler implements IScheduler {
  constructor(public ctx: SchedulerCtx) {
    this.jobs = []
  }
}
```

* active components that run some kind of process should expose a `start()` method that the top-level application can call, after the application has prepared itself to create components on demand.  The start method can also return a function that acts as a "stop()" to be called in some kind of reverse order.

```
class Scheduler implements IScheduler, Startable {
  start() {
    ... start background task ...
    return ()=>{
      ... stop background task ...
    }
  }
}

interface Startable {
  start(): OnStop|void
}

type OnStop = ()=>void

class App {
  ...
  constructor(ctx: AppCtx) {}

  start() {
    this.startService(this.scheduler)
  }

  onStops: Array<OnStop> = []
  startService(startable: Startable) {
    const onStop = startable.start()
    if (onStop) {
      this.onStops.unshift(onStop)
    }
  }
  stop() {
    for(const onStop of this.onStops) {
      onStop.stop()
    }
  }
}
```


