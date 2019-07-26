---
title: "Running multi-site, heterogeneous, compute that our users manage: Queue Managers evolution and lessons"
date: 2019-07-26
description: "QueueManager Evolution"
pageDescription : "The QCArchive Queue Managers and how they have evolved"
type : "featured-post"
image : "images/QCArchiveLogo.svg"
author : "Levi Naden"
authorImage : "images/molssi-logo.svg"
authorDescription : "QC Archive Developer"
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

QCArchive at its core is a database for storing, organizing, and indexing quantum chemistry calculations. Although there 
is a need to add existing data manually, adding every calculation to the database manually would be a tedious, and error 
prone way of inputting data, especially for new calculations. Fortunately, QCArchive has a compute queueing, 
and provision system running on the Fractal Servers by default. Preparing and staging these jobs required plenty 
of engineering on its own, but today I wanted to talk about the other major barrier: consuming and actually running 
the jobs. 

How hard would it be for you, the reader, to run a single command line task from your computer which had an input file 
of options and parameters? Likely not a challenging task if you knew what the "task" (and required program) in question 
was. 
Independent of any computational resources needed for the task, you would expect this task to run, generate its output, 
and report success or failure. 

Let's step the task requirements up now: what if the task in question needed more resources (CPU cores, 
memory, etc.) than your computer had to complete in a reasonable time, or even run? You could find a more powerful 
computer to run the task on, or you could take advantage of a cluster or cloud solution available to most the 
computational sciences in some capacity. Whether it be a local university cluster, a private cluster, a government 
funded supercomputing center, or a cloud provided compute server; options are available to run this more resource 
intensive task. However, not only did this introduce new hardware to the difficulty, it also introduced additional 
complexity to running the computation in the form of physical or remote machine access and multi-user machines which 
might have a job queueing system. But, for one job, you can overcome this by manually if need be.

Now let's jump a few steps and bring this to the QCArchive level of problems. We now have 100,000 to 10,000,000 order 
of magnitude of tasks which need to be computed at some point. Some of these tasks might be able to be run on desktop 
power machines, some might need a hundred cores and TB of memory. These tasks range in the which program they need to 
run in, and may be submitted from users somewhere in the Seychelles or in the Anchorage, Alaska USA. Further, someone 
may have their own Fractal instance and want to compute their own proprietary jobs on secure hardware. Take this 
scale of jobs into account, combine it with the problem of diverse hardware and scheduling software of any compute site, 
and finally remember that the compute can be handled by different software who's inputs and outputs have to be parsed 
for QCArchive format; and it makes the question I posed at the onset daunting: How hard would it be to compute these 
jobs? With all of this in mind, the question QCArchive set out to answer was "How do we solve this?"


{{< image src="../../static/images/blog/QCQuManagerNetwork.png" srcAlt="../../static/images/blog/QCQuManagerNetwork.png" >}}

QCArchive's answer to the question was to solve each problem in abstract, isolated layers. Attempting to come up with 
a monolithic solution to these problems is unfeasible. We did not have the expertise nor the resources to dedicate to 
coming up with a universal compute solution which runs on all hardware with all intermediary software at all scales. 
However, we can instead break up the steps into smaller chunks and leverage existing tools who do have the expertise on 
any one layer. The image above shows the communication levels between Fractal's Server, and the actual compute being 
carried out, however, the list below talks about the layers and how they are isolated from each other as far as their 
duties. 

1. The Fractal Server prepares tasks in a flat list and knows what software is required to execute the compute.
    * Once a task is handed off, the Server locks it internally so nothing else can compute them while it waits. 
2. The Queue Manager starts on the hardware the user wants to run and consumes jobs from the Server.
    * The Queue Manager tells the Server what compute software it detects so it can only get tasks it can run.
    * Tasks are formatted in the Queue Manager and handed off to Adapter for actual task deployment on the hardware.
