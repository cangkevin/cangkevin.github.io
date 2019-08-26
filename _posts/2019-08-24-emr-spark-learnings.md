---
layout: post
title: "Maximizing Spark on EMR Utilization in a Dynamic Environment"
author: Kevin Cang
excerpt_separator: <!--preview-->
tags: ['Spark', 'EMR', 'Optimization']
---
For the past year, I had the opportunity to work on an enterprise-wide data ingestion platform. One core aspect of this platform was powered by Spark on YARN running on EMR clusters. Prior to working on this platform, I had zero knowledge on the aforementioned technologies.

There was a period of time when the platform was costing hundreds of thousands of dollars per month, with the worst peak at just over $1 million. Despite my lack of knowledge at the time, I knew that it was preposterous that such expenditures could be justified. I simply believed there is no way our platform should be costing this much.
Hence, I saw the opportunity for learning had presented itself which provoked me to jump straight into the rabbit hole of finding how to optimize this expensive platform.

<!--preview-->
# Grasping the situation
First, I had to get an idea of what the utilization of the EMR clusters looked like. As with any engineering problem in corporate, if management can't be convinced that it's a problem to begin, then traction towards performing the work won't be gained.

[Ganglia](http://ganglia.sourceforge.net/) was the tool of choice I selected since EMR offered support for installing the application and it's a rather stable software for distributed monitoring. Using this tool directly on our production clusters proved my speculation correct: our EMR clusters were underutilized by a significant amount.
We operated four clusters that handled various use cases. On average, our EMR cluster that receives the most traffic was achieving about 20% sustained CPU utilization and roughly 30% memory utilization. At this point, it was obvious that we had a problem that was some combination of an overprivisioning and underutilization.

Next, I had to understand why our cluster utilization was so low. In order to understand that, I had to understand the various use cases that our platform typically handles, which gets into the question of how do our users actually utilize our platform.
It turns out that our platform covered an uncomfortably broad range of use cases: various Spark applications (I'll refer to these as "jobs") that rolled through could be handling data whose size could range anywhere between tens of MBs to several TBs. Additionally, some jobs only called for validation of the data itself with a simple passthrough if it's valid while some applications required intensive computations to be carried out. Another significant aspect was the scale that we were operating at: we had reached the point where we processing somewhere between 40k-50k jobs per day that were unevenly distributed across four clusters.

Also, for some reasons unknown, there was a lack of tuning in our Spark settings and configurations; a lot of them were left on default values, which is definitely not suitable for an enterprise-wide platform in a production environment.

Given these discoveries, I knew that there were plenty of opportuniies to tune the platform. But I was not exactly confident that typical conventions for tuning Spark configurations were going to be enough; my preliminary research into tuning Spark applications [makes an assumption that we're processing one job at a time](https://aws.amazon.com/blogs/big-data/best-practices-for-successfully-managing-memory-for-apache-spark-applications-on-amazon-emr/), which was not the case for us and would not have sufficied in keeping up with the volume that we were handling. However, it was a starting point to begin the optimization work.


# Spark optimizations
## Executor configuration
[The method of tuning executor settings often show up when you run a Google search on tuning Spark applications](https://blog.cloudera.com/how-to-tune-your-apache-spark-jobs-part-2/). It is a simple but rather fool-proof way to get increased utilization of your cluster and it only relies on understanding how much compute resources you have available and simple arithmetic.
### Executor cores (`spark.executor.cores`)
It's typically recommended to pick 2 to 5 cores. 1 core is not recommended since you will not be able to take advantage of processing multiple tasks in the same JVM and having more than 5 cores will start to be detrimental towards HDFS throughput.
A nice rule of thumb to keep in mind is to pick a number that evenly divides the number of total cores an a single instance. For instance, if you're running r4.8xlarge (32 cores) instances in your cluster, then it would be good to pick 2 or 4 as the executor cores configuration. This helps in reducing, if not eliminating, fragmentation when YARN allocates cores for each container.

The above is directly applicable if you're using EMRs because AWS actually [populates the configuration value `yarn.nodemanager.resource.cpu-vcores` to be the total number of CPU cores of the underlying EC2 instance](https://docs.aws.amazon.com/emr/latest/ReleaseGuide/emr-hadoop-task-config.html). In general, you should base your selection on what the actual value of `yarn.nodemanager.resource.cpu-vcores` is.
### Executor memory (`spark.executor.memory` and `spark.executor.memoryOverhead`)
Once the number of executor cores has been chosen, executor memory can be computed by:
\\[m_{e,optimal}=\frac{\mbox{yarn.nodemanager.resource.memory-mb}}{\bigg(\frac{\mbox{yarn.nodemanager.resource.cpu-vcores}}{\mbox{spark.executor.cores}}\bigg)}\\]

In short, the amount of memory offered to YARN on a single instance divided by the theoretical max number of executors that can be allocated on a single instance, assuming each executor allocates `spark.executor.cores` each. The value of this calculation is optimal in terms of allocating the underlying instance's resources and reducing fragmentation on the memory side of things.

One important thing to remember is that this optimal value is for the __entire__ YARN container allocation. Hence, it should be ensured that: \\[\mbox{spark.executor.memory} + \mbox{spark.executor.memoryOverhead} = m_{e,optimal}\\]

## Driver configuration
Driver configuration will often depend on the what you are doing in your Spark application and whether you use a lot of driver intensive operations in your code. In the context of our situation, we did not engage in driver-intensive operations. Hence, we proceeded with deriving configurations in a similar manner as the executor configurations.
### Driver cores (`spark.driver.cores`)
In most cases, you should be able to get away allocating just 1 core for the driver. The driver itself shouldn't really participate in activity that requires a significant amount of cores. However as a reminder, __this is highly dependent on your application's use case__.
### Driver memory (`spark.driver.memory`)
Following the same method as deriving executor memory, driver memory can be computed by:
\\[m_{d,optimal}=\frac{\mbox{yarn.nodemanager.resource.memory-mb}}{\bigg(\frac{\mbox{yarn.nodemanager.resource.cpu-vcores}}{\mbox{spark.driver.cores}}\bigg)}\\]

Once again, \\(m_{d,optimal}\\) is the optimal value for the __entire__ YARN container allocation for the driver. Hence, it should be ensured that:
\\[\mbox{spark.driver.memory} + \mbox{spark.driver.memoryOverhead} = m_{d,optimal}\\]

## Benefit of executor and driver configurations
Following the above executor and driver configurations, this provides a good compromise in having balanced executors that can process both small and large jobs and efficiently allocating cluster resources, ensuring that you're able to leverage every single core and GB of memory that you are paying for.

## Dynamic allocation
[When provisioning an EMR, dynamic allocation is enabled by default](https://docs.aws.amazon.com/emr/latest/ReleaseGuide/emr-spark-configure.html). In our situation, it made sense to go ahead with using dynamic allocation due to dynamic nature of our workload (i.e. significant spike in traffic around midnight during business days and very little traffic during the weekends).
I couldn't see a strong reason for going towards a fixed amount of executors for every single job. Doing so would lead to overprovisioning of cluster resources for small jobs, which would negatively impact cluster utilization. On the opposite end of the spectrum, it would lead to underprovisioning for larger jobs, which would negatively affect how quickly the job would get processed. This would be unacceptable in cases where the completion of the job would often trigger downstream processes or it has to adhere to a certain SLA.

Majority of the default settings for dynamic allocation were fine. However, there was one configuration that needed careful consideration, especially if the clusters are operating on a shared environment model for tens of thousands of jobs throughout the day.
### Establishing an upper bound (`spark.dynamicAllocation.maxExecutors`)
[By default, this configuration is set to `infinity`](https://spark.apache.org/docs/latest/configuration.html#dynamic-allocation) which is probably not a good idea as it can lead to a scenario where a number of large jobs running concurrently could potentially occupy the entire cluster. This will increase latency for other jobs as they would be waiting for resources to become available either through auto-scaling or the release of resources from jobs that finish. Auto-scaling would impact our costs and waiting on jobs to finish could potentially cause SLA breaches, both being not ideal scenarios.

To handle this scenario, `spark.dynamicAllocation.maxExecutors` needed to be set. Some guidelines for setting this value:
1. Consider how much a single application can occupy the cluster
    - If the cluster size is fixed, think in terms of the max amount of resources (i.e. no more than 10 percent of the cluster resources).
    - If the cluster size is dynamic (clusters with auto-scaling), think in terms of max amount of resources when the cluster is at its minimum and maximum auto-scaling limit (i.e. no more than 10 percent of cluster resources at maximum size and no more than 20 percent of cluster resources at minimum size).
2. Convert the percentages into their corresponding cores and memory allocation values
3. Convert the corresponding cores and memory allocation values to the number of executors

The above guidelines should help you arrive at what value makes sense for the maximum number of executors a single application should take.

# Hadoop YARN optimizations
## Capacity Scheduler
Hadoop offers a [capacity scheduler](https://hadoop.apache.org/docs/current/hadoop-yarn/hadoop-yarn-site/CapacityScheduler.html) that is designed for multi-tenacy and maximizing utilization of the cluster. Much like anything in the Hadoop and Spark ecosystem, there are plenty of configuration knobs to tinker with. However, there were two configurations that we explicitly chose to incorporate.
### `yarn.scheduler.capacity.resource-calculator`
This configuration is used to control how resources are compared in the scheduler. EMR sets the default to be `DefaultResourceCalculator` which is implementation that only observes cluster memory. `DominantResourceCalculator` is another implementation that factors in multiple resources such as memory and cores. It is usually a good idea to use `DominantResourceCalculator` as purely observing cluster memory has its limitations.
### `yarn.scheduler.maximum-am-resource-percent`
By default, EMR sets the value of this configuration to `0.5`, which means that potentially half of the cluster resources can be allocated towards to ApplicationMasters i.e. the drivers of Spark applications. This particular configuration is a bit tricky as there's no ideal value. Set it too low and the number of applications running on the cluster concurrently would be very small; set it too high and you run the risk of a scenario where majority of cluster resources are allocated for the drivers and only a small amount of resources remain for actual allocation of executors.

 Some guidelines to reach a reasonable value:
- If at any point in time the amount of applications running concurrently on the cluster is likely to be small, then it would make sense to lean towards a smaller percentage and vice-versa.
- If applications tend to take up a lot of executor resources, then it would make sense to lean towards a smaller percentage and vice-versa.

# EMR configuration optimizations
## Enabling auto-scaling
Autoscaling is rather necessary for achieving healthy utilization and cost savings, especially for dynamic workloads. Strategies for auto-scaling in EMR clusters shouldn't be approached in the same way as setting up an auto-scaling policy for a group of EC2 instances; [this wonderful AWS blog post provides insight as to configuring optimal auto-scaling configurations for EMRs](https://aws.amazon.com/blogs/big-data/best-practices-for-resizing-and-automatic-scaling-in-amazon-emr/).

Some guidelines:
- Understanding HDFS utilization of your clusters can help with whether auto-scaling should be fine-tuned for either core nodes or task nodes
- Avoid using a single metric for auto-scaling triggers; do take into consider a combination of metrics instead

# Results of optimization
As a result of applying the above optimizations, I was able to shed off nearly $400k per month on our EC2 costs! Not bad considering the changes weren't terribly complex and there were still plenty of things to tweak to drive down the costs even more.

# Future areas of work
Although the optimization work had a profound impact on both costs and scale, I believe that this could have been pushed even further if we had better tools for monitoring and analytics in place. The following would be things that I feel would have been nice to have, if not essential:
- Detailed monitoring of various metrics associated with EMR cluster as well as on each individual node in the cluster
- Data on the resource usage of jobs that roll through the clusters to which analysis can be done to understand various peaks and valleys of traffic on various time scales (i.e. hourly, daily, monthly)

Having the above can help you see things on a more granular level, especially in production, which can yield insights that can give you confidence to carry out even further optimization work.
