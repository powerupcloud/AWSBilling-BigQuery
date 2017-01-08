Most of the accounts you would like to analyzed tend to have consolidated billing enabled. So The queries below assume such scenario. If you have a standalone account, all you need to do is remove the `CASE` statements from `SELECT` and `GROUP BY`. So How do you find How many linked accounts you have? Do a `SELECT DISTINCT linkedaccountid FROM yourtable`

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
       blendedcost,
       unblendedcost
FROM [powerupcloud-production:awscost.ailcost]
WHERE recordtype = 'AccountTotal'
```
