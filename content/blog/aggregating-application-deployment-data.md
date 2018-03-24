+++
author = "Scott Guymer"
categories = []
date = "2018-03-24T11:39:23+00:00"
description = ""
draft = true
featured = ""
featuredalt = ""
featuredpath = ""
linktitle = ""
title = "Aggregating application deployment data"
type = "post"

+++
Any company of a decent size and age that creates software at any scale will have gone through a number of "technical evolutions". These come along as you hire new people, create new products or just better ways of doing things come along. The kind of changes I'm talking about are the adoption of different technologies over time and the introduction of new ones. This cuts right across different aspects of software, from languages to frameworks to techniques to tooling. We end up collecting an number of each for various reasons. It may even be a conscious choice where we see that one technology is better suited to solving a particular problem than another or one system is simply not as important and don't need the same investment. The reasons here aren't too important, the fact is you are in this situation and will be for a foreseeable amount of time. 

When it comes to deployment software there are often lots of changes that have happened over time and more than likely more than one tool that is currently in use across your teams.

I know within the team i work in we have about 4

* Jenkins
* GitLab Pipelines
* Heroku Pipelines
* Octopus Deploy

The issue we have is that we have no insight across the teams into what is deploy, to where, when and how. There is very little reporting built into these tools to see across all of your applications and even if there were you would have to switch between applications to see any of this and implementations would likely differ wildly.

What we wanted to do was to bring all of this information together into one place so that we can have some sort of visibility and enable some level of insight into the applications that are being deployed across teams.

## Issues

As you can imagine there is no way that these different systems were going to be sending me any standard format of data. Not even a little bit..

GitLab

Heroku

Within heroku each app is tied to a pipeline which has a stage. There is the notion of coupling an app to a pipeline and stage. However none of this information comes over within the release webhook which is focussed primarily on apps and not pipelines. And at present there doesnt seem to be any webhooks associated to pipeline actions specifically. So we would have to query the specific pipeline couplings for an app after we receive the webhook from the app deployment. Thankfully there is a pipelines API that allows us to query the pipeline couplings from the appId 

[https://devcenter.heroku.com/articles/platform-api-reference#pipeline-coupling-info-by-app](https://devcenter.heroku.com/articles/platform-api-reference#pipeline-coupling-info-by-app "query the pipeline couplings from the appId ")

This looks something like this, with the addition of an Auth header.

    $ curl -n https://api.heroku.com/apps/$APP_ID_OR_NAME/pipeline-couplings \
      -H "Accept: application/vnd.heroku+json; version=3"

## Rationalising the data

From the data above i spent some time thinking about what data was strictly needed to do some basic reporting and insight onto what 

* Date
* Some unique ID
* Environment
* Status
* System

Future nice to haves

* Deployment length - to help identify long and problematic deploys
* Explicit commit sha to track back to source code
* 