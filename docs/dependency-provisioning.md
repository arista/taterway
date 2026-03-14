# Dependency Provisioning

Each program component, such as a function or a class, is explicitly passed any dependencies that it requires to do its job.  Components don't "reach out" to global entities or static methods, but instead rely solely on what's been passed in.  So when a component is designed, it not only defines what parameters it expects, but also defines a **Deps** object that declares its dependencies.  The component should then expect to be passed its required parameters and Deps object.

For example (in TypeScript):

```
function uploadDirectory(public props: UploadDirectoryProps, ctx: UploadDirectoryDeps) {
  ctx.directorySource.readDir(props.dirname)
  ...
  ctx.uploadTarget.writeFile(...)
}

interface UploadDirectoryProps {
  dirname: string
  dest: string
}

interface UploadDirectoryDeps {
  directorySource: IDirectorySource
  uploadTarget: IUploadTarget
}
```

Here the uploadDirectory function expects its dependencies to be passed as an UploadDirectoryDeps function.  Similarly for a class:

```
class Scheduler implements IScheduler {
  constructor(public ctx: SchedulerDeps) {
    this.jobs = []
  }
  
  pollNextJob() {
    if (this.ctx.timeService.currentTime >= this.nextJob.time) {
      ...
    }
  }

  ...
}

interface SchedulerDeps {
  timeService: ITimeService
}

interface IScheduler {
  addJob(...)
  addPeriodicJob(...)
  ...
}
```

Here the class is constructed with a SchedulerDeps, which contains all of the dependencies that might be used by any method in the class.

A "provisioning service" is then responsible for assembling the Deps objects required for each component.  This is typically the job of the top-level "App" class.  That class would create concrete instances of classes that satisfy the interfaces required by various components.  For example:

```
class App {
  timeService: TimeService
  scheduler: Scheduler
  directorySource: DirectorySource
  uploadTarget: UploadTarget
  
  constructor() {
    this.timeService = new TimeService(...)
    this.scheduler = new Scheduler({
      timeService: this.timeService
    })
    ...
  }
}
```

## Call Provisioning

When a component is doing its work, it may rely on other components that require their own Deps objects.  For example, a `PeriodicUploader` might need to periodically call the `uploadDirectory` function above, which requires an `UploadDirectoryDeps` to be passed in.  The `PeriodicUploader` is not expected to assemble that Deps on its own - in fact, it shouldn't even "see" that Deps.  Instead, the `PeriodicUploader` declares one of its dependencies to be a curried form of the `uploadDirectory` function that does not require that deps parameter:

```
function uploadDirectory(props: UploadDirectoryProps, ctx: UploadDirectoryDeps): void {...}

type IUploadDirectory = (props: UploadDirectoryProps): void


class PeriodicUploader {
  constructor(public props: PeriodicUploaderProps, ctx: PeriodicUploaderDeps) {...}
  
  start() {
    const {dirname, dest, period} = this.props
    const {scheduler, uploadDirectory} = this.ctx
    scheduler.addPeriodicJob(period, ()=>{
      uploadDirectory({dirname, dest})
    })
  }
}

interface PeriodicUploaderProps {
  dirname: string
  dest: string
  period: number
}

interface PeriodicUploaderDeps {
  scheduler: IScheduler
  uploadDirectory: IUploadDirectory
}
```

This allows the `PeriodicUploader` to make use of the `uploadDirectory` function, without knowing anything about its `UploadDirectoryDeps` requirement.  Instead, that requirement would be fulfilled by the top-level App service:

```
class App {
  ...
  uploadDirectory: IUploadDirectory
  
  constructor() {
    ...
    this.uploadDirectory = (props) => uploadDirectory(props, {
      directorySource: this.directorySource,
      uploadTarget: this.uploadTarget,
    })
  }
}
```

Now the `App` is able to provide a form of the `uploadDirectory` function to any component that needs it.

Similarly, a component might need to create an instance of a class, but find that the class declares a Deps with dependencies.  For example, consider a `startPeriodicUploads` function that needs to create a new `PeriodicUploader` class.  In this case, the `PeriodicUploader` needs to define of "curried" constructor that doesn't require the Deps.  This can be done by declaring a factory method:

```
class PeriodicUploader {...}

type CreatePeriodicUploader = (props: PeriodicUploaderProps): IPeriodicUploader
```

The `startPeriodicUploads` can then use this:

```
function startPeriodicUploads(ctx: StartPeriodicUploadsDeps) {
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

interface StartPeriodicUploadsDeps {
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

## Combining Props and Deps

The above examples assume that the component is responsible for distinguishing between what are "parameters", and what are "dependencies".  Sometimes, however, this distinction isn't clear.  For example, the `period` for the `PeriodicUploader` might sometimes be passed in directly by the caller, or it might be provisioned by the App from some configuration variable.

From the component's perspective, it just cares about getting the `period`.  It's not particularly interested in declaring that the period be considered a "parameter" or a "dependency".  From the component's perspective, the parameters and the deps are really just one bag of **Context** that the function needs:

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

The same is true for function calls - they would generally expect a single "ctx" parameter that contains everything the function needs to do its work, without trying to distinguish between a "parameter" and a "dependency".  That distinction really is a matter to be decided by how the component is used, not necessarily the component builder.

