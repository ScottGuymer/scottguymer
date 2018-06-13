+++
categories = ["category", "subcategory"]
date = "2018-04-11T13:51:05+00:00"
draft = true
keywords = ["tech"]
tags = ["monitoring", "instrumentation", "kubernetes", "OpenTracing", "Prometheus"]
title = "Monitoring in a cloud native world"

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

We need to then collect all of these log together into one place and at a minimum be able to search them for important things. Even better if we can set up things like alerts for certain types of logs and the frequency at which they appear.

So log aggregation is the definition of a standard and also a way of bringing them all into one place.

## Instrumentation

Its important that we need teams to apply thought and engineering effort to implementing the right instrumentation for their application. We should move away from the traditional APM style vendors (New Relic, AppD etc) in favour of having each application explicitly have to implement the instrumentation of the metrics it cares most about. This forces development teams to have to think about the way their application operates and what is important to the application during its operation and what is important for understanding the usefulness or success of the application itself. Measuring CPU and RAM are just the context for all other metrics we wish to measure.

Prometheus is emerging as a defacto standard for instrumenting distributed applications, especially those running on kubernetes. I encourage you to take a look at the [prometheus project](https://prometheus.io) it has some great features and encourages you to think differently about not just the metrics you wish to collect but how to collect them at scale without incurring significant performance penalties

## Tracing

Tracing is the act of measuring the way you 

# What can we do?

We need to provide a robust, scalable and opinionated way of collecting application insight across your entire deployment. This will provide the sorts of insight into applications that let us operate them with utter and complete confidence. This should come "for free" as much as possible as a benefit of using the platform itself. And should be entirely baked into your architecture so that everyone is on the same page.

Developers should also be able to implement their own instrumentation within the application and have the platform able to aggregate this information with 0 effort from the team. In this way we can build custom instrumentation into application for metrics that each application cares about. These metrics should then be available to be queried and visualised. Preferable alongside the performance metrics so that issues can be correlated.

## Power of peer review

One of the major not so obvious benefits in this approach is that we are "writing code" for our metrics. This means that it will have to go through the standard peer review process your teams use, in lots of teams this is sort of pull request within git. This means that this code has to be reviewed, discussed and refactored before it makes it into your application.