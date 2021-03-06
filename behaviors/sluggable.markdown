---
layout: documentation
title: Sluggable Behavior
---

# Sluggable Behavior #

The `sluggable` behavior allows a model to offer a human readable identifier that can be used for search engine friendly URLs.

## Basic Usage ##

In the `schema.xml`, use the `<behavior>` tag to add the `sluggable` behavior to a table:

```xml
<table name="post">
  <column name="id" required="true" primaryKey="true" autoIncrement="true" type="INTEGER" />
  <column name="title" type="VARCHAR" required="true" primaryString="true" />
  <behavior name="sluggable" />
</table>
```

Rebuild your model, insert the table creation sql again, and you're ready to go. The model now has an additional getter for its slug, which is automatically set before the object is saved:

```php
<?php
$p1 = new Post();
$p1->setTitle('Hello, World!');
$p1->save();
echo $p1->getSlug(); // 'hello-world'
```

By default, the behavior uses the string representation of the object to build the slug. In the example above, the `title` column is defined as `primaryString`, so the slug uses this column as a base string. The string is then cleaned up in order to allow it to appear in a URL. In the process, blanks and special characters are replaced by a dash, and the string is lowercased.

>**Tip**<br />The slug is unique by design. That means that if you create a new object and that the behavior calculates a slug that already exists, the string is modified to be unique:

```php
<?php
$p2 = new Post();
$p2->setTitle('Hello, World!');
$p2->save();
echo $p2->getSlug(); // 'hello-world-1'
```

The generated model query offers a `findOneBySlug()` method to easily retrieve a model object based on its slug:

```php
<?php
$p = PostQuery::create()->findOneBySlug('hello-world');
```

## Parameters ##

By default, the behavior adds one column to the model. If this column is already described in the schema, the behavior detects it and doesn't add it a second time. The behavior parameters allow you to use custom patterns for the slug composition. The following schema illustrates a complete customization of the behavior:

```xml
<table name="post">
  <column name="id" required="true" primaryKey="true" autoIncrement="true" type="INTEGER" />
  <column name="title" type="VARCHAR" required="true" primaryString="true" />
  <column name="url" type="VARCHAR" size="100" />
  <behavior name="sluggable">
    <parameter name="slug_column" value="url" />
    <parameter name="slug_pattern" value="/posts/{Title}" />
    <parameter name="replace_pattern" value="/[^\w\/]+/u" />
    <parameter name="replacement" value="-" />
    <parameter name="separator" value="/" />
    <parameter name="permanent" value="true" />
    <parameter name="scope_column" value="" />
  </behavior>
</table>
```

Whatever `slug_column` name you choose, the `sluggable` behavior always adds the following proxy methods, which are mapped to the correct column:

```php
<?php
$post->getSlug();         // returns $post->url
$post->setSlug($slug);    // $post->url = $slug
```

The `slug_pattern` parameter is the rule used to build the raw slug based on the object properties. Any substring enclosed between brackets '{}' is turned into a getter, so the `Post` class generates slugs as follows:

```php
<?php
protected function createRawSlug()
{
  return '/posts/' . $this->getTitle();
}
```

Incidentally, that means that you can use names that don't match a real column phpName, as long as your model provides a getter for it.

The `replace_pattern` parameter is a regular expression that shows all the characters that will end up replaced by the `replacement` parameter. In the above example, special characters like '!' or ':' are replaced by '-', but not letters, digits, nor '/'.

The `separator` parameter is the character that separates the slug from the incremental index added in case of non-unicity. Set as '/', it makes `Post` objects sharing the same title have the following slugs:

```text
'posts/hello-world'
'posts/hello-world/1'
'posts/hello-world/2'
...
```

A `permanent` slug is not automatically updated when the fields that constitute it change. This is useful when the slug serves as a permalink, that should work even when the model object properties change. Note that you can still manually change the slug in a model using the `permanent` setting by calling `setSlug()`;

## Further Customization ##

The slug is generated by the object when it is saved, via the `createSlug()` method. This method does several operations on a simple string:

```php
<?php
protected function createSlug()
{
  // create the slug based on the `slug_pattern` and the object properties
  $slug = $this->createRawSlug();
  // truncate the slug to accomodate the size of the slug column
  $slug = $this->limitSlugSize($slug);
  // add an incremental index to make sure the slug is unique
  $slug = $this->makeSlugUnique($slug);

  return $slug;
}

protected function createRawSlug()
{
  // here comes the string composition code, generated according to `slug_pattern`
  $slug = 'posts/' . $this->cleanupSlugPart($this->getTitle());
  // cleanupSlugPart() cleans up the slug part
  // based on the `replace_pattern` and `replacement` parameters

  return $slug;
}
```

You can override any of these methods in your model class, in order to implement a custom slug logic.
