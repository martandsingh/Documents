
# Unlock Massive Savings: Top Strategies to Slash Your Databricks Monthly Costs!
![data-science-partner-databricks-290x100-4c25-d@2x](https://github.com/user-attachments/assets/87ce2f0f-a9b7-4042-a283-cbec6596bab2)

```So one day you wakeup and have your coffee as always. It is just a wonderful day like any other day until you see a budget exceeding alert for your data platform. Believe me!! that will wake you up faster than your freshly brewed coffee. Somehow you gather the courage to open the email and find out your monthly budget exceeded by more than 60%. What will you do?```

Saving on Databricks costs is crucial for several reasons:

1. **Budget Efficiency**: Databricks, as a powerful data platform, can become quite expensive, especially for large-scale operations. Reducing costs helps companies manage their budgets more effectively and allocate resources to other critical areas.
2. **Competitive Advantage**: Lower operational costs can enhance a company's competitive edge by allowing them to invest more in innovation, marketing, and customer service, rather than spending on infrastructure.
3. **Scalability**: Effective cost management ensures that as data needs grow, companies can scale their operations without encountering prohibitive costs. This scalability is essential for businesses that anticipate expansion.
4. **Profit Margins**: For businesses with tight profit margins, every dollar saved on operational expenses directly impacts the bottom line. Cost savings from Databricks can significantly improve overall profitability.
5. **Resource Allocation**: Reducing expenses frees up budget for other strategic initiatives, such as research and development, talent acquisition, or new technology investments.
6. **Predictable Spending**: By optimizing costs, companies can achieve more predictable and manageable monthly expenses, making financial planning and forecasting more accurate.
7. **Value Maximization**: Effective cost management ensures that the investment in Databricks is delivering maximum value. It allows businesses to leverage the platform's capabilities without overspending.

Overall, cost savings in Databricks is about achieving a balance between harnessing the platform’s full potential and maintaining financial prudence.


Using the wrong resources in Databricks can lead to high billing in several ways:

1. **Over-Provisioned Clusters**: Selecting larger or more powerful clusters than necessary for a given task can result in paying for unused or underutilized compute capacity. For example, using high-memory or high-CPU instances for simple, less demanding jobs leads to higher costs without additional benefits.
2. **Under-Provisioned Clusters**: Conversely, using clusters that are too small or underpowered for the workload can lead to inefficiencies and slower processing times. This can cause jobs to take longer to complete, potentially leading to higher costs due to extended resource usage.
3. **Improper Cluster Sizing**: Choosing instances with more resources than required for specific tasks can cause unnecessary expenses. For instance, running a data analysis job that only needs basic compute power on a high-performance instance results in paying for excess capacity.
4. **Non-Optimized Data Storage**: Using inefficient storage solutions or not managing data lifecycle properly can lead to higher storage costs. Storing data in premium tiers or not using features like data compression and optimization can add to expenses.
5. **Inappropriate Instance Types**: Different types of compute instances have varying costs. Using specialized or high-performance instance types for tasks that don’t require them can be more expensive. For example, using GPU-optimized instances for non-GPU tasks unnecessarily increases costs.
6. **Inefficient Query Design**: Running poorly optimized or inefficient queries can consume more compute resources and time, leading to higher costs. Queries that are not well-structured or involve scanning large amounts of data can significantly impact the billing.
7. **Persistent Clusters**: Keeping clusters running continuously when they’re not needed can lead to high costs. For example, running a cluster 24/7 when it’s only needed for specific time windows wastes resources and drives up the bill.
8. **High-Availability Features**: Implementing high-availability and redundancy features, such as having multiple clusters for failover or using multiple availability zones, can increase costs. If these features are not necessary for the workload, they can lead to additional expenses.
9. **Improper Scaling**: Not setting up auto-scaling correctly can lead to either over-provisioning or under-provisioning of resources. Over-provisioning results in paying for more compute power than needed, while under-provisioning can cause delays and extended job runtimes.
10. **Unmanaged Data Transfers**: If data is frequently transferred in and out of the Databricks environment or between different cloud regions, it can incur additional data egress and transfer charges.

To avoid high billing, it’s crucial to carefully assess and select appropriate resources based on the specific needs of your workloads. Regularly monitoring usage, optimizing queries, and adjusting resource configurations can help manage costs effectively.

```This document is related to understand mainly on how choosing a wrong cluster configuration or type can lead a very high OPEX cost. Here we will discuss how can you manage your jobs, compute and multiple environment.```

```Before going further, we must understand that the whole article is based on a few assmptions:```
1. We are following lakehouse architecture, though these practices are common and will be helpful for all the usecases.
2. The organization has different databricks workspaces for dev, staging and production.
3. The organization is using SQL warehouse cluster.


```Before going to the databricks clusters and reduce the cluster capacity, we should understand first what is wrong with the whole incidence?```

# How to monitor databricks usage?
Databricks provides you demo dashboards which are quite helpful when it comes to monitor your workspace usage. You may not see those dashboard by default in your "dashboard" tab. To enable that you must run below commands:
#### import dbdemos
#### dbdemos.install('uc-04-system-tables', catalog='main', schema='billing_forecast')
Above command will enable demo dashboard. It will create a sql warehouse cluster, if you already have one you can delete that later and use the existing one.

# Where to find these dashboards?
You can go to the dashboard tab in databricks as shown below:

![image](https://github.com/user-attachments/assets/bf3dd228-1910-4954-a797-bca3a07ec40f)

You will see a list of dashboard with [dbdemos] prefix. 

![image](https://github.com/user-attachments/assets/34c3773d-1d0f-4dad-8fb8-ad5428d01afb)

You can find the details here:
https://www.databricks.com/resources/demos/tutorials/governance/system-tables

You can also monitor your billing using system tables. To use these tables you should have at least select permissions on these tables. If you dont have that, you may have to contact your databricks admin.
Below query uses:
1. system.billing.usage - for more details https://docs.databricks.com/en/admin/system-tables/billing.html 
2. system.billing.list_prices - for more details https://docs.databricks.com/en/admin/system-tables/pricing.html

Below query uses billing usage and pricing table to determine daily cost for different workspaces. Before executing don't forget to replace you staging & production workspace id respectively. If you dont have multiple env then you can remove the case statement (Environment column).
``` sql
SELECT
  (
    CASE workspace_id
      WHEN '{staging-workspace-id}' THEN 'Staging'
      WHEN '{production-workspace-id}' THEN 'Production'
      ELSE 'Development'
    END
  ) as Environment,
  workspace_id,
  usage.usage_date,
  SUM(
    usage.usage_quantity * list_prices.pricing.effective_list.default
  ) as `Total Dollar Cost`,
  (
    SUM(
      usage.usage_quantity * list_prices.pricing.effective_list.default
    ) - LAG (
      SUM(
        usage.usage_quantity * list_prices.pricing.effective_list.default
      )
    ) OVER (
      PARTITION BY
        workspace_id
      ORDER BY
        usage.usage_date
    )
  ) / SUM(
    usage.usage_quantity * list_prices.pricing.effective_list.default
  ) AS AmountChange
FROM
  system.billing.usage
  JOIN system.billing.list_prices ON list_prices.sku_name = usage.sku_name
WHERE
  usage.usage_end_time >= list_prices.price_start_time
  AND (
    list_prices.price_end_time IS NULL
    OR usage.usage_end_time < list_prices.price_end_time
  )
  AND usage.usage_date BETWEEN DATE_FORMAT (DATE_ADD (DAY, -60, CURRENT_DATE()), 'yyyy-MM-dd') AND DATE_FORMAT  (CURRENT_DATE(), 'yyyy-MM-dd')
GROUP BY
  workspace_id,
  usage.usage_date
ORDER BY
  usage.usage_date
```

# What is a databricks compute?
Databricks compute refers to the selection of computing resources available in the Databricks workspace. Users need access to compute to run data engineering, data science, and data analytics workloads, such as production ETL pipelines, streaming analytics, ad-hoc analytics, and machine learning.

Users can either connect to existing compute or create new compute if they have the proper permissions.
You can view the compute you have access to using the Compute section of the workspace:

## Types of compute
These are the types of compute available in Databricks:
1. **All-Purpose compute**: Provisioned compute used to analyze data in notebooks. You can create, terminate, and restart this compute using the UI, CLI, or REST API.
2. **Job compute**: Provisioned compute used to run automated jobs. The Databricks job scheduler automatically creates a job compute whenever a job is configured to run on new compute. The compute terminates when the job is complete. You cannot restart a job compute. See Use Databricks compute with your jobs.
3. **Instance pools**: Compute with idle, ready-to-use instances, used to reduce start and autoscaling times. You can create this compute using the UI, CLI, or REST API.

Reference: https://docs.gcp.databricks.com/en/compute/index.html#:~:text=These%20are%20the%20types%20of%20compute%20available%20in,used%20to%20reduce%20start%20and%20autoscaling%20times.%20

# What can be the potential issues?
1. Are you using all-purpose cluster/compute to run your workflows or scheduled jobs?
2. Is you all-prupose cluster or SQL warehouse compute running all the time?
3. Have you scheduled frequent alerts which can cause the warehouse compute to run for a longer period of the time?
4. Can you reduce the workflow frequency to reduce the usage?
5. Is your cluster overpowered?
6. Do you really need cluster for staging or production?
7. Can you use job compute?
8. Did you notice your worker nodes?


# What are the best practices to avoid all these problems?

##  Are you using all-purpose cluster/compute to run your workflows or scheduled jobs?
A common mistake among the developers is that they use an interactive or all-purpose cluster to run databricks workflow or scheduled notebooks. One should always use job compute or job pool to run databricks workflow. The job compute is cheaper and will die as soon as job finishes. 

## Is you all-prupose cluster or SQL warehouse compute running all the time?
Do not forget to "terminate after" setting for your compute. This setting will terminate your cluster after a set period of time if it is unused by any developer or job. I personally like to terminate my cluster after 60 minutes of inactivity. You can set as per your own use cases. 

## Have you scheduled frequent alerts which can cause the warehouse compute to run for a longer period of the time?
Databricks is growing day by day and providing you new amazing features. The alert mechanism is also one of them but new feature means one more addition to your monthly billing. Databricks alerts uses a SQL query to send you frequent alert based on a certain condition. To perform this operation it uses SQL warehouse cluster. If you set frequent alerts, your SQL warehouse compute may run for a longer period of time than you desire. This will increase your billing amount as warehouse clusters are quite expensive. Sometimes you need to run a script which involve terminate and load, i would prefer to create a notebook and schedule it with a job cluter.

##  Can you reduce the workflow frequency to reduce the usage?
More your workflow runs, more it costs you. So before scheduling your pipeline it is important to decide a correct frequency so that it solves your business problem and dont hits you hard on your OPEX side. So it is always better to schedule your pipeline with the correct frequency. 

## Is your cluster overpowered?
You can track your cluster memory used, data processed and other metrics. This will help you to guide if your cluster is overpower. You can set a threshold and monitor if you compute power is exceeding that or not.
![image](https://github.com/user-attachments/assets/bedbca6b-855a-48a7-b387-1edea6d59d4f)

![image](https://github.com/user-attachments/assets/131d1e90-1f13-4521-a030-9e8ca1926659)


## Do you really need cluster for staging or production?
In most of the cases, organizations keep their staging and production read-only or provide access to limited person only. As a data engineer or data scientist you will be developing ETL/ELT pipeline, ML notebooks on development env and move it to staging and production using CI/CD. The workflows on staging and production will use job pool or job compute to run. Most of the companies dont do direct development on stagin and production (definitely). So in that case think twice before you create a interactive cluster on staging or production env. 

## Can you use job compute?
Databricks pools are a set of idle, ready-to-use instances. When cluster nodes are created using the idle instances, cluster start and auto-scaling times are reduced. If the pool has no idle instances, the pool expands by allocating a new instance from the instance provider in order to accommodate the cluster’s request. 

## Did you notice your worker nodes?
When you create a compute, we have to be very careful about number of worker nodes. As each node is a different machine which is eventually increasing your cost. So be very mindful about the number of worker nodes. Try to limit it as per your requirement.

```Note: Above mentioned techniques are based on my personal experience. If you have ever faced any situation like this and have a resolution, feel free to contirbute to the document.```

Read more about job pool https://docs.databricks.com/en/compute/pool-index.html#:~:text=Databricks%20pools%20are%20a%20set%20of%20idle%2C%20ready-to-use,provider%20in%20order%20to%20accommodate%20the%20cluster%E2%80%99s%20request.

Thank you!! follow me on linkedin for more. linkedin.com/in/martandsays
