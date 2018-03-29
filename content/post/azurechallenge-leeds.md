+++
categories = ["category", "subcategory"]
date = "2018-03-29T13:27:50+00:00"
keywords = ["tech"]
tags = ["kubernetes", "azure"]
title = "AzureChallenge Leeds"

+++
This week I attended the Azure Challenge in Leeds hosted by Microsoft at the Hilton Leeds City. This was a 2 day event focussing on Kubernetes in Azure. There was very little information about the format and content and it felt like it was being kept under wraps!

The only info we were given was

> * Install the Azure CLI 2.0
> * Install Docker
> * Install Kubectl
> * Install Postman - this is optional but useful

I arrived a bit late and sat down with Team 8 that consisted of Lee Dyche, Paul Latham and David Betteridge.

## This was not your average workshop...

It quickly became clear that this wasn't your average workshop and we were in for something good.

> We are not going to be giving your any instructions....

This was a good start.

We were just given a list of goals to achieve to get a cluster up and kick off the application on it and after that there was just a list of challenges. Each with a couple of links to some reading. We were going to learn a lot today...

## Starting off

We had to make a couple of decisions up front

* Use [AKS](https://docs.microsoft.com/en-us/azure/aks/) or [ACS Engine](https://github.com/Azure/acs-engine)?
* What size nodes did we want?

These were pretty simple, we picked AKS as it would get us up and running quicker and some mid range nodes with 1 CPU and 8Gb RAM.

The cluster came up fairly quickly with just one command line and about 10 minutes to wait.

We then had to get the pre-created application up and running in the cluster. All we had was a couple of references to docker hub and github. A couple of yaml files and helm commands later and we were in business.

We added our site to the traffic manager that was being monitored for uptime and we were live!!

## Getting more complex

Now started the challenges. There were lots of them.

There were a few things we knew we had to do out of the list and they were pretty obvious

* Monitoring
* CI/CD

But there were so many choices. I picked monitoring to tackle while Lee and David tackled CI/CD. I decided to start with [Azure OMS](https://docs.microsoft.com/en-us/azure/aks/tutorial-kubernetes-monitor) and see what sort of visibility that gave us. But why stop at one monitoring system? So I installed [DataDog](https://www.datadoghq.com/blog/monitor-kubernetes-docker/) too so I could compare and contrast the information available. (I have since tried Prometheus Operator too, which is great!)

![](/uploads/2018/03/29/k8s-dash-2.png)

So we now had some visibility into our cluster and some way to build and deploy things to it!

## Then it got testing

It wasn't too long before things started going wrong!!

It took a while for us to realise that there was some testing going on and it was taking our cluster down!! Panic!

We had a big problem with some of our components and they had crashed hard. We had some serious downtime and went from first to last in the uptime stakes! :(

![](/uploads/2018/03/29/DumpsterFire2.jpg)

It took us a while to get back up and running and we spent some time investigating the issues we had so they didn't happen again. We decided to switch come components out for Azure services so we could scale and tune them better.

At this point we also decided that we probably needed a dev namespace for testing our improvements.

It was the end of day 1 already! We would have to come back tomorrow and continue tuning our system.

## The Competition

We all came back together the next day and discussed our plan of attack over a coffee. We implemented some more changes and made a couple of changes to our resource profile and implemented some changes to the web front end components.

A couple more tests later and things were hanging together nicely. We seemed to be handling the load to a certain point and then the performance was just tailing off and hitting the floor as something became overloaded.

We took some time to investigate the metrics to see where out bottleneck was that was causing our performance hit. We found that our persistence layer simply couldn't keep up with the number of incoming requests. We made a couple more tweaks and asked for another test.

The result were great. We jumped straight into first place, with a bit of a lead! But it just wasn't enough for us, we wanted more power.

## Performance improvements

We made some pretty decent performance tweaks to our platform. We tuned the Kubernetes resource request and limits to pack more instances into our servers. We also made sure that we had pre-scaled our service in anticipation of the testing we also configured the autoscaler scale up the service more aggressively.

On the Azure side we made sure that our services were tuned a bit better. We spent a lot of time looking at the metrics and tweaking the settings.

We quickly figured out that the persistence was still our big bottleneck. So we did some tweaking of the config to try get more throughput. That should get us some more performance from the front end. Time for another test.. This was going to be good...

![](/uploads/2018/03/29/photofinish.jpg)

It was a draw.... Exactly the same test results as Team 7!!!

## Team8 FTW!

There was going to be a decider... One final test to separate the teams. Again the persistence was still seeming like our big bottleneck. It seemed liked we were already at our maximum performance for the instance, it took us a while to realise that there was still some more tweaking we could do... We decided to go for it and made a quick last minute update to the configuration, out simply we turned the dial up to 11.

![](/uploads/2018/03/29/volume-11-smushed.jpg)

It was the final test.... This was it..

The results were awesome! We were in first place by quite a margin with low response times and very few errors. We had served 1000s of requests during the test and our requests per second were the highest on the leaderboard. We had won in the performance stakes by quite a margin! Even our availability was great for the 2 days of the event.

Our prize was a copy of [Kubernetes: Up and Running](http://amzn.eu/hh5St3h) each. (Which would have been good on day 1) I would highly recommend this book as your first port of call should you want to get up to speed with Kubernetes. Its simple, concise and covers all the bases. As a bonus its co-authored by the founders of Kubernetes so you are getting the info straight from the experts!

The event was awesome. The format was new for me and was a great forum for learning and experimenting and who doesn't like a bit of competition. The challenges were just enough to learn a lot about Kubernetes and Azure over the 2 days.

I want to say a big thanks to [Marcus Robinson](https://twitter.com/techdiction) and the team from Microsoft for hosting the event and inviting us.