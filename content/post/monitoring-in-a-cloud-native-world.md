+++
categories = ["category", "subcategory"]
date = "2018-04-11T13:51:05+00:00"
draft = true
keywords = ["tech"]
tags = ["monitoring", "instrumentation", "kubernetes"]
title = "Monitoring in a cloud native world"

+++
> APM products aren't part of your solution they are part of your problem

Dont get me wrong they are for the most part great products and they do absolutely serve a purpose. But as we move towards a much more distributed world with lots of smaller interconnected applications that are frequently updated and dynamically scaled I'm not sure that these traditional products are the solution.

<!--more-->

Provide a robust, scalable and opinionated way of collecting instrumentation across your entire deployment. This will provide the sorts of metrics from applications. This should come "for free" as much as possible as a benefit of using the platform itself.

 

It should be capable of collecting Performance Metrics from applications running upon it. With 0 effort from the application itself. The developers of the applications should get this as an expectation from the platform.

 

Developers should also be able to implement their own instrumentation within the application and have the platform able to aggregate this information with 0 effort from the team. In this way we can build custom instrumentation into application for metrics that each application cares about. These metrics should then be available to be queried and visualised. Preferable alongside the performance metrics so that issues can be correlated.

 

We should move away from the traditional APM style vendors (New Relic, AppD etc) in favour of having each application explicitly have to implement the reporting of the metrics it cares most about. This forces development teams to have to think about the way their application operates and what is important to the application during its operation and what is important for monitoring the usefulness or success of the application itself. Rather than just relying on a 3rd party "tool" to take care of this and no thought being put into it.