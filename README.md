# mdb-sa-labs-idx

First, restore the follwing [dump](https://s3.eu-west-3.amazonaws.com/sylvain.chambon/saescape/shipments.tar.gz) in a Atlas M10 or a local database with mongorestore.
On Atlas:

```
mongorestore --uri 'mongodb+srv://search-clu.e8fmq.mongodb.net/game' shipments.bson \
   --username='jane.doe' \
   --password='mysuperstrongpwd'
```

# Lab 1: Single Field

For an on-demand audit requirement, the business team is asking us to retrieve shipments to China involving Nitrogen-4-Nickel-Ni-Carbon delivered to Sirius Black.

As this query will be executed every minute, an index must be created to handle it. Some constraints force us to create a Single Field index only.

What is the best Single Field index in this case?

> With simple count queries, just find the most selective field.

Explain why by leveraging explain() of each option

> Create 3 single field indexes (one for each field). Use "hint" to force the selection of a given index while executing the query and compare explain outputs (totalKeysExamined).

Does sort (1, -1) matter in a single field index?

> No, traversing the B-Tree will have the same cost regardless of sort.

Create a Single Field index on each field (to, recipient, material) and execute explain("queryPlanner") and explain("executionStats").

What do these commands achieve?

> explain("queryPlanner") details the plan selected by the query optimizer.

![image](https://user-images.githubusercontent.com/102281652/223132357-2152c163-3b14-4bd1-9476-0ffc87cbad17.png)

> explain("executionStats") details the execution of the winning plan.

What is the “AND_SORTED” stage?

> Indexes intersection

As an optimization, a software developer is asking you if it is fine to replace the current _id (which is a classic ObjectId) with a functional identifier they have.
This functional identifier is unique and will support a large portion of reads (getById is ~50% of total reads).

Here is the new _id generation pattern proposed by the customer: <fromCountryCode>-<toCountryCode>-<materialCode>-<recipientNumber>
Here is a sample: cn-fr-c21h30o2-765766
The recipient number is coming from an Oracle Sequence in their CRM solution.

What should be our recommendation in this case?

> If your data doesn’t have a naturally occurring unique field (often the case), go ahead and use the default ObjectId for _ids. However, if your data does have a unique field and you don’t need the properties of an ObjectId, then go ahead and override the default _id—use your own unique value. This saves a bit of space and is particularly useful if you were going to index your unique id, as this will save you an entire index in space and resources (a very significant savings).
> There are a couple reasons not to use your own _id that you should consider: first, you must be very confident that it is unique or be willing to handle duplicate key exceptions. Second, you should keep in mind the tree structure of an index and how random or non-random your insertion order will be. ObjectIds have an excellent insertion order as far as the index tree is concerned: they always are increasing, meaning they are always being inserted at the right edge of the B-tree. This, in turn, means that MongoDB only has to keep the right edge of the B-tree in memory.
Conversely, a random value in the _id field means that _ids will be inserted all over the tree. Then the machine must move a page of the index into memory, update a tiny piece of it, then probably ignore it until it slides out of memory again. This is less efficient.

# Lab 2: Compound indexes
   
For a new use case, we need to provide the list of distinct materials shipped to a given country for a given recipient.

What is the best compound index to support this query ?
What if we want to sort by material in alphabetical order?
   
> Compound index with "to", "recipient" and "material" (for nice a covered query)

```
[
  {
    $match: {
      to: "China",
      recipient: "Hedwig",
    },
  },
  {
    $group: {
      _id : "$material"
    }
  },
  {
    $sort : { _id : 1 }
  }
]
```

What if we reverse to and recipient in the “E” part of the index?
   
> If the order in the E does not always matter, having a look at the order can help to handle more queries (prefixes). => We want to hit a compound index as much and as often as possible.
> On the other hand Lower cardinality first may significantly improve prefix compression causing the index to take up less space on disk and in RAM.

Why does the index direction (asc/desc) matter in a compound index ?

> Finding a record in a BTree takes O(Log(n)) time. Finding a range of records in order is only OLog(n) + k where k is the number of records to return.
If the records are out of order, the cost could be as high as OLog(n) * k

# Lab 3: Multikey
   
Write a query with the proper index to get the total quantity for shipments to a given country where agents contain a list of given agents.

> Index the agents array and write an aggregation with a $all match.

For the next version of the application, the customer will expect a huge growth of the agents array.
In average, documents will contain 50k agents but some documents can embed N x 100K agents.

What is the impact on our previous index?
   
> Each item of the array will be persisted as a key in the index. Expect to have a significant growth of the multikey index impacting the writes.
   
Is it still valid to keep it?
What could be our recommendation?

> https://www.mongodb.com/developer/products/mongodb/schema-design-anti-pattern-massive-arrays/
> http://www.askasya.com/post/largeembeddedarrays/

# Lab 4: Covered queries
   
Write a query to retrieve the most recent shipment date to “China” for “Hedwig”.

```
db.shipments.find({ to: "China", recipient: "Hedwig" }, { _id: 0, date: 1 }).sort({ date: -1 }).limit(1)

> Do not forget to project _id : 0 

With the appropriate index, you should be able to have a covered query.
How can I detect that the query is covered by an index with explain()?
   
> No FETCH stage!
   
# Lab 5: TTL
   
What are the drawbacks of using TTL indexes?

> No management of the internal deletion batch (execution time, limit etc) that can trigger unwanted IOPS peaks.
   
In which cases TTL indexes can be leveraged?
   
> Small and short living documents
   
Describe a valid alternative to TTL indexes to handle physical deletes.
⇒ You can propose an algorithm.  

> Implement a batch. Fetch expired documents _ids (leverage a simple date field index) with a $limit. Then build some chunks of _ids and perform a simple deleteMany with an array if _ids. You can schedule the batch to run during appropriate hours and proper $limit and chunk sizes.
