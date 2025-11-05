---
title: Partitioning
---

For performance and to facilitate incremental updates, the summary table is
partitioned according to the property tagged with the `#[Analytics\Partition]`
attribute. This property must be a Doctrine embeddable that implements
`Partition`.

:::warning

The partitioning term used here is different from the partitioning term used in
the database.

:::

## Best Practices, or TLDR;

If the source entity uses an auto-incrementing integer primary key, use this
partitioning scheme:

```php
use Doctrine\ORM\Mapping as ORM;
use Rekalogika\Analytics\Core\Metadata as Analytics;
use Rekalogika\Analytics\Core\Entity\BaseSummary;
use Rekalogika\Analytics\Core\Partition\DefaultIntegerPartition;
use Rekalogika\Analytics\Core\ValueResolver\PropertyValue;

class YourSummary extends BaseSummary
{
    // highlight-start
    #[ORM\Embedded()]
    #[Analytics\Partition(new PropertyValue('id'))]
    private DefaultIntegerPartition $partition;
    // highlight-end
}
```

If your source entity uses UUIDv7 (or ULID) as the primary key, use this
partitioning scheme:

```php
use Doctrine\ORM\Mapping as ORM;
use Rekalogika\Analytics\Core\Metadata as Analytics;
use Rekalogika\Analytics\Core\Partition\UuidV7IntegerPartition;
use Rekalogika\Analytics\Core\Entity\BaseSummary;
use Rekalogika\Analytics\Uuid\ValueResolver\StringUuidToTruncatedInteger;

class YourSummary extends BaseSummary
{
    // highlight-start
    #[ORM\Embedded()]
    #[Analytics\Partition(new StringUuidToTruncatedInteger('id'))]
    private UuidV7IntegerPartition $partition;
    // highlight-end
}
```

## Concepts

A property of the source entity is designated the **partitioning key**. The key
is used to partition the data. The key is usually the primary key of the source
entity, but not necessarily so. The key must be monotonic, or always increasing,
but not necessarily unique. And the key must be permanent, it must not change in
the lifetime of the entity.

Partitioning is divided into **levels**. Each level consists of multiple
**partition** of the same length, one after the other. Levels are indicated by a
number. A lower level has a shorter length than a higher level.

A partition is indicated by the **level** and the **key**. A partition of a
level consists of several partitions of the lower level, except the lowermost
level.

Records from the source entity are grouped by a specific lowest level partition
according to the **partitioning key**, and rolled up into that partition. Then,
eventually, the lowest level accumulates enough partitions, and in turn they are
rolled up into the next higher level partition. And so on, until the highest
level is reached.

If new source entities are added, they will be rolled up to the newest lowest
level partition, and the framework does not need to reprocess the entire summary
table.

If changes are detected in the old records, the lowest partition is marked as
dirty. The framework will reprocess the dirty partition, then mark the higher
level partition as dirty, and so on, until it bubbles up to the highest level.
Again, the framework does not need to reprocess the entire summary table.

## Available Partitioning Strategies

### `DefaultIntegerPartition`

Suitable for partitioning auto-incrementing integer primary keys. It partitions
using 11, 22, 33, 44, and 55 bits of width. A 11-bit partition aggregates up to
2048 records.

### `UuidV7IntegerPartition`

Suitable for partitioning UUIDv7 (also ULID) primary keys. It should be coupled
by a `StringUuidToTruncatedInteger` value resolver that truncates the 128-bit
UUID to a 48-bit integer.

UUIDv7 (and ULID) stores the creation time in the first 48 bits. So, the widths
of each level correspond to the following intervals:

- 22 bits: corresponds to 1.165 hour interval
- 27 bits: corresponds to 1.6 days interval
- 32 bits: corresponds to 50 days interval
- 37 bits: corresponds to 4.3 years interval

It means the lowest level will contain rolled-up records of those created within
1.165 hour interval. The second level will contain rolled-up records of those
created within 1.6 days interval, and so on.

### Custom Integer Partition

You can create a custom integer partition by extending `IntegerPartition`.

### Custom Non-Integer Partition

You might be able to create your own non-integer partition by implementing the
`Partition` interface, but currently this is untested and unsupported.

