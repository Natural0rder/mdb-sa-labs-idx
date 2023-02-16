# mdb-sa-labs-idx

First, restore the dump in a Atlas M10 or a local database with mongorestore.
On Atlas:

```
mongorestore --uri 'mongodb+srv://search-clu.e8fmq.mongodb.net/game' shipments.bson \
   --username='jane.doe' \
   --password='mysuperstrongpwd'
```

# Lab 1: Single Field

For an on-demand audit requirement, the business team is asking us to retrieve shipments to China involving Nitrogen-4-Nickel-Ni-Carbon delivered to Sirius Black.

As this query will be executed every minute, an index must be created to handle it. Some constraints force us to create a Single Field index only.

- What is the best Single Field index in this case?
- Explain why by leveraging explain()

