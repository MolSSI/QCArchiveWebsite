---
title: "The state of QCArchive: March 2019"
date: 2019-03-23
description: "QCArchives current stat "
pageDescription : "Current state of QCA"
type : "post"
image : "images/collaborators/OpenFFVector.png"
imageAlt : "images/collaborators/OpenFFVector.svg"
author : "Levi Naden"
authorImage : "images/people/nadenhead.jpg"
authorDescription : "Levi is a MolSSI Software Scientist and a QCA developer"
#authorSocial : 
#  - icon : "tf-ion-social-facebook"
#    URL : "#"
#    
#  - icon : "tf-ion-social-twitter"
#    URL : "#"
#    
#  - icon : "tf-ion-social-instagram"
#    URL : "#"
#    
#  - icon : "tf-ion-social-dribbble"
#    URL : "#"
---

QCArchive has entered closed beta and is now computing production calculations in as part of the Open Forcefiled 
Initiative's thrusts.

We had entered closed beta at the start of this month and have since then created compute managers attached to each of  
the Virginia Tech ARC clusters, several Intel Knights Landing machines available to MolSSI, and our collaborators at 
the Open Forcefiled Initiative have created managers on the Memorial Sloan Kettering' and UC Berkly's HPC HPC 
cluster. All the managers are computing jobs and adding data to the central Fractal database. 

> Managers are easy to start up from command line and a config file

{{< image src="../../images/blog/manager.gif" srcAlt="../../images/blog/manager.gif" >}}

The manager creation process has been vastly simplified with an improved CLI for starting them based on a config file. 
We the inclusion of Dask, configuring your own managers to execute jobs on your own cluster is much simpler than it was 
before the beta.

We welcome community feedback on this new feature to help make it better from both ease of use and features.
