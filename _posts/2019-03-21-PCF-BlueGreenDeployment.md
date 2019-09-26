---
layout:     post
title:      PivotalCloudFoundry(二)
subtitle:   PCF Blue-Green Deployment
date:       2019-03-21
author:     LANY
header-img: img/post-20190306-bg.jpg
catalog: true
tags:
    - Pivotal
    - PAAS
    - Cloud Foundry
---
# PCF Blue-GreenDeployment

## What's Blue-GreenDeployment?

> Blue-green deployment is a technique that reduces downtime and risk by running two identical production environments called Blue and Green.

> At any time, only one of the environments is live, with the live environment serving all production traffic. For this example, Blue is currently live and Green is idle.

> As you prepare a new version of your software, deployment and the final stage of testing takes place in the environment that is not live: in this example, Green. Once you have deployed and fully tested the software in Green, you switch the router so all incoming requests now go to Green instead of Blue. Green is now live, and Blue is idle.

> This technique can eliminate downtime due to application deployment. In addition, blue-green deployment reduces risk: if something unexpected happens with your new version on Green, you can immediately roll back to the last version by switching back to Blue.


## Blue-Green Deployment whih Cloud Foundary Example
Prerequisites:

* need know Deploying with Application Mantfest.yml.
[Detail](http://docs.pivotal.io/pivotalcf/1-12/devguide/deploy-apps/manifest.html)
* must have the CLI and MAVEN installed.
* must provious for simple application that you push.

### Step 1: Push App

Use the CLI push the application,name the "Blue" with the hostname 'example-demo'.

```   
$ cf push Blue -n example-demo
```
As shown in the graphic below:

* Blue now is running on PCF.
* The CF route sends all traffic for example-demo.apps.xkpcf.local traffic to Blue.

![blue](https://raw.githubusercontent.com/HiLany/HiLany.github.io/master/img/post-2019-0321-2.png)

### Step 2: Update App and Push

Make a change to your application.First,replace the word "Blue" with "Green",
then use `mvn package` to repackage your application, use `cf push` to push your app again,name the "Green" with the hostname 'example-demo-temp'.

```
$cf push Green -n example-demo-temp
```
As shown in the graphic below:

* Blue and Green are both running on PCF.
* The CF Router continue sends all traffic for example-demo.apps.xkpcf.local traffic to Blue.The Route also sends any traffic for example-demo-temp.apps.xkpcf.local to Green.

![blue-green](https://raw.githubusercontent.com/HiLany/HiLany.github.io/master/img/post-2019-0321-1.png) 

### Step 3: Map Origin route to Green

Now, both of application you push are running on PCF,switch the router so all incoming requests go to the Green app and the Blue app. Do this by mapping the original URL route (example-demo.apps.xkpcf.local) to the Green application using the cf map-route command.

```
$cf map-route Green apps.xkpcf.local -n example-demo
```

After command:

* The CF Router continues sending traffic for example-demo.apps.xkpcf.local to Green.
* Within a few seconds,the CF Router begins load balancing traffic for example-demo.apps.xkpcf.local between Blue and Green.

![map](https://raw.githubusercontent.com/HiLany/HiLany.github.io/master/img/post-2019-0321-4.png)

### Step 4 : Unmap route to Blue

Once you verify Green is running as you excepted,you need stop routing requests to Blue, use command below:

```
$cf unmap-route Blue apps.xkpcf.local -n example-demo
```
After command ,the CF router stops sending requests to Blue and All Traffic for example-demo.apps.xkpcf.local is sent to Green.

![unmap](https://raw.githubusercontent.com/HiLany/HiLany.github.io/master/img/post-2019-0321-5.png)

### Step 5 : Remove temporary route to Green

You can use `cf unmap-route` to remove the temporary route `example-demo-temp.apps.xkpcf.local` from Green.

```
$cf unmap-route Green apps.xkpcf.local -n example-demo-temp
```
The route can be deleted using cf delete-route or reserved for later use. You can also decommission Blue, or keep it in case you need to roll back your changes.
![green](https://raw.githubusercontent.com/HiLany/HiLany.github.io/master/img/post-2019-0321-3.png)

Congratulations! you have finshed this exercise.

    
   