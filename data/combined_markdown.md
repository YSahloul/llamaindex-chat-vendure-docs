---
title: "Vendure API Reference"
---

This section contains reference documentation for the various APIs exposed by Vendure.

:::info
All of the information in this section is generated directly from the Vendure source code. You can jump directly
to the source file using the links below each heading.

![Source links](./links.webp)
:::

### TypeScript API

These are the classes, interfaces and other TypeScript object which are used when **writing plugins** or custom business logic.

### Core Plugins

These are the TypeScript APIs for the core Vendure plugins.

### GraphQL API

These are the GraphQL APIs you will use to build your storefront (Shop API) or admin integrations (Admin API).

### Admin UI API

These are the Angular components and services you can use when building Admin UI extensions.


---
title: 'Collections'
---

import Tabs from '@theme/Tabs';
import TabItem from '@theme/TabItem';

[`Collections`](/reference/typescript-api/entities/collection/) are used to categorize and organize your catalog. A collection
contains multiple product variants, and a product variant can belong to multiple collections. Collections can be nested to
create a hierarchy of categories, which is typically used to create a menu structure in the storefront.

![Collections](./collections.webp)

Collections are not _only_ used as the basis of storefront navigation. They are a general-purpose organization tool which can be used
for many purposes, such as:

- Creating a collection of "new arrivals" which is used on the homepage.
- Creating a collection of "best sellers" which is used to display a list of popular products.
- Creating a collection of "sale items" which is used to apply a discount to all products in the collection, via a promotion.

## Collection filters

The specific product variants that belong to a collection are determined by the collection's [`CollectionFilters`](/reference/typescript-api/configuration/collection-filter/).
A collection filter is a piece of logic which is used to determine whether a product variant should be included in the collection. By default, Vendure
includes a number of collection filters:

- **Filter by facet values**: Include all product variants which have a specific set of facet values.
- **Filter by product variant name**: Include all product variants whose name matches a specific string.
- **Manually select product variants**: Allows manual selection of individual product variants.
- **Manually select products**: Allows manual selection of entire products, and then includes all variants of those products.

![Collection filters](./collection-filters.webp)

