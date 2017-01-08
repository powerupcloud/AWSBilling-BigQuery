Most of the accounts you would like to analyzed tend to have consolidated billing enabled. So The queries below assume such scenario. If you have a standalone account, all you need to do is remove the `CASE` statements from `SELECT` and `GROUP BY`. So How do you find How many linked accounts you have? Do a `SELECT DISTINCT linkedaccountid FROM yourtable` if you are using Standard SQL mode in BigQuery. Else, you need to do a `SELECT linkedaccountid, count(*) FROM yourtable GROUP BY linkedaccountid` in legacy SQL mode. That kind of simulates `SELECT DISTINCT` behaviour. Good thing is, you can switch Standard vs legacy SQL in BigQuery depending on your needs. 

# Month to Date Spend By Account

```sql
SELECT payeraccountid,
       linkedaccountid,
       CASE
           WHEN linkedaccountid = 'xxxxxxxxxx37' THEN 'Account-Prod'
           WHEN linkedaccountid = 'xxxxxxxxxx43' THEN 'Account-Dev'
           WHEN linkedaccountid ='xxxxxxxxxx09' THEN 'Account-Stage'
           ELSE 'SomethingElse'
       END AS accountname,
       blendedcost
FROM [powertracker.mytable]
WHERE recordtype = 'AccountTotal'
```
# Cost By AWS Service, By Account
What is the cost breakdown of each AWS service like EC2, RDS, CloudFront, S3 by account as of today?

```sql
#Use FLOAT64 instead of FLOAT if you are using Standard SQL
SELECT SUM(CAST(blendedcost AS FLOAT)) AS cost,
       productname,
       CASE
           WHEN linkedaccountid = 'xxxxxxxxxx37' THEN 'Account-Prod'
           WHEN linkedaccountid = 'xxxxxxxxxx43' THEN 'Account-Dev'
           WHEN linkedaccountid ='xxxxxxxxxx09' THEN 'Account-Stage'
           ELSE 'SomethingElse'
       END AS accountname
FROM powertracker.mytable
WHERE productname <> ''
  --AND blendedcost > 0
GROUP BY accountname,
         productname
ORDER BY accountname
```

# How Much Does EC2 cost by Account?

```sql

SELECT SUM(CAST(blendedcost AS FLOAT)) as EC2Cost,
       CASE
           WHEN linkedaccountid = 'xxxxxxxxxx37' THEN 'Account-Prod'
           WHEN linkedaccountid = 'xxxxxxxxxx43' THEN 'Account-Dev'
           WHEN linkedaccountid ='xxxxxxxxxx09' THEN 'Account-Stage'
           ELSE 'SomethingElse'
       END AS accountname
FROM powertracker.mytable
WHERE productname = 'Amazon Elastic Compute Cloud'
GROUP BY accountname
```

# Cost By Internal Services
When I say Internal services, I mean cost allocation tags you have enabled. Below query assumes only one cost allocation tag: `user_servicename`. If you have more, you should add more in SELECT and GROUP BY

```sql

#If AS FLOAT doesn't work, please use AS FLOAT64
SELECT SUM(CAST(blendedcost AS FLOAT)) AS Cost,  
           user_servicename,
           WHEN linkedaccountid = 'xxxxxxxxxx37' THEN 'Account-Prod'
           WHEN linkedaccountid = 'xxxxxxxxxx43' THEN 'Account-Dev'
           WHEN linkedaccountid ='xxxxxxxxxx09' THEN 'Account-Stage'
           ELSE 'SomethingElse'
       END AS accountname
FROM powertracker.mytable WHERE user_servicename <> ''
GROUP BY   user_servicename, accountname 
ORDER BY Cost DESC;
```

