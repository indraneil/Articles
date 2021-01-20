# How we created a disaster recovery strategy for a large service
## Preface
I spent 1.5 years re-creating a disaster recovery (DR) story for a widely available service which runs on Azure. They had a pre-existing process but exercising it took days and they still ended up with data consistency issues.

Here are my notes on what it took to get them to where they finally ended up (a reliably running disaster recovery in under four hours).
It took dedicated work from a dozen engineers (out of 300) over 1.5 years for it to get there. _The cost of infrastructure effectively doubled, but we moved the service from 3-9s availability to 4-9s availability along the way._

## The original problem statement
I stumbled upon the problem because I ported their compute story [from Azure PAAS to Azure VMSS](https://github.com/indraneil/VmssFromPaas/blob/master/Introduction.md) + Service fabric.
> When I moved a scale unit to the new stack using the preexisting DR process, it failed spectacularly when tried.

It relegated my work to only new scale units, which wasn't a very compelling story because that meant straddling 2 stacks indefinitely.
That made me examine the problem and see if it could be solved so that it helped everyone.

### Understanding the service
The service is split up into regional scale units, each of which is a collection of micro-services running on a common [service fabric based compute cluster](https://docs.microsoft.com/en-us/azure/service-fabric/service-fabric-azure-clusters-overview).
- The micro-services use independent storages and code repositories and talk to each other using HTTPS. Quite a few microservices ran with [in-memory stateful data](https://docs.microsoft.com/en-us/azure/service-fabric/service-fabric-reliable-services-reliable-collections), which meant that data in the service was the primary source of truth.
- They share common libraries around logging/authentication/service lifecycle/storage etc.
- They share the same CI/CD frameworks, and monitoring dashboards, but all other decisions are taken independently.

### What appeared to be the problem
- Each micro-service had its own process around bootstrapping itself into a new region, especially around populating key vaults with certificates and secrets.
- They had taken different approaches to resiliency (using geo replication, availability zones etc.).
- There wasn't an automated way to locate the build numbers, and map them to the repositories and release branches for each micro-service that needed to be redeployed in the case of a DR.
- When doing just compute DRs, different services seemed to take varying amounts of time to start up (up to 5+ hours), load data and run data consistency checks.
- Finally, our services (including monitoring infrastructure) didn't understand the concept of DR or the need for a passive mode. So, we often ended up with more writer processes than expected, or the monitoring infrastructure would flag suspicious behavior (new certificate/security groups, capacity fluctuations etc.) which fired alerts and slowed down the DR.

### Concerns from leadership
- The problems added up to DR process never starting and ending at the advertised time. This made it impossible for leadership to sign off on public communication. The optics were especially bad since we could be rooting out data consistency issues for days after the procedure was completed.
- And yet, not having a working DR story meant that we were at the mercy of Azure availability. Azure had some notable outages that spanned days in 2018. This service had no choice but to wait for availability to recover and ride out the startup storms that followed. They estimated that they were not meeting their 99.9 SLA in significant number of regions due to this.
- Another observation was that some of these problems overlapped with setting up new scale units in new regions. The micro-services had to locate their own capacity in new regions and the release team had to coordinate deployments which all added up to months of work. The leadership hoped that we could solve this problem at the same time (since a DR involved creating new resources as well).

## Cut to the chase - what we did to solve the issues
### A multi-stage pipeline for DR and scale out
- **Downtime for unplanned DR:** Do whatever is needed to fence the old cluster, so that it does not interfere with setting up the new cluster. This step runs for unplanned DR and is pass-through for planned DR.
- **Prereqs:** This includes steps that are needed to bootstrap the failover (like deploy the new cluster, get a list of builds and stage them).
- **Setting up passive cluster:** Install the microservices in passive mode and the copy data for storages (if we are failing over data as well).
- **Downtime for planned DR:** This will involve doing a graceful fencing of the old cluster - it runs for planned DR and is pass-through for unplanned DR.
- **Restore data:** Load the data needed for the passive apps in the new cluster.
- **Activate new cluster:** Reconfigure the microservices to become active in the new cluster. Update DNS to point to this cluster.
- **Post DR steps:** Sanity test the new cluster, enable alerts that may have been suppressed etc.

This pipeline was made to be idempotent and supported the common plugins that were used in Microservice deployments. It brought 2 pieces of innovation and 1 process changes.
- It read off a storage to detect the build numbers/repositories/branches for each micro-service and could gather their builds. Normal runs of micro-service deployments would add their entries into the same storage.
- It could take the list of builds, gather them, and deploy them as instructed to a cluster, as a batch with appropriate logs and retries.
- It could be stopped and resumed from any point. It supported adding new steps as we discovered them. Every new step started out as a link to a wiki page explaining the steps. We committed to porting each step to an automated one ASAP.

### Getting engineers to trust the pipeline
We ran 3 steps in the pipeline (prereqs, setting up the passive cluster, restore the data) multiple times over the next few months.
It allowed us to exercise several things.
- Confirming that we can reliably discover builds and deploy them quickly. This benefited the scale out work.
- We ran several passive cluster tests to confirm (with network traces) that the passive micro-services didn't initiate any reads or writes unless instructed. We found several bugs and blocked them in the storage library layers. We also found ways to tweak the monitoring infrastructure.
- We discovered improvements in the data load steps so that they sped up, became more reliable and thus faster (these now take under 2 hours)

### Conformity in our data replication and failover story
We convinced leadership that we needed to get all micro-services to agree to the same data replication story. We evaluated all storage types used and designated an owner for each storage type.
- Wherever possible we recommended the following order of data redundancy ([GZRS > GRS > ZRS](https://docs.microsoft.com/en-us/azure/storage/common/storage-redundancy))
- The owner was asked to create a data replication and failover story.
	- The replication story could be on-demand (mandatory) and/or continuous (default preferred). 
	- It was the owner's responsibility to come up with a deployment script that could setup replication in the recommended configuration. The micro-services were instructed to include the changes in their official deployments even if they were not planning on using them yet.
	- The owner would also design a failover script that the micro-services would include in their deployment. 

### Conformity in regional footprint
The micro-services had opportunistically selected regions, so over the years, a scale unit spanned several nearby azure regions which made it more susceptible to regional outages.
- We picked hero region pairs for each scale unit. These regions were guaranteed to have availability zones.
- All storages in a scale unit were expected to use replication in hero region pairs.
- We grandfathered in exceptions for micro-services but got buy-off from leadership about removing these at the first available opportunity.
  - The regional footprint of each microservice was captured/driven from a central repository, so the DR team had complete visibility into who was where.
  - It allowed us to answer questions on which services were impacted in case of a regional outage, and where we would move them when needed.

### Getting engineers to trust the pipeline (part 2)
We ran the on-demand replication and failover steps in the pipeline multiple times over the next few months. We got leadership to approve running this on the team's selfhost cluster, which we used for bi-weekly signoffs. We caused several outages and found numerous bugs, but the fixes started piling up at this point. At the end of this period, we had proved several things.
- We could force an on-demand data replication for every storage we wanted.
- We could failover storages and run the service with remote storages.
- Services could be deployed in passive mode and stay passive indefinitely.
- We could failover compute and restore data in 2 hours or less.
- We fixed all data consistency issues we discovered.

## What happened next?
All this took about 12 months. In the meanwhile, we had created a few new scale units in the desired configuration (hero region pairs, use of availability zones, new compute stack, all storages with replication enabled) and run them safely. We finally had leadership approval to try failing over the smallest scale unit with real customers to this configuration. It went off very well. We wrapped things up with 2 hours of customer observed downtime. Over the next few months, we picked the next smallest scale unit at a time and failed it over. 

Eventually, things sped up and in the next 8 months we failed over about 25 scale units. An astonishing number of customers and their users were migrated, but no one saw a downtime bigger than 4 hours.

The move the new compute stack saved us money, but multiple availability zones, geo replication of data etc. increased the COGS of the service. The net effect was a combination of doubling the COGS, but increasing the resilience of the system. 

I eventually moved on to doing something else, but it was an awesome learning experience.
