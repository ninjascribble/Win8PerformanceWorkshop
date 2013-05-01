Windows 8 App Performance Workshop
==================================

Presenter: Sharif Farag, Pricipal Group Program Manager

Agenda
------

### May 1, 2013

- Intro
- Win 8 App Performance Fundamentals
- Tools Introduction
- Tools Overview
- WWA / XAML Breakouts

### May 2, 2013

- WWA / XAML Walkthrough Lab
- Hands on Performance Tuning

Introduction: Why Performance Matters
-------------------------------------

- Mobile device improvements are behind desktop PC performance
- Engineering best practices are still the best way to tweak performance
- "Performance is a feature you design for"

### Top 4 reasons for bad reviews

1. App Freezes
2. Crashes
3. Slow to respond
4. Heavy battery usage

### Performance design approach

1. Learning
2. Planning
3. Instrumentation
4. Measurement
5. Monitoring & Analysis

### Goals

- Learn the tools
- Develop a mental model for how things *should* perform

### Three pillars of performance

- Fast
- Fluid
- Efficient

Fast
----

- App launch, navigation, changing orientation, "snappy"
- "User expectations are hardware independent"

### Critical Path Analysis

1. Begin with the "Big Picture" to identify the critical path

2. Identify each "Phase" on the critical path
    - Break down the Big Picture
    - Outline phases with custom instrumentation (ETW)

3. Drill down in each Phase of the critical path
    - ID the resource being blocked (starving)
    - ID the resources that might be responsible

4. Optimize
    - Ask: Can we not do this? (eliminate)
    - Ask: Can we do it earlier? (cache, persist)
    - Ask: Can we do it later? (queue)
    - Ask: Can we do it more efficiently? (batch, hash)

### UI Framework on the critical path

But what if it's not your code that's causing the problem? Most of the time on
the critical path may be spent on the UI Thread in framework/system code.

The UI Thread is the key to responsiveness, and spends time completing framework
tasks on behalf of your application. On top of this, the size of your DOM
(element tree) contributes to the time it takes to execute these tasks.

### The Display Pipeline

These are the steps the framework takes when rendering the UI. Assume that the
entire available DOM is being rendered, not just the visible parts of the DOM.

1. Load/Parse/Build the DOM

2. Apply formatting and layout styles
    - Compute CSS, apply templates
    - Measure and arrange
    - Build display for hit testing & rastarization

3. Display/Rastarization
    - Walk the DOM and rastarize the elements

4. Composition
    - Walk the DOM (again) to render and present

### UI Framework Invalidations

...And we do it all again, and again every time something changes visually
(invalidates the current UI).

- __Format / Layout__ invalidations requires measurement and reflow. Costly.
- __Display / Composition__ invalidations do not impact layout. Less costly.

Basically, if it takes up space, then it's going to take more time.

### Managing UI Framework Costs

#### Reduce Complexity

