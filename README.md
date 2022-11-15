# edb_failover_slots

An extension that makes logical replication slots practically usable across
physical failover.

This extension does the following:

- copy any missing slots from primary to standby
- remove any slots from standby that are not found on primary
- periodically synchronize position of slots on standby based on primary
- ensure that selected standbys receive data before any of the logical slot
  walsenders can send data to consumers

PostgreSQL 11 on higher is required.

## Configuration options

The extension itself must be added to `shared_preload_libraries` on both the
primary instance as well as any standby that is used for high availability
(failover or switchover) purposes.

The behavior of edb_failover_slots is configurable using these configuration
options (set in `postgresql.conf`).

### edb_failover_slots.synchronize_slot_names

This standby option allows setting which logical slots should be synchronized
to this physical standby. It's a comma-separated list of slot filters.

A slot filter is defined as  `key:value` pair (separated by colon) where `key`
can be one of:

 - `name` - specifies to match exact slot name
 - `name_like` - specifies to match slot name against SQL `LIKE` expression
 - `plugin` - specifies to match slot plugin name against the value

The `key` can be omitted and will default to `name` in that case.

For example, `'my_slot_name,plugin:test_decoding'` will
synchronize the slot named "my_slot_name" and any slots that use the test_decoding plugin.

If this is set to an empty string, no slots will be synchronized to this physical
standby.

The default value is `'name_like:%'`, which means all logical replication slots
will be synchronized.


### edb_failover_slots.drop_extra_slots

This standby option controls what happens to extra slots on the standby that are
not found on the primary using the `edb_failover_slots.synchronize_slot_names` filter.
If it's set to true (which is the default), they will be dropped, otherwise
they will be kept.

### edb_failover_slots.primary_dsn

A standby option for specifying the connection string to use to connect to the
primary when fetching slot information.

If empty (default), then use same connection string as `primary_conninfo`.

Note that `primary_conninfo` cannot be used if there is a `password` field in
the connection string because it gets obfuscated by PostgreSQL and
edb_failover_slots can't actually see the password. In this case,
`edb_failover_slots.primary_dsn` must be configured.

### edb_failover_slots.standby_slot_names

This option is typically used in failover configurations to ensure that the
failover-candidate streaming physical replica(s) have received and flushed
all changes before they ever become visible to any subscribers. That guarantees
that a commit cannot vanish on failover to a standby for the consumer of a logical
slot.

Replication slots whose names are listed in the comma-separated
`edb_failover_slots.standby_slot_names` list are treated specially by the
walsender on the primary.

Logical replication walsenders will ensure that all local changes are sent and
flushed to the replication slots in `edb_failover_slots.standby_slot_names`
before the walsender sends those changes for the logical replication slots.
Effectively, it provides a synchronous replication barrier between the named
list of slots and all the consumers of logically decoded streams from walsender.

Any replication slot may be listed in `edb_failover_slots.standby_slot_names`;
both logical and physical slots work, but it's generally used for physical
slots.

Without this safeguard, two anomalies are possible where a commit can be
received by a subscriber and then vanish from the provider on failover because
the failover candidate hadn't received it yet:

* For 1+ subscribers, the subscriber may have applied the change but the new
  provider may execute new transactions that conflict with the received change,
  as it never happened as far as the provider is concerned;

and/or

* For 2+ subscribers, at the time of failover, not all subscribers have applied
  the change. The subscribers now have inconsistent and irreconcilable states
  because the subscribers that didn't receive the commit have no way to get it
  now.

Setting `edb_failover_slots.standby_slot_names` will (by design) cause subscribers to
lag behind the provider if the provider's failover-candidate replica(s) are not
keeping up. Monitoring is thus essential.

### edb_failover_slots.standby_slots_min_confirmed

Controls how many of the `edb_failover_slots.standby_slot_names` have to
confirm before we send data through the logical replication
slots. Setting -1 (the default) means to wait for all entries in
`edb_failover_slots.standby_slot_names`.
