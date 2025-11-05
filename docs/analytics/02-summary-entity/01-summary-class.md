---
title: Summary Class
---

A summary entity is an entity that contains pre-aggregated data from the source
entity. To work with this package, you need to create one or more summary
entities for each source entity that you want to analyze.

## Example Source Entity

This is the example of an entity that we would like to analyze. Here, we have an
`Order`:

```php
use Doctrine\DBAL\Types\Types;
use Doctrine\ORM\Mapping as ORM;

#[ORM\Entity()]
class Order
{
    #[ORM\Id]
    #[ORM\GeneratedValue]
    #[ORM\Column]
    private ?int $id = null;

    #[ORM\ManyToOne()]
    private ?Item $item = null;

    #[ORM\ManyToOne()]
    private ?Customer $customer = null;

    #[ORM\Column(type: Types::DATETIME_MUTABLE)]
    private ?\DateTimeInterface $time = null;

    // setters, getters and other logic are omitted for brevity
}
```

The source entity must have a key field that has the following properties:

* Permanent, the value never changes in the lifetime of the entity.
* Monotonic, always increases.

Most of the time, you should be able to use the primary key of the entity as the
key field.

## The Summary Entity

This is an example summary entity for the above `Order` entity. A summary table
is a standard Doctrine entity with additional attributes that define how the
data is rolled up from the source entity:

```php
use Brick\Money\Money;
use Doctrine\DBAL\Types\Types;
use Doctrine\ORM\Mapping as ORM;
use Rekalogika\Analytics\Core\AggregateFunction\Count;
use Rekalogika\Analytics\Core\AggregateFunction\Sum;
use Rekalogika\Analytics\Core\Metadata as Analytics;
use Rekalogika\Analytics\Core\Partition\DefaultIntegerPartition;
use Rekalogika\Analytics\Core\Entity\BaseSummary;
use Rekalogika\Analytics\Core\ValueResolver\IdentifierValue;
use Rekalogika\Analytics\Core\ValueResolver\PropertyValue;
use Rekalogika\Analytics\Time\Dimension\System\GregorianDateWithHour
use Rekalogika\Analytics\Time\Metadata\TimeProperties;
use Symfony\Component\Translation\TranslatableMessage;

#[ORM\Entity()]
#[Analytics\Summary(
    sourceClass: Order::class,
    label: new TranslatableMessage('Orders'),
)]
class OrderSummary extends BaseSummary
{
    // 1. Partition

    #[ORM\Embedded()]
    #[Analytics\Partition(new PropertyValue('id'))]
    private DefaultIntegerPartition $partition;

    // 2. Dimensions

    #[ORM\Embedded()]
    #[Analytics\Dimension(
        source: new PropertyValue('time'),
        label: new TranslatableMessage('Time'),
    )]
    #[TimeProperties(
        sourceTimeZone: new \DateTimeZone('UTC'),
        summaryTimeZone: new \DateTimeZone('Asia/Jakarta'),
    )]
    private GregorianDateWithHour $time;

    #[ORM\ManyToOne()]
    #[Analytics\Dimension(
        source: new IdentifierValue('customer.country'),
        label: new TranslatableMessage('Customer Country'),
    )]
    private ?Country $customerCountry = null;

    #[ORM\ManyToOne()]
    #[Analytics\Dimension(
        source: new IdentifierValue('customer.country.region'),
        label: new TranslatableMessage('Customer Region'),
    )]
    private ?Region $customerRegion = null;

    #[ORM\Column(enumType: Gender::class, nullable: true)]
    #[Analytics\Dimension(
        source: new PropertyValue('customer.gender'),
        label: new TranslatableMessage('Customer Gender'),
    )]
    private ?Gender $customerGender = null;

    // 3. Measures

    #[ORM\Column(type: Types::INTEGER)]
    #[Analytics\Measure(
        function: new Sum('item.price'),
        label: new TranslatableMessage('Price'),
    )]
    private ?int $price = null;

    #[ORM\Column(type: Types::INTEGER)]
    #[Analytics\Measure(
        function: new Count('id'),
        label: new TranslatableMessage('Count'),
    )]
    private ?int $count = null;

    // 4. An example getter with business logic

    public function getPrice(): ?Money
    {
        if ($this->price === null) {
            return null;
        }

        return Money::ofMinor($this->price, 'EUR');
    }
}
```