- Number of elements
- Number of templates
- Avoid complex layouts
- Use absolute positioning (because it doesn't require flow recalculation)
- Virtualize content

#### Maximize batching & scheduling

- Be aware of every single paint / layout calculation on the critical path
- Stage changes outside of the live element tree
- Commit changes in intentional large chunks to reduce the number of recalcs
- Layout and display only what you expect / require. _Be intentional_

Fluid
-----

### Key Concepts

Vsync / Refresh rate is 60fps. That means that animations / transitions must be
updated at the same cadence in order to seem "smooth" (~16ms)

### Composition

Desktop Composition (DWM) process is responsible for screen updates.

- Composes application generated surfaces every vsync
- CPU work is directly related to the complexity of the "scene"
- GPU work is bound by read/write memory bandwidth

### Independent animation / manipulation

In general, the UI thread is too slow and too busy to keep up the 60fps rate.
E.g. animation will impact panning performance, and vice versa.

Display / Composition invalidations can be re-rendered with DWM, which means
they are rendered on an independent thread. These invalidations include opacity,
translation, color, etc... but not anything that might cause a reflow.

_If an element requires a format or layout invalidation, then its
children cannot benefit from the independent animation thread._

### Render-ahead & Virtualization

The UI thread will try to render virtualized content that falls outside the view
to try to improve panning experience. If it's too busy, or starved for resources
then it will fail to render ahead, and cause a blank view on pan.

### Overdraw / Layers

More complex scenes require more of the UI thread. DWM will have to compose many
layers for every frame of animation if there are elements that are not opaque
due to partial transparency or opacity < 1.0.

#### Managing overdraw

- Turn on the overdraw heat map (how?)
- To ensure consistent flow, avoid > 2 layers
- Use solid background images
- Be mindful of using large container divs with empty space
- Use the HTML / DWM frames tables to see the visuals in the scene

Efficient
---------

### Battery life

Frequent CPU use will impact battery life, including:

- Frequent animations without user interaction
- Polling a web service frequently
- Querying the GPS sensor frequently

### Memory

Lower memory footprint improves responsiveness, nad reduces the likelyhood of
your app being terminated. It's best to release resources that are not needed
right away instead of keeping them around for later.

#### Benchmarks

- < 100MB: Mail, Calendar, Calculator
- 100-150MB: News readers, casual games
- 150-250MB: Media, Video conferencing
- > 250MB: Games

### Disk footprint

Minimize the number of resources in your application bundle, and keep them
appropriate to screen size, locale, etc...

#### Benchmarks

- < 50MB: Ideal for most apps
- < 100MB: For large apps such as games

Visual Studio Performance Tools
-------------------------------

_Note: Running JavaScript apps in debug mode will adversely affect performance!_

### Memory analysis

Debug -> Javascript Analysis -> Memory

Shipped in November with update #1. Helps to understand your app's memory usage
at any given time, and identify the objects that are monopolizing your memory
pool.

The tool forces garbage collection when you take a snapshot, so the analysis
it generates is always from a clean state.

### Profiling UI responsiveness

Debug -> Javascript Analysis -> UI Performance

Shipped in March with quarterly update #2. Logs CPU activity and FPS at a given
time.

### Profiling code execution

Debug -> Start Performance Analysis

Help to find the app's hottest code paths, and ID which functions are most
responsible for performance degradation.

#### Inclusive time

The total time from when the function was entered to when it was exited.

#### Exclusive time

The amount of time spent executing just the function. This is the flag-raiser.

Windows Performance Toolkit
---------------------------

_(Why would you get a program manager to speak to a bunch of devs?)_

### Terminology

- Windows Performance Toolkit (WPT)
- Windows Performance Recorder (WPR)
- Windows Performance Analyzer (WPA)

### Windows Performance Recorder (WPR)

The tool tha captures a trace for a scenario you want to investigate.

Before capturing a trace, you can configure WPR to inspect only specific
resources. You can also configure the scenario that it will be analyzing, 
including audio/video glitches, Internet Explorer behavior, and WAA or XAML app.

### Windows Performance Analysis (WPA)

The tool that will analyze the WPR trace for performance bottlenecks.

#### Graph explorer

Display system resource usage by type (computation, storage, memory, power...).
Drag graphs from this view into the Analysis view for inspection.

#### Analysis

- WWAHost.exe is the application launch process_
- Right-click on a process and "filter to selection"
- Table view: Left of gold bar are "group by" columns
- Configure a Symbol Path and Trace -> Load Symbols
- Profiles are trace-agnostic. Save profiles and apply them to later traces.
- There's also a default profile for modern app analysis.
- Can compare two traces in a Comparative Analysis view.
- ...But, while the table will be a diff view, the graph will not.
- "Just traverse the entire thing and make it work."

#### Online resources

- Search: "Build WPA"
- http:\\ietestdrive.com

WWA Breakout
------------

"Webworkers usually slow down the performance of a web application." Wat?

"Downloading / Decoding images may take some time, but don't impact your perf."

"No performance differences between IE10 rendering and WinJS rendering."

### Normal page processing workflow

1. Networking (Request)
2. Parsers (Response)
3. DOM Tree
4. Formatting
5. Layout
6. Display Everything
7. Script execution

Loop 4-7 on every animation frame ("vblank").

__Display Tree__ is basically a rendering cache that includes the parsed server
response, DOM tree, and formatting/layout information. The DWM process processes
this on each frame tick so that it doesn't have to go through the whole process
again.

Vsync target is 60fps, but Trident doesn't drop frames. If your animation takes
longer than 16ms to process, then you'll get a lower FPS. All work is triggered
by the animation loop ("vblank"). Lower FPS impacts more than just animation
performance!

### Deferred loading of JS assets in Win8 Apps

- Load async
- Defer property
- Load necessary scripts in views
