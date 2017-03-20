Eventhough the Metry API is built to be used on the fly, you can use the API to continously export energy data to your own database. 
This is useful if you already have legacy code.

Are you developing a new energy-app? Use the Metry API directly - it will save you a lot of time!

# Preparations

You need an API-key or a developer account - both are free!

# Continously export data

Create a script that **runs once per day** and perform the following tasks

1. Request active meters from Metry
2. Map meters not yet mapped
3. Request data for all mapped meters

## 1. Request active meters from Metry

``
  GET /api/v2/meters?revoked=false&box=active
``  

The request above will fetch all active meters from the user's account. The `revoked` and `box` params makes sure that you
only get meters where energy data is currently accessible.

NOTE: The list will be paginated if more than 50 meters are available, so don't forget to loop though the pages using
the `skip` params as [described here](http://docs.metry.apiary.io/#introduction/list-responses-and-pagination)

## 2. Map meters not yet mapped

Now you will have to map each meter to the meters in your own databases. This is basically done by storing each
meter's `root._id` together with the existing meters in your database. 

The process of mapping each Metry meter with the existing meters in your database usually requires human interaction. Therefore you should provide the user an interface where this
can be accomplished. Remember that you can help the user with suggestions using the meta data available for each meter 
(`ean`, `name`, `address`, `tags`, `type`).

**Can't allow human interaction?** Ask the user to perform the mapping in Metry portal by adding a tag for each meter. The
tag should be whatever identifier you have for each existing meter in your database.

## 3. Request data for all mapped meters

Now you can start to export the energy data by looping through all the meters found in step 1. 
For each meter you make a request that looks something like this:

``
https://app.metry.io/api/2.0/consumptions/5417d7a9fd27c650008b5d94/month/201603?metrics=metrics
``

Check out [the consumption documentation](http://docs.metry.apiary.io/#reference/consumption/get-consumption) to
understand how the query is structured. Note: use the `root._id` as **meter_id**.

### Common pit-falls and questions

#### What period should I ask for?
Each meter has a `consumption_stats` attribute where you can see exactly what data that is available.
If it is that first time data is requested you may using this to fetch all available data. In other cases requests data from the last 
available value in your own database to the previous month, day or hour.

#### What granularity should I ask for?
The `consumption_stats`provides you with information about available data. As a minimum, 
Metry will always provide monthly values for metric energy. The availability of all other metrics and granularities will vary.

#### Sum of hour not always same as month
You cannot assume that a month value will be the same as the underlying day or 
hour values. If there are gaps in the hour data, the month values will be higher than the sum of hour values. Metry stores the values
separately, which allows this feature.

#### Available meters changes over time**
Remember that a consumption request that worked one day, may fail the next day because the user chose to 
trash the meter from Metry. The Metry API is using standard HTTP responses [response codes](http://docs.metry.apiary.io/#introduction/error-handling)
to describe the error.

Use this to filter errors that are 

- caused by user action (403)
- caused by your self (400)
- cased by Metry (500)

#### Understand how a Metry-meter corresponds to your meters
A meter at Metry *is defined as a point where data is collected*. 
It is equivalent to Swedish definitions "anläggnignsid" or "leveranspunkt". One meter at Metry may include all of the following items;

- month values
- day values
- hour values
- reading values
- multiple metrics for each value (month, day, hour and readings)

# One-time export

Since the continuous export script only requests new values, historical values added after the first request will 
never be requested. The same issue applies to updated values (while Metry tries to only collect correct values there will be
rare cases where values are updated to replace an incorrect value).

To handle this scenarios, you should allow the user to force data request for a specific meter and period and
overwrite existing values. This will make life easier for you as well as your user.
