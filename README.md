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



