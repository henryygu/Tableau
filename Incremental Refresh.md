# Incremental Refreshes on Tableau

## Problem

Long dataset >23 million rows and is refreshing slowly. Aim to get the refresh below 15 minutes.

## Can we use Tableau incremental refresh?



Tableau’s incremental refresh works by looking through the datasource and finding new values. It does this by using the Unique ID column and selecting for ID > max(unique ID) = ID > 3.
These new values are then appended to the data source.
In the previous example
 
For example, if the first refresh happens at 9am, the max of unique id is 3.
| Site Name | Metric Name | Metric Value | Unique ID |
|-----------|-------------|--------------|-----------|
| Site A    | Metric 1    |            1 |         1 |
| Site C    | Metric 1    |            1 |         2 |
| Site D    | Metric 1    |            1 |         3 |


Then the second refresh  at 9:15am, Tableau is looking for a Unique ID of 4 or higher.
| Site Name | Metric Name | Metric Value | Unique ID |
|-----------|-------------|--------------|-----------|
| Site A    | Metric 1    |            2 |         1 |
| Site B    | Metric 1    |            1 |         2 |
| Site C    | Metric 1    |            1 |         2 |
| Site D    | Metric 1    |            1 |         4 |


The resulting output within Tableau would look like, where Site B is completed missed since it’s unique ID is 2, and Site D is duplicated twice, but each one has a different Unique ID.
| Site Name | Metric Name | Metric Value | Unique ID |
|-----------|-------------|--------------|-----------|
| Site A    | Metric 1    |            1 |         1 |
| Site C    | Metric 1    |            1 |         2 |
| Site D    | Metric 1    |            1 |         2 |
| Site D    | Metric 1    |            1 |         4 |


Also note that since Tableau only considers the UniqueID column, in this scenario, Site A has resubmitted during the second refresh a value of “2 (green highlight)”. This will not be picked up by Tableau and the dashboard will always display 1, until a full refresh is carried out.

## Load Date Time
Tableau’s incremental refresh can also utilise a Date time field. Instead of looking for the max number, it can look of the max Date Time. The logic is similar to the above with the Unique ID. Ideally every change would be flagged with a new Date Time field, however this is unrealistic should there be modifications to previously submitted data (site B issue above) as the SQL server would need to compare the current refresh and last refresh to determine this.

One solution is to set that only data in the last 60 days can be modified. This means that by using Load Date Time, we would be extracting the **last 60 days worth of data in each incremental refresh**, which would cause the values to be duplicated over and over again. The data prior to the last 60 days (ie from min(date) to today minus 60 days) would not have their Load Date Time changed.

### The outcome:
![2023-04-06-06-13-00-07-WINWORD_JEZovRKEgf](/assets/2023-04-06-06-13-00-07-WINWORD_JEZovRKEgf.png)

Collection data sources match perfectly, and refresh times are less than 5 minutes. The one outstanding issues that we could not fix was the extremely slow load times of the dashboard and the slow performance when selecting filters. As such, this was deemed to be not acceptable and abandoned.

### The issue with Load Date Time

To filter out all the duplicates, we essentially need to check for each combination of Site and Metric, what is the Maximum value we have for Load Date Time. This creates a huge overhead for calculation as tableau needs to calculate this for each of the 22 million rows. This gets more difficult as each 15 minute refresh is technically adding 1.4 million rows. 

By the end of one day in UAT, we were at approximately 40 million rows and UAT was only refreshing every hour.

### [Load_Date_Time_Filter]

```Tableau
IF ATTR(DATETRUNC('day', [Load Date Time])) = today() THEN 
    MAX([Load Date Time])=ATTR([Load Date Time])
ELSE TRUE
END
```

## Hyper API?


import pyodbc 
server = 'APSC2A....'
database = 'mydb'
username = 'myusername'
password = 'mypassword'
cnxn = pyodbc.connect('DRIVER={ODBC Driver 17 for SQL Server};SERVER='+server+';DATABASE='+database+';UID='+username+';PWD='+ password)
cursor = cnxn.cursor()

#Sample select query
cursor.execute("SELECT * FROM vw_nhsiAneDashboardMetricsTransposed WHERE datediff(day,period, today()) < 60")
for row in cursor.fetchall():
    print row
 
