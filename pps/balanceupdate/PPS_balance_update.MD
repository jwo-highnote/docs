# Problem
On Aug 17, we received 8.8K/m (146 tps) verification mainly from Pingpong traffic spike. This exceeded the limit app-txn-transaction service can handle and we saw traffic dropping. Ideally, we will want to support upto 200 tps as our upstream app-iso limit.

## Analysis
### Schema
Indices are expensive for update. Indices with the same leading column (public_ledger_id) can be redundant and add unnecessary overhead for insert.  
#### tx_ledger_entry
* `tx_ledger_entry_idx ON tx_ledger_entry(public_ledger_id);`
* `tx_ledger_entry_ledger_id_idx ON tx_ledger_entry(public_ledger_id, create_commit_date DESC) STORING (change_amount, impact);`
#### tx_balance
* `tx_balance_as_of_idx ON tx_balance(public_ledger_id,  as_of_date DESC, update_sequence DESC) STORING (balance_amount, balance_currency_code);`
* `unique tx_balance_v2_udx ON tx_balance(public_ledger_id, update_sequence DESC) STORING (balance_amount, balance_currency_code, public_tenant_id, public_party_id, as_of_date);`
#### tx_processed_ledger_entry & tx_balance
These two tables seems holds redundant information however have different index
`UNIQUE INDEX tx_processed_ledger_entry_udx ON tx_processed_ledger_entry(processed_ledger_entry_id);`
`tx_balance_public_ledger_entry_id ON tx_balance(public_ledger_entry_id); `
We can add unique constrain on `tx_balance` table itself and remove the `tx_process_ledger_entry` table

#### Cardinality
Also note that currently public_ledger_id has very low cardinality and skewed data shape: the top 4 account accountes for 50% of all data. 
In the above indices, the secondary columns are montonically incrementing (update_sequence, as_of_date). This likely cause hotspot when 1 client is under heavy load (need to verify with spanner insight).
The following SQL is used to gather tx_balance data shape.
```
The following queries are used to gather the data distribution

top 4:

SELECT count(*) FROM `bay1-pj-prod-live-data-plat.transaction.tx_balance` 
WHERE public_account_id 
in ('ac_og2203c37e11119249a28ea76440fe45a70a',
'ac_c0224167e5ad2031453786f1ba9d536efe24',
'ac_og22b84062e8e0af41cf9426b7e469bec09d',
'ac_ba22ced9378be03c461482c408713086b54e') 
AND commit_timestamp>=TIMESTAMP("2023-01-01")

1193900
------------------------------------------------------
total count

SELECT count(*) 
FROM `bay1-pj-prod-live-data-plat.transaction.tx_balance` 
WHERE commit_timestamp>=TIMESTAMP("2023-01-01")

2692164
```
### Workflow (Sequence Diagram)
![uml](http://www.plantuml.com/plantuml/svg/dP6nJiGm44JxV4LmJKJl0Xs7hg2Wf8uSUmX6pjhiseZyFKu84aKKeUiiQzxJpkv9ik5Sfj3Sv3TCQ5CmdXfbKkgyHDNThy_CXiJOx2Dr8oUhTAi4vDhyHQhPZrJK3VhFjzBAau2phbWVu4Pm3yeytxQG7f_rtKiiR56Ta2x54C3TUnnsPGpZz7_8L8OxAKD5rfdF9XYoU4dEP5mMja9RPgzmmBYvj8qaPK9paDcRzAE0P_tgoPtd9wT_gb-MvkewbPULVWZB3kPdut2bQE4V2pCeRsiv51tSj9vq-3MSGAvCuHi0)

* Multiple read queries before the insert of balance (see sequence diagram). we potentially can remove findById. 
* tx_balance_v2_udx prevent concurrent write, we might want to analyze if that is cost effective comparing holding a "ledger_entry" lock
* tx_process_ledger_entries seems hold information can be derived from tx_balance. Its additional value is the unique index on ledger_entry_id to ensure idempotent.

# Proposals
## App Optimization
#### Remove findById call
Include all info in the payload of event_source so no need to get ledger entry details through `findById`
* Pros: Save extra query
* Cons: Potentially make it hard for payload schema evolution, we need to make sure the schema is forward compatible.
#### Remove isEntryAlreadyMade
Currently, our schema ensures idempotency with unique constrain. At the same time, application also query `isEntryAlreadyMade` for idem check before insert. This check likely is designed for pubsub redelivery use case. Unless this is very frequent, we likely better off with an optimistic approach
* Proposal:
  * Remove this idem check
  * Application handle unique constrain error.
  * Application maintain a bloom-filter ledger_entry_id. 
    * This bloom-filter is reloaded upon unique constrain error with last n minutes depends on the ledger entry commit timestamp.
    * Since pub/sub deliver in order, we only need to load entries >= current entry_id commit_timestamp into bloom-filter for the ledger
    * All balance insert to update bloom-filter
  * Application lookup bloom-filter rather than spanner for idem check
* Pros: 
  * Significantly reduce the query for idem check.
  * The unique index ensures data integrity.
* Cons:
  * More complicated than the current implementation
  * Unique constrain violation will trigger bloom-filter reload. This should only happen during pubsub (or es-poller) redelivery, assuming it is relatively infrequent.
#### Batch insert with mutation
* Pros: 
  * Update rows in batch is more efficient and we can further optimized with [partioning](https://cloud.google.com/spanner/docs/bulk-loading#partition-by-key)
* Cons:
  * Logic can be more complicated, e.g. we need to manage `update sequence` in app layer and handle redelivery/idem differently.

## Schema Optimization
### Remove indices already covered
We might need to modify application to [force index](https://cloud.google.com/spanner/docs/secondary-indexes#index-directive) to sql if necessary
```
FROM MyTable@{FORCE_INDEX=MyTableIndex}
```
* Remove `tx_ledger_entry_idx ON tx_ledger_entry(public_ledger_id);`

### Remove tx_processed_ledger_entry
We can make `tx_balance(public_ledger_entry_id)` and switch application to use tx_balance instead of tx_processed_ledger_entry

### Weak Entity Schema Design
tx_balance, tx_process_ledger_entry, tx_entry_details are weak entity to ledger_entry (i.e. 1-1 mapping to ledger_entry)
We can design those table's primary key as ledger_entry_id with pre/post-fix so there will be no need to an extra unique index.

### ledger_id design
The relevant indices usually driven from ledger_id, which has low cardinality and second driving column monotonically increments. 
If we observe hotspot (1 server cpu higher than others) during the spike we might want to change the ledger_id design to be prefixed rather postfixed with account id to spread the load. See schema [design best practice](https://cloud.google.com/spanner/docs/schema-design#primary-key-prevent-hotspots)
`ac_ba2200042104ff014ea989a08211295e4460_a034`->`ac_a034_ba2200042104ff014ea989a08211295e4460`
`ac_ba2200042104ff014ea989a08211295e4460_a036`->`ac_a036_ba2200042104ff014ea989a08211295e4460`
`ac_ba2200042104ff014ea989a08211295e4460_a040`->`ac_a040_ba2200042104ff014ea989a08211295e4460`

Spanner indices are essentially tables. Currently ledgers of the same account likely will hit the index with a hotspot as they are within close range.

# Decisions
TBD