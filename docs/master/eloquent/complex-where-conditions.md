# Complex Where Conditions

**Experimental: not enabled by default, not guaranteed to be stable.**

Adding query conditions ad-hoc can be cumbersome and limiting when you require
manifold ways to filter query results.
Lighthouse's `WhereConditions` extension can give advanced query capabilities to clients
and allow them to apply complex, dynamic WHERE conditions to queries.

## Setup

For Laravel 11 and higher, add the service provider to your `bootstrap/providers.php`:

```php
return [
    // ...
    \Nuwave\Lighthouse\WhereConditions\WhereConditionsServiceProvider::class,
]
```

For other versions, register the service provider `Nuwave\Lighthouse\WhereConditions\WhereConditionsServiceProvider`,
see [registering providers in Laravel](https://laravel.com/docs/providers#registering-providers).

Install the dependency [mll-lab/graphql-php-scalars](https://github.com/mll-lab/graphql-php-scalars):

```shell
composer require mll-lab/graphql-php-scalars
```

## Usage

You can use this feature through a set of schema directives that enhance fields
with advanced filter capabilities.

### @whereConditions

```graphql
"""
Add a dynamically client-controlled WHERE condition to a fields query.
"""
directive @whereConditions(
  """
  Restrict the allowed column names to a well-defined list.
  This improves introspection capabilities and security.
  Mutually exclusive with `columnsEnum`.
  """
  columns: [String!]

  """
  Use an existing enumeration type to restrict the allowed columns to a predefined list.
  This allows you to re-use the same enum for multiple fields.
  Mutually exclusive with `columns`.
  """
  columnsEnum: String

  """
  Reference a method that applies the client given conditions to the query builder.

  Expected signature: `(
      \Illuminate\Database\Query\Builder|\Illuminate\Database\Eloquent\Builder $builder,
      array<string, mixed> $whereConditions
  ): void`

  Consists of two parts: a class name and a method name, separated by an `@` symbol.
  If you pass only a class name, the method name defaults to `__invoke`.
  """
  handler: String = "\\Nuwave\\Lighthouse\\WhereConditions\\WhereConditionsHandler"
) on ARGUMENT_DEFINITION
```

> This directive only works if the field resolver passes its builder through a call to `$resolveInfo->enhanceBuilder()`.
> Built-in field resolver directives that query the database do this, such as [@all](../api-reference/directives.md#all) or [@hasMany](../api-reference/directives.md#hasmany).

You can apply this directive on any field that performs an Eloquent query:

```graphql
type Query {
  people(
    where: _ @whereConditions(columns: ["age", "type", "haircolour", "height"])
  ): [Person!]! @all
}

type Person {
  id: ID!
  age: Int!
  height: Int!
  type: String!
  hair_colour: String!
}
```

Lighthouse automatically generates definitions for an `Enum` type and an `Input` type
that are restricted to the defined columns, so you do not have to specify them by hand.
The blank type named `_` will be changed to the actual type.
Here are the types that will be included in the compiled schema:

```graphql
"Dynamic WHERE conditions for the `where` argument of the query `people`."
input QueryPeopleWhereWhereConditions {
  "The column that is used for the condition."
  column: QueryPeopleWhereColumn

  "The operator that is used for the condition."
  operator: SQLOperator = EQ

  "The value that is used for the condition."
  value: Mixed

  "A set of conditions that requires all conditions to match."
  AND: [QueryPeopleWhereWhereConditions!]

  "A set of conditions that requires at least one condition to match."
  OR: [QueryPeopleWhereWhereConditions!]

  "Check whether a relation exists. Extra conditions or a minimum amount can be applied."
  HAS: QueryPeopleWhereWhereConditionsRelation
}

"Allowed column names for the `where` argument of the query `people`."
enum QueryPeopleWhereColumn {
  AGE @enum(value: "age")
  TYPE @enum(value: "type")
  HAIRCOLOUR @enum(value: "haircolour")
  HEIGHT @enum(value: "height")
}

"Dynamic HAS conditions for WHERE condition queries."
input QueryPeopleWhereWhereConditionsRelation {
  "The relation that is checked."
  relation: String!

  "The comparison operator to test against the amount."
  operator: SQLOperator = GTE

  "The amount to test."
  amount: Int = 1

  "Additional condition logic."
  condition: QueryPeopleWhereWhereConditions
}
```

Alternatively to the `columns` argument, you can also use `columnsEnum` in case you
want to re-use a list of allowed columns. Here's how your schema could look like:

```graphql
type Query {
  allPeople(where: _ @whereConditions(columnsEnum: "PersonColumn")): [Person!]!
    @all

  paginatedPeople(
    where: _ @whereConditions(columnsEnum: "PersonColumn")
  ): [Person!]! @paginated
}

"A custom description for this custom enum."
enum PersonColumn {
  AGE @enum(value: "age")
  TYPE @enum(value: "type")
  HAIRCOLOUR @enum(value: "haircolour")
  HEIGHT @enum(value: "height")
}
```

Lighthouse will still automatically generate the necessary input types.
Instead of creating enums for the allowed columns, it will simply use the existing `PersonColumn` enum.

It is recommended to either use the `columns` or the `columnsEnum` argument.
When you don't define any allowed columns, clients can specify arbitrary column names as a `String`.
This approach should by taken with care, as it carries
potential performance and security risks and offers little type safety.

A simple query for a person who is exactly 42 years old would look like this:

```graphql
{
  people(where: { column: AGE, operator: EQ, value: 42 }) {
    name
  }
}
```

Note that the operator defaults to `EQ` (`=`) if not given, so you could
also omit it from the previous example and get the same result.

The following query gets actors over age 37 who either have red hair or are at least 150cm:

```graphql
{
  people(
    where: {
      AND: [
        { column: AGE, operator: GT, value: 37 }
        { column: TYPE, value: "Actor" }
        {
          OR: [
            { column: HAIRCOLOUR, value: "red" }
            { column: HEIGHT, operator: GTE, value: 150 }
          ]
        }
      ]
    }
  ) {
    name
  }
}
```

Some operators require passing lists of values - or no value at all. The following
query gets people that have no hair and blue-ish eyes:

```graphql
{
  people(
    where: {
      AND: [
        { column: HAIRCOLOUR, operator: IS_NULL }
        { column: EYES, operator: IN, value: ["blue", "aqua", "turquoise"] }
      ]
    }
  ) {
    name
  }
}
```

Using `null` as argument value does not have any effect on the query.
This query would retrieve all persons without any condition:

```graphql
{
  people(where: null) {
    name
  }
}
```

### @whereHasConditions

```graphql
"""
Allows clients to filter a query based on the existence of a related model, using
a dynamically controlled `WHERE` condition that applies to the relationship.
"""
directive @whereHasConditions(
  """
  The Eloquent relationship that the conditions will be applied to.

  This argument can be omitted if the argument name follows the naming
  convention `has{$RELATION}`. For example, if the Eloquent relationship
  is named `posts`, the argument name must be `hasPosts`.
  """
  relation: String

  """
  Restrict the allowed column names to a well-defined list.
  This improves introspection capabilities and security.
  Mutually exclusive with `columnsEnum`.
  """
  columns: [String!]

  """
  Use an existing enumeration type to restrict the allowed columns to a predefined list.
  This allows you to re-use the same enum for multiple fields.
  Mutually exclusive with `columns`.
  """
  columnsEnum: String

  """
  Reference a method that applies the client given conditions to the query builder.

  Expected signature: `(
      \Illuminate\Database\Query\Builder|\Illuminate\Database\Eloquent\Builder $builder,
      array<string, mixed> $whereConditions
  ): void`

  Consists of two parts: a class name and a method name, separated by an `@` symbol.
  If you pass only a class name, the method name defaults to `__invoke`.
  """
  handler: String = "\\Nuwave\\Lighthouse\\WhereConditions\\WhereConditionsHandler"
) on ARGUMENT_DEFINITION
```

This directive works very similar to [@whereConditions](#whereconditions), except that
the conditions are applied to a relation sub query:

```graphql
type Query {
  people(
    hasRole: _ @whereHasConditions(columns: ["name", "access_level"])
  ): [Person!]! @all
}

type Role {
  name: String!
  access_level: Int
}
```

Again, Lighthouse will auto-generate an `input` and `enum` definition for your query:

```graphql
"Dynamic WHERE conditions for the `hasRole` argument of the query `people`."
input QueryPeopleHasRoleWhereConditions {
  "The column that is used for the condition."
  column: QueryPeopleHasRoleColumn

  "The operator that is used for the condition."
  operator: SQLOperator = EQ

  "The value that is used for the condition."
  value: Mixed

  "A set of conditions that requires all conditions to match."
  AND: [QueryPeopleHasRoleWhereConditions!]

  "A set of conditions that requires at least one condition to match."
  OR: [QueryPeopleHasRoleWhereConditions!]
}

"Allowed column names for the `hasRole` argument of the query `people`."
enum QueryPeopleHasRoleColumn {
  NAME @enum(value: "name")
  ACCESS_LEVEL @enum(value: "access_level")
}
```

A simple query for a person who has an access level of at least 5, through one of
their roles, looks like this:

```graphql
{
  people(hasRole: { column: ACCESS_LEVEL, operator: GTE, value: 5 }) {
    name
  }
}
```

You can also query for relationship existence without any condition; simply use an empty object as argument value.
This query would retrieve all persons that have a role:

```graphql
{
  people(hasRole: {}) {
    name
  }
}
```

Just like with the [@whereCondition](../api-reference/directives.md#whereconditions) directive, using `null` as argument value does not have any effect on the query.
This query would retrieve all persons, no matter if they have a role or not:

```graphql
{
  people(hasRole: null) {
    name
  }
}
```

## Custom operator

If Lighthouse's default `SQLOperator` does not fit your use case, you can register a custom operator class.
This may be necessary if your database uses different SQL operators then Lighthouse's default,
or you want to extend/restrict the allowed operators.

First create a class that implements `Nuwave\Lighthouse\WhereConditions\Operator`. For example:

```php
namespace App\GraphQL;

use Nuwave\Lighthouse\WhereConditions\Operator;

final class CustomSQLOperator implements Operator { ... }
```

An `Operator` has two responsibilities:

- provide an `enum` definition that will be used throughout the schema
- handle client input and apply the operators to the query builder

To tell Lighthouse to use your custom operator class, you have to bind it in a service provider:

```php
namespace App\GraphQL;

use App\GraphQL\CustomSQLOperator;
use Illuminate\Support\ServiceProvider;
use Nuwave\Lighthouse\WhereConditions\Operator;

final class GraphQLServiceProvider extends ServiceProvider
{
    public function register(): void
    {
        $this->app->bind(Operator::class, CustomSQLOperator::class);
    }
}
```

Don't forget to [register your new service provider in Laravel](https://laravel.com/docs/providers#registering-providers).
Make sure to add it after Lighthouse's service provider:

```diff
    \Nuwave\Lighthouse\WhereConditions\WhereConditionsServiceProvider::class,
+   \App\GraphQL\GraphQLServiceProvider::class,
```

## Custom handler

If you want to take advantage of the schema generation that [@whereConditions](#whereconditions)
and [@whereHasConditions](#wherehasconditions) provide, but customize the application of arguments
to the query builder, you can provide a custom handler.

```graphql
type Query {
  people(
    where: _ @whereConditions(columns: ["age"], handler: "App\\MyCustomHandler")
  ): [Person!]! @all
}
```

When a client passes `where`, your handler will be called with the query builder and
the passed conditions:

```php
namespace App;

use Illuminate\Database\Eloquent\Builder;

final class MyCustomHandler
{
    /**
     * @param  array<string, mixed>  $whereConditions
     */
    public function __invoke(Builder $builder, array $whereConditions): void
    {
        // TODO make calls to $builder depending on $whereConditions
    }
}
```
