# Group & Total

__Minimum MongoDB Version:__ 4.2


## Scenario

A user wants to scan through a collection, filtering only records within a specific date range, and then grouping the records by a recurring field's value, accumulating counts, totals and the array of details from each record in the group.

In this example, a collection of _orders_, from shop purchases for the year 2020 only will be searched for. The records will then be grouped by customer ID, capturing, for 2020, each customer's first purchase date, the number of orders they made, the total value of all their orders added together and a list of their individual order items. Essentially what is produced is a report of orders made by each customer in 2020.


## Sample Data Population

Drop the old version of the database (if it exists) and then populate a new `orders` collection with 9 order documents spanning 2019-2021, for 3 different unique customers:

```javascript
use group-and-total;
db.dropDatabase();

// Create index for a orders collection
db.orders.createIndex({'orderdate': -1});

// Insert 9 records into the orders collection
db.orders.insertMany([
  {
    'customer_id': 'elise_smith@myemail.com',
    'orderdate': ISODate('2020-05-30T08:35:52Z'),
    'value': NumberDecimal('231.43'),
  },
  {
    'customer_id': 'elise_smith@myemail.com',
    'orderdate': ISODate('2020-01-13T09:32:07Z'),
    'value': NumberDecimal('99.99'),
  },
  {
    'customer_id': 'oranieri@warmmail.com',
    'orderdate': ISODate('2020-01-01T08:25:37Z'),
    'value': NumberDecimal('63.13'),
  },
  {
    'customer_id': 'tj@wheresmyemail.com',
    'orderdate': ISODate('2019-05-28T19:13:32Z'),
    'value': NumberDecimal('2.01'),
  },  
  {
    'customer_id': 'tj@wheresmyemail.com',
    'orderdate': ISODate('2020-11-23T22:56:53Z'),
    'value': NumberDecimal('187.99'),
  },
  {
    'customer_id': 'tj@wheresmyemail.com',
    'orderdate': ISODate('2020-08-18T23:04:48Z'),
    'value': NumberDecimal('4.59'),
  },
  {
    'customer_id': 'elise_smith@myemail.com',
    'orderdate': ISODate('2020-12-26T08:55:46Z'),
    'value': NumberDecimal('48.50'),
  },
  {
    'customer_id': 'tj@wheresmyemail.com',
    'orderdate': ISODate('2021-02-29T07:49:32Z'),
    'value': NumberDecimal('1024.89'),
  },
  {
    'customer_id': 'elise_smith@myemail.com',
    'orderdate': ISODate('2020-10-03T13:49:44Z'),
    'value': NumberDecimal('102.24'),
  },
]);
```


## Aggregation Pipeline(s)

Define a single pipeline ready to perform the aggregation:

```javascript
var pipeline = [
  // Match only orders made in 2020
  {'$match': {
    'orderdate': {
      '$gte': ISODate('2020-01-01T00:00:00Z'),
      '$lt': ISODate('2021-01-01T00:00:00Z'),
    }
  }},
  
  // Sort by order date ascending (required to pick out 'first_purchase_date' below)
  {'$sort': {
    'orderdate': 1,
  }},      

  // Group by customer
  {'$group': {
    '_id': '$customer_id',
    'first_purchase_date': {'$first': '$orderdate'},
    'total_value': {'$sum': '$value'},
    'total_orders': {'$sum': 1},
    'orders': {'$push': {'orderdate': '$orderdate', 'value': '$value'}},
  }},
  
  // Sort by each customer's first purchase date
  {'$sort': {
    'first_purchase_date': 1,
  }},    
  
  // Set customer's ID to be value of the field that was grouped on
  {'$set': {
    'customer_id': '$_id',
  }},
  
  // Omit unwanted field
  {'$unset': [
    '_id',
  ]},   
];
```


## Execution

Execute the aggregation using the defined pipeline and also view its explain plan:

```javascript
db.orders.aggregate(pipeline);
```

```javascript
db.orders.explain('executionStats').aggregate(pipeline);
```


## Expected Results

Three documents should be returned, representing the three customers, each showing the customer's first purchase date, the total value of all their orders, the number of orders they made and a list of each order's detail, for orders placed in 2020 only, as shown below:

```javascript
[
  {
    customer_id: 'oranieri@warmmail.com',
    first_purchase_date: 2020-01-01T08:25:37.000Z,
    total_value: Decimal128("63.13"),
    total_orders: 1,
    orders: [
      {orderdate: 2020-01-01T08:25:37.000Z, value: Decimal128("63.13")}
    ]
  },
  {
    customer_id: 'elise_smith@myemail.com',
    first_purchase_date: 2020-01-13T09:32:07.000Z,
    total_value: Decimal128("482.16"),
    total_orders: 4,
    orders: [
      {orderdate: 2020-01-13T09:32:07.000Z, value: Decimal128("99.99")},
      {orderdate: 2020-05-30T08:35:52.000Z, value: Decimal128("231.43")},
      {orderdate: 2020-10-03T13:49:44.000Z, value: Decimal128("102.24")},
      {orderdate: 2020-12-26T08:55:46.000Z, value: Decimal128("48.50")}
    ]
  },
  {
    customer_id: 'tj@wheresmyemail.com',
    first_purchase_date: 2020-08-18T23:04:48.000Z,
    total_value: Decimal128("192.58"),
    total_orders: 2,
    orders: [
      {orderdate: 2020-08-18T23:04:48.000Z, value: Decimal128("4.59")},
      {orderdate: 2020-11-23T22:56:53.000Z, value: Decimal128("187.99")
      }
    ]
  }
]
```


## Observations & Comments

 * __Double Sort Use.__ It is necessary to perform a sort on the order date both before and after the group stage. The sort before the group is required because the group stage uses a `$first` group accumulator to capture just the first order's `orderdate` value for each customer being grouped. The sort after the group is required because the act of having just grouped on customer ID will mean that the records are no longer sorted by purchase date for the records coming out of the group stage.
 
 * __Renaming Group.__ Towards the end of the pipeline you will see what is a common pattern for pipelines that use `$group`, consisting of a combination of `$set`+`$unset` stages, to essentially take the group's key (which is always called `_id`) and substitute it with a more meaningful name in the result (`customer_id` in this case).
 
 * __Lossless Decimals.__ You may notice that a `Decimal()` function has been used to ensure the order amounts in the inserted records are using a lossless decimal type, [IEEE 754 decimal128](https://docs.mongodb.com/manual/tutorial/model-monetary-data/). In this example, if a JSON _float_ or _double_ type is used instead, the result order totals will suffer from loss of precision. For example, for the customer `elise_smith@myemail.com` the `total_value` result will have the value shown in the second line below, rather than the first line, if a _double_ type was used:

```javascript
// Desired result (achieved by using decimal128 types)
total_value: Decimal128("482.16")

// Result that occurs if using float or double types instead
total_value: 482.15999999999997
```
