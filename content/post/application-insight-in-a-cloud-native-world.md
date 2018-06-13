+++
categories = ["category", "subcategory"]
date = "2018-06-13T15:44:05+01:00"
keywords = ["tech"]
tags = ["application insight", "monitoring", "kubernetes", "OpenTracing", "Prometheus", "logging", "tracing"]
title = "Application Insight in a cloud native world"

+++
> APM products aren't part of your solution they are part of your problem

Don't get me wrong they are for the most part great products and they do absolutely serve a purpose. But as we move towards a much more distributed world with lots of smaller interconnected applications that are frequently updated and dynamically scaled I'm not sure that these traditional products are the solution. Certainly not in the product format they are in currently where you pay per "host" or "agent".

In my experience there is also the issue that APM solutions seem to provide a false sense of comfort for teams. "We have installed XYZ and we have monitoring covered"often with little or no thought put into _what_ we are even interested in monitoring within our applications. Of course they all come with sensible defaults and give you lots of interesting metrics and graphs to look at and they will for sure give you _some_ insight into what it going on within your application. Which in the surface of it look great and give you loads of confidence.

I don't think its good enough to just install the agent and tick the box. We need to commit a good amount of headspace to what our application should look like in production, what is important to it and also what can and will go wrong that we want to be able to know about and understand. This helps us operate our applications with all of the right information to know our app is not just working but working correctly.

# What does Application Insight look like?

My current thinking is splitting "application insight" into 3 distinct but interrelated problems:

* Log Aggregation
* Instrumentation
* Tracing

We often think of these things collectively as "monitoring" but I would go further and say that monitoring is what we do with this information, how we aggregate it and use it to give us insight into what is going on. Part of this is ensuring that the system lets us know when it is encountering issues, preferably before they happen.

Lets break this down and look at the different parts

## Log Aggregation

Logging is everywhere, everything logs our some sort if information. We need to treat this as a stream of events and collect them all together. Its important that although applications don't all have to log the exact same things they need to be doing it in as compatible a format as possible. The most accessible format for most applications is to log to JSON. This has the benefit of being a wide standard and one that can be enriched and adjusted as needed.

Its important that we focus our logs on things that the application does functionally and not concern ourselves too much with framework or language implementation details. For example we should be logging the fact that a transaction occurred (or failed) and important information about that transaction not concerning ourselves with things like the fact that the application invoked certain methods (we will cover that in tracing later).

We need to then collect all of these log together into one place and at a minimum be able to search them for important things. Even better if we can set up things like alerts for certain types of logs and the frequency at which they appear.

So log aggregation is the definition of a standard and also a way of bringing them all into one place. In a kubernetes implementation this is actually quite a trivial thing to do and there are lots of great solutions available out there. Most of them run some sort of collector as DaemonSet with an agent on each node in the cluster. The defacto open source solution in this space is ElasticSearch - FluentD - Kibana.

## Instrumentation

Its important that we need teams to apply thought and engineering effort to implementing the right instrumentation for their application. We should move away from the traditional APM style vendors in favour of having each application explicitly have to implement the instrumentation of the metrics it cares most about. This forces development teams to have to think about the way their application operates and what is important to the application during its operation and what is important for understanding the usefulness or success of the application itself. Measuring CPU and RAM are just the context for all other metrics we wish to measure.

Prometheus is emerging as a defacto standard for instrumenting distributed applications, especially those running on kubernetes. I encourage you to take a look at the [prometheus project](https://prometheus.io) it has some great features and encourages you to think differently about not just the metrics you wish to collect but how to collect them at scale without incurring significant performance penalties.

To enable us to instrument our code we need to import the Prometheus SDK which is a set of language level features that allow you to create measurements at any point in your code using the built in types of measure that prometheus supports. As prometheus is a standard rather than an implementation there are SDKs available in lots of common languages and means you can use is across your estate no matter what your mix of languages and frameworks.

Adding specific code to our applications in order to instrument them may seem a bit odd at first and not something we might want to do. However, think of this in the same way you use log lines, they just appear in the code and have little or nothing to do with the actual execution. Being able to understand our app in operation is just as important as having "clean code".

## Tracing

Tracing is the act of measuring the way your program executes and is specific to your implementation. This is useful because it shows us the different moving parts internally to our applications and how they interact.

With distributed systems we need to further concern ourselves with how your application interacts with other applications that compose your wider system. We need to be able to trace execution between boundaries and get insight into what happened. If each application did this individually this would be a very tedious process as we would have to try and identify individual executions in a sea of data.

Distributed tracing specifications such as [OpenTracing](http://opentracing.io/) define how this should work in an abstract manner that means you can add tracing code to your application without tying yourself into a specific vendor like you might with more traditional solutions. The OpenTracing API is a standardised spec that is implemented in some 9 languages so far with many differing implementation being available both open source and commercially.

With OpenTracing we wrap parts of our program execution in "spans" that measure its execution. We can add further contextual information to the span in the form of tags or logs. A typical trace may look something like this in c#

    using (IScope scope2 = trace.BuildSpan("httpbinrequest").StartActive(finishSpanOnDispose: true))
    {
    	await $"https://httpbin.org/delay/{requestTime}".GetAsync();
    }

Its important to understand that the implementations of OpenTracing compliant tracers such as [Jaeger](https://www.jaegertracing.io/) rely only on language features and install like an SDK. In this way they rely less on "plugging" into the runtime and extracting information without your application needing to be aware. This is a great result that makes using later language versions a lot easier and removes a huge upgrade issue when say moving from Java 7 to Java 8 and having to wait for a new agent from your APM vendor of choice. It also means that should you choose an unsupported language it is fairly trivial to create a working  implementation.

# The result

We need to be able to provide a robust, scalable and opinionated way of collecting application insight across your entire deployment. This will provide the sorts of insight into applications that let us operate them with utter and complete confidence. We need to make this as easy as possible for teams to implement and ensure that it is very much thought of as a benefit to the team. It should be entirely baked into our architecture so that everyone is on the same page.

Developers need to be asked to implement their own instrumentation within the application so that thought needs to be put into "what is important here" in that way we are forcing developers to think about operations at design time. In this way we define what "insight" each application cares about.

## Power of peer review

One of the major not so obvious benefits in this approach is that we are "writing code" for our insight. This means that it will have to go through the standard peer review process your teams use, in lots of teams this is sort of pull request within git. This means that this code has to be reviewed, discussed and refactored before it makes it into your application.

## Deployment

The other great benefit is that all of this is deployed with your application. Each and every change is deployed as part of your built application. Meaning that all changes have to go through the same pipeline and changes don't have to be done in a different system or control panel. It bring control of the insight back into the code itself and back into your Git repository. Changes can be made and tested on branches without affecting your production configuration and can be tied back to specific deployments and changes.