# Cost By Internal Services - On a Given Day
What if I would like to know how much my each cost allocation tag costed me on a given day this month? Easy.
```sql
SELECT SUM(CAST(blendedcost AS FLOAT)) AS Cost,
           user_servicename,
           WHEN linkedaccountid = 'xxxxxxxxxx37' THEN 'Account-Prod'
           WHEN linkedaccountid = 'xxxxxxxxxx43' THEN 'Account-Dev'
           WHEN linkedaccountid ='xxxxxxxxxx09' THEN 'Account-Stage'
           ELSE 'SomethingElse'
       END AS accountname
FROM powertracker.mytable WHERE user_servicename <> ''
AND DATE(TIMESTAMP(usagestartdate)) = '2017-01-01'
GROUP BY   user_servicename,
       accountname
ORDER BY Cost DESC;
```
# Drill Through Of a Particular Service
Ok, I found that a service called `Galactus` is costing me too much. How do I find what resources consist of `Galactus` cost allocation tag and how much they cost me? 

```sql

SELECT productname,
       usagetype,
       OPERATION,
       availabilityzone,
       reservedinstance,
       itemdescription,
       SUM(CAST(blendedcost AS FLOAT)) AS Cost,
       resourceid
FROM [Powertracker.mytable]
WHERE user_servicename = 'galactus'
  AND linkedaccountid = 'xxxxxxxxxx43'
  AND CAST(blendedcost AS FLOAT) > 0
GROUP BY usagetype,
         productname,
         usagetype,
         OPERATION,
         availabilityzone,
         reservedinstance,
         itemdescription,
         resourceid
ORDER BY Cost DESC
```

# Daily  - Cost By AWS Service

What if I need to find a cost breakdown by AWS services? 
```sql
SELECT productname,
       SUM(CAST(blendedcost AS FLOAT)) AS Cost
FROM [Powertracker.mytable]
WHERE linkedaccountid = 'xxxxxxxxxx37'
AND DAY(TIMESTAMP(usagestartdate)) = 2 #this is the day of the month you are interested in
GROUP BY productname
ORDER BY Cost DESC
```

# Daily - EC2 Instances Cost

If I need the cost for EC2 instances ONLY, on a given day. 

```sql
SELECT 
       resourceid,
       SUM(blendedcost) AS EC2InstanceCost
FROM [Powertracker.mytable]
WHERE productname = 'Amazon Elastic Compute Cloud'
  AND linkedaccountid = 'xxxxxxxxxx37'
  AND OPERATION = 'RunInstances'
  AND itemdescription NOT LIKE '%data transfer%'
  DAY(TIMESTAMP(usagestartdate)) = 2
GROUP BY resourceid
```

# Total Cost of Reserved Instances and On-demand Instances on a Given Day

```sql
--Total Cost for Reserved Instances
SELECT
       SUM(CAST(blendedcost AS FLOAT)) AS ReserverdInstCost
FROM [Powertracker.mytable]
WHERE productname = 'Amazon Elastic Compute Cloud'
AND reservedinstance = 'Y'
AND linkedaccountid = 'xxxxxxxxxx37'
AND operation = 'RunInstances'
AND itemdescription NOT LIKE '%data transfer%'
AND DAY(TIMESTAMP(usagestartdate)) = 2
ORDER BY Cost DESC

--Total Cost of On-Demand Instances
SELECT
       SUM(CAST(blendedcost AS FLOAT)) AS OnDemandCost
FROM [Powertracker.mytable]
WHERE productname = 'Amazon Elastic Compute Cloud'
AND reservedinstance = 'N'
AND linkedaccountid = 'xxxxxxxxxx37'
AND operation = 'RunInstances'
AND itemdescription NOT LIKE '%data transfer%'
AND DAY(TIMESTAMP(usagestartdate)) = 2
ORDER BY Cost DESC
```

# How Much Are My EBS Volumes Costing Me?

```sql
SELECT resourceid, SUM(CAST(blendedcost AS FLOAT)) AS EBSCost
FROM [Powertracker.mytable]
WHERE productname = 'Amazon Elastic Compute Cloud'
AND resourceid CONTAINS 'vol-'
AND AND DAY(TIMESTAMP(usagestartdate)) = 2 #remove this if you want a monthly aggregate
GROUP BY resourceid
Order by Cost DESC
```
# How Much is RDS costing?

```sql
SELECT resourceid, SUM(CAST(blendedcost AS FLOAT)) AS RDSCost
FROM [Powertracker.mytable]
WHERE productname = 'Amazon RDS Service'
AND linkedaccountid = 'xxxxxxxxxx37'
GROUP BY resourceid
```

