+++
categories = ["category", "subcategory"]
date = "2018-03-29T13:27:50+00:00"
draft = true
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

> We are not going to be giving your any instructions....

This was a good start.

We were just given a list of goals to achieve to get a cluster up and kick off the application on it and after that there was just a list of challenges. Each with a couple of links to some reading. We were going to learn a lot today...

## Starting off

We had to make a couple of decisions up front

* Use AKS or ACS?
* What size nodes did we want?

These were pretty simple, we picked AKS as it would get us up and running quicker and some mid range nodes with 1 CPU and 8Gb RAM.

The cluster came up fairly quickly with just one command line and about 10 minutes to wait.

We then had to get the pre-created application up and running in the cluster. All we had was a couple of references to docker hub and github. A couple of yaml files and helm commands later and we were in business!!

We added our site to the traffic manager that was being monitored for uptime and we were live!!

## Getting more complex

Now started the challenges. There were lots of them.

There were a few things we knew we had to do out of the list and they were pretty obvious

* Monitoring
* CI/CD

But there were so many choices! I picked monitoring to tackle while Lee and David tackled CI/CD. I decided to start with Azure OMS and see what sort of visibility that gave us. But why stop at one monitoring system! So I installed DataDog too so i could compare and contrast the information available. (I have since tried Prometheus Operator too, which is great!)

So we now had some visibility into our cluster and some way to build and deploy things to it!

## Load testing

It wasn't too long before things started going wrong!! 

It took a while for us to realise that we were being secretly load tested and it was taking our cluster down!! Panic!

We had a big problem with some of our components and they had crashed hard. We had some serious downtime and went from first to last in the uptime stakes! :(

It took us a while to get back up and running and we spent some time investigating the issues we had so they didn't happen again. We decided to switch come components out for Azure services so we could scale and tune them better. We ended up with an EventHub and CosmosDB instance.

At this point we also decided that we probably needed a dev namespace for testing our improvements.

It was the end of day 1 already! We would have to come back tomorrow and continue tuning our s

## The Competition

A couple more load tests later and things were hanging together, 

## Performance improvements

We made some pretty decent performance tweaks to our platform. We tuned the kubernetes resource request and limits to pack more instances into our servers. We also made sure that we had pre-scaled our service to take the load and made the autoscaler scale up the service more aggressively. 

On the Azure side we made sure that our event hub was scaled up to a decent amount of throughput units and that auto inflating was enabled to inflate to the maximum possible.

We quickly figured out that cosmos was still our big bottleneck. So we did some tweaking of the partitioning to make it partition and get more throughput. That should get us some more performance. Time for another load test.. This was going to be good...

![](/uploads/2018/03/29/photofinish.jpg)

It was a draw.... Exactly the same stats as Team 7!!!

## Team8 FTW!

There was going to be a decider... One final load test to separate the teams. Cosmos was still seeming like our big bottleneck. It seemed liked we were already at our maximum throughout units for the instance, it took us a while to realise that if we clicked unlimited when creating our collection we could go to 100,000 tpu... We quickly updated the collection to unlimited and turned the dial up to 11.

![](/uploads/2018/03/29/volume-11-smushed.jpg)

It was the final load test.... This was it.. The decider.

In the end we won a copy of [Kubernetes: Up and Running](http://amzn.eu/hh5St3h) each! I would highly recommend this book as your first port of call should you want to get up to speed with kubernetes. Its simple, concise and covers all the bases. As a bonus its co-authored by the founders of kubernetes so you are getting the info straight from the horses mouth!