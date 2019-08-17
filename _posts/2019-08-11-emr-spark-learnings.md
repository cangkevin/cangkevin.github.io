---
layout: post
title: "Maximizing Spark on EMR Utilization in a Dynamic Environment"
author: Kevin Cang
excerpt_separator: <!--preview-->
tags: ['Spark', 'EMR', 'Optimization']
---
For the past year, I had the opportunity to work on an enterprise-wide data ingestion platform. One core aspect of this platform was powered by Spark on YARN running on Amazon EMR clusters. Prior to working on this platform, I had no knowledge on Spark, YARN, or EMR.

There was a period of time when the platform was costing hundreds of thousands of dollars per month to operate, with the worst peak at just over $1 million. Even with my lack of working knowledge of the technolgy at the time, I knew that it was preposterous that such expenditures could be justified; I simply believed that there is no way that this should be costing this much.
Once I had convinced myself, the opportunity for learning and cost savings had presented itself and provoked me to jump straight into the rabbit hole of finding how to optimize this expensive platform.

<!--preview-->
# Grasping the situation
First thing to be done was to get a snapshot of what the utilization of the EMR clusters looked like. As with any engineering problem in corporate, if management can't be convinced that it's a problem to begin, then traction towards performing the work won't be gained.

[Ganglia](http://ganglia.sourceforge.net/) was the tool of choice I selected since AWS offered support for installing the application and it's a well-polished software for distributed monitoring. Using this tool directly on our production clusters proved my speculation correct: that our EMR clusters were underutilized by a significant amount.
On average, our EMR cluster that receives the most traffic was achieving about 20% sustained CPU utilization and roughly 30% memory utilization. At this point, it can be acknowledged that we have a big problem that is some combination of an overprivisioning and underutilization.

The next step was to understand why our cluster utilization was so low. In order to understand that, we had to understand the various use cases that the platform typically encounters, which starts to get into the question of how do our users actually utilize our platform.
It turns out that our platform covered an uncomfortably broad range of use cases: various Spark applications (I'll refer to these as "jobs") that rolled through could be handling data whose size could range from tens of MBs to several TBs. Additionally, some jobs only called for validation of the data itself with a simple passthrough if it's valid while some applications required intensive computations to be carried out. Another significant aspect was the scale that we were operating at: we had reached the point where we processing somewhere between 40k-50k jobs per day that were unevenly distributed across 4 clusters.

Also, for some reasons unknown, there was a lack of configurations tuning in our Spark settings; a lot of the Spark configurations were default values, which is definitely not suitable in an enterprise environment.

Given these discoveries, I knew that there was plenty of opportunity to tune and improve the platform. But I was not exactly confident that typical conventions for tuning Spark configurations were going to be enough; typical tuning makes a large assumption that we're processing one job at a time, which was not the case at the time and would not have sufficied in keeping up with the volume that we were processing at. However, it was at least a starting point to begin the optimization work.


# Spark optimizations
## Executor configuration
The method of tuning executor settings often show up when you run a Google search on tuning Spark applications. It is a simple but rather fool-proof way to get increased utilization of your cluster and it only relies on understanding how much compute resources you have available and simple arithmetic.
### Executor cores (`spark.executor.cores`)
It's typically recommended to pick 2 to 5 cores. 1 core is not recommended since you will not be able to take advantage of processing multiple tasks in the same JVM and having more than 5 cores will start to be detrimental towards HDFS throughput.
A nice rule of thumb to keep in mind is to pick a number that evenly divides the number of total cores an a single instance. For instance, if you're running r4.8xlarge (32 cores) instances in your cluster, then it would be good to pick 2 or 4 as the executor cores configuration. This helps in reducing, if not eliminating, fragmentation when YARN allocates cores for each container.

The above is directly applicable if you're using EMRs because AWS actually populates the configuration value `yarn.nodemanager.resource.cpu-vcores` to be the total number of CPU cores of the instance. In general, you should base your selection on what the actual value of `yarn.nodemanager.resource.cpu-vcores` is.
### Executor memory (`spark.executor.memory` and `spark.executor.memoryOverhead`)
Once the number of executor cores has been chosen, executor memory can be computed by:
\\[m_{e,optimal}=\frac{\mbox{yarn.nodemanager.resource.memory-mb}}{\bigg(\frac{\mbox{yarn.nodemanager.resource.cpu-vcores}}{\mbox{spark.executor.cores}}\bigg)}\\]
In short, the amount of memory offered to YARN on a single instance divided by the theoretical max number of executors that can be allocated on a single instance, assuming each executor allocates `spark.executor.cores` each. The value from this calculation is optimal in terms of allocating the underlying instance's resources and also helps to reduce fragmentation on the memory side of things.

One important thing to note is that this optimal value is for the __entire__ YARN container allocation. Hence, it should be ensured that: \\[\mbox{spark.executor.memory} + \mbox{spark.executor.memoryOverhead} = m_{e,optimal}\\]

## Driver configuration
Driver configuration will often depend on the what you are doing in your Spark application and whether you use a lot of driver intensive operations in your code. In the context of this situation, we did not engage in very driver-intensive operations and hence proceeded with deriving configurations in a similar manner as the executor configurations.
### Driver cores (`spark.driver.cores`)
In most cases, you should be able to get away allocating just 1 core for the driver. The driver itself shouldn't really participate in activity that requires a significant amount of resources.
### Driver memory (`spark.driver.memory`)
Following the same method as deriving executor memory, driver memory can be computed by:
\\[m_{d,optimal}=\frac{\mbox{yarn.nodemanager.resource.memory-mb}}{\bigg(\frac{\mbox{yarn.nodemanager.resource.cpu-vcores}}{\mbox{spark.driver.cores}}\bigg)}\\]

Once again, \\(m_{d,optimal}\\) is the optimal value for the __entire__ YARN container allocation for the driver. Hence, it should be ensured that:
\\[\mbox{spark.driver.memory} + \mbox{spark.driver.memoryOverhead} = m_{d,optimal}\\]

## Benefit of executor and driver configurations
Following the above executor and driver configurations, this provides a good compromise in having balanced executors that can process both small and large jobs and efficiently allocating cluster resources, ensuring that you're able to leverage every single core and GB of memory that you are paying for.

## Dynamic allocation
When provisioning an EMR, dynamic allocation is enabled by default. In our situation, it made sense to go ahead with using dynamic allocation due to broad range of jobs that the platform processed.
I couldn't see a strong reason for going towards fixed amount of executors for each and every single job. Doing so would lead to overprovisioning of cluster resources for small jobs, which would negatively impact cluster utilization. On the opposite end of the spectrum, it would lead to underprovisioning for larger jobs, which would negatively affect how quickly the job would get processed. This would be unacceptable in cases where the completion of the job would often trigger downstream processes or it has to adhere to a certain SLA.

Majority of the default settings for dynamic allocation were fine. However, there was one exception that needed careful consideration, especially if the clusters were operating in a shared environment model for tens of thousands of jobs coming throughout the day.
### Establishing an upper bound (`spark.dynamicAllocation.maxExecutors`)
By default, this configuration is set to `infinity` which is probably not a good idea as it can lead to a scenario of a number of large jobs running concurrently that could potentially occupy the entire cluster. This then increases latency for other jobs as they would be waiting for resources to become available and could potentially cause SLA breaches.

To handle this potential scenario, `spark.dynamicAllocation.maxExecutors` needed to be set. Some guidelines for picking a reasonable value:
- If the cluster size is fixed, then it might be reasonable to think about how much a single application can occupy a cluster (i.e. no more than 10 percent of the cluster).