## Sections

### 1. Partition

The `partition` attribute is used to define how the data is partitioned. The
default `DefaultIntegerPartition` here should be sufficient for
auto-incrementing primary key of the source entity.

Read more about partitions in the [partitioning](partitioning) section.

### 2. Dimensions

These are the properties that have distinct, descriptive values. You will use
these properties to filter and group the data. Using SQL as an analogy, these
are the fields that you would use in a `GROUP BY` and/or `WHERE` clause.

All dimensions are indicated by the `#[Analytics\Dimension]` attribute. The most
important argument is the `source` argument. This argument is used to resolve
the value of the dimension from the source entity. A `PropertyValue`
points to the value of a property in the source entity. An `IdentifierValue`
points to a related entity.

A dimension can be hierarchical, like the `time` dimension above. A hierarchical
dimension is modeled using a Doctrine embeddable. Inside the class, the time
dimension is further divided into `year`, `month`, `day`, and more.

Read more about dimensions in the [dimensions](dimensions) section.

### 3. Measures

These are the properties that you want to aggregate. Using SQL as an analogy,
you would use these fields in a `SUM`, `COUNT`, and other aggregate functions.

Measures are indicated by the `#[Analytics\Measure]` attribute. The most
important argument is the `function` argument. It is used to define the
aggregation function.

Read more about measures in the [measures](measures) section.

## 4. Getters

Just like a regular Doctrine entity, you can define getters in the summary
entity. You can also have simple business logic in these getters. For example,
the `getPrice()` getter above converts the price to a `Money` object.

## Labels and Translations

All the items in the summary entity have a `label` attribute that accepts a
string or a `TranslatableInterface`. These labels are used in the user interface
to identify the item, for example in a table header or a chart legend.

If a `TranslatableInterface` is used, the label will be translated using the
Symfony translation component.

## Indexing

The framework automatically creates indexes for the summary table. You don't
need to create any indexes manually.

## Changing Summary Entity

A summary entity should not be changed after it is created and populated.

If you need to change the summary entity, you should create a new one, refresh
the data and wait until it is completed, and then retire the old one. If you
anticipate that you will have to change the summary entity, we suggest
date-coding the summary entity class name, for example `OrderSummary20250115`.

## Summary Entity is an Entity but not an Entity

A summary entity is defined as a Doctrine entity. But it is mainly for defining
the structure of the summary table and the summarization behavior. You will
never interact with a real instance of the summary entity. Instead, you query
the summary table using the `SummaryManager`, and get the result not in the
form of a summary entity.

An event listener is installed to prevent you from accidentally persisting,
updating, or deleting a summary entity.

## `HasQueryBuilderModifier`

The framework works by creating a `QueryBuilder` to roll-up the data from the
source table to the summary table. If you need to modify this `QueryBuilder`, you
can make the summary entity implement the `HasQueryBuilderModifier` interface.

```php
use Doctrine\ORM\QueryBuilder;
use Rekalogika\Analytics\Contracts\Summary\HasQueryBuilderModifier;

class OrderSummary extends Summary implements HasQueryBuilderModifier
{
    // ...

    // highlight-start
    public function modifyQueryBuilder(QueryBuilder $queryBuilder): void
    {
        $queryBuilder->andWhere('root.id > 100000');
    }
    // highlight-end

    // ...
}
```

With the above example, the resulting summary table will only consider the
orders with an ID greater than 100000.
