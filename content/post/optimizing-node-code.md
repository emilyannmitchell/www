---
title: Tips for optimizing slow code in Node
date: '2016-02-06T16:49:04-06:00'
strongloopURL: 'https://strongloop.com/strongblog/tips-optimizing-slow-code-in-nodejs/'
tags:
  - performance
  - optimization
  - nodejs
---

Node.js programs can be slow due to CPU or IO bound operations. On the CPU side, typically there is a "hot path" (a code that is visited often) that is not optimized. On the IO side, limits imposed by either the underlying OS or Node itself may be at fault. Or, a slow application may have nothing to do with Node; instead, an outside resource, like database queries or a slow API call, may not be optimized.

In this article, we will focus on identifying and optimizing CPU heavy operations in our codebase. We will explore taking profiles of our production application in order to analyze them and make changes to improve efficiency.

Avoiding heavy CPU usage is especially important for servers[^1] due to Node's single-threaded nature. Time spent on the CPU takes away time for servicing other requests. If you notice your application is responding slowly and the CPU is consistently higher for the process, profiling your application helps find bottlenecks and bring your program back to a speedy state.

# Profiling applications

Duplicating sluggish application issues that occur in production is hard and time consuming. Thankfully, you don't have to do that. You can gather profile data on the production servers themselves to analyze offline. Let's look at a few ways to do that.

## Using kernel level tools

First, you can use kernel level tools, such as DTrace (Solaris, BSD), perf (Linux), and XPerf (Windows), to gather stack traces from running processes and then generate [flame graphs](http://www.brendangregg.com/FlameGraphs/cpuflamegraphs.html). Kernel level profiling has minimal impact on a running process. Flame graphs are generated as SVG with the ability to zoom in and out of the call stacks. Yunong Xiao from Netflix has an excellent [talk](https://www.youtube.com/watch?v=O1YP8QP9gLA) and [post](http://yunong.io/2015/11/23/generating-node-js-flame-graphs/) for Linux perf to learn more about this technique. If you need to maintain high throughput in your production application, use this method.

{{< figure src="/_media/flame-graph.png" title="Source: <https://cloudup.com/cE3eGxFHGif>" >}}

## Using the V8 profiler

Another option is tapping into the [V8 profiler](https://github.com/node-inspector/v8-profiler) directly. This method shares the same process with your application, so it could impact performance. For that reason, only run the profiler when you experience the problem in order to capture the relevant output. The perk of this method: you can use all of Chrome's profiling tools with the generated output (including flame graphs) to investigate.

To instrument your application run:

```sh
npm install v8-profiler --save
```

Then, add this code to your application:

```javascript
const profiler = require('v8-profiler')
const fs = require('fs')
var profilerRunning = false

function toggleProfiling () {
  if (profilerRunning) {
    const profile = profiler.stopProfiling()
    console.log('stopped profiling')
    profile.export()
      .pipe(fs.createWriteStream('./myapp-'+Date.now()+'.cpuprofile'))
      .once('error',  profiler.deleteAllProfiles)
      .once('finish', profiler.deleteAllProfiles)
    profilerRunning = false
    return
  }
  profiler.startProfiling()
  profilerRunning = true
  console.log('started profiling')
}

process.on('SIGUSR2', toggleProfiling)
```

The output is written to a file in the current working directory for the process (make sure it's writable!). Since this is a programmatic API, you can trigger it however you'd like (web endpoint, IPC, etc.). You also can trigger it whenever you like, if you have a hunch about when things get sluggish. Setting up automatic triggers can be helpful to avoid babysitting your application, but it requires forethought on when and how long to capture.

Once you've collected your profile data, [load it up](https://docs.strongloop.com/display/SLC/CPU+profiling#CPUprofiling-ViewingCPUprofiledata) it in the Chrome Dev Tools and start [analyzing](https://developers.google.com/web/tools/chrome-devtools/profile/rendering-tools/js-execution?hl=en)!

{{< figure src="/_media/chrome-flame-graph.png" >}}

## Using a process manager

Although utilizing the V8 profiler directly is powerful and customizable, it does invade your code base and adds another dependency to your project which may not be desirable. An alternative is to use a process manager that can wrap your application with tools when you need them. One option is the `slc` command-line tool from StrongLoop.

First, run `npm install strongloop -g`. Then run:

```sh
slc start [/path/to/app]
```

This will start your application wrapped in a process manager that allows you to take CPU profiles on demand. To verify a proper start and obtain the application id, run:

```sh
slc ctl
```

You will get an output similar to this:

```
Service ID: 1
Service Name: my-sluggish-app
Environment variables:
    Name      Value
    NODE_ENV  production
Instances:
    Version  Agent version  Debugger version  Cluster size  Driver metadata
     5.0.1       2.0.2            1.0.0             1             N/A
Processes:
        ID      PID   WID  Listening Ports  Tracking objects?  CPU profiling?  Tracing?  Debugging?
    1.1.61022  61022   0
    1.1.61023  61023   1     0.0.0.0:3000
```

Locate the process id for your application. In this example, it is `1.1.61023`. Now, we can start profiling whenever we want by running:

```sh
slc ctl cpu-start 1.1.61023
```

When we feel we have captured the sluggish behavior, we can stop the profiler by running:

```sh
slc ctl cpu-stop 1.1.61023
```

This will write a file to disk:

```
CPU profile written to `node.1.1.61023.cpuprofile`, load into Chrome Dev Tools
```

And that's it. You can load it into Chrome just like the V8 profiler.

## Making the right choice

In this article, I presented three options for capturing production CPU usage in Node. So which one should you use? Here are some thoughts to help narrow down that decision:

1. I need to profile for long periods of time: use kernel tools.
2. I want to use Chrome Developer Tools: use V8 profiler or a process manager.
3. I want to trigger profiles for certain actions in my application: use V8 profiler.
4. I can't have performance impacted in my application: use kernel tools.
5. I want my applications ready for profiling without having to instrument each one: use a process manager.

[^1]: Command line applications typically don't have these requirements.