3. An Adapter is run underneath the Queue Manager which takes tasks and knows how to interface with the actual hardware 
   platform. These are things like [Parsl](https://parsl.readthedocs.io/en/latest/index.html), 
   [Dask Jobqueue](https://jobqueue.dask.org/en/latest/), 
   [Balsam](https://balsam.readthedocs.io/en/latest/), etc.
   * Adapters are intentionally NOT software developed or maintained bt the QCArchive team. This is software which 
     is publicly available and developed by 3rd parties for the express purpose of running on remote or cluster 
     hardware.
   * The Adapters are chosen to interface with Schedulers for cluster such as SLURM, PBS/Torque, and LSF.
   * The Queue Managers provide an interface to the Adapter software, and submit tasks to await the individual adapters 
     to return their result.
4. QCEngine is then executed on the hardware allocated by the Adapters and formats the task input data into a way 
   which the actual Quantum Chemistry software can interpret and execute.
5. The Quantum Chemistry software is then run with the inputs and the result is then passed back up the stack 
   all the way to the server
   
Each of the layers above exists in isolation from one another and only interfaces with adjacent layers to pass data 
back and forth. By designing the Queue Manager stack in this way, we can rely on the 3rd party software to handle the 
daunting challenge of deploying on very diverse hardware scenarios. With some engineering on the Server side, multiple 
Queue Managers can connect at the same time, allowing for different hardware and software requirements to be consumed 
concurrently with just different setup and configuration per manager.

{{< image src="../../static/images/blog/QCQuManagerBasic.png" srcAlt="../../static/images/blog/QCQuManagerBasic.png" >}}

This approach is not without its problems however. There is no magic Adapter which will work in every hardware setting. 
Nor is there a miracle auto-hardware-and-software detection algorithm to know the configuration of every machine and 
software stack. There is still a user setup phase required, but we have made making improvements over the past months 
to make it easier than ever to get up and going. 

The initial setup of the Queue Managers relied purely on a CLI and a pre-existing Fireworks installation or simple Pool
executor for computation. Since then, we have added an 
[option rich YAML configuration file](https://qcfractal.readthedocs.io/en/latest/managers_samples.html) format which 
relies on a common language  for hardware configurations to translate between Adapters. We have Adapters for 
[Parsl](https://parsl.readthedocs.io/en/latest/index.html), [Dask Jobqueue](https://jobqueue.dask.org/en/latest/), 
[Balsam](https://balsam.readthedocs.io/en/latest/), [FireWorks](https://materialsproject.github.io/fireworks/), and the 
native [Python ProcessPoolExecutor](https://docs.python.org/3/library/concurrent.futures.html).

The isolated nature of the layers has also meant that users have only needed to interact with the Queue Manager layer to
provide their own resources for QCArchive task consumption. This has trimmed adoption time per-site to be on the order 
of hours rather than days as the users only have to learn one layer (in combination with the knowledge of their local 
hardware), and us developers have been able to focus on small areas of code when bugs need fixed and features are 
required. With that, we have seen users spin up Queue Managers at Virginia Tech's ARC four clusters, UC Irvine HPC 
cluster, GA Tech PACE clusters, and Memorial Sloan Kettering Cancer Center's HPCs, contributing in total 
**{FILL IN MISSING AND TOTAL CPU/JOB TIME}**. We have also had a user start a manager on the ORNL Titan supercomputer. 
The very diverse setups have also led to varying manager configurations. For one user, it was easier for them and their 
system to submit jobs directly to their cluster's scheduler, then start many Process Pool Executor Queue Managers from 
within those jobs. Although the conditions which made this easier are an edge case, its still a valid Queue Manager 
setup and let us see if our Server can handle hundreds of managers all connecting at once (it can without issue).

Our [Balsam adapter](https://balsam.readthedocs.io/en/latest/) provided evidence that our software stack is not only 
stable at scale, but is great for consuming large scale compute as well. Balsam is a HPC Workflows and Edge Service 
tool which is running on the Argonne Leadership Computing Facility supercomputer Theta and is designed to help users 
manage complex workflows, and better optimize consumption of the many-node jobs that Theta requires. With the help from 
[Misha Salim](https://www.alcf.anl.gov/staff-directory/misha-salim) at Argonne, we were able to write a Queue Adapter 
which would submit jobs to the Balsam service. Because Theta required that several hundred whole nodes 
be requested at minimum, this would be a chance for us to test our software stack's ability to not only serve and 
receive tasks at a high volume, but also could we consume the vast resources available to us? The answer to both 
questions was a resounding "yes." Within six hours of wall clock time, we submitted a total of **{FILL IN}** tasks to the 
802 nodes system consuming almost **{300,000 CHECK AND FILL IN}** core hours. At one point in the run we even had to 
populate the queue with more tasks because the one Queue Manager running through Balsam on Theta was consuming and 
completing them so fast we almost drained the server.

Overall, we think our approach to the diverse hardware and software resources has been and will continue to be 
successful. Users are able to spin up Queue Managers with minimal input from us developers by leveraging their 
knowledge of their systems to configure a Manger once and go. We've been able to run high volumes of both compute and 
Queue Manager counts through our stack with almost no modifications to the Server side. There have been hurdles along 
the way as we have found bugs in the code and needed to make the Queue Managers' interfaces to the Adapter layer more 
diverse to suit everyone's needs. However, with active user feed back and much appreciated patience, we have been able 
to resolve most of those as they have come up. 

When we first made the Queue Managers available, the MolSSI public instance of the Fractal Server only had **{FILL IN}** 
tasks computed, run mostly by us, the developers. Since **{FILL IN DATE}** when we started requesting our early adopters 
to use our Queue Manager, we have now added **{FILL IN}** computations of to our database representing 
**{FILL IN}** core hours of compute time across **{FILL IN}** physical site and we only expect that number to increase. 