It is also possible to create your own custom collection filters, which can be used to implement more complex logic. See the section on [creating a collection filter](#creating-a-collection-filter) for more details.

### Filter inheritance

When a collection is nested within another collection, the child collection can inherit the parent's collection filters. This means that the child collection
will _combine_ its own filters with the parent's filters.

![Filter inheritance](./filter-inheritance.webp)

In the example above, we have a parent collection "Menswear", with a child collection "Mens' Casual". The parent collection has a filter which includes all
product variants with the "clothing" and "mens" facet values. The child collection is set to inherit the parent's filters, and has an additional filter which
includes all product variants with the "casual" facet value.

Thus, the child collection will include all product variants which have the "clothing", "mens" and "casual" facet values.

:::note
When filter inheritance is enabled, a child collection will contain a **subset** of the product variants of its parent collection.

In order to create a child collection which contains product variants _not_ contained by the parent collection, you must disable filter inheritance
in the child collection.
:::

### Creating a collection filter

You can create your own custom collection filters with the [`CollectionFilter`](/reference/typescript-api/configuration/collection-filter/) class. This class
is a [configurable operation](/guides/developer-guide/strategies-configurable-operations/#configurable-operations) where the specific
filtering logic is implemented in the `apply()` method passed to its constructor.

The `apply()` method receives an instance of the [TypeORM SelectQueryBuilder](https://typeorm.io/select-query-builder) which should have filtering logic
added to it using the `.andWhere()` method.

Here's an example of a collection filter which filters by SKU:

```ts title="src/config/sku-collection-filter.ts"
import { CollectionFilter, LanguageCode } from '@vendure/core';

export const skuCollectionFilter = new CollectionFilter({
    args: {
        // The `args` object defines the user-configurable arguments
        // which will get passed to the filter's `apply()` function.
        sku: {
            type: 'string',
            label: [{ languageCode: LanguageCode.en, value: 'SKU' }],
            description: [
                {
                    languageCode: LanguageCode.en,
                    value: 'Matches any product variants with an SKU containing this value',
                },
            ],
        },
    },
    code: 'variant-sku-filter',
    description: [{ languageCode: LanguageCode.en, value: 'Filter by matching SKU' }],

    // This is the function that defines the logic of the filter.
    apply: (qb, args) => {
        // Sometimes syntax differs between database types, so we use
        // the `type` property of the connection options to determine
        // which syntax to use.
        const LIKE = qb.connection.options.type === 'postgres' ? 'ILIKE' : 'LIKE';

        return qb.andWhere(`productVariant.sku ${LIKE} :sku`, {
            sku: `%${args.sku}%`
        });
    },
});
```

In the `apply()` method, the product variant entity is aliased as `'productVariant'`.

This custom filter is then added to the defaults in your config:

```ts title="src/vendure-config.ts"
import { defaultCollectionFilters, VendureConfig } from '@vendure/core';
import { skuCollectionFilter } from './config/sku-collection-filter';

export const config: VendureConfig = {
    // ...
    catalogOptions: {
        collectionFilters: [
            ...defaultCollectionFilters,
            // highlight-next-line
            skuCollectionFilter
        ],
    },
};
```

:::info
To see some more advanced collection filter examples, you can look at the source code of the
[default collection filters](https://github.com/vendure-ecommerce/vendure/blob/master/packages/core/src/config/catalog/default-collection-filters.ts).
:::


---
title: "Customers"
---

A [`Customer`](/reference/typescript-api/entities/customer/) is a person who can buy from your shop. A customer can have one or more
[`Addresses`](/reference/typescript-api/entities/address/), which are used for shipping and billing.

If a customer has registered an account, they will have an associated [`User`](/reference/typescript-api/entities/user/). The user
entity is used for authentication and authorization. **Guest checkouts** are also possible, in which case a customer will not have a user.

:::info
See the [Auth guide](/guides/core-concepts/auth/#customer-auth) for a detailed explanation of the relationship between
customers and users.
:::

![Customer](./customer.webp)

Customers can be organized into [`CustomerGroups`](/reference/typescript-api/entities/customer-group/). These groups can be used in
logic relating to promotions, shipping rules, payment rules etc. For example, you could create a "VIP" customer group and then create
a promotion which grants members of this group free shipping. Or a "B2B" group which is used in a custom tax calculator to
apply a different tax rate to B2B customers.


---
title: "Email & Notifications"
---

A typical ecommerce application needs to notify customers of certain events, such as when they place an order or
when their order has been shipped. This is usually done via email, but can also be done via SMS or push notifications.

## Email

Email is the most common way to notify customers of events, so a default Vendure installation includes our [EmailPlugin](/reference/core-plugins/email-plugin).

The EmailPlugin by default uses [Nodemailer](https://nodemailer.com/about/) to send emails via a variety of
different transports, including SMTP, SendGrid, Mailgun, and more.
The plugin is configured with a list of [EmailEventHandlers](/reference/core-plugins/email-plugin/email-event-handler) which are responsible for
sending emails in response to specific events.

:::note
This guide will cover some of the main concepts of the EmailPlugin, but for a more in-depth look at how to configure
and use it, see the [EmailPlugin API docs](/reference/core-plugins/email-plugin).
:::

Here's an illustration of the flow of an email being sent:

![Email plugin flow](./email-plugin-flow.webp)

All emails are triggered by a particular [Event](/guides/developer-guide/events/) - in this case when the state of an
Order changes. The EmailPlugin ships with a set of [default email handlers](https://github.com/vendure-ecommerce/vendure/blob/master/packages/email-plugin/src/default-email-handlers.ts),
one of which is responsible for sending "order confirmation" emails.

### EmailEventHandlers

Let's take a closer look at a simplified version of the `orderConfirmationHandler`:

```ts
import { OrderStateTransitionEvent } from '@vendure/core';
import { EmailEventListener, transformOrderLineAssetUrls, hydrateShippingLines } from '@vendure/email-plugin';

// The 'order-confirmation' string is used by the EmailPlugin to identify
// which template to use when rendering the email.
export const orderConfirmationHandler = new EmailEventListener('order-confirmation')
    .on(OrderStateTransitionEvent)
    // Only send the email when the Order is transitioning to the
    // "PaymentSettled" state and the Order has a customer associated with it.
    .filter(
        event =>
            event.toState === 'PaymentSettled'
            && !!event.order.customer,
    )
    // We commonly need to load some additional data to be able to render the email
    // template. This is done via the `loadData()` method. In this method we are
    // mutating the Order object to ensure that product images are correctly
    // displayed in the email, as well as fetching shipping line data from the database.
    .loadData(async ({ event, injector }) => {
        transformOrderLineAssetUrls(event.ctx, event.order, injector);
        const shippingLines = await hydrateShippingLines(event.ctx, event.order, injector);
        return { shippingLines };
    })
    // Here we are setting the recipient of the email to be the
    // customer's email address.
    .setRecipient(event => event.order.customer!.emailAddress)
    // We can interpolate variables from the EmailPlugin's configured
    // `globalTemplateVars` object.
    .setFrom('{{ fromAddress }}')
    // We can also interpolate variables made available by the
    // `setTemplateVars()` method below
    .setSubject('Order confirmation for #{{ order.code }}')
    // The object returned here defines the variables which are
    // available to the email template.
    .setTemplateVars(event => ({ order: event.order, shippingLines: event.data.shippingLines }))
```

To recap:

- The handler listens for a specific event
- It optionally filters those events to determine whether an email should be sent
- It specifies the details of the email to be sent, including the recipient, subject, template variables, etc.

The full range of methods available when setting up an EmailEventHandler can be found in the [EmailEventHandler API docs](/reference/core-plugins/email-plugin/email-event-handler).

### Email variables

In the example above, we used the `setTemplateVars()` method to define the variables which are available to the email template.
Additionally, there are global variables which are made available to _all_ email templates & EmailEventHandlers. These are
defined in the `globalTemplateVars` property of the EmailPlugin config:

```ts title="src/vendure-config.ts"
import { VendureConfig } from '@vendure/core';
import { EmailPlugin } from '@vendure/email-plugin';

export const config: VendureConfig = {
    // ...
    plugins: [
        EmailPlugin.init({
            // ...
            // highlight-start
            globalTemplateVars: {
                fromAddress: '"MyShop" <noreply@myshop.com>',
                verifyEmailAddressUrl: 'https://www.myshop.com/verify',
                passwordResetUrl: 'https://www.myshop.com/password-reset',
                changeEmailAddressUrl: 'https://www.myshop.com/verify-email-address-change'
            },
            // highlight-end
        }),
    ],
};
```

### Email integrations

The EmailPlugin is designed to be flexible enough to work with many different email services. The default
configuration uses Nodemailer to send emails via SMTP, but you can easily configure it to use a different
transport. For instance:

- [AWS SES](https://www.vendure.io/marketplace/aws-ses)
- [SendGrid](https://www.vendure.io/marketplace/sendgrid)

## Other notification methods

The pattern of listening for events and triggering some action in response is not limited to emails. You can
use the same pattern to trigger other actions, such as sending SMS messages or push notifications. For instance,
let's say you wanted to create a plugin which sends an SMS message to the customer when their order is shipped.

:::note
This is just a simplified example to illustrate the pattern.
:::

```ts title="src/plugins/sms-plugin/sms-plugin.ts"
import { OnModuleInit } from '@nestjs/common';
import { PluginCommonModule, VendurePlugin, EventBus } from '@vendure/core';
import { OrderStateTransitionEvent } from '@vendure/core';

// A custom service which sends SMS messages
// using a third-party SMS provider such as Twilio.
import { SmsService } from './sms.service';

@VendurePlugin({
    imports: [PluginCommonModule],
    providers: [SmsService],
})
export class SmsPlugin implements OnModuleInit {
    constructor(
        private eventBus: EventBus,
        private smsService: SmsService,
    ) {}

    onModuleInit() {
        this.eventBus
            .ofType(OrderStateTransitionEvent)
            .filter(event => event.toState === 'Shipped')
            .subscribe(event => {
                this.smsService.sendOrderShippedMessage(event.order);
            });
    }
}
```

---
title: "Images & Assets"
---

[`Assets`](/reference/typescript-api/entities/asset/) are used to store files such as images, videos, PDFs, etc. Assets can be
assigned to **products**, **variants** and **collections** by default. By using [custom fields](/guides/developer-guide/custom-fields/) it is
possible to assign assets to other entities. For example, for implementing customer profile images.

The handling of assets in Vendure is implemented in a modular way, allowing you full control over the way assets
are stored, named, imported and previewed.

![Asset creation and retrieval](./asset-flow.webp)

1. An asset is created by uploading an image. Internally the [`createAssets` mutation](/reference/graphql-api/admin/mutations/#createassets) will be executed.
2. The [`AssetNamingStrategy`](/reference/typescript-api/assets/asset-naming-strategy/) is used to generate file names for the source image and the preview. This is useful for normalizing file names as well as handling name conflicts.
3. The [`AssetPreviewStrategy`](/reference/typescript-api/assets/asset-preview-strategy) generates a preview image of the asset. For images, this typically involves creating a version with constraints on the maximum dimensions. It could also be used to e.g. generate a preview image for uploaded PDF files, videos or other non-image assets (such functionality would require a custom `AssetPreviewStrategy` to be defined).
4. The source file as well as the preview image are then passed to the [`AssetStorageStrategy`](/reference/typescript-api/assets/asset-storage-strategy) which stores the files to some form of storage. This could be the local disk or an object store such as AWS S3 or Minio.
5. When an asset is later read, e.g. when a customer views a product detail page which includes an image of the product, the `AssetStorageStrategy` can be used to
read the file from the storage location.


## AssetServerPlugin

Vendure comes with the `@vendure/asset-server-plugin` package pre-installed. This provides the [`AssetServerPlugin`](/reference/core-plugins/asset-server-plugin/) which provides many advanced features to make working with
assets easier.

The plugin provides a ready-made set of strategies for handling assets, but also allows you to replace these defaults with
your own implementations. For example, here are instructions on how to replace the default storage strategy with one
that stores your assets on AWS S3 or Minio: [configure S3 asset storage](/reference/core-plugins/asset-server-plugin/s3asset-storage-strategy#configures3assetstorage)

It also features a powerful image transformation API, which allows you to specify the dimensions, crop, and image format
using query parameters.

:::info
See the [AssetServerPlugin docs](/reference/core-plugins/asset-server-plugin/) for a detailed description of all the features.
:::

## Asset Tags

Assets can be tagged. A [`Tag`](/reference/typescript-api/entities/tag/) is a simple text label that can be applied to an asset. An asset can have multiple tags or none. Tags are useful for organizing assets, since assets are otherwise organized as a flat list with no concept of a directory structure.

![Asset tags](./asset-tags.webp)


---
title: "Money & Currency"
---

In Vendure, monetary values are stored as **integers** using the **minor unit** of the selected currency.
For example, if the currency is set to USD, then the integer value `100` would represent $1.00.
This is a common practice in financial applications, as it avoids the rounding errors that can occur when using floating-point numbers.

For example, here's the response from a query for a product's variant prices:

```json
{
  "data": {
    "product": {
      "id": "42",
      "variants": [
        {
          "id": "74",
          "name": "Bonsai Tree",
          "currencyCode": "USD",
          // highlight-start
          "price": 1999,
          "priceWithTax": 2399,
          // highlight-end
        }
      ]
    }
  }
}
```

In this example, the tax-inclusive price of the variant is `$23.99`.

:::info
To illustrate the problem with storing money as decimals, imagine that we want to add the price of two items:

- Product A: `$1.21`
- Product B: `$1.22`

We should expect the sum of these two amounts to equal `$2.43`. However, if we perform this addition in JavaScript (and the same
holds true for most common programming languages), we will instead get `$2.4299999999999997`!

For a more in-depth explanation of this issue, see [this StackOverflow answer](https://stackoverflow.com/a/3730040/772859)
:::

## Displaying monetary values

When you are building your storefront, or any other client that needs to display monetary values in a human-readable form,
you need to divide by 100 to convert to the major currency unit and then format with the correct decimal & grouping dividers.

In JavaScript environments such as browsers & Node.js, we can take advantage of the excellent [`Intl.NumberFormat` API](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Intl/NumberFormat).

Here's a function you can use in your projects:

```ts title="src/utils/format-currency.ts"
export function formatCurrency(value: number, currencyCode: string, locale?: string) {
    const majorUnits = value / 100;
    try {
        // Note: if no `locale` is provided, the browser's default
        // locale will be used.
        return new Intl.NumberFormat(locale, {
            style: 'currency',
            currency: currencyCode,
        }).format(majorUnits);
    } catch (e: any) {
        // A fallback in case the NumberFormat fails for any reason
        return majorUnits.toFixed(2);
    }
}
```

If you are building an Admin UI extension, you can use the built-in [`LocaleCurrencyPipe`](/reference/admin-ui-api/pipes/locale-currency-pipe/):

```html title="src/plugins/my-plugin/ui/components/my-component/my.component.html"

```

## Support for multiple currencies

Vendure supports multiple currencies out-of-the-box. The available currencies must first be set at the Channel level
(see the [Channels, Currencies & Prices section](/guides/core-concepts/channels/#channels-currencies--prices)), and then
a price may be set on a `ProductVariant` in each of the available currencies.

When using multiple currencies, the [ProductVariantPriceSelectionStrategy](/reference/typescript-api/configuration/product-variant-price-selection-strategy/)
is used to determine which of the available prices to return when fetching the details of a `ProductVariant`. The default strategy
is to return the price in the currency of the current session request context, which is determined firstly by any `?currencyCode=XXX` query parameter
on the request, and secondly by the `defaultCurrencyCode` of the Channel.

## The GraphQL `Money` scalar

In the GraphQL APIs, we use a custom [`Money` scalar type](/reference/graphql-api/admin/object-types/#money) to represent
all monetary values. We do this for two reasons:

1. The built-in `Int` type is that the GraphQL spec imposes an upper limit of
`2147483647`, which in some cases (especially currencies with very large amounts) is not enough.
2. Very advanced use-cases might demand more precision than is possible with an integer type. Using our own custom
scalar gives us the possibility of supporting more precision.

Here's how the `Money` scalar is used in the `ShippingLine` type:

```graphql
type ShippingLine {
    id: ID!
    shippingMethod: ShippingMethod!
    // highlight-start
    price: Money!
    priceWithTax: Money!
    discountedPrice: Money!
    discountedPriceWithTax: Money!
    // highlight-end
    discounts: [Discount!]!
}
```

If you are defining custom GraphQL types, or adding fields to existing types (see the [Extending the GraphQL API doc](/guides/developer-guide/extend-graphql-api/)),
then you should also use the `Money` scalar for any monetary values.


## The `@Money()` decorator

When [defining new database entities](/guides/developer-guide/database-entity//), if you need to store a monetary value, then rather than using the TypeORM `@Column()`
decorator, you should use Vendure's [`@Money()` decorator](/reference/typescript-api/money/money-decorator).

Using this decorator allows Vendure to correctly store the value in the database according to the configured `MoneyStrategy` (see below).

```ts title="src/plugins/quote/entities/quote.entity.ts"
import { DeepPartial } from '@vendure/common/lib/shared-types';
import { VendureEntity, Order, EntityId, Money, CurrencyCode, ID } from '@vendure/core';
import { Column, Entity, ManyToOne } from 'typeorm';

@Entity()
class Quote extends VendureEntity {
    constructor(input?: DeepPartial<Quote>) {
        super(input);
    }

    @ManyToOne(type => Order)
    order: Order;

    @EntityId()
    orderId: ID;

    @Column()
    text: string;

    // highlight-start
    @Money()
    value: number;
    // highlight-end

    // Whenever you store a monetary value, it's a good idea to also
    // explicitly store the currency code too. This makes it possible
    // to support multiple currencies and correctly format the amount
    // when displaying the value.
    @Column('varchar')
    currencyCode: CurrencyCode;

    @Column()
    approved: boolean;
}
```

## Advanced configuration: MoneyStrategy

For advanced use-cases, it is possible to configure aspects of how Vendure handles monetary values internally by defining
a custom [`MoneyStrategy`](/reference/typescript-api/money/money-strategy/).

The `MoneyStrategy` allows you to define:

- How the value is stored and retrieved from the database
- How rounding is applied internally

For example, in addition to the [`DefaultMoneyStrategy`](/reference/typescript-api/money/default-money-strategy), Vendure
also provides the [`BigIntMoneyStrategy`](/reference/typescript-api/money/big-int-money-strategy) which stores monetary values
using the `bigint` data type, allowing much larger amounts to be stored.

Here's how you would configure your server to use this strategy:

```ts title="src/vendure-config.ts"
import { VendureConfig, BigIntMoneyStrategy } from '@vendure/core';

export const config: VendureConfig = {
    // ...
    entityOptions: {
        moneyStrategy: new BigIntMoneyStrategy(),
    }
}
```


---
title: "Products"
---

Your catalog is composed of [`Products`](/reference/typescript-api/entities/product/) and [`ProductVariants`](/reference/typescript-api/entities/product-variant/).
A `Product` always has _at least one_ `ProductVariant`. You can think of the product as a "container" which includes a name, description, and images that apply to all of
its variants.

Here's a visual example, in which we have a "Hoodie" product which is available in 3 sizes. Therefore, we have
3 variants of that product:

![Products and ProductVariants](./products-variants.webp)

Multiple variants are made possible by adding one or more [`ProductOptionGroups`](/reference/typescript-api/entities/product-option-group) to
the product. These option groups then define the available [`ProductOptions`](/reference/typescript-api/entities/product-option)

If we were to add a new option group to the example above for "Color", with 2 options, "Black" and "White", then in total
we would be able to define up to 6 variants:

- Hoodie Small Black
- Hoodie Small White
- Hoodie Medium Black
- Hoodie Medium White
- Hoodie Large Black
- Hoodie Large White

:::info
When a customer adds a product to their cart, they are adding a specific `ProductVariant` to their cart, not the `Product` itself.
It is the `ProductVariant` that contains the SKU ("stock keeping unit", or product code) and price information.
:::

## Product price and stock

The `ProductVariant` entity contains the price and stock information for a product. Since a given product variant can have more
than one price, and more than one stock level (in the case of multiple warehouses), the `ProductVariant` entity contains
relations to one or more [`ProductVariantPrice`](/reference/typescript-api/entities/product-variant-price) entities and
one or more [`StockLevel`](/reference/typescript-api/entities/stock-level) entities.

![Price and stock](./product-relations.webp)

## Facets

[`Facets`](/reference/typescript-api/entities/facet/) are used to add structured labels to products and variants. A facet has
one or more [`FacetValues`](/reference/typescript-api/entities/facet-value/). Facet values can be assigned to products or
product variants.

For example, a "Brand" facet could be used to label products with the brand name, with each facet value representing a different brand. You can
also use facets to add other metadata to products and variants such as "New", "Sale", "Featured", etc.

![Facets and FacetValues](./facets.webp)

These are the typical uses of facets in Vendure:

- As the **basis of [Collections](/guides/core-concepts/collections)**, in order to categorize your catalog.
- To **filter products** in the storefront, also known as "faceted search". For example, a customer is on the "hoodies" collection
page and wants to filter to only show Nike hoodies.
- For **internal logic**, such as a promotion that applies to all variants with the "Summer Sale" facet value, or a shipping calculation
that applies a surcharge to all products with the "Fragile" facet value. Such facets can be set to be private so that they
are not exposed to the storefront.


---
title: "Taxes"
showtoc: true
---

E-commerce applications need to correctly handle taxes such as sales tax or value added tax (VAT). In Vendure, tax handling consists of:

* **Tax categories** Each ProductVariant is assigned to a specific TaxCategory. In some tax systems, the tax rate differs depending on the type of good. For example, VAT in the UK has 3 rates, "standard" (most goods), "reduced" (e.g. child car seats) and "zero" (e.g. books).
* **Tax rates** This is the tax rate applied to a specific tax category for a specific [Zone](/reference/typescript-api/entities/zone/). E.g., the tax rate for "standard" goods in the UK Zone is 20%.
* **Channel tax settings** Each Channel can specify whether the prices of product variants are inclusive of tax or not, and also specify the default Zone to use for tax calculations.
* **TaxZoneStrategy** Determines the active tax Zone used when calculating what TaxRate to apply. By default, it uses the default tax Zone from the Channel settings.
* **TaxLineCalculationStrategy** This determines the taxes applied when adding an item to an Order. If you want to integrate a 3rd-party tax API or other async lookup, this is where it would be done.

## API conventions

In the GraphQL API, any type which has a taxable price will split that price into two fields: `price` and `priceWithTax`. This pattern also holds for other price fields, e.g.

```graphql
query {
  activeOrder {
    ...on Order {
      lines {
        linePrice
        linePriceWithTax
      }
      subTotal
      subTotalWithTax
      shipping
      shippingWithTax
      total
      totalWithTax
    }
  }
}
```

In your storefront, you can therefore choose whether to display the prices with or without tax, according to the laws and conventions of the area in which your business operates.

## Calculating tax on order lines

When a customer adds an item to the Order, the following logic takes place:

1. The price of the item, and whether that price is inclusive of tax, is determined according to the configured [OrderItemPriceCalculationStrategy](/reference/typescript-api/orders/order-item-price-calculation-strategy/).
2. The active tax Zone is determined based on the configured [TaxZoneStrategy](/reference/typescript-api/tax/tax-zone-strategy/).
3. The applicable TaxRate is fetched based on the ProductVariant's TaxCategory and the active tax Zone determined in step 1.
4. The `TaxLineCalculationStrategy.calculate()` of the configured [TaxLineCalculationStrategy](/reference/typescript-api/tax/tax-line-calculation-strategy/) is called, which will return one or more [TaxLines](/reference/graphql-api/admin/object-types/#taxline).
5. The final `priceWithTax` of the order line is calculated based on all the above.

## Calculating tax on shipping

The taxes on shipping is calculated by the [ShippingCalculator](/reference/typescript-api/shipping/shipping-calculator/) of the Order's selected [ShippingMethod](/reference/typescript-api/entities/shipping-method/).

## Configuration

This example shows the default configuration for taxes (you don't need to specify this in your own config, as these are the defaults):

```ts title="src/vendure-config.ts"
import {
    DefaultTaxLineCalculationStrategy,
    DefaultTaxZoneStrategy,
    DefaultOrderItemPriceCalculationStrategy,
    VendureConfig
} from '@vendure/core';

export const config: VendureConfig = {
  taxOptions: {
    taxZoneStrategy: new DefaultTaxZoneStrategy(),
    taxLineCalculationStrategy: new DefaultTaxLineCalculationStrategy(),
  },
  orderOptions: {
    orderItemPriceCalculationStrategy: new DefaultOrderItemPriceCalculationStrategy()
  }
}

```


---
title: "Error Handling"
showtoc: true
---

import Tabs from '@theme/Tabs';
import TabItem from '@theme/TabItem';
import Stackblitz from '@site/src/components/Stackblitz';

Errors in Vendure can be divided into two categories:

* Unexpected errors
* Expected errors

These two types have different meanings and are handled differently from one another.

## Unexpected Errors

This type of error occurs when something goes unexpectedly wrong during the processing of a request. Examples include internal server errors, database connectivity issues, lacking permissions for a resource, etc. In short, these are errors that are *not supposed to happen*.

Internally, these situations are handled by throwing an Error:

```ts
const customer = await this.findOneByUserId(ctx, user.id);
// in this case, the customer *should always* be found, and if
// not then something unknown has gone wrong...
if (!customer) {
    throw new InternalServerError('error.cannot-locate-customer-for-user');
}
```

In the GraphQL APIs, these errors are returned in the standard `errors` array:

```json
{
  "errors": [
    {
      "message": "You are not currently authorized to perform this action",
      "locations": [
        {
          "line": 2,
          "column": 2
        }
      ],
      "path": [
        "me"
      ],
      "extensions": {
        "code": "FORBIDDEN"
      }
    }
  ],
  "data": {
    "me": null
  }
}
```
So your client applications need a generic way of detecting and handling this kind of error. For example, many http client libraries support "response interceptors" which can be used to intercept all API responses and check the `errors` array.

:::note
GraphQL will return a `200` status even if there are errors in the `errors` array. This is because in GraphQL it is still possible to return good data _alongside_ any errors.
:::

Here's how it might look in a simple Fetch-based client:

```ts title="src/client.ts"
export function query(document: string, variables: Record

</Tabs>

### Handling ErrorResults in plugin code

If you are writing a plugin which deals with internal Vendure service methods that may return ErrorResults,
then you can use the `isGraphQlErrorResult()` function to check whether the result is an ErrorResult:

```ts
import { Injectable} from '@nestjs/common';
import { isGraphQlErrorResult, Order, OrderService, OrderState, RequestContext } from '@vendure/core';

@Injectable()
export class MyService {

    constructor(private orderService: OrderService) {}

    async myMethod(ctx: RequestContext, order: Order, newState: OrderState) {
        const transitionResult = await this.orderService.transitionToState(ctx, order.id, newState);
        if (isGraphQlErrorResult(transitionResult)) {
            // The transition failed with an ErrorResult
            throw transitionResult;
        } else {
            // TypeScript will correctly infer the type of `transitionResult` to be `Order`
            return transitionResult;
        }
    }
}
```

### Handling ErrorResults in client code

Because we know all possible ErrorResult that may occur for a given mutation, we can handle them in an exhaustive manner. In other
words, we can ensure our storefront has some sensible response to all possible errors. Typically this will be done with
a `switch` statement:

```ts
const result = await query(APPLY_COUPON_CODE, { code: 'INVALID-CODE' });

switch (result.applyCouponCode.__typename) {
    case 'Order':
        // handle success
        break;
    case 'CouponCodeExpiredError':
        // handle expired code
        break;
    case 'CouponCodeInvalidError':
        // handle invalid code
        break;
    case 'CouponCodeLimitError':
        // handle limit error
        break;
    default:
        // any other ErrorResult can be handled with a generic error message
}
```

If we combine this approach with [GraphQL code generation](/guides/storefront/codegen/), then TypeScript will even be able to
help us ensure that we have handled all possible ErrorResults:

```ts
// Here we are assuming that the APPLY_COUPON_CODE query has been generated
// by the codegen tool, and therefore has the
// type `TypedDocumentNode<ApplyCouponCode, ApplyCouponCodeVariables>`.
const result = await query(APPLY_COUPON_CODE, { code: 'INVALID-CODE' });

switch (result.applyCouponCode.__typename) {
    case 'Order':
        // handle success
        break;
    case 'CouponCodeExpiredError':
        // handle expired code
        break;
    case 'CouponCodeInvalidError':
        // handle invalid code
        break;
    case 'CouponCodeLimitError':
        // handle limit error
        break;
    default:
        // highlight-start
        // this line will cause a TypeScript error if there are any
        // ErrorResults which we have not handled in the switch cases
        // above.
        const _exhaustiveCheck: never = result.applyCouponCode;
        // highlight-end
}
```

## Live example

Here is a live example which the handling of unexpected errors as well as ErrorResults:




---
title: "Events"
---

Vendure emits events which can be subscribed to by plugins. These events are published by the [EventBus](/reference/typescript-api/events/event-bus/) and
likewise the `EventBus` is used to subscribe to events.

An event exists for virtually all significant actions which occur in the system, such as:

- When entities (e.g. `Product`, `Order`, `Customer`) are created, updated or deleted
- When a user registers an account
- When a user logs in or out
- When the state of an `Order`, `Payment`, `Fulfillment` or `Refund` changes

A full list of the available events follows.

## Event types



</div>

## Subscribing to events

To subscribe to an event, use the `EventBus`'s `.ofType()` method. It is typical to set up subscriptions in the `onModuleInit()` or `onApplicationBootstrap()`
lifecycle hooks of a plugin or service (see [NestJS Lifecycle events](https://docs.nestjs.com/fundamentals/lifecycle-events).

Here's an example where we subscribe to the `ProductEvent` and use it to trigger a rebuild of a static storefront:

```ts title="src/plugins/storefront-build/storefront-build.plugin.ts"
import { OnModuleInit } from '@nestjs/common';
import { EventBus, ProductEvent, PluginCommonModule, VendurePlugin } from '@vendure/core';

import { StorefrontBuildService } from './services/storefront-build.service';

@VendurePlugin({
    imports: [PluginCommonModule],
})
export class StorefrontBuildPlugin implements OnModuleInit {
    constructor(
        // highlight-next-line
        private eventBus: EventBus,
        private storefrontBuildService: StorefrontBuildService) {}

    onModuleInit() {
        // highlight-start
        this.eventBus
            .ofType(ProductEvent)
            .subscribe(event => {
                this.storefrontBuildService.triggerBuild();
            });
        // highlight-end
    }
}
```

:::info
The `EventBus.ofType()` and related `EventBus.filter()` methods return an RxJS `Observable`.
This means that you can use any of the [RxJS operators](https://rxjs-dev.firebaseapp.com/guide/operators) to transform the stream of events.

For example, to debounce the stream of events, you could do this:

```ts
// highlight-next-line
import { debounceTime } from 'rxjs/operators';

// ...

this.eventBus
    .ofType(ProductEvent)
     // highlight-next-line
    .pipe(debounceTime(1000))
    .subscribe(event => {
        this.storefrontBuildService.triggerBuild();
    });
```
:::

### Subscribing to multiple event types

Using the `.ofType()` method allows us to subscribe to a single event type. If we want to subscribe to multiple event types, we can use the `.filter()` method instead:

```ts title="src/plugins/my-plugin/my-plugin.plugin.ts"
import { Injectable, OnModuleInit } from '@nestjs/common';
import { EventBus, PluginCommonModule, VendurePlugin, ProductEvent, ProductVariantEvent } from '@vendure/core';

@VendurePlugin({
    imports: [PluginCommonModule],
})
export class MyPluginPlugin implements OnModuleInit {
    constructor(private eventBus: EventBus) {}

    onModuleInit() {
        this.eventBus
            // highlight-start
            .filter(event =>
                event instanceof ProductEvent || event instanceof ProductVariantEvent)
            // highlight-end
            .subscribe(event => {
                // the event will be a ProductEvent or ProductVariantEvent
            });
    }
}
```

## Publishing events

You can publish events using the `EventBus.publish()` method. This is useful if you want to trigger an event from within a plugin or service.

For example, to publish a `ProductEvent`:

```ts title="src/plugins/my-plugin/services/my-plugin.service.ts"
import { Injectable } from '@nestjs/common';
import { EventBus, ProductEvent, RequestContext, Product } from '@vendure/core';

@Injectable()
export class MyPluginService {
    constructor(private eventBus: EventBus) {}

    async doSomethingWithProduct(ctx: RequestContext, product: Product) {
        // ... do something
        // highlight-next-line
        this.eventBus.publish(new ProductEvent(ctx, product, 'updated'));
    }
}
```

## Creating custom events

You can create your own custom events by extending the [`VendureEvent`](/reference/typescript-api/events/vendure-event) class. For example, to create a custom event which is triggered when a customer submits a review, you could do this:

```ts title="src/plugins/reviews/events/review-submitted.event.ts"
import { ID, RequestContext, VendureEvent } from '@vendure/core';
import { ProductReviewInput } from '../types';

/**
 * @description
 * This event is fired whenever a ProductReview is submitted.
 */
export class ReviewSubmittedEvent extends VendureEvent {
    constructor(
        public ctx: RequestContext,
        public input: ProductReviewInput,
    ) {
        super();
    }
}
```

The event would then be published from your plugin's `ProductReviewService`:

```ts title="src/plugins/reviews/services/product-review.service.ts"
import { Injectable } from '@nestjs/common';
import { EventBus, ProductReviewService, RequestContext } from '@vendure/core';

import { ReviewSubmittedEvent } from '../events/review-submitted.event';
import { ProductReviewInput } from '../types';

@Injectable()
export class ProductReviewService {
    constructor(private eventBus: EventBus, private productReviewService: ProductReviewService) {}

    async submitReview(ctx: RequestContext, input: ProductReviewInput) {
        // highlight-next-line
        this.eventBus.publish(new ReviewSubmittedEvent(ctx, input));
        // handle creation of the new review
        // ...
    }
}
```

### Entity events

There is a special event class [`VendureEntityEvent`](/reference/typescript-api/events/vendure-entity-event) for events relating to the creation, update or deletion of entities. Let's say you have a custom entity (see [defining a database entity](/guides/developer-guide/database-entity//)) `BlogPost` and you want to trigger an event whenever a new `BlogPost` is created, updated or deleted:

```ts title="src/plugins/blog/events/blog-post-event.ts"
import { ID, RequestContext, VendureEntityEvent } from '@vendure/core';
import { BlogPost } from '../entities/blog-post.entity';
import { CreateBlogPostInput, UpdateBlogPostInput } from '../types';

type BlogPostInputTypes = CreateBlogPostInput | UpdateBlogPostInput | ID | ID[];

/**
 * This event is fired whenever a BlogPost is added, updated
 * or deleted.
 */
export class BlogPostEvent extends VendureEntityEvent<BlogPost[], BlogPostInputTypes> {
    constructor(
        ctx: RequestContext,
        entity: BlogPost,
        type: 'created' | 'updated' | 'deleted',
        input?: BlogPostInputTypes,
    ) {
        super(entity, type, ctx, input);
    }
}
```

Using this event, you can subscribe to all `BlogPost` events, and for instance filter for only the `created` events:

```ts title="src/plugins/blog/blog-plugin.ts"
import { Injectable, OnModuleInit } from '@nestjs/common';
import { EventBus, PluginCommonModule, VendurePlugin } from '@vendure/core';
import { filter } from 'rxjs/operators';

import { BlogPostEvent } from './events/blog-post-event';

@VendurePlugin({
    imports: [PluginCommonModule],
    // ...
})
export class BlogPlugin implements OnModuleInit {
    constructor(private eventBus: EventBus) {}

    onModuleInit() {
        this.eventBus
            // highlight-start
            .ofType(BlogPostEvent).pipe(
                filter(event => event.type === 'created'),
            )
            .subscribe(event => {
                const blogPost = event.entity;
                // do something with the newly created BlogPost
            });
            // highlight-end
    }
}
```


---
title: 'Plugins'
sidebar_position: 6
---

import Tabs from '@theme/Tabs';
import TabItem from '@theme/TabItem';

The heart of Vendure is its plugin system. Plugins not only allow you to instantly add new functionality to your
Vendure server via third-part npm packages, they are also the means by which you build out the custom business
logic of your application.

Plugins in Vendure allow one to:

- Modify the VendureConfig object, such as defining custom fields on existing entities.
- Extend the GraphQL APIs, including modifying existing types and adding completely new queries and mutations.
- Define new database entities and interact directly with the database.
- Interact with external systems that you need to integrate with.
- Respond to events such as new orders being placed.
- Trigger background tasks to run on the worker process.

… and more!

In a typical Vendure application, custom logic and functionality is implemented as a set of plugins
which are usually independent of one another. For example, there could be a plugin for each of the following:
wishlists, product reviews, loyalty points, gift cards, etc.
This allows for a clean separation of concerns and makes it easy to add or remove functionality as needed.

## Core Plugins

Vendure provides a set of core plugins covering common functionality such as assets handling,
email sending, and search. For documentation on these, see the [Core Plugins reference](/reference/core-plugins/).

## Plugin basics

Here's a bare-minimum example of a plugin:

```ts title="src/plugins/avatar-plugin/avatar.plugin.ts"
import { LanguageCode, PluginCommonModule, VendurePlugin } from '@vendure/core';

@VendurePlugin({
    imports: [PluginCommonModule],
    configuration: config => {
        config.customFields.Customer.push({
            type: 'string',
            name: 'avatarUrl',
            label: [{ languageCode: LanguageCode.en, value: 'Avatar URL' }],
            list: true,
        });
        return config;
    },
})
export class AvatarPlugin {}
```

This plugin does one thing only: it adds a new custom field to the `Customer` entity.

The plugin is then imported into the `VendureConfig`:

```ts title="src/vendure-config.ts"
import { VendureConfig } from '@vendure/core';
import { AvatarPlugin } from './plugins/avatar-plugin/avatar.plugin';

export const config: VendureConfig = {
    // ...
    // highlight-next-line
    plugins: [AvatarPlugin],
};
```

The key feature is the `@VendurePlugin()` decorator, which marks the class as a Vendure plugin and accepts a configuration
object on the type [`VendurePluginMetadata`](/reference/typescript-api/plugin/vendure-plugin-metadata/).

A VendurePlugin is actually an enhanced version of a [NestJS Module](https://docs.nestjs.com/modules), and supports
all the metadata properties that NestJS modules support:

- `imports`: Allows importing other NestJS modules in order to make use of their exported providers.
- `providers`: The providers (services) that will be instantiated by the Nest injector and that may
    be shared across this plugin.
- `controllers`: Controllers allow the plugin to define REST-style endpoints.
- `exports`: The providers which will be exported from this plugin and made available to other plugins.

Additionally, the `VendurePlugin` decorator adds the following Vendure-specific properties:

- `configuration`: A function which can modify the `VendureConfig` object before the server bootstraps.
- `shopApiExtensions`: Allows the plugin to extend the GraphQL Shop API with new queries, mutations, resolvers & scalars.
- `adminApiExtensions`: Allows the plugin to extend the GraphQL Admin API with new queries, mutations, resolvers & scalars.
- `entities`: Allows the plugin to define new database entities.
- `compatibility`: Allows the plugin to declare which versions of Vendure it is compatible with.

:::info
Since a Vendure plugin is a superset of a NestJS module, this means that many NestJS modules are actually
valid Vendure plugins!
:::

## Plugin lifecycle

Since a VendurePlugin is built on top of the NestJS module system, any plugin (as well as any providers it defines)
can make use of any of the [NestJS lifecycle hooks](https://docs.nestjs.com/fundamentals/lifecycle-events):

- onModuleInit
- onApplicationBootstrap
- onModuleDestroy
- beforeApplicationShutdown
- onApplicationShutdown

:::caution
Note that lifecycle hooks are run in both the server and worker contexts.
If you have code that should only run either in the server context or worker context,
you can inject the [ProcessContext provider](/reference/typescript-api/common/process-context/).
:::

### Configure

Another hook that is not strictly a lifecycle hook, but which can be useful to know is the [`configure` method](https://docs.nestjs.com/middleware#applying-middleware) which is
used by NestJS to apply middleware. This method is called _only_ for the server and _not_ for the worker, since middleware relates
to the network stack, and the worker has no network part.

```ts
import { MiddlewareConsumer, NestModule } from '@nestjs/common';
import { EventBus, PluginCommonModule, VendurePlugin } from '@vendure/core';
import { MyMiddleware } from './api/my-middleware';

@VendurePlugin({
    imports: [PluginCommonModule]
})
export class MyPlugin implements NestModule {

  configure(consumer: MiddlewareConsumer) {
    consumer
      .apply(MyMiddleware)
      .forRoutes('my-custom-route');
  }
}
```

## Writing your first plugin

In Vendure **plugins** are used to extend the core functionality of the server. Plugins can be pre-made functionality that you can install via npm, or they can be custom plugins that you write yourself.

For any unit of functionality that you need to add to your project, you'll be creating a Vendure plugin. By convention, plugins are stored in the `plugins` directory of your project. However, this is not a requirement, and you are free to arrange your plugin files in any way you like.

```txt
├──src
    ├── index.ts
    ├── vendure-config.ts
    ├── plugins
        ├── reviews-plugin
        ├── cms-plugin
        ├── wishlist-plugin
        ├── stock-sync-plugin
```

:::info
For a complete working example of a Vendure plugin, see the [real-world-vendure Reviews plugin](https://github.com/vendure-ecommerce/real-world-vendure/tree/master/src/plugins/reviews)

You can also use the [Vendure CLI](/guides/developer-guide/cli) to quickly scaffold a new plugin.

If you intend to write a shared plugin to be distributed as an npm package, see the [vendure plugin-template repo](https://github.com/vendure-ecommerce/plugin-template)
:::

In this guide, we will implement a simple but fully-functional **wishlist plugin** step-by-step. The goal of this plugin is to allow signed-in customers to add products to a wishlist, and to view and manage their wishlist.

### Step 1: Create the plugin file

We'll start by creating a new directory to house our plugin, add create the main plugin file:

```txt
├──src
    ├── index.ts
    ├── vendure-config.ts
    ├── plugins
        // highlight-next-line
        ├── wishlist-plugin
            // highlight-next-line
            ├── wishlist.plugin.ts
```

```ts title="src/plugins/reviews-plugin/reviews.plugin.ts"
import { PluginCommonModule, VendurePlugin } from '@vendure/core';

@VendurePlugin({
    imports: [PluginCommonModule],
})
export class WishlistPlugin {}
```

The `PluginCommonModule` will be required in all plugins that you create. It contains the common services that are exposed by Vendure Core, allowing you to inject them into your plugin's services and resolvers.

### Step 2: Define an entity

Next we will define a new database entity to store the wishlist items. Vendure uses [TypeORM](https://typeorm.io/) to manage the database schema, and an Entity corresponds to a database table.

First let's create the file to house the entity:

```txt
├── wishlist-plugin
    ├── wishlist.plugin.ts
    ├── entities
        // highlight-next-line
        ├── wishlist-item.entity.ts
```

By convention, we'll store the entity definitions in the `entities` directory of the plugin. Again, this is not a requirement, but it is a good way to keep your plugin organized.

```ts title="src/plugins/wishlist-plugin/entities/wishlist-item.entity.ts"
import { DeepPartial, ID, ProductVariant, VendureEntity, EntityId } from '@vendure/core';
import { Column, Entity, ManyToOne } from 'typeorm';

@Entity()
export class WishlistItem extends VendureEntity {
    constructor(input?: DeepPartial

</Tabs>




</Tabs>

We can then query the wishlist items:




</Tabs>

And finally, we can test removing an item from the wishlist:



</Tabs>



---
title: 'Strategies & Configurable Operations'
sidebar_position: 4
---

Vendure is built to be highly configurable and extensible. Two methods of providing this extensibility are **strategies** and **configurable operations**.

## Strategies

A strategy is named after the [Strategy Pattern](https://en.wikipedia.org/wiki/Strategy_pattern), and is a way of providing
a pluggable implementation of a particular feature. Vendure makes heavy use of this pattern to delegate the implementation
of key points of extensibility to the developer.

Examples of strategies include:

- [`OrderCodeStrategy`](/reference/typescript-api/orders/order-code-strategy/) - determines how order codes are generated
- [`StockLocationStrategy`](/reference/typescript-api/products-stock/stock-location-strategy/) - determines which stock locations are used to fulfill an order
- [`ActiveOrderStrategy`](/reference/typescript-api/orders/active-order-strategy/) - determines how the active order in the Shop API is selected
- [`AssetStorageStrategy`](/reference/typescript-api/assets/asset-storage-strategy/) - determines where uploaded assets are stored
- [`GuestCheckoutStrategy`](/reference/typescript-api/orders/guest-checkout-strategy/) - defines rules relating to guest checkouts
- [`OrderItemPriceCalculationStrategy`](/reference/typescript-api/orders/order-item-price-calculation-strategy/) - determines how items are priced when added to the order
- [`TaxLineCalculationStrategy`](/reference/typescript-api/tax/tax-line-calculation-strategy/) - determines how tax is calculated for an order line

As an example, let's take the [`OrderCodeStrategy`](/reference/typescript-api/orders/order-code-strategy/). This strategy
determines how codes are generated when new orders are created. By default, Vendure will use the built-in `DefaultOrderCodeStrategy`
which generates a random 16-character string.

What if you need to change this behavior? For instance, you might have an existing back-office system that is responsible
for generating order codes, which you need to integrate with. Here's how you would do this:

```ts title="src/config/my-order-code-strategy.ts"
import { OrderCodeStrategy, RequestContext } from '@vendure/core';
import { OrderCodeService } from '../services/order-code.service';

export class MyOrderCodeStrategy implements OrderCodeStrategy {

    private orderCodeService: OrderCodeService;

    init(injector) {
        this.orderCodeService = injector.get(OrderCodeService);
    }

    async generate(ctx: RequestContext): string {
        return this.orderCodeService.getNewOrderCode();
    }
}
```

:::info

All strategies can be make use of existing services by using the `init()` method. This is because all strategies
extend the underlying [`InjectableStrategy` interface](/reference/typescript-api/common/injectable-strategy). In
this example we are assuming that we already created an `OrderCodeService` which contains all the specific logic for
connecting to our backend service which generates the order codes.

:::

We then need to pass this custom strategy to our config:

```ts title="src/vendure-config.ts"
import { VendureConfig } from '@vendure/core';
import { MyOrderCodeStrategy } from '../config/my-order-code-strategy';

export const config: VendureConfig = {
    // ...
    orderOptions: {
        // highlight-next-line
        orderCodeStrategy: new MyOrderCodeStrategy(),
    },
}
```

### Strategy lifecycle

Strategies can use two optional lifecycle methods:

- `init(injector: Injector)` - called during the bootstrap phase when the server or worker is starting up.
This is where you can inject any services which you need to use in the strategy. You can also perform any other setup logic
needed, such as instantiating a connection to an external service.
- `destroy()` - called during the shutdown of the server or worker. This is where you can perform any cleanup logic, such as
closing connections to external services.

### Passing options to a strategy

Sometimes you might want to pass some configuration options to a strategy.
For example, imagine you want to create a custom [`StockLocationStrategy`](/reference/typescript-api/products-stock/stock-location-strategy/) which
selects a location within a given proximity to the customer's address. You might want to pass the maximum distance to the strategy
in your config:

```ts title="src/vendure-config.ts"
import { VendureConfig } from '@vendure/core';
import { MyStockLocationStrategy } from '../config/my-stock-location-strategy';

export const config: VendureConfig = {
    // ...
    catalogOptions: {
        // highlight-next-line
        stockLocationStrategy: new MyStockLocationStrategy({ maxDistance: 100 }),
    },
}
```

This config will be passed to the strategy's constructor:

```ts title="src/config/my-stock-location-strategy.ts"
import  { ID, ProductVariant, RequestContext, StockLevel, StockLocationStrategy } from '@vendure/core';

export class MyStockLocationStrategy implements StockLocationStrategy {

    constructor(private options: { maxDistance: number }) {}

    getAvailableStock(
        ctx: RequestContext,
        productVariantId: ID,
        stockLevels: StockLevel[]
    ): ProductVariant[] {
        const maxDistance = this.options.maxDistance;
        // ... implementation omitted
    }
}
```

## Configurable Operations

Configurable operations are similar to strategies in that they allow certain aspects of the system to be customized. However,
the main difference is that they can also be _configured_ via the Admin UI. This allows the store owner to make changes to the
behavior of the system without having to restart the server.

So they are typically used to supply some custom logic that needs to accept configurable arguments which can change
at runtime.

Vendure uses the following configurable operations:

- [`CollectionFilter`](/reference/typescript-api/configuration/collection-filter/) - determines which products are included in a collection
- [`PaymentMethodHandler`](/reference/typescript-api/payment/payment-method-handler/) - determines how payments are processed
- [`PromotionCondition`](/reference/typescript-api/promotions/promotion-condition/) - determines whether a promotion is applicable
- [`PromotionAction`](/reference/typescript-api/promotions/promotion-action/) - determines what happens when a promotion is applied
- [`ShippingEligibilityChecker`](/reference/typescript-api/shipping/shipping-eligibility-checker/) - determines whether a shipping method is available
- [`ShippingCalculator`](/reference/typescript-api/shipping/shipping-calculator/) - determines how shipping costs are calculated

Whereas strategies are typically used to provide a single implementation of a particular feature, configurable operations
are used to provide a set of implementations which can be selected from at runtime.

For example, Vendure ships with a set of default CollectionFilters:

```ts title="default-collection-filters.ts"
export const defaultCollectionFilters = [
    facetValueCollectionFilter,
    variantNameCollectionFilter,
    variantIdCollectionFilter,
    productIdCollectionFilter,
];
```

When setting up a Collection, you can choose from these available default filters:

![CollectionFilters](./collection-filters.webp)

When one is selected, the UI will allow you to configure the arguments for that filter:

![CollectionFilters args](./collection-filters-args.webp)

Let's take a look at a simplified implementation of the `variantNameCollectionFilter`:

```ts title="variant-name-collection-filter.ts"
import { CollectionFilter, LanguageCode } from '@vendure/core';

export const variantNameCollectionFilter = new CollectionFilter({
    args: {
        operator: {
            type: 'string',
            ui: {
                component: 'select-form-input',
                options: [
                    { value: 'startsWith' },
                    { value: 'endsWith' },
                    { value: 'contains' },
                    { value: 'doesNotContain' },
                ],
            },
        },
        term: { type: 'string' },
    },
    code: 'variant-name-filter',
    description: [{ languageCode: LanguageCode.en, value: 'Filter by product variant name' }],
    apply: (qb, args) => {
        // ... implementation omitted
    },
});
```

Here are the important parts:

- Configurable operations are **instances** of a pre-defined class, and are instantiated before being passed to your config.
- They must have a `code` property which is a unique string identifier.
- They must have a `description` property which is a localizable, human-readable description of the operation.
- They must have an `args` property which defines the arguments which can be configured via the Admin UI. If the operation has no arguments,
then this would be an empty object.
- They will have one or more methods that need to be implemented, depending on the type of operation. In this case, the `apply()` method
is used to apply the filter to the query builder.

### Configurable operation args

The `args` property is an object which defines the arguments which can be configured via the Admin UI. Each property of the `args`
object is a key-value pair, where the key is the name of the argument, and the value is an object which defines the type of the argument
and any additional configuration.

As an example let's look at the `dummyPaymentMethodHandler`, a test payment method which we ship with Vendure core:

```ts title="dummy-payment-method.ts"
import { PaymentMethodHandler, LanguageCode } from '@vendure/core';

export const dummyPaymentHandler = new PaymentMethodHandler({
    code: 'dummy-payment-handler',
    description: [/* omitted for brevity */],
    args: {
        automaticSettle: {
            type: 'boolean',
            label: [
                {
                    languageCode: LanguageCode.en,
                    value: 'Authorize and settle in 1 step',
                },
            ],
            description: [
                {
                    languageCode: LanguageCode.en,
                    value: 'If enabled, Payments will be created in the "Settled" state.',
                },
            ],
            required: true,
            defaultValue: false,
        },
    },
    createPayment: async (ctx, order, amount, args, metadata, method) => {
        // Inside this method, the `args` argument is type-safe and will be
        // an object with the following shape:
        // {
        //   automaticSettle: boolean
        // }

        // ... implementation omitted
    },
})
```

The following properties are used to configure the argument:

#### type



[`ConfigArgType`](/reference/typescript-api/configurable-operation-def/config-arg-type)

The following types are available: `string`, `int`, `float`, `boolean`, `datetime`, `ID`.

#### label



[`LocalizedStringArray`](/reference/typescript-api/configurable-operation-def/localized-string-array/)

A human-readable label for the argument. This is used in the Admin UI.

#### description



[`LocalizedStringArray`](/reference/typescript-api/configurable-operation-def/localized-string-array/)

A human-readable description for the argument. This is used in the Admin UI as a tooltip.

#### required



`boolean`

Whether the argument is required. If `true`, then the Admin UI will not allow the user to save the configuration
unless a value has been provided for this argument.

#### defaultValue



`any` (depends on the `type`)

The default value for the argument. If not provided, then the argument will be `undefined` by default.

#### list



`boolean`

Whether the argument is a list of values. If `true`, then the Admin UI will allow the user to add multiple values
for this argument. Defaults to `false`.

#### ui



Allows you to specify the UI component that will be used to render the argument in the Admin UI, by specifying
a `component` property, and optional properties to configure that component.

```ts
{
    args: {
        operator: {
            type: 'string',
            ui: {
                component: 'select-form-input',
                options: [
                    { value: 'startsWith' },
                    { value: 'endsWith' },
                    { value: 'contains' },
                    { value: 'doesNotContain' },
                ],
            },
        },
    }
}
```

A full description of the available UI components can be found in the [Custom Fields guide](/guides/developer-guide/custom-fields/#custom-field-ui).

### Injecting dependencies

Configurable operations are instantiated before being passed to your config, so the mechanism for injecting dependencies
is similar to that of strategies: namely you use an optional `init()` method to inject dependencies into the operation instance.

The main difference is that the injected dependency cannot then be stored as a class property, since you are not defining
a class when you define a configurable operation. Instead, you can store the dependency as a closure variable.

Here’s an example of a ShippingCalculator that injects a service which has been defined in a plugin:

```ts title="src/config/custom-shipping-calculator.ts"
import { Injector, ShippingCalculator } from '@vendure/core';
import { ShippingRatesService } from './shipping-rates.service';

// We keep reference to our injected service by keeping it
// in the top-level scope of the file.
let shippingRatesService: ShippingRatesService;

export const customShippingCalculator = new ShippingCalculator({
    code: 'custom-shipping-calculator',
    description: [],
    args: {},

    init(injector: Injector) {
        // The init function is called during bootstrap, and allows
        // us to inject any providers we need.
        shippingRatesService = injector.get(ShippingRatesService);
    },

    calculate: async (order, args) => {
        // We can now use the injected provider in the business logic.
        const { price, priceWithTax } = await shippingRatesService.getRate({
            destination: order.shippingAddress,
            contents: order.lines,
        });

        return {
            price,
            priceWithTax,
        };
    },
});
```






---
title: 'The API Layer'
sidebar_position: 1
---

import Tabs from '@theme/Tabs';
import TabItem from '@theme/TabItem';

Vendure is a headless platform, which means that all functionality is exposed via GraphQL APIs. The API can be thought of
as a number of layers through which a request will pass, each of which is responsible for a different aspect of the
request/response lifecycle.

## The journey of an API call

Let's take a basic API call and trace its journey from the client to the server and back again.




</Tabs>


:::note

If you have your local development server running, you can try this out by opening the GraphQL Playground in your browser:

[http://localhost:3000/shop-api](http://localhost:3000/shop-api)

:::

![./Vendure_docs-api_request.webp](./Vendure_docs-api_request.webp)


## Middleware

"Middleware" is a term for a function which is executed before or after the main logic of a request. In Vendure, middleware
is used to perform tasks such as authentication, logging, and error handling. There are several types of middleware:

### Express middleware

At the lowest level, Vendure makes use of the popular Express server library. [Express middleware](https://expressjs.com/en/guide/using-middleware.html)
can be added to the sever via the [`apiOptions.middleware`](/reference/typescript-api/configuration/api-options#middleware) config property. There are hundreds of tried-and-tested Express
middleware packages available, and they can be used to add functionality such as CORS, compression, rate-limiting, etc.

Here's a simple example demonstrating Express middleware which will log a message whenever a request is received to the
Admin API:

```ts title="src/vendure-config.ts"
import { VendureConfig } from '@vendure/core';
import { RequestHandler } from 'express';

/**
* This is a custom middleware function that logs a message whenever a request is received.
*/
const myMiddleware: RequestHandler = (req, res, next) => {
    console.log('Request received!');
    next();
};

export const config: VendureConfig = {
    // ...
    apiOptions: {
        middleware: [
            {
                // We will execute our custom handler only for requests to the Admin API
                route: 'admin-api',
                handler: myMiddleware,
            }
        ],
    },
};
```

### NestJS middleware

You can also define [NestJS middleware](https://docs.nestjs.com/middleware) which works like Express middleware but also
has access to the NestJS dependency injection system.

```ts title="src/vendure-config.ts"
import { VendureConfig, ConfigService } from '@vendure/core';
import { Injectable, NestMiddleware } from '@nestjs/common';
import { Request, Response, NextFunction } from 'express';


@Injectable()
class MyNestMiddleware implements NestMiddleware {
    // Dependencies can be injected via the constructor
    constructor(private configService: ConfigService) {}

    use(req: Request, res: Response, next: NextFunction) {
        console.log(`NestJS middleware: current port is ${this.configService.apiOptions.port}`);
        next();
    }
}

export const config: VendureConfig = {
    // ...
    apiOptions: {
        middleware: [
            {
                route: 'admin-api',
                handler: MyNestMiddleware,
            }
        ],
    },
};
```

NestJS allows you to define specific types of middleware including [Guards](https://docs.nestjs.com/guards),
[Interceptors](https://docs.nestjs.com/interceptors), [Pipes](https://docs.nestjs.com/pipes) and [Filters](https://docs.nestjs.com/exception-filters).

Vendure uses a number of these mechanisms internally to handle authentication, transaction management, error handling and
data transformation.

### Global NestJS middleware

Guards, interceptors, pipes and filters can be added to your own custom resolvers and controllers
using the NestJS decorators as given in the NestJS docs. However, a common pattern is to register them globally via a
[Vendure plugin](/guides/developer-guide/plugins/):

```ts title="src/plugins/my-plugin/my-plugin.ts"
import { VendurePlugin } from '@vendure/core';
import { APP_GUARD, APP_FILTER, APP_INTERCEPTOR  } from '@nestjs/core';

// Some custom NestJS middleware classes which we want to apply globally
import { MyCustomGuard, MyCustomInterceptor, MyCustomExceptionFilter } from './my-custom-middleware';

@VendurePlugin({
    // ...
    providers: [
        // This is the syntax needed to apply your guards,
        // interceptors and filters globally
        {
            provide: APP_GUARD,
            useClass: MyCustomGuard,
        },
        {
            provide: APP_INTERCEPTOR,
            useClass: MyCustomInterceptor,
        },
        {
            provide: APP_FILTER,
            useClass: MyCustomExceptionFilter,
        },
    ],
})
export class MyPlugin {}
```

Adding this plugin to your Vendure config `plugins` array will now apply these middleware classes to all requests.

```ts title="src/vendure-config.ts"
import { VendureConfig } from '@vendure/core';
import { MyPlugin } from './plugins/my-plugin/my-plugin';

export const config: VendureConfig = {
    // ...
    plugins: [
        MyPlugin,
    ],
};
```



### Apollo Server plugins
Apollo Server (the underlying GraphQL server library used by Vendure) allows you to define
[plugins](https://www.apollographql.com/docs/apollo-server/integrations/plugins/) which can be used to hook into various
stages of the GraphQL request lifecycle and perform tasks such as data transformation. These are defined via the
[`apiOptions.apolloServerPlugins`](/reference/typescript-api/configuration/api-options#apolloserverplugins) config property.



## Resolvers

A "resolver" is a GraphQL concept, and refers to a function which is responsible for returning the data for a particular
field. In Vendure, a resolver can also refer to a class which contains multiple resolver functions. For every query or
mutation, there is a corresponding resolver function which is responsible for returning the requested data (and performing
side-effect such as updating data in the case of mutations).

Here's a simplified example of a resolver function for the `product` query:

```ts
import { Query, Resolver, Args } from '@nestjs/graphql';
import { Ctx, RequestContext, ProductService } from '@vendure/core';

@Resolver()
export class ShopProductsResolver {

     constructor(private productService: ProductService) {}

     @Query()
     product(@Ctx() ctx: RequestContext, @Args() args: { id: string }) {
         return this.productService.findOne(ctx, args.id);
     }
}
```

- The `@Resolver()` decorator marks this class as a resolver.
- The `@Query()` decorator marks the `product()` method as a resolver function.
- The `@Ctx()` decorator injects the [`RequestContext` object](/reference/typescript-api/request/request-context/), which contains information about the
current request, such as the current user, the active channel, the active language, etc. The `RequestContext` is a key part
of the Vendure architecture, and is used throughout the application to provide context to the various services and plugins. In general, your
resolver functions should always accept a `RequestContext` as the first argument, and pass it through to the services.
- The `@Args()` decorator injects the arguments passed to the query, in this case the `id` that we provided in our query.

As you can see, the resolver function is very simple, and simply delegates the work to the `ProductService` which is
responsible for fetching the data from the database.

:::tip
In general, resolver functions should be kept as simple as possible,
and the bulk of the business logic should be delegated to the service layer.
:::

## API Decorators

Following the pattern of NestJS, Vendure makes use of decorators to control various aspects of the API. Here are the
important decorators to be aware of:

### `@Resolver()`

This is exported by the `@nestjs/graphql` package. It marks a class as a resolver, meaning that its methods can be used
to resolve the fields of a GraphQL query or mutation.

```ts title="src/plugins/wishlist/api/wishlist.resolver.ts"
import { Resolver } from '@nestjs/graphql';

// highlight-next-line
@Resolver()
export class WishlistResolver {
    // ...
}
```

### `@Query()`

This is exported by the `@nestjs/graphql` package. It marks a method as a resolver function for a query. The method name
should match the name of the query in the GraphQL schema, or if the method name is different, a name can be provided as
an argument to the decorator.

```ts title="src/plugins/wishlist/api/wishlist.resolver.ts"
import { Query, Resolver } from '@nestjs/graphql';

@Resolver()
export class WishlistResolver {

    // highlight-next-line
    @Query()
    wishlist() {
        // ...
    }
}
```

### `@Mutation()`

This is exported by the `@nestjs/graphql` package. It marks a method as a resolver function for a mutation. The method name
should match the name of the mutation in the GraphQL schema, or if the method name is different, a name can be provided as
an argument to the decorator.

```ts title="src/plugins/wishlist/api/wishlist.resolver.ts"
import { Mutation, Resolver } from '@nestjs/graphql';

@Resolver()
export class WishlistResolver {

    // highlight-next-line
    @Mutation()
    addItemToWishlist() {
        // ...
    }
}
```

### `@Allow()`

The [`Allow` decorator](/reference/typescript-api/request/allow-decorator) is exported by the `@vendure/core` package. It is used to control access to queries and mutations. It takes a list
of [Permissions](/reference/typescript-api/common/permission/) and if the current user does not have at least one of the
permissions, then the query or mutation will return an error.

```ts title="src/plugins/wishlist/api/wishlist.resolver.ts"
import { Mutation, Resolver } from '@nestjs/graphql';
import { Allow, Permission } from '@vendure/core';

@Resolver()
export class WishlistResolver {

    @Mutation()
    // highlight-next-line
    @Allow(Permission.UpdateCustomer)
    updateCustomerWishlist() {
        // ...
    }
}
```

### `@Transaction()`

The [`Transaction` decorator](/reference/typescript-api/request/transaction-decorator/) is exported by the `@vendure/core` package. It is used to wrap a resolver function in a database transaction. It is
normally used with mutations, since queries typically do not modify data.

```ts title="src/plugins/wishlist/api/wishlist.resolver.ts"
import { Mutation, Resolver } from '@nestjs/graphql';
import { Transaction } from '@vendure/core';

@Resolver()
export class WishlistResolver {

    // highlight-next-line
    @Transaction()
    @Mutation()
    addItemToWishlist() {
        // if an error is thrown here, the
        // entire transaction will be rolled back
    }
}
```

:::note
The `@Transaction()` decorator _only_ works when used with a `RequestContext` object (see the `@Ctx()` decorator below).

This is because the `Transaction` decorator stores the transaction context on the `RequestContext` object, and by passing
this object to the service layer, the services and thus database calls can access the transaction context.
:::

### `@Ctx()`

The [`Ctx` decorator](/reference/typescript-api/request/ctx-decorator/) is exported by the `@vendure/core` package. It is used to inject the
[`RequestContext` object](/reference/typescript-api/request/request-context/) into a resolver function. The `RequestContext` contains information about the
current request, such as the current user, the active channel, the active language, etc. The `RequestContext` is a key part
of the Vendure architecture, and is used throughout the application to provide context to the various services and plugins.

```ts title="src/plugins/wishlist/api/wishlist.resolver.ts"
import { Mutation, Resolver } from '@nestjs/graphql';
import { Ctx, RequestContext } from '@vendure/core';

@Resolver()
export class WishlistResolver {

    @Mutation()
    // highlight-next-line
    addItemToWishlist(@Ctx() ctx: RequestContext) {
        // ...
    }
}
```

:::tip
As a general rule, _always_ use the `@Ctx()` decorator to inject the `RequestContext` into your resolver functions.
:::

### `@Args()`

This is exported by the `@nestjs/graphql` package. It is used to inject the arguments passed to a query or mutation.

Given the a schema definition like this:

```graphql
extend type Mutation {
    addItemToWishlist(variantId: ID!): Wishlist
}
```

The resolver function would look like this:

```ts title="src/plugins/wishlist/api/wishlist.resolver.ts"
import { Mutation, Resolver, Args } from '@nestjs/graphql';
import { Ctx, RequestContext, ID } from '@vendure/core';

@Resolver()
export class WishlistResolver {

    @Mutation()
    addItemToWishlist(
        @Ctx() ctx: RequestContext,
        // highlight-next-line
        @Args() args: { variantId: ID }
    ) {
        // ...
    }
}
```

As you can see, the `@Args()` decorator injects the arguments passed to the query, in this case the `variantId` that we provided in our query.

## Field resolvers

So far, we've seen examples of resolvers for queries and mutations. However, there is another type of resolver which is
used to resolve the fields of a type. For example, given the following schema definition:

```graphql
type WishlistItem {
    id: ID!
    // highlight-next-line
    product: Product!
}
```

The `product` field is a relation to the `Product` type. The `product` field resolver
would look like this:

```ts title="src/plugins/wishlist/api/wishlist-item.resolver.ts"
import { Parent, ResolveField, Resolver } from '@nestjs/graphql';
import { Ctx, RequestContext } from '@vendure/core';

import { WishlistItem } from '../entities/wishlist-item.entity';

// highlight-next-line
@Resolver('WishlistItem')
export class WishlistItemResolver {

    // highlight-next-line
    @ResolveField()
    product(
        @Ctx() ctx: RequestContext,
        // highlight-next-line
        @Parent() wishlistItem: WishlistItem
    ) {
        // ...
    }
}
```

Note that in this example, the `@Resolver()` decorator has an argument of `'WishlistItem'`. This tells NestJS that
this resolver is for the `WishlistItem` type, and that when we use the `@ResolveField()` decorator, we are defining
a resolver for a field of that type.

In this example we're defining a resolver for the `product` field of the `WishlistItem` type. The
`@ResolveField()` decorator is used to mark a method as a field resolver. The method name should match the name of the
field in the GraphQL schema, or if the method name is different, a name can be provided as an argument to the decorator.

## REST endpoints

Although Vendure is primarily a GraphQL-based API, it is possible to add REST endpoints to the API. This is useful if
you need to integrate with a third-party service or client application which only supports REST, for example.

Creating a REST endpoint is covered in detail in the [Add a REST endpoint
guide](/guides/developer-guide/rest-endpoint/).


---
title: 'The Service Layer'
sidebar_position: 2
---

The service layer is the core of the application. This is where the business logic is implemented, and where
the application interacts with the database. When a request comes in to the API, it gets routed to a resolver
which then calls a service method to perform the required operation.

![../the-api-layer/Vendure_docs-api_request.webp](../the-api-layer/Vendure_docs-api_request.webp)

:::info
Services are classes which, in NestJS terms, are [providers](https://docs.nestjs.com/providers#services). They
follow all the rules of NestJS providers, including dependency injection, scoping, etc.
:::

Services are generally scoped to a specific domain or entity. For instance, in the Vendure core, there is a [`Product` entity](/reference/typescript-api/entities/product),
and a corresponding [`ProductService`](/reference/typescript-api/services/product-service) which contains all the methods for interacting with products.

Here's a simplified example of a `ProductService`, including an implementation of the `findOne()` method that was
used in the example in the [previous section](/guides/developer-guide/the-api-layer/#resolvers):

```ts title="src/services/product.service.ts"
import { Injectable } from '@nestjs/common';
import { IsNull } from 'typeorm';
import { ID, Product, RequestContext, TransactionalConnection, TranslatorService } from '@vendure/core';

@Injectable()
export class ProductService {

    constructor(private connection: TransactionalConnection,
                private translator: TranslatorService){}

    /**
     * @description
     * Returns a Product with the given id, or undefined if not found.
     */
    async findOne(ctx: RequestContext, productId: ID): Promise<Product | undefined> {
        const product = await this.connection.findOneInChannel(ctx, Product, productId, ctx.channelId, {
            where: {
                deletedAt: IsNull(),
            },
        });
        if (!product) {
            return;
        }
        return this.translator.translate(product, ctx);
    }

    // ... other methods
    findMany() {}
    create() {}
    update() {}
}
```

- The `@Injectable()` decorator is a [NestJS](https://docs.nestjs.com/providers#services) decorator which allows the service
    to be injected into other services or resolvers.
- The `constructor()` method is where the dependencies of the service are injected. In this case, the `TransactionalConnection`
    is used to access the database, and the `TranslatorService` is used to translate the Product entity into the current
    language.

## Using core services

All the internal Vendure services can be used in your own plugins and scripts. They are listed in the [Services API reference](/reference/typescript-api/services/) and
can be imported from the `@vendure/core` package.

To make use of a core service in your own plugin, you need to ensure your plugin is importing the `PluginCommonModule` and
then inject the desired service into your own service's constructor:

```ts title="src/my-plugin/my.plugin.ts"
import { PluginCommonModule, VendurePlugin } from '@vendure/core';
import { MyService } from './services/my.service';

@VendurePlugin({
    // highlight-start
    imports: [PluginCommonModule],
    providers: [MyService],
    // highlight-end
})
export class MyPlugin {}
```

```ts title="src/my-plugin/services/my.service.ts"
import { Injectable } from '@nestjs/common';
import { ProductService } from '@vendure/core';

@Injectable()
export class MyService {

    // highlight-next-line
    constructor(private productService: ProductService) {}

    // you can now use the productService methods
}
```

## Accessing the database

One of the main responsibilities of the service layer is to interact with the database. For this, you will be using
the [`TransactionalConnection` class](/reference/typescript-api/data-access/transactional-connection/), which is a wrapper
around the [TypeORM `DataSource` object](https://typeorm.io/data-source-api). The primary purpose of the `TransactionalConnection`
is to ensure that database operations can be performed within a transaction (which is essential for ensuring data integrity), even
across multiple services. Furthermore, it exposes some helper methods which make it easier to perform common operations.

:::info

Always pass the `RequestContext` (`ctx`) to the `TransactionalConnection` methods. This ensures the operation occurs within
any active transaction.

:::

There are two primary APIs for accessing data provided by TypeORM: the **Find API** and the **QueryBuilder API**.

### The Find API

This API is the most convenient and type-safe way to query the database. It provides a powerful type-safe way to query
including support for eager relations, pagination, sorting, filtering and more.

Here are some examples of using the Find API:

```ts title="src/services/item.service.ts"
import { Injectable } from '@nestjs/common';
import { ID, RequestContext, TransactionalConnection } from '@vendure/core';
import { IsNull } from 'typeorm';
import { Item } from '../entities/item.entity';

@Injectable()
export class ItemService {

    constructor(private connection: TransactionalConnection) {}

    findById(ctx: RequestContext, itemId: ID): Promise<Item | null> {
        return this.connection.getRepository(ctx, Item).findOne({
            where: { id: itemId },
        });
    }

    findByName(ctx: RequestContext, name: string): Promise<Item | null> {
        return this.connection.getRepository(ctx, Item).findOne({
            where: {
                // Multiple where clauses can be specified,
                // which are joined with AND
                name,
                deletedAt: IsNull(),
            },
        });
    }

    findWithRelations() {
        return this.connection.getRepository(ctx, Item).findOne({
            where: { name },
            relations: {
                // Join the `item.customer` relation
                customer: true,
                product: {
                    // Here we are joining a nested relation `item.product.featuredAsset`
                    featuredAsset: true,
                },
            },
        });
    }

    findMany(ctx: RequestContext): Promise<Item[]> {
        return this.connection.getRepository(ctx, Item).find({
            // Pagination
            skip: 0,
            take: 10,
            // Sorting
            order: {
                name: 'ASC',
            },
        });
    }
}
```

:::info

Further examples can be found in the [TypeORM Find Options documentation](https://typeorm.io/find-options).

:::

### The QueryBuilder API

When the Find API is not sufficient, the QueryBuilder API can be used to construct more complex queries. For instance,
if you want to have a more complex `WHERE` clause than what can be achieved with the Find API, or if you want to perform
sub-queries, then the QueryBuilder API is the way to go.

Here are some examples of using the QueryBuilder API:

```ts title="src/services/item.service.ts"
import { Injectable } from '@nestjs/common';
import { ID, RequestContext, TransactionalConnection } from '@vendure/core';
import { Brackets, IsNull } from 'typeorm';
import { Item } from '../entities/item.entity';

@Injectable()
export class ItemService {

    constructor(private connection: TransactionalConnection) {}

    findById(ctx: RequestContext, itemId: ID): Promise<Item | null> {
        // This is simple enough that you should prefer the Find API,
        // but here is how it would be done with the QueryBuilder API:
        return this.connection.getRepository(ctx, Item).createQueryBuilder('item')
            .where('item.id = :id', { id: itemId })
            .getOne();
    }

    findManyWithSubquery(ctx: RequestContext, name: string) {
        // Here's a more complex query that would not be possible using the Find API:
        return this.connection.getRepository(ctx, Item).createQueryBuilder('item')
            .where('item.name = :name', { name })
            .andWhere(
                new Brackets(qb1 => {
                    qb1.where('item.state = :state1', { state1: 'PENDING' })
                       .orWhere('item.state = :state2', { state2: 'RETRYING' });
                }),
            )
            .orderBy('item.createdAt', 'ASC')
            .getMany();
    }
}
```

:::info

Further examples can be found in the [TypeORM QueryBuilder documentation](https://typeorm.io/select-query-builder).

:::

### Working with relations

One limitation of TypeORM's typings is that we have no way of knowing at build-time whether a particular relation will be
joined at runtime. For instance, the following code will build without issues, but will result in a runtime error:

```ts
const product = await this.connection.getRepository(ctx, Product).findOne({
    where: { id: productId },
});
if (product) {
    // highlight-start
    console.log(product.featuredAsset.preview);
    // ^ Error: Cannot read property 'preview' of undefined
    // highlight-end
}
```

This is because the `featuredAsset` relation is not joined by default. The simple fix for the above example is to use
the `relations` option:

```ts
const product = await this.connection.getRepository(ctx, Product).findOne({
    where: { id: productId },
    // highlight-next-line
    relations: { featuredAsset: true },
});
```
or in the case of the QueryBuilder API, we can use the `leftJoinAndSelect()` method:

```ts
const product = await this.connection.getRepository(ctx, Product).createQueryBuilder('product')
    // highlight-next-line
    .leftJoinAndSelect('product.featuredAsset', 'featuredAsset')
    .where('product.id = :id', { id: productId })
    .getOne();
```

### Using the EntityHydrator

But what about when we do not control the code which fetches the entity from the database? For instance, we might be implementing
a function which gets an entity passed to it by Vendure. In this case, we can use the [`EntityHydrator`](/reference/typescript-api/data-access/entity-hydrator/)
to ensure that a given relation is "hydrated" (i.e. joined) before we use it:

```ts
import { EntityHydrator, ShippingCalculator } from '@vendure/core';

let entityHydrator: EntityHydrator;

const myShippingCalculator = new ShippingCalculator({
    // ... rest of config omitted for brevity
    init(injector) {
        entityHydrator = injector.get(EntityHydrator);
    },
    calculate: (ctx, order, args) => {
      // highlight-start
      // ensure that the customer and customer.groups relations are joined
      await entityHydrator.hydrate(ctx, order, { relations: ['customer.groups' ]});
      // highlight-end

      if (order.customer?.groups?.some(g => g.name === 'VIP')) {
        // ... do something special for VIP customers
      } else {
        // ... do something else
      }
    },
});
```

### Joining relations in built-in service methods

Many of the core services allow an optional `relations` argument in their `findOne()` and `findMany()` and related methods.
This allows you to specify which relations should be joined when the query is executed. For instance, in the [`ProductService`](/reference/typescript-api/services/product-service)
there is a `findOne()` method which allows you to specify which relations should be joined:

```ts
const productWithAssets = await this.productService
    .findOne(ctx, productId, ['featuredAsset', 'assets']);
```



---
title: 'Worker & Job Queue'
sidebar_position: 5
---

The Vendure Worker is a Node.js process responsible for running computationally intensive
or otherwise long-running tasks in the background. For example, updating a
search index or sending emails. Running such tasks in the background allows
the server to stay responsive, since a response can be returned immediately
without waiting for the slower tasks to complete.

Put another way, the Worker executes **jobs** which have been placed in the **job queue**.

![Worker & Job Queue](./worker-job-queue.webp)

## The worker

The worker is started by calling the [`bootstrapWorker()`](/reference/typescript-api/worker/bootstrap-worker/) function with the same
configuration as is passed to the main server `bootstrap()`. In a standard Vendure installation, this is found
in the `index-worker.ts` file:

```ts title="src/index-worker.ts"
import { bootstrapWorker } from '@vendure/core';
import { config } from './vendure-config';

bootstrapWorker(config)
    .then(worker => worker.startJobQueue())
    .catch(err => {
        console.log(err);
    });
```

### Underlying architecture

The Worker is a NestJS standalone application. This means it is almost identical to the main server app,
but does not have any network layer listening for requests. The server communicates with the worker
via a “job queue” architecture. The exact implementation of the job queue is dependent on the
configured [`JobQueueStrategy`](/reference/typescript-api/job-queue/job-queue-strategy/), but by default
the worker polls the database for new jobs.

### Multiple workers

It is possible to run multiple workers in parallel to better handle heavy loads. Using the
[`JobQueueOptions.activeQueues`](/reference/typescript-api/job-queue/job-queue-options#activequeues) configuration, it is even possible to have particular workers dedicated
to one or more specific types of jobs. For example, if your application does video transcoding,
you might want to set up a dedicated worker just for that task:

```ts title="src/transcoder-worker.ts"
import { bootstrapWorker, mergeConfig } from '@vendure/core';
import { config } from './vendure-config';

const transcoderConfig = mergeConfig(config, {
    jobQueueOptions: {
      activeQueues: ['transcode-video'],
    }
});

bootstrapWorker(transcoderConfig)
  .then(worker => worker.startJobQueue())
  .catch(err => {
    console.log(err);
  });
```

### Running jobs on the main process

It is possible to run jobs from the Job Queue on the main server. This is mainly used for testing
and automated tasks, and is not advised for production use, since it negates the benefits of
running long tasks off of the main process. To do so, you need to manually start the JobQueueService:

```ts title="src/index.ts"
import { bootstrap, JobQueueService } from '@vendure/core';
import { config } from './vendure-config';

bootstrap(config)
    .then(app => app.get(JobQueueService).start())
    .catch(err => {
        console.log(err);
        process.exit(1);
    });
```

### ProcessContext

Sometimes your code may need to be aware of whether it is being run as part of a server or worker process.
In this case you can inject the [`ProcessContext`](/reference/typescript-api/common/process-context/) provider and query it like this:

```ts title="src/plugins/my-plugin/services/my.service.ts"
import { Injectable, OnApplicationBootstrap } from '@nestjs/common';
import { ProcessContext } from '@vendure/core';

@Injectable()
export class MyService implements OnApplicationBootstrap {
    constructor(private processContext: ProcessContext) {}

    onApplicationBootstrap() {
        if (this.processContext.isServer) {
            // code which will only execute when running in
            // the server process
        }
    }
}
```

## The job queue

Vendure uses a [job queue](https://en.wikipedia.org/wiki/Job_queue) to handle the processing of certain tasks which are typically too slow to run in the
normal request-response cycle. A normal request-response looks like this:

![Regular request response](./Vendure_docs-job-queue.webp)

In the normal request-response, all intermediate tasks (looking up data in the database, performing business logic etc.)
occur before the response can be returned. For most operations this is fine, since those intermediate tasks are very fast.

Some operations however will need to perform much longer-running tasks. For example, updating the search index on
thousands of products could take up to a minute or more. In this case, we certainly don’t want to delay the response
until that processing has completed. That’s where a job queue comes in:

![Request response with job queue](./Vendure_docs-job-queue-2.webp)

### What does Vendure use the job queue for?

By default, Vendure uses the job queue for the following tasks:

- Re-building the search index
- Updating the search index when changes are made to Products, ProductVariants, Assets etc.
- Updating the contents of Collections
- Sending transactional emails

### How does the Job Queue work?

This diagram illustrates the job queue mechanism:

![Job queue sequence](./Vendure_docs-job-queue-3.webp)

The server adds jobs to the queue. The worker then picks up these jobs from the queue and processes them in sequence,
one by one (it is possible to increase job queue throughput by running multiple workers or by increasing the concurrency
of a single worker).

### JobQueueStrategy

The actual queue part is defined by the configured [`JobQueueStrategy`](/reference/typescript-api/job-queue/job-queue-strategy/).

If no strategy is defined, Vendure uses an [in-memory store](/reference/typescript-api/job-queue/in-memory-job-queue-strategy/)
of the contents of each queue. While this has the advantage
of requiring no external dependencies, it is not suitable for production because when the server is stopped, the entire
queue will be lost and any pending jobs will never be processed. Moreover, it cannot be used when running the worker
as a separate process.

A better alternative is to use the [DefaultJobQueuePlugin](/reference/typescript-api/job-queue/default-job-queue-plugin/)
(which will be used in a standard `@vendure/create` installation), which configures Vendure to use the [SqlJobQueueStrategy](/reference/typescript-api/job-queue/sql-job-queue-strategy).
This strategy uses the database as a queue, and means that even if the Vendure server stops, pending jobs will be persisted and upon re-start, they will be processed.

It is also possible to implement your own JobQueueStrategy to take advantage of other technologies.
Examples include RabbitMQ, Google Cloud Pub Sub & Amazon SQS. It may make sense to implement a custom strategy based on
one of these if the default database-based approach does not meet your performance requirements.

### Job Queue Performance

It is common for larger Vendure projects to define multiple custom job queues, When using the [DefaultJobQueuePlugin](/reference/typescript-api/job-queue/default-job-queue-plugin/)
with many queues, performance may be impacted. This is because the `SqlJobQueueStrategy` uses polling to check for
new jobs in the database. Each queue will (by default) query the database every 200ms. So if there are 10 queues,
this will result in a constant 50 queries/second.

In this case it is recommended to try the [BullMQJobQueuePlugin](/reference/core-plugins/job-queue-plugin/bull-mqjob-queue-plugin/),
which uses an efficient push-based strategy built on Redis.

## Using Job Queues in a plugin

If your plugin involves long-running tasks, you can also make use of the job queue.

:::info
A real example of this can be seen in the [EmailPlugin source](https://github.com/vendure-ecommerce/vendure/blob/master/packages/email-plugin/src/plugin.ts)
:::

Let's say you are building a plugin which allows a video URL to be specified, and then that video gets transcoded into a format suitable for streaming on the storefront. This is a long-running task which should not block the main thread, so we will use the job queue to run the task on the worker.

First we'll add a new mutation to the Admin API schema:

```ts title="src/plugins/product-video/api/api-extensions.ts"
import gql from 'graphql-tag';

export const adminApiExtensions = gql`
  extend type Mutation {
    addVideoToProduct(productId: ID! videoUrl: String!): Job!
  }
`;
```

The resolver looks like this:


```ts title="src/plugins/product-video/api/product-video.resolver.ts"
import { Args, Mutation, Resolver } from '@nestjs/graphql';
import { Allow, Ctx, RequestContext, Permission, RequestContext } from '@vendure/core'
import { ProductVideoService } from '../services/product-video.service';

@Resolver()
export class ProductVideoResolver {

    constructor(private productVideoService: ProductVideoService) {}

    @Mutation()
    @Allow(Permission.UpdateProduct)
    addVideoToProduct(@Ctx() ctx: RequestContext, @Args() args: { productId: ID; videoUrl: string; }) {
        return this.productVideoService.transcodeForProduct(
            args.productId,
            args.videoUrl,
        );
    }
}
```
The resolver just defines how to handle the new `addVideoToProduct` mutation, delegating the actual work to the `ProductVideoService`.

### Creating a job queue

The [`JobQueueService`](/reference/typescript-api/job-queue/job-queue-service/) creates and manages job queues. The queue is created when the
application starts up (see [NestJS lifecycle events](https://docs.nestjs.com/fundamentals/lifecycle-events)), and then we can use the `add()` method to add jobs to the queue.

```ts title="src/plugins/product-video/services/product-video.service.ts"
import { Injectable, OnModuleInit } from '@nestjs/common';
import { JobQueue, JobQueueService, ID, Product, TransactionalConnection } from '@vendure/core';
import { transcode } from 'third-party-video-sdk';

@Injectable()
class ProductVideoService implements OnModuleInit {

    private jobQueue: JobQueue<{ productId: ID; videoUrl: string; }>;

    constructor(private jobQueueService: JobQueueService,
                private connection: TransactionalConnection) {
    }

    async onModuleInit() {
        this.jobQueue = await this.jobQueueService.createQueue({
            name: 'transcode-video',
            process: async job => {
                // Inside the `process` function we define how each job
                // in the queue will be processed.
                // In this case we call out to some imaginary 3rd-party video
                // transcoding API, which performs the work and then
                // returns a new URL of the transcoded video, which we can then
                // associate with the Product via the customFields.
                const result = await transcode(job.data.videoUrl);
                await this.connection.getRepository(Product).save({
                    id: job.data.productId,
                    customFields: {
                        videoUrl: result.url,
                    },
                });
                // The value returned from the `process` function is stored as the "result"
                // field of the job (for those JobQueueStrategies that support recording of results).
                //
                // Any error thrown from this function will cause the job to fail.
                return result;
            },
        });
    }

    transcodeForProduct(productId: ID, videoUrl: string) {
        // Add a new job to the queue and immediately return the
        // job itself.
        return this.jobQueue.add({productId, videoUrl}, {retries: 2});
    }
}
```

Notice the generic type parameter of the `JobQueue`:

```ts
JobQueue<{ productId: ID; videoUrl: string; }>
```

This means that when we call `jobQueue.add()` we must pass in an object of this type. This data will then be available in the `process` function as the `job.data` property.

:::note
The data passed to `jobQueue.add()` must be JSON-serializable, because it gets serialized into a string when stored in the job queue. Therefore you should
avoid passing in complex objects such as `Date` instances, `Buffer`s, etc.
:::

The `ProductVideoService` is in charge of setting up the JobQueue and adding jobs to that queue. Calling

```ts
productVideoService.transcodeForProduct(id, url);
```

will add a transcoding job to the queue.

:::tip
Plugin code typically gets executed on both the server _and_ the worker. Therefore, you sometimes need to explicitly check
what context you are in. This can be done with the [ProcessContext](/reference/typescript-api/common/process-context/) provider.
:::


Finally, the `ProductVideoPlugin` brings it all together, extending the GraphQL API, defining the required CustomField to store the transcoded video URL, and registering our service and resolver. The [PluginCommonModule](/reference/typescript-api/plugin/plugin-common-module/) is imported as it exports the `JobQueueService`.

```ts title="src/plugins/product-video/product-video.plugin.ts"
import gql from 'graphql-tag';
import { PluginCommonModule, VendurePlugin } from '@vendure/core';
import { ProductVideoService } from './services/product-video.service';
import { ProductVideoResolver } from './api/product-video.resolver';
import { adminApiExtensions } from './api/api-extensions';

@VendurePlugin({
    imports: [PluginCommonModule],
    providers: [ProductVideoService],
    adminApiExtensions: {
        schema: adminApiExtensions,
        resolvers: [ProductVideoResolver]
    },
    configuration: config => {
        config.customFields.Product.push({
            name: 'videoUrl',
            type: 'string',
        });
        return config;
    }
})
export class ProductVideoPlugin {}
```
### Subscribing to job updates

When creating a new job via `JobQueue.add()`, it is possible to subscribe to updates to that Job (progress and status changes). This allows you, for example, to create resolvers which are able to return the results of a given Job.

In the video transcoding example above, we could modify the `transcodeForProduct()` call to look like this:

```ts title="src/plugins/product-video/services/product-video.service.ts"
import { Injectable, OnModuleInit } from '@nestjs/common';
import { of } from 'rxjs';
import { map, catchError } from 'rxjs/operators';
import { ID, Product, TransactionalConnection } from '@vendure/core';

@Injectable()
class ProductVideoService implements OnModuleInit {
    // ... omitted (see above)

    transcodeForProduct(productId: ID, videoUrl: string) {
        const job = await this.jobQueue.add({productId, videoUrl}, {retries: 2});

        return job.updates().pipe(
            map(update => {
                // The returned Observable will emit a value for every update to the job
                // such as when the `progress` or `status` value changes.
                Logger.info(`Job ${update.id}: progress: ${update.progress}`);
                if (update.state === JobState.COMPLETED) {
                    Logger.info(`COMPLETED ${update.id}: ${update.result}`);
                }
                return update.result;
            }),
            catchError(err => of(err.message)),
        );
    }
}
```

If you prefer to work with Promises rather than Rxjs Observables, you can also convert the updates to a promise:

```ts
const job = await this.jobQueue.add({ productId, videoUrl }, { retries: 2 });

return job.updates().toPromise()
  .then(/* ... */)
  .catch(/* ... */);
```


---
title: Introducing GraphQL
---
import Playground from '@site/src/components/Playground';
import Tabs from '@theme/Tabs';
import TabItem from '@theme/TabItem';

:::info
Vendure uses [GraphQL](https://graphql.org/) as its API layer.

This is an introduction to GraphQL for those who are new to it. If you are already familiar with GraphQL, you may choose
to skip this section.
:::

## What is GraphQL?

From [graphql.org](https://graphql.org/):

> GraphQL is a query language for APIs and a runtime for fulfilling those queries with your existing data. GraphQL provides a complete and understandable description of the data in your API, gives clients the power to ask for exactly what they need and nothing more, makes it easier to evolve APIs over time, and enables powerful developer tools.

To put it simply: GraphQL allows you to fetch data from an API via _queries_, and to update data via _mutations_.

Here's a GraphQL query which fetches the product with the slug "football":



</Tabs>

Here's an example mutation operation to update the first customer's email:



</Tabs>

Operations can also have a **name**, which, while not required, is recommended for real applications as it makes debugging easier
(similar to having named vs anonymous functions in JavaScript), and also allows you to take advantage of code generation tools.

Here's the above query with a name:

```graphql
// highlight-next-line
query GetCustomers {
  customers {
    id
    name
    email
  }
}
```

### Variables

Operations can also have **variables**. Variables are used to pass input values into the operation. In the example `updateCustomerEmail` mutation
operation above, we are passing an input object specifying the `customerId` and `email`. However, in that example they are hard-coded into the
operation. In a real application, you would want to pass those values in dynamically.

Here's how we can re-write the above mutation operation to use variables:




</Tabs>

### Fragments

A **fragment** is a reusable set of fields on an object type. Let's define a fragment for the `Customer` type that
we can re-use in both the query and the mutation:

```graphql
fragment CustomerFields on Customer {
  id
  name
  email
}
```

Now we can re-write the query and mutation operations to use the fragment:

```graphql
query GetCustomers{
  customers {
    ...CustomerFields
  }
}
```

```graphql
mutation UpdateCustomerEmail($input: UpdateCustomerEmailInput!) {
  updateCustomerEmail(input: $input) {
    ...CustomerFields
  }
}
```

You can think of the syntax as similar to the JavaScript object spread operator (`...`).

### Union types

A **union type** is a special type which can be one of a number of other types. Let's say for example that when attempting to update a customer's
email address, we want to return an error type if the email address is already in use. We can update our schema to model this as a union type:

```graphql
type Mutation {
  updateCustomerEmail(input: UpdateCustomerEmailInput!): UpdateCustomerEmailResult!
}

// highlight-next-line
union UpdateCustomerEmailResult = Customer | EmailAddressInUseError

type EmailAddressInUseError {
  errorCode: String!
  message: String!
}
```

:::info
In Vendure, we use this pattern for almost all mutations. You can read more about it in the [Error Handling guide](/guides/developer-guide/error-handling/).
:::

Now, when we perform this mutation, we need alter the way we select the fields in the response, since the response could be one of two types:




</Tabs>

The `__typename` field is a special field available on all types which returns the name of the type. This is useful for
determining which type was returned in the response in your client application.

:::tip
The above operation could also be written to use the `CustomerFields` fragment we defined earlier:
```graphql
mutation UpdateCustomerEmail($input: UpdateCustomerEmailInput!) {
  updateCustomerEmail(input: $input) {
    // highlight-next-line
    ...CustomerFields
    ... on EmailAddressInUseError {
      errorCode
      message
    }
  }
}
```
:::

### Resolvers

The schema defines the _shape_ of the data, but it does not define _how_ the data is fetched. This is the job of the resolvers.

A resolver is a function which is responsible for fetching the data for a particular field. For example, the `customers` field on the `Query` type
would be resolved by a function which fetches the list of customers from the database.

To get started with Vendure's APIs, you don't need to know much about resolvers beyond this basic understanding. However,
later on you may want to write your own custom resolvers to extend the API. This is covered in the [Extending the GraphQL API guide](/guides/developer-guide/extend-graphql-api/).

## Querying data

Now that we have a basic understanding of the GraphQL type system, let's look at how we can use it to query data from the Vendure API.

In REST terms, a GraphQL query is equivalent to a GET request. It is used to fetch data from the API. Queries should not change any
data on the server.

This is a GraphQL Playground running on a real Vendure server. You can run the query by clicking the "play" button in the
middle of the two panes.

<Playground api="shop" document={`
query {
  product(slug: "football") {
    id
    name
    slug
  }
}
`} />

Let's get familiar with the schema:

1. Hover your mouse over any field to see its type, and in the case of the `product` field itself, you'll see documentation about what it does.
2. Add a new line after `slug` and press `Ctrl / ⌘` + `space` to see the available fields. At the bottom of the field list, you'll see the type of that field.
3. Try adding the `description` field and press play. You should see the product's description in the response.
4. Try adding `variants` to the field list. You'll see a red warning in the left edge, and hovering over `variants` will inform
you that it must have a selection of subfields. This is because the `variants` field refers to an **object type**, so we must select
which fields of that object type we want to fetch. For example:
  ```graphql
  query {
    product(slug: "football") {
      id
      name
      slug
      variants {
        // highlight-start
        # Sub-fields are required for object types
        sku
        priceWithTax
        // highlight-end
      }
    }
  }
  ```

## IDE plugins

Plugins are available for most popular IDEs & editors which provide auto-complete and type-checking for GraphQL operations
as you write them. This is a huge productivity boost, and is **highly recommended**.

- [GraphQL extension for VS Code](https://marketplace.visualstudio.com/items?itemName=GraphQL.vscode-graphql)
- [GraphQL plugin for IntelliJ](https://plugins.jetbrains.com/plugin/8097-graphql) (including WebStorm)

## Code generation

Code generation means the automatic generation of TypeScript types based on your GraphQL schema and your GraphQL operations. This is a very powerful feature that allows you to write your code in a type-safe manner, without you needing to manually write any types for your API calls.

For more information see the [GraphQL Code Generation guide](/guides/storefront/codegen).

## Further reading

This is just a very brief overview, intended to introduce you to the main concepts you'll need to build with Vendure.
There are many more language features and best practices to learn about which we did not cover here.

Here are some resources you can use to gain a deeper understanding of GraphQL:

- The official [Introduction to GraphQL on graphql.org](https://graphql.org/learn/) is comprehensive and easy to follow.
- For a really fundamental understanding, see the [GraphQL language spec](https://spec.graphql.org/).
- If you like to learn from videos, the [graphql.wtf](https://graphql.wtf/) series is a great resource.


---
title: "Try the API"
---

import Playground from '@site/src/components/Playground';

Once you have successfully installed Vendure locally following the [installation guide](/guides/getting-started/installation),
it's time to try out the API!

:::note
This guide assumes you chose to populate sample data when installing Vendure.

You can also follow along with these example using the public demo playground at [demo.vendure.io/shop-api](https://demo.vendure.io/shop-api)
:::

## GraphQL Playground

The Vendure server comes with a GraphQL Playground which allows you to explore the API and run queries and mutations. It is
a great way to explore the API and to get a feel for how it works.

In this guide, we'll be using the GraphQL Playground to run queries and mutations against the Shop and Admin APIs. At each
step, you paste the query or mutation into the left-hand pane of the Playground, and then click the "Play" button to run it.
You'll then see the response in the right-hand pane.

![Graphql playground](./playground.webp)

:::note
Before we start using the GraphQL Playground, we need to alter a setting to make sure it is handling session cookies
correctly.

In the "settings" panel, we need to change the `request.credentials` setting from `omit` to `include`:

![Graphql playground setup](./playground-setup.webp)
:::

## Shop API

The Shop API is the public-facing API which is used by the storefront application.

Open the GraphQL Playground at [http://localhost:3000/shop-api](http://localhost:3000/shop-api).

### Fetch a list of products

Let's start with a **query**. Queries are used to fetch data. We will make a query to get a list of products.

<Playground api="shop" document={`
query {
  products {
    totalItems
    items {
      id
      name
    }
  }
}
`} />


Note that the response only includes the properties we asked for in our query (id and name). This is one of the key benefits
of GraphQL - the client can specify exactly which data it needs, and the server will only return that data!

Let's add a few more properties to the query:

<Playground api="shop" document={`
query {
  products {
    totalItems
    items {
      id
      name
      slug
      description
      featuredAsset {
        id
        preview
      }
    }
  }
}
`} />

You should see that the response now includes the `slug`, `description` and `featuredAsset` properties. Note that the
`featuredAsset` property is itself an object, and we can specify which properties of that object we want to include in the
response. This is another benefit of GraphQL - you can "drill down" into the data and specify exactly which properties you
want to include.

Now let's add some arguments to the query. Some queries (and most mutations) can accept argument, which you put in parentheses
after the query name. For example, let's fetch the first 5 products:

<Playground api="shop" document={`
query {
  products(options: { take: 5 }) {
    totalItems
    items {
      id
      name
    }
  }
}
`} />

On running this query, you should see just the first 5 results being returned.

Let's add a more complex argument: this time we'll filter for only those products which contain the string "shoe" in the
name:

<Playground api="shop" document={`
query {
  products(options: {
    filter: { name: { contains: "shoe" } }
  }) {
    totalItems
    items {
      id
      name
    }
  }
}
`} />

### Add a product to an order

Next, let's look at a **mutation**. Mutations are used to modify data on the server.

Here's a mutation which adds a product to an order:

<Playground api="shop" document={`
mutation {
  addItemToOrder(productVariantId: 42, quantity: 1) {
    ...on Order {
      id
      code
      totalQuantity
      totalWithTax
      lines {
        productVariant {
          name
        }
        quantity
        linePriceWithTax
      }
    }
    ...on ErrorResult {
      errorCode
      message
    }
  }
}
`} />

This mutation adds a product variant with ID `42` to the order. The response will either be an `Order` object, or an `ErrorResult`.
We use a special syntax called a **fragment** to specify which properties we want to include in the response. In this case,
we are saying that if the response is an `Order`, we want to include the `id`, `code`, `totalQuantity`, `totalWithTax` etc., and
if the response is an `ErrorResult`, we want to include the `errorCode` and `message`.

Running this mutation a second time should show that the quantity of the product in the order has increased by 1. If not,
then the session is not being persisted correctly (see the note earlier in this guide about the `request.credentials` setting).

:::info
For more information about `ErrorResult` and the handling of errors in Vendure, see the [Error Handling guide](/guides/developer-guide/error-handling).
:::

## Admin API

The Admin API exposes all the functionality required to manage the store. It is used by the Admin UI, but can also be used
by integrations and custom scripts.

:::note
The examples in this section are not interactive, due to security settings on our demo server,
but you can paste them into your local GraphQL playground.
:::

Open the GraphQL Playground at [http://localhost:3000/admin-api](http://localhost:3000/admin-api).

### Logging in

Most Admin API operations are restricted to authenticated users. So first of all we'll need to log in.

```graphql title="Admin API"
mutation {
  login(username: "superadmin", password: "superadmin") {
    ...on CurrentUser {
      id
      identifier
    }
    ...on ErrorResult {
      errorCode
      message
    }
  }
}
```

### Fetch a product

The Admin API exposes a lot more information about products than you can get from the Shop API:

```graphql title="Admin API"
query {
  product(id: 42) {
    // highlight-next-line
    enabled
    name
    variants {
      id
      name
      // highlight-next-line
      enabled
      prices {
        currencyCode
        price
      }
      // highlight-start
      stockLevels {
        stockLocationId
        stockOnHand
        stockAllocated
      }
      // highlight-end
    }
  }
}
```

:::info
GraphQL is statically typed and uses a **schema** containing information about all the available queries, mutations and types. In the
GraphQL playground, you can explore the schema by clicking the "docs" tab on the right hand side.

![Graphql playground docs](./playground-docs.webp)
:::


---
title: "Digital Products"
---

import Tabs from '@theme/Tabs';
import TabItem from '@theme/TabItem';

Digital products include things like ebooks, online courses, and software. They are products that are delivered to the customer electronically, and do not require
physical shipping.

This guide will show you how you can add support for digital products to Vendure.

## Creating the plugin

:::info
The complete source of the following example plugin can be found here: [example-plugins/digital-products](https://github.com/vendure-ecommerce/vendure/tree/master/packages/dev-server/example-plugins/digital-products)
:::

### Define custom fields

If some products are digital and some are physical, we can distinguish between them by adding a customField to the ProductVariant entity.

```ts title="src/plugins/digital-products/digital-products-plugin.ts"
import { LanguageCode, PluginCommonModule, VendurePlugin } from '@vendure/core';

@VendurePlugin({
    imports: [PluginCommonModule],
    configuration: config => {
        // highlight-start
        config.customFields.ProductVariant.push({
            type: 'boolean',
            name: 'isDigital',
            defaultValue: false,
            label: [{ languageCode: LanguageCode.en, value: 'This product is digital' }],
            public: true,
        });
        // highlight-end
        return config;
    },
})
export class DigitalProductsPlugin {}
```

We will also define a custom field on the `ShippingMethod` entity to indicate that this shipping method is only available for digital products:

```ts title="src/plugins/digital-products/digital-products-plugin.ts"
import { LanguageCode, PluginCommonModule, VendurePlugin } from '@vendure/core';

@VendurePlugin({
    imports: [PluginCommonModule],
    configuration: config => {
        // config.customFields.ProductVariant.push({ ... omitted
        // highlight-start
        config.customFields.ShippingMethod.push({
            type: 'boolean',
            name: 'digitalFulfilmentOnly',
            defaultValue: false,
            label: [{ languageCode: LanguageCode.en, value: 'Digital fulfilment only' }],
            public: true,
        });
        // highlight-end
        return config;
    },
})
```

Lastly we will define a custom field on the `Fulfillment` entity where we can store download links for the digital products. If your own implementation you may
wish to handle this part differently, e.g. storing download links on the `Order` entity or in a custom entity.

```ts title="src/plugins/digital-products/digital-products-plugin.ts"
import { LanguageCode, PluginCommonModule, VendurePlugin } from '@vendure/core';

@VendurePlugin({
    imports: [PluginCommonModule],
    configuration: config => {
        // config.customFields.ProductVariant.push({ ... omitted
        // config.customFields.ShippingMethod.push({ ... omitted
        // highlight-start
        config.customFields.Fulfillment.push({
            type: 'string',
            name: 'downloadUrls',
            nullable: true,
            list: true,
            label: [{ languageCode: LanguageCode.en, value: 'Urls of any digital purchases' }],
            public: true,
        });
        // highlight-end
        return config;
    },
})
```

### Create a custom FulfillmentHandler

The `FulfillmentHandler` is responsible for creating the `Fulfillment` entities when an Order is fulfilled. We will create a custom handler which
is responsible for performing the logic related to generating the digital download links.

In your own implementation, this may look significantly different depending on your requirements.

```ts title="src/plugins/digital-products/config/digital-fulfillment-handler.ts"
import { FulfillmentHandler, LanguageCode, OrderLine, TransactionalConnection } from '@vendure/core';
import { In } from 'typeorm';

let connection: TransactionalConnection;

/**
 * @description
 * This is a fulfillment handler for digital products which generates a download url
 * for each digital product in the order.
 */
export const digitalFulfillmentHandler = new FulfillmentHandler({
    code: 'digital-fulfillment',
    description: [
        {
            languageCode: LanguageCode.en,
            value: 'Generates product keys for the digital download',
        },
    ],
    args: {},
    init: injector => {
        connection = injector.get(TransactionalConnection);
    },
    createFulfillment: async (ctx, orders, lines) => {
        const digitalDownloadUrls: string[] = [];

        const orderLines = await connection.getRepository(ctx, OrderLine).find({
            where: {
                id: In(lines.map(l => l.orderLineId)),
            },
            relations: {
                productVariant: true,
            },
        });
        for (const orderLine of orderLines) {
            if (orderLine.productVariant.customFields.isDigital) {
                // This is a digital product, so generate a download url
                const downloadUrl = await generateDownloadUrl(orderLine);
                digitalDownloadUrls.push(downloadUrl);
            }
        }
        return {
            method: 'Digital Fulfillment',
            trackingCode: 'DIGITAL',
            customFields: {
                downloadUrls: digitalDownloadUrls,
            },
        };
    },
});

function generateDownloadUrl(orderLine: OrderLine) {
    // This is a dummy function that would generate a download url for the given OrderLine
    // by interfacing with some external system that manages access to the digital product.
    // In this example, we just generate a random string.
    const downloadUrl = `https://example.com/download?key=${Math.random().toString(36).substring(7)}`;
    return Promise.resolve(downloadUrl);
}
```

### Create a custom ShippingEligibilityChecker

We want to ensure that the digital shipping method is only applicable to orders containing at least one digital product.
We do this with a custom ShippingEligibilityChecker:

```ts title="src/plugins/digital-products/config/digital-shipping-eligibility-checker.ts"
import { LanguageCode, ShippingEligibilityChecker } from '@vendure/core';

export const digitalShippingEligibilityChecker = new ShippingEligibilityChecker({
    code: 'digital-shipping-eligibility-checker',
    description: [
        {
            languageCode: LanguageCode.en,
            value: 'Allows only orders that contain at least 1 digital product',
        },
    ],
    args: {},
    check: (ctx, order, args) => {
        const digitalOrderLines = order.lines.filter(l => l.productVariant.customFields.isDigital);
        return digitalOrderLines.length > 0;
    },
});
```

### Create a custom ShippingLineAssignmentStrategy

When adding shipping methods to the order, we want to ensure that digital products are correctly assigned to the digital shipping
method, and physical products are not.

```ts title="src/plugins/digital-products/config/digital-shipping-line-assignment-strategy.ts"
import {
    Order,
    OrderLine,
    RequestContext,
    ShippingLine,
    ShippingLineAssignmentStrategy,
} from '@vendure/core';

/**
 * @description
 * This ShippingLineAssignmentStrategy ensures that digital products are assigned to a
 * ShippingLine which has the `isDigital` flag set to true.
 */
export class DigitalShippingLineAssignmentStrategy implements ShippingLineAssignmentStrategy {
    assignShippingLineToOrderLines(
        ctx: RequestContext,
        shippingLine: ShippingLine,
        order: Order,
    ): OrderLine[] | Promise

</Tabs>

If the "digital download" shipping method is eligible, it should be set as a shipping method along with any other method
required by any physical products in the order.



</Tabs>


---
title: "Paginated lists"
---

import Tabs from '@theme/Tabs';
import TabItem from '@theme/TabItem';

Vendure's list queries follow a set pattern which allows for pagination, filtering & sorting. This guide will demonstrate how
to implement your own paginated list queries.

## API definition

Let's start with defining the GraphQL schema for our query. In this example, we'll image that we have defined a [custom entity](/guides/developer-guide/database-entity/) to
represent a `ProductReview`. We want to be able to query a list of reviews in the Admin API. Here's how the schema definition
would look:

```ts title="src/plugins/reviews/api/api-extensions.ts"
import gql from 'graphql-tag';

export const adminApiExtensions = gql`
type ProductReview implements Node {
  id: ID!
  createdAt: DateTime!
  updatedAt: DateTime!
  product: Product!
  productId: ID!
  text: String!
  rating: Float!
}

type ProductReviewList implements PaginatedList {
  items: [ProductReview!]!
  totalItems: Int!
}

# Generated at run-time by Vendure
input ProductReviewListOptions

extend type Query {
   productReviews(options: ProductReviewListOptions): ProductReviewList!
}
`;
```

Note that we need to follow these conventions:

- The type must implement the `Node` interface, i.e. it must have an `id: ID!` field.
- The list type must be named `

</Tabs>

In the above example, we are querying the first 10 reviews, sorted by `createdAt` in descending order, and filtered to only
include reviews with a rating between 3 and 5.


---
title: "Managing the Active Order"
---

import Stackblitz from '@site/src/components/Stackblitz';

The "active order" is what is also known as the "cart" - it is the order that is currently being worked on by the customer.

An order remains active until it is completed, and during this time it can be modified by the customer in various ways:

- Adding an item
- Removing an item
- Changing the quantity of items
- Applying a coupon
- Removing a coupon

This guide will cover how to manage the active order.

## Define an Order fragment

Since all the mutations that we will be using in this guide require an `Order` object, we will define a fragment that we can
reuse in all of our mutations.

```ts title="src/fragments.ts"
const ACTIVE_ORDER_FRAGMENT = /*GraphQL*/`
fragment ActiveOrder on Order {
  __typename
  id
  code
  couponCodes
  state
  currencyCode
  totalQuantity
  subTotalWithTax
  shippingWithTax
  totalWithTax
  discounts {
    description
    amountWithTax
  }
  lines {
    id
    unitPriceWithTax
    quantity
    linePriceWithTax
    productVariant {
      id
      name
      sku
    }
    featuredAsset {
      id
      preview
    }
  }
  shippingLines {
    shippingMethod {
      description
    }
    priceWithTax
  }
}`
```

:::note
The `__typename` field is used to determine the type of the object returned by the GraphQL server. In this case, it will always be `'Order'`.

Some GraphQL clients such as Apollo Client will automatically add the `__typename` field to all queries and mutations, but if you are using a different client you may need to add it manually.
:::

This fragment can then be used in subsequent queries and mutations by using the `...` spread operator in the place where an `Order` object is expected.
You can then embed the fragment in the query or mutation by using the `${ACTIVE_ORDER_FRAGMENT}` syntax:

```ts title="src/queries.ts"
import { ACTIVE_ORDER_FRAGMENT } from './fragments';

export const GET_ACTIVE_ORDER = /*GraphQL*/`
  query GetActiveOrder {
    activeOrder {
      // highlight-next-line
      ...ActiveOrder
    }
  }
  // highlight-next-line
  ${ACTIVE_ORDER_FRAGMENT}
`;
```

For the remainder of this guide, we will list just the body of the query or mutation, and assume that the fragment is defined and imported as above.

## Get the active order

This fragment can then be used in subsequent queries and mutations. Let's start with a query to get the active order using the [`activeOrder` query](/reference/graphql-api/shop/queries#activeorder):

```graphql
query GetActiveOrder {
  activeOrder {
    ...ActiveOrder
  }
}
```

## Add an item

To add an item to the active order, we use the [`addItemToOrder` mutation](/reference/graphql-api/shop/mutations/#additemtoorder), as we have seen in the [Product Detail Page guide](/guides/storefront/product-detail/).

```graphql
mutation AddItemToOrder($productVariantId: ID!, $quantity: Int!) {
  addItemToOrder(productVariantId: $productVariantId, quantity: $quantity) {
    ...ActiveOrder
    ... on ErrorResult {
      errorCode
      message
    }
    ... on InsufficientStockError {
      quantityAvailable
      order {
        ...ActiveOrder
      }
    }
  }
}
```

:::info
If you have defined any custom fields on the `OrderLine` entity, you will be able to pass them as a `customFields` argument to the `addItemToOrder` mutation.
See the [Configurable Products guide](/guides/how-to/configurable-products/) for more information.
:::

## Remove an item

To remove an item from the active order, we use the [`removeOrderLine` mutation](/reference/graphql-api/shop/mutations/#removeorderline),
and pass the `id` of the `OrderLine` to remove.

```graphql
mutation RemoveItemFromOrder($orderLineId: ID!) {
  removeOrderLine(orderLineId: $orderLineId) {
    ...ActiveOrder
    ... on ErrorResult {
      errorCode
      message
    }
  }
}
```

## Change the quantity of an item

To change the quantity of an item in the active order, we use the [`adjustOrderLine` mutation](/reference/graphql-api/shop/mutations/#adjustorderline).

```graphql
mutation AdjustOrderLine($orderLineId: ID!, $quantity: Int!) {
  adjustOrderLine(orderLineId: $orderLineId, quantity: $quantity) {
    ...ActiveOrder
    ... on ErrorResult {
        errorCode
        message
    }
  }
}
```

:::info
If you have defined any custom fields on the `OrderLine` entity, you will be able to update their values by passing a `customFields` argument to the `adjustOrderLine` mutation.
See the [Configurable Products guide](/guides/how-to/configurable-products/) for more information.
:::

## Applying a coupon code

If you have defined any [Promotions](/guides/core-concepts/promotions/) which use coupon codes, you can apply the a coupon code to the active order
using the [`applyCouponCode` mutation](/reference/graphql-api/shop/mutations/#applycouponcode).

```graphql
mutation ApplyCouponCode($couponCode: String!) {
  applyCouponCode(couponCode: $couponCode) {
    ...ActiveOrder
    ... on ErrorResult {
      errorCode
      message
    }
  }
}
```

## Removing a coupon code

To remove a coupon code from the active order, we use the [`removeCouponCode` mutation](/reference/graphql-api/shop/mutations/#removecouponcode).

```graphql
mutation RemoveCouponCode($couponCode: String!) {
  removeCouponCode(couponCode: $couponCode) {
    ...ActiveOrder
    ... on ErrorResult {
      errorCode
      message
    }
  }
}
```

## Live example

Here is a live example which demonstrates adding, updating and removing items from the active order:




---
title: "Checkout Flow"
---

import Tabs from '@theme/Tabs';
import TabItem from '@theme/TabItem';

Once the customer has added the desired products to the active order, it's time to check out.

This guide assumes that you are using the [default OrderProcess](/guides/core-concepts/orders/#the-order-process), so
if you have defined a custom process, some of these steps may be slightly different.

:::note
In this guide, we will assume that an `ActiveOrder` fragment has been defined, as detailed in the
[Managing the Active Order guide](/guides/storefront/active-order/#define-an-order-fragment), but for the purposes of
checking out the fragment should also include `customer` `shippingAddress` and `billingAddress` fields.
:::

## Add a customer

Every order must be associated with a customer. If the customer is not logged in, then the [`setCustomerForOrder`](/reference/graphql-api/shop/mutations/#setcustomerfororder) mutation must be called. This will create a new Customer record if the provided email address does not already exist in the database.

:::note
If the customer is already logged in, then this step is skipped.
:::



</Tabs>

## Set the shipping address

The [`setOrderShippingAddress`](/reference/graphql-api/shop/mutations/#setordershippingaddress) mutation must be called to set the shipping address for the order.




</Tabs>

If the customer is logged in, you can check their existing addresses and pre-populate an address form if an existing address is found.




</Tabs>

## Set the shipping method

Now that we know the shipping address, we can check which shipping methods are available with the [`eligibleShippingMethods`](/reference/graphql-api/shop/queries/#eligibleshippingmethods) query.



</Tabs>

The results can then be displayed to the customer so they can choose the desired shipping method. If there is only a single
result, then it can be automatically selected.

The desired shipping method's id is the passed to the [`setOrderShippingMethod`](/reference/graphql-api/shop/mutations/#setordershippingmethod) mutation.

```graphql
mutation SetShippingMethod($id: [ID!]!) {
    setOrderShippingMethod(shippingMethodId: $id) {
        ...ActiveOrder
        ...on ErrorResult {
            errorCode
            message
        }
    }
}
```

## Add payment

The [`eligiblePaymentMethods`](/reference/graphql-api/shop/queries/#eligiblepaymentmethods) query can be used to get a list of available payment methods.
This list can then be displayed to the customer, so they can choose the desired payment method.




</Tabs>

Next, we need to transition the order to the `ArrangingPayment` state. This state ensures that no other changes can be made to the order
while the payment is being arranged. The [`transitionOrderToState`](/reference/graphql-api/shop/mutations/#transitionordertostate) mutation is used to transition the order to the `ArrangingPayment` state.



</Tabs>

At this point, your storefront will use an integration with the payment provider to collect the customer's payment details, and then
the exact sequence of API calls will depend on the payment integration.

The [`addPaymentToOrder`](/reference/graphql-api/shop/mutations/#addpaymenttoorder) mutation is the general-purpose mutation for adding a payment to an order.
It accepts a `method` argument which must corresponde to the `code` of the selected payment method, and a `metadata` argument which is a JSON object containing any additional information required by that particular integration.

For example, the demo data populated in a new Vendure installation includes a "Standard Payment" method, which uses the [`dummyPaymentHandler`](/reference/typescript-api/payment/dummy-payment-handler) to simulate a payment provider. Here's how you would add a payment using this method:




</Tabs>

Other payment integrations have specific setup instructions you must follow:

### Stripe

Our [`StripePlugin docs`](/reference/core-plugins/payments-plugin/stripe-plugin/) describe how to set up your checkout to use Stripe.

### Braintree

Our [`BraintreePlugin` docs](/reference/core-plugins/payments-plugin/braintree-plugin/) describe how to set up your checkout to use Braintree.

### Mollie

Our [`MolliePlugin` docs](/reference/core-plugins/payments-plugin/mollie-plugin/) describe how to set up your checkout to use Mollie.

### Other payment providers

For more information on how to integrate with a payment provider, see the [Payment](/guides/core-concepts/payment/) guide.

## Display confirmation

Once the checkout has completed, the order will no longer be considered "active" (see the [`OrderPlacedStrategy`](/reference/typescript-api/orders/order-placed-strategy/)) and so the [`activeOrder`](/reference/graphql-api/shop/queries/#activeorder) query will return `null`. Instead, the [`orderByCode`](/reference/graphql-api/shop/queries/#orderbycode) query can be used to retrieve the order by its code to display a confirmation page.



</Tabs>

:::info
By default Vendure will only allow access to the order by code for the first 2 hours after the order is placed if the customer is not logged in.
This is to prevent a malicious user from guessing order codes and viewing other customers' orders. This can be configured via the [`OrderByCodeAccessStrategy`](/reference/typescript-api/orders/order-by-code-access-strategy/).
:::




---
title: "Storefront GraphQL Code Generation"
---

Code generation means the automatic generation of TypeScript types based on your GraphQL schema and your GraphQL operations.
This is a very powerful feature that allows you to write your code in a type-safe manner, without you needing to manually
write any types for your API calls.

To do this, we will use [Graphql Code Generator](https://the-guild.dev/graphql/codegen).

:::note
This guide is for adding codegen to your storefront. For a guide on adding codegen to your backend Vendure plugins or UI extensions, see the [Plugin Codegen](/guides/how-to/codegen/) guide.
:::

## Installation

Follow the installation instructions in the [GraphQL Code Generator Quick Start](https://the-guild.dev/graphql/codegen/docs/getting-started/installation).

Namely:

```bash
npm i graphql
npm i -D typescript @graphql-codegen/cli

npx graphql-code-generator init

npm install
```

During the `init` step, you'll be prompted to select various options about how to configure the code generation.

- Where is your schema?: Use `http://localhost:3000/shop-api` (unless you have configured a different GraphQL API URL)
- Where are your operations and fragments?: Use the appropriate glob pattern for you project. For example, `src/**/*.{ts,tsx}`.
- Select `codegen.ts` as the name of the config file.

## Configuration

The `init` step above will create a `codegen.ts` file in your project root. Add the highlighted lines:

```ts title="codegen.ts"

import type { CodegenConfig } from '@graphql-codegen/cli';

const config: CodegenConfig = {
  overwrite: true,
  schema: 'http://localhost:3000/shop-api',
  documents: 'src/**/*.graphql.ts',
  generates: {
    'src/gql/': {
      preset: 'client',
      plugins: [],
      // highlight-start
      config: {
        scalars: {
            // This tells codegen that the `Money` scalar is a number
            Money: 'number',
        },
        namingConvention: {
            // This ensures generated enums do not conflict with the built-in types.
            enumValues: 'keep',
        },
      }
      // highlight-end
    },
  }
};

export default config;
```

## Running Codegen

During the `init` step, you will have installed a `codegen` script in your `package.json`. You can run this script to
generate the TypeScript types for your GraphQL operations.

:::note
Ensure you have the Vendure server running before running the codegen script.
:::

```bash
npm run codegen
```

This will generate a `src/gql` directory containing the TypeScript types for your GraphQL operations.

## Use the `graphql()` function

If you have existing GraphQL queries and mutations in your application, you can now use the `graphql()` function
exported by the `src/gql/index.ts` file to execute them. If you were previously using the `gql` tagged template function,
replace it with the `graphql()` function.

```ts title="src/App.tsx"
import { useQuery } from '@tanstack/react-query';
import request from 'graphql-request'
// highlight-next-line
import { graphql } from './gql';

// highlight-start
// GET_PRODUCTS will be a `TypedDocumentNode` type,
// which encodes the types of the query variables and the response data.
const GET_PRODUCTS = graphql(`
// highlight-end
    query GetProducts($options: ProductListOptions) {
        products(options: $options) {
            items {
                id
                name
                slug
                featuredAsset {
                    preview
                }
            }
        }
    }
`);

export default function App() {
  // highlight-start
  // `data` will now be correctly typed
  const { isLoading, data } = useQuery({
  // highlight-end
    queryKey: ['products'],
    queryFn: async () =>
      request(
        'http://localhost:3000/shop-api',
        // highlight-start
        GET_PRODUCTS,
        {
        // The variables will also be correctly typed
        options: { take: 3 },
        }
        // highlight-end
      ),
  });

  if (isLoading) return ;

  return data ? (
    data.products.items.map(({ id, name, slug, featuredAsset }) => (
      
        
      </div>
    ))
  ) : (
    <>Loading...</>
  );
}
```

In the above example, the type information all works out of the box because the `graphql-request` library from v5.0.0
has built-in support for the [`TypedDocumentNode`](https://github.com/dotansimha/graphql-typed-document-node) type,
as do the latest versions of most of the popular GraphQL client libraries, such as Apollo Client & Urql.

:::note
In the documentation examples on other pages, we do not assume the use of code generation in order to keep the examples as simple as possible.
:::


---
title: Connect to the API
---

import Tabs from '@theme/Tabs';
import TabItem from '@theme/TabItem';
import Stackblitz from '@site/src/components/Stackblitz';

The first thing you'll need to do is to connect your storefront app to the **Shop API**. The Shop API is a GraphQL API
that provides access to the products, collections, customer data, and exposes mutations that allow you to add items to
the cart, checkout, manage customer accounts, and more.

:::tip
You can explore the Shop API by opening the GraphQL Playground in your browser at
[`http://localhost:3000/shop-api`](http://localhost:3000/shop-api) when your Vendure
server is running locally.
:::

## Select a GraphQL client

GraphQL requests are made over HTTP, so you can use any HTTP client such as the [Fetch API](https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API) to make requests to the Shop API. However, there are also a number of specialized GraphQL clients which can make working with GraphQL APIs easier. Here are some popular options:

* [Apollo Client](https://www.apollographql.com/docs/react): A full-featured client which includes a caching layer and React integration.
* [urql](https://formidable.com/open-source/urql/): The highly customizable and versatile GraphQL client for React, Svelte, Vue, or plain JavaScript
* [graphql-request](https://github.com/jasonkuhrt/graphql-request): Minimal GraphQL client supporting Node and browsers for scripts or simple apps
* [TanStack Query](https://tanstack.com/query/latest): Powerful asynchronous state management for TS/JS, React, Solid, Vue and Svelte, which can be combined with `graphql-request`.


## Managing Sessions

Vendure supports two ways to manage user sessions: **cookies** and **bearer token**. The method you choose depends on your requirements, and is specified by the [`authOptions.tokenMethod` property](/reference/typescript-api/auth/auth-options/#tokenmethod) of the VendureConfig. By default, both are enabled on the server:

```ts title="src/vendure-config.ts"
import { VendureConfig } from '@vendure/core';

export const config: VendureConfig = {
    // ...
    authOptions: {
        // highlight-next-line
        tokenMethod: ['bearer', 'cookie'],
    },
};
```

### Cookie-based sessions

Using cookies is the simpler approach for browser-based applications, since the browser will manage the cookies for you automatically.

1. Enable the `credentials` option in your HTTP client. This allows the browser to send the session cookie with each request.

    For example, if using a fetch-based client (such as [Apollo client](https://www.apollographql.com/docs/react/recipes/authentication/#cookie)) you would set `credentials: 'include'` or if using [XMLHttpRequest](https://developer.mozilla.org/en-US/docs/Web/API/XMLHttpRequest/withCredentials), you would set `withCredentials: true`

2. When using cookie-based sessions, you should set the [`authOptions.cookieOptions.secret` property](/reference/typescript-api/auth/cookie-options#secret) to some secret string which will be used to sign the cookies sent to clients to prevent tampering. This string could be hard-coded in your config file, or (better) reside in an environment variable:

    ```ts title="src/vendure-config.ts"
    import { VendureConfig } from '@vendure/core';

    export const config: VendureConfig = {
        // ...
        authOptions: {
            tokenMethod: ['bearer', 'cookie'],
            // highlight-start
            cookieOptions: {
                secret: process.env.COOKIE_SESSION_SECRET
            }
            // highlight-end
        }
    }
    ```

:::caution
**SameSite cookies**

When using cookies to manage sessions, you need to be aware of the SameSite cookie policy. This policy is designed to prevent cross-site request forgery (CSRF) attacks, but can cause problems when using a headless storefront app which is hosted on a different domain to the Vendure server. See [this article](https://web.dev/samesite-cookies-explained/) for more information.
:::

### Bearer-token sessions

Using bearer tokens involves a bit more work on your part: you'll need to manually read response headers to get the token, and once you have it you'll have to manually add it to the headers of each request.

The workflow would be as follows:

1. Certain mutations and queries initiate a session (e.g. logging in, adding an item to an order etc.). When this happens, the response will contain a HTTP header which [by default is called `'vendure-auth-token'`](/reference/typescript-api/auth/auth-options#authtokenheaderkey).
2. So your http client would need to check for the presence of this header each time it receives a response from the server.
3. If the `'vendure-auth-token'` header is present, read the value and store it because this is your bearer token.
4. Attach this bearer token to each subsequent request as `Authorization: Bearer 
;
    if (error) return ;

    return data.products.items.map(({ id, name, slug, featuredAsset }) => (
        
            
        </div>
    ));
}

```

</TabItem>

 );
```

</TabItem>
</Tabs>

Here's a live version of this example:



As you can see, the basic implementation with `fetch` is quite straightforward. However, it is also lacking some features that other,
dedicated client libraries will provide.

### Apollo Client

Here's an example configuration for [Apollo Client](https://www.apollographql.com/docs/react/) with a React app.

Follow the [getting started instructions](https://www.apollographql.com/docs/react/get-started) to install the required
packages.



,
);
```

</TabItem>
;
    if (error) return ;

    return data.products.items.map(({ id, name, slug, featuredAsset }) => (
        
            
        </div>
    ));
}
```

</TabItem>
</Tabs>


Here's a live version of this example:



### TanStack Query

Here's an example using [@tanstack/query](https://tanstack.com/query/latest) in combination with [graphql-request](https://github.com/jasonkuhrt/graphql-request) based on [this guide](https://tanstack.com/query/v4/docs/react/graphql).

Note that in this example we have also installed the [`@graphql-typed-document-node/core` package](https://github.com/dotansimha/graphql-typed-document-node), which allows the
client to work with TypeScript code generation for type-safe queries.



;

    return data ? (
        data.products.items.map(({ id, name, slug, featuredAsset }) => (
            
                
            </div>
        ))
    ) : (
        <>Loading...</>
    );
}
```

</TabItem>



    </StrictMode>
);
```

</TabItem>
</Tabs>

Here's a live version of this example:





---
title: "Customer Accounts"
---

import Tabs from '@theme/Tabs';
import TabItem from '@theme/TabItem';

Customers can register accounts and thereby gain the ability to:

- View past orders
- Store multiple addresses
- Maintain an active order across devices
- Take advantage of plugins that expose functionality to registered customers only, such as wishlists & loyalty points.

## Querying the active customer

The [`activeCustomer` query](/reference/graphql-api/shop/queries/#activecustomer) will return a [`Customer`](/reference/graphql-api/shop/object-types#customer) object if the customer is registered and logged in, otherwise it will return `null`. This can be used in the storefront header for example to
determine whether to display a "sign in" or "my account" link.




</Tabs>

## Logging in and out

The [`login` mutation](/reference/graphql-api/shop/mutations#login) is used to attempt to log in using email address and password.
Given correct credentials, a new authenticated session will begin for that customer.






</Tabs>

The [`logout` mutation](/reference/graphql-api/shop/mutations#logout) will end an authenticated customer session.




</Tabs>

:::note
The `login` mutation, as well as the following mutations related to registration & password recovery only
apply when using the built-in [`NativeAuthenticationStrategy`](/reference/typescript-api/auth/native-authentication-strategy/).

If you are using alternative authentication strategies in your storefront, you would use the [`authenticate` mutation](/reference/graphql-api/shop/mutations/#authenticate) as covered in the [External Authentication guide](/guides/core-concepts/auth/#external-authentication).
:::

## Registering a customer account

The [`registerCustomerAccount` mutation](/reference/graphql-api/shop/mutations/#registercustomeraccount) is used to register a new customer account.

There are three possible registration flows:
If [`authOptions.requireVerification`](/reference/typescript-api/auth/auth-options/#requireverification) is set to `true` (the default):

1. **The Customer is registered _with_ a password**. A verificationToken will be created (and typically emailed to the Customer). That
verificationToken would then be passed to the verifyCustomerAccount mutation _without_ a password. The Customer is then
verified and authenticated in one step.
2. **The Customer is registered _without_ a password**. A verificationToken will be created (and typically emailed to the Customer). That
verificationToken would then be passed to the verifyCustomerAccount mutation _with_ the chosen password of the Customer. The Customer is then
verified and authenticated in one step.

If `authOptions.requireVerification` is set to `false`:

3. The Customer _must_ be registered _with_ a password. No further action is needed - the Customer is able to authenticate immediately.

Here's a diagram of the second scenario, where the password is supplied during the _verification_ step.

![Verification flow](./verification.webp)

Here's how the mutations would look for the above flow:





</Tabs>

Note that in the variables above, we did **not** specify a password, as this will be done at the verification step.
If a password _does_ get passed at this step, then it won't be needed at the verification step. This is a decision
you can make based on the desired user experience of your storefront.

Upon registration, the [EmailPlugin](/reference/core-plugins/email-plugin/) will generate an email to the
customer containing a link to the verification page. In a default Vendure installation this is set in the vendure config file:

```ts title="src/vendure-config.ts"
EmailPlugin.init({
    route: 'mailbox',
    handlers: defaultEmailHandlers,
    templatePath: path.join(__dirname, '../static/email/templates'),
    outputPath: path.join(__dirname, '../static/email/output'),
    globalTemplateVars: {
        fromAddress: '"Vendure Demo Store" 


</Tabs>

## Password reset

Here's how to implement a password reset flow. It is conceptually very similar to the verification flow described above.

![Password reset flow](./pw-reset.webp)

A password reset is triggered by the [`requestPasswordReset` mutation](/reference/graphql-api/shop/mutations/#requestpasswordreset):




</Tabs>

Again, this mutation will trigger an event which the EmailPlugin's default email handlers will pick up and send
an email to the customer. The password reset page then needs to get the token from the url and pass it to the
[`resetPassword` mutation](/reference/graphql-api/shop/mutations/#resetpassword):





</Tabs>


---
title: "Listing Products"
---

import Tabs from '@theme/Tabs';
import TabItem from '@theme/TabItem';
import Stackblitz from '@site/src/components/Stackblitz';

Products are listed when:

- Displaying the contents of a collection
- Displaying search results

In Vendure, we usually use the `search` query for both of these. The reason is that the `search` query is optimized
for high performance, because it is backed by a dedicated search index. Other queries such as `products` or `Collection.productVariants` _can_ also
be used to fetch a list of products, but they need to perform much more complex database queries, and are therefore slower.

## Listing products in a collection

Following on from the [navigation example](/guides/storefront/navigation-menu/), let's assume that a customer has
clicked on a collection item from the menu, and we want to display the products in that collection.

Typically, we will know the `slug` of the selected collection, so we can use the `collection` query to fetch the
details of this collection:




</Tabs>

The collection data can be used to render the page header.

Next, we can use the `search` query to fetch the products in the collection:





</Tabs>

:::note
The key thing to note here is that we are using the `collectionSlug` input to the `search` query. This ensures
that the results all belong to the selected collection.
:::

Here's a live demo of the above code in action:



## Product search

The `search` query can also be used to perform a full-text search of the products in the catalog by passing the
`term` input:




</Tabs>

:::tip
You can also limit the full-text search to a specific collection by passing the
`collectionSlug` or `collectionId` input.
:::

## Faceted search

The `search` query can also be used to perform faceted search. This is a powerful feature which allows customers
to filter the results according to the facet values assigned to the products & variants.

By using the `facetValues` field, the search query will return a list of all the facet values which are present
in the result set. This can be used to render a list of checkboxes or other UI elements which allow the customer
to filter the results.




</Tabs>

These facet values can then be used to filter the results by passing them to the `facetValueFilters` input

For example, let's filter the results to only include products which have the "Nikkon" brand. Based on our last
request we know that there should be 2 such products, and that the `facetValue.id` for the "Nikkon" brand is `11`.

```json
{
  "facetValue": {
    // highlight-next-line
    "id": "11",
    "name": "Nikkon",
    "facet": {
      "id": "2",
      "name": "brand"
    }
  },
  // highlight-next-line
  "count": 2
}
```

Here's how we can use this information to filter the results:

:::note
In the next example, rather than passing each individual variable (skip, take, term) as a separate argument,
we are passing the entire `SearchInput` object as a variable. This allows us more flexibility in how
we use the query, as we can easily add or remove properties from the input object without having to
change the query itself.
:::





</Tabs>

:::info
The `facetValueFilters` input can be used to specify multiple filters, combining each with either `and` or `or`.

For example, to filter by both the "Camera" **and** "Nikkon" facet values, we would use:

```json
{
  "facetValueFilters": [
    { "and": "9" },
    { "and": "11" }
  ]
}
```

To filter by "Nikkon" **or** "Sony", we would use:

```json
{
  "facetValueFilters": [
    { "or": ["11", "15"] }
  ]
}
```
:::

Here's a live example of faceted search. Try searching for terms like "shoe", "plant" or "ball".



## Listing custom product data

If you have defined custom fields on the `Product` or `ProductVariant` entity, you might want to include these in the
search results. With the [`DefaultSearchPlugin`](/reference/typescript-api/default-search-plugin/) this is
not possible, as this plugin is designed to be a minimal and simple search implementation.

Instead, you can use the [`ElasticsearchPlugin`](/reference/core-plugins/elasticsearch-plugin/) which
provides advanced features which allow you to index custom data. The Elasticsearch plugin is designed as a drop-in
replacement for the `DefaultSearchPlugin`, so you can simply swap out the plugins in your `vendure-config.ts` file.


---
title: "Navigation Menu"
---

import Tabs from '@theme/Tabs';
import TabItem from '@theme/TabItem';
import Stackblitz from '@site/src/components/Stackblitz';

A navigation menu allows your customers to navigate your store and find the products they are looking for.

Typically, navigation is based on a hierarchy of [collections](/guides/core-concepts/collections/). We can get the top-level
collections using the `collections` query with the `topLevelOnly` filter:




</Tabs>

## Building a navigation tree

The `collections` query returns a flat list of collections, but we often want to display them in a tree-like structure.
This way, we can build up a navigation menu which reflects the hierarchy of collections.

First of all we need to ensure that we have the `parentId` property on each collection.

```graphql title="Shop API"
query GetAllCollections {
  collections(options: { topLevelOnly: true }) {
    items {
      id
      slug
      name
      // highlight-next-line
      parentId
      featuredAsset {
        id
        preview
      }
    }
  }
}
```

Then we can use this data to build up a tree structure. The following code snippet shows how this can be done in TypeScript:

```ts title="src/utils/array-to-tree.ts"
export type HasParent = { id: string; parentId: string | null };
export type TreeNode<T extends HasParent> = T & {
    children: Array<TreeNode<T>>;
};
export type RootNode<T extends HasParent> = {
    id?: string;
    children: Array<TreeNode<T>>;
};

/**
 * Builds a tree from an array of nodes which have a parent.
 * Based on https://stackoverflow.com/a/31247960/772859, modified to preserve ordering.
 */
export function arrayToTree<T extends HasParent>(nodes: T[]): RootNode<T> {
    const topLevelNodes: Array<TreeNode<T>> = [];
    const mappedArr: { [id: string]: TreeNode<T> } = {};

    // First map the nodes of the array to an object -> create a hash table.
    for (const node of nodes) {
        mappedArr[node.id] = { ...(node as any), children: [] };
    }

    for (const id of nodes.map((n) => n.id)) {
        if (mappedArr.hasOwnProperty(id)) {
            const mappedElem = mappedArr[id];
            const parentId = mappedElem.parentId;
            if (!parent) {
                continue;
            }
            // If the element is not at the root level, add it to its parent array of children.
            const parentIsRoot = !mappedArr[parentId];
            if (!parentIsRoot) {
                if (mappedArr[parentId]) {
                    mappedArr[parentId].children.push(mappedElem);
                } else {
                    mappedArr[parentId] = { children: [mappedElem] } as any;
                }
            } else {
                topLevelNodes.push(mappedElem);
            }
        }
    }
    const rootId = topLevelNodes.length ? topLevelNodes[0].parentId : undefined;
    return { id: rootId, children: topLevelNodes };
}
```

## Live example

Here's a live demo of the above code in action:




---
title: "Product Detail Page"
---

import Tabs from '@theme/Tabs';
import TabItem from '@theme/TabItem';
import Stackblitz from '@site/src/components/Stackblitz';

The product detail page (often abbreviated to PDP) is the page that shows the details of a product and allows the user to add it to their cart.

Typically, the PDP should include:

- Product name
- Product description
- Available product variants
- Images of the product and its variants
- Price information
- Stock information
- Add to cart button

## Fetching product data

Let's create a query to fetch the required data. You should have either the product's `slug` or `id` available from the
url. We'll use the `slug` in this example.




</Tabs>

This single query provides all the data we need to display our PDP.

## Formatting prices

As explained in the [Money & Currency guide](/guides/core-concepts/money/), the prices are returned as integers in the
smallest unit of the currency (e.g. cents for USD). Therefore, when we display the price, we need to divide by 100 and
format it according to the currency's formatting rules.

In the demo at the end of this guide, we'll use the [`formatCurrency` function](/guides/core-concepts/money/#displaying-monetary-values)
which makes use of the browser's `Intl` API to format the price according to the user's locale.

## Displaying images

If we are using the [`AssetServerPlugin`](/reference/core-plugins/asset-server-plugin/) to serve our product images (as is the default), then we can take advantage
of the dynamic image transformation abilities in order to display the product images in the correct size and in
and optimized format such as WebP.

This is done by appending a query string to the image URL. For example, if we want to use the `'large'` size preset (800 x 800)
and convert the format to WebP, we'd use a url like this:

```tsx

```

An even more sophisticated approach would be to make use of the HTML [`
    );
}
```

## Adding to the order

To add a particular product variant to the order, we need to call the [`addItemToOrder`](/reference/graphql-api/shop/mutations/#additemtoorder) mutation.
This mutation takes the `productVariantId` and the `quantity` as arguments.





</Tabs>

There are some important things to note about this mutation:

- Because the `addItemToOrder` mutation returns a union type, we need to use a [fragment](/guides/getting-started/graphql-intro/#fragments) to specify the fields we want to return.
In this case we have defined a fragment called `UpdatedOrder` which contains the fields we are interested in.
- If any [expected errors](/guides/developer-guide/error-handling/) occur, the mutation will return an `ErrorResult` object. We'll be able to
see the `errorCode` and `message` fields in the response, so that we can display a meaningful error message to the user.
- In the special case of the `InsufficientStockError`, in addition to the `errorCode` and `message` fields, we also get the `quantityAvailable` field
which tells us how many of the requested quantity are available (and have been added to the order). This is useful information to display to the user.
The `InsufficientStockError` object also embeds the updated `Order` object, which we can use to update the UI.
- The `__typename` field can be used by the client to determine which type of object has been returned. Its value will equal the name
of the returned type. This means that we can check whether `__typename === 'Order'` in order to determine whether the mutation was successful.

## Live example

Here's an example that brings together all of the above concepts:




---
title: "Storefront Starters"
---

Since building an entire Storefront from scratch can be a daunting task, we have prepared a few starter projects that you can use as a base for your own storefront.

These starters provide basic functionality including:

- Product listing
- Product details
- Search with facets
- Cart functionality
- Checkout flow
- Account management
- Styling with Tailwind

The idea is that you clone the starter project and then customize it to your needs.

:::note
Prefer to build your own solution? Follow the rest of the guides in this section to learn how to build a Storefront from scratch.
:::

## Remix Storefront

- 🔗 [remix-storefront.vendure.io](https://remix-storefront.vendure.io/)
- 💻 [github.com/vendure-ecommerce/storefront-remix-starter](https://github.com/vendure-ecommerce/storefront-remix-starter)

[Remix](https://remix.run/) is a React-based full-stack JavaScript framework which focuses on web standards, modern web app UX, and which helps you build better websites.

Our official Remix Storefront starter provides you with a lightning-fast, modern storefront solution which can be deployed to any of the popular cloud providers like Vercel, Netlify, or Cloudflare Pages.

![Remix Storefront](./remix-storefront.webp)

## Qwik Storefront

- 🔗 [qwik-storefront.vendure.io](https://qwik-storefront.vendure.io/)
- 💻 [github.com/vendure-ecommerce/storefront-qwik-starter](https://github.com/vendure-ecommerce/storefront-qwik-starter)

[Qwik](https://qwik.builder.io/) is a cutting-edge web framework that offers unmatched performance.

Our official Qwik Storefront starter provides you with a lightning-fast, modern storefront solution which can be deployed to any of the popular cloud providers like Vercel, Netlify, or Cloudflare Pages.

![Qwik Storefront](./qwik-storefront.webp)

## Angular Storefront

- 🔗 [angular-storefront.vendure.io](https://angular-storefront.vendure.io/)
- 💻 [github.com/vendure-ecommerce/storefront-angular-starter](https://github.com/vendure-ecommerce/storefront-angular-starter)

[Angular](https://angular.io/) is a popular, stable, enterprise-grade framework made by Google.

Our official Angular Storefront starter is a modern Progressive Web App that uses Angular Universal server-side rendering.

![Angular Storefront](./angular-storefront.webp)



