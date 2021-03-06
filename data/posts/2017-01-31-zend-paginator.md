---
layout: post
title: Paginating data collections with zend-paginator
date: 2017-01-31T11:15:00-06:00
author: Enrico Zimuel
url_author: http://www.zimuel.it
permalink: /blog/2017-01-31-zend-paginator.html
categories:
- blog
- components
- zend-paginator

---

[zend-paginator](https://docs.zendframework.com/zend-paginator/) is a flexible
component for paginating collections of data and presenting that data to users.

[Pagination](https://en.wikipedia.org/wiki/Pagination) is a standard UI solution
to manage the visualization of lists of items, like a list of posts in a blog
or a list of products in an online store.

zend-paginator is very popular among Zend Framework developers, and it's often
used with [zend-view](https://docs.zendframework.co/zend-view/), thanks to
the pagination control view helper zend-view provides.

It can be used also with other template engines. In this article, I will
demonstrate how to use it with [Plates](http://platesphp.com/).

## Usage of zend-paginator

The component can be installed via [composer](https://getcomposer.org/), using
the following command:

```bash
$ composer require zendframework/zend-paginator
```

To consume the paginator component, we need a collection of items.
zend-paginator ships with several different adapters for common collection
types:

- *ArrayAdapter*, which works with PHP arrays;
- *Callback*, which allows providing callbacks for obtaining counts of items and lists of items;
- *DbSelect*, to work with a SQL collection (using [zend-db](https://docs.zendframework.com/zend-db/));
- *DbTableGateway*, to work with a Table Data Gateway (using the TableGateway
  feature from zend-db.
- *Iterator*, to work with any [Iterator](http://php.net/manual/en/class.iterator.php) instance.

If your collection does not fit one of these adapters, you can create a custom
adapter. To do so, you will need to implement
`Zend\Paginator\Adapter\AdapterInterface`, which defines two methods:

- `count() : int`
- `getItems(int $offset, int $itemCountPerPage) : array`

Each adapter need to return the total number of items in the collection,
implementing the `count()` method, and a portion (a *page*) of items starting
from `$offset` position with a size of `$itemCountPerPage` per page.

With these two methods we can use zend-paginator with any type of collection.

For instance, imagine we need to paginate a collection of blog posts and we
have a `Posts` class that manage all the posts. We can implement an adapter
like this:

```php
require 'vendor/autoload.php';

use Zend\Paginator\Adapter\AdapterInterface;
use Zend\Paginator\Paginator;
use Zend\Paginator\ScrollingStyle\Sliding;

class Posts implements AdapterInterface
{
    protected $posts = [];

    public function __construct()
    {
      // Read posts from file/database/whatever
    }

    public function count()
    {
        return count($this->posts);
    }

    public function getItems($offset, $itemCountPerPage)
    {
        return array_slice($this->posts, $offset, $itemCountPerPage);
    }
}

$posts = new Posts();
$paginator = new Paginator($posts);

Paginator::setDefaultScrollingStyle(new Sliding());
$paginator->setCurrentPageNumber(1);
$paginator->setDefaultItemCountPerPage(8);

foreach ($paginator as $post) {
  // Iterate on each post
}

$pages = $paginator->getPages();
var_dump($pages);
```

In this example, we created a zend-paginator adapter using a custom `Posts`
class. This class stores the collection of posts using a protected array
(`$posts`). This adapter is then passed to an instance of `Paginator`.

When creating a `Paginator`, we need to configure its behavior.
The first setting is the scrolling style. In the example above, we used the
[Sliding](https://github.com/zendframework/zend-paginator/blob/master/src/ScrollingStyle/Sliding.php)
style, a Yahoo!-like scrolling style that positions the current page number as
close as possible to the center of the page range.

![Scrolling style](/img/blog/usage-rendering-control.png)

> Note: the `Sliding` scrolling style is the default style used by zend-paginator. We need
> to set it explicitly using `Paginator::setDefaultScrollingStyle()` only if we
> do not use [zend-servicemanager](https://github.com/zendframework/zend-servicemanager)
> as a plugin manager. Otherwise, the scrolling style is loaded by default from
> the plugin manager.

The other two configuration values are the current page number and the number
of items per page. In the example above, we started from page 1, and we count 8
items per page.

We can then iterate on the `$paginator` object to retrieve the post of the
current page in the collection.

At the end, we can retrieve the information regarding the previous page, the
next page, the total items in the collection, and more. To get these values
we need to call the `getPages()` method. We will obtain an object like this:

```php
object(stdClass)#81 (13) {
  ["pageCount"]=>
  int(3)
  ["itemCountPerPage"]=>
  int(8)
  ["first"]=>
  int(1)
  ["current"]=>
  int(1)
  ["last"]=>
  int(3)
  ["next"]=>
  int(2)
  ["pagesInRange"]=>
  array(3) {
    [1]=>
    int(1)
    [2]=>
    int(2)
    [3]=>
    int(3)
  }
  ["firstPageInRange"]=>
  int(1)
  ["lastPageInRange"]=>
  int(3)
  ["currentItemCount"]=>
  int(8)
  ["totalItemCount"]=>
  int(19)
  ["firstItemNumber"]=>
  int(1)
  ["lastItemNumber"]=>
  int(8)
}
```

Using this information, we can easily build an HTML footer to navigate across
the collection.

> Note: using zend-view, we can consume the [paginationControl](https://docs.zendframework.com/zend-paginator/usage/#rendering-pages-with-view-scripts)
> helper, which emits an HTML pagination bar.

In the next section, I'll demonstrate using the [Plates](http://platesphp.com/)
template engine.

## An example using Plates

Plates implements templates using native PHP; it is fast and easy to use,
without any additional meta language; it is just PHP.

In our example, we will create a Plates template to paginate a collection of
data using zend-paginator. We will use [bootstrap](http://getbootstrap.com/) as
the UI framework.

For purposes of this example, blog posts will be accessible via the following URL:

```
/blog[/page/{page:\d+}]
```

where `[/page/{page:\d+}]` represents the optional page number (using the regexp
  `\d+` to validate only digits). If we open the `/blog` URL we will get the
first page of the collection. To return the second page we need to connect to
`/blog/page/2`, third page to `/blog/page/3`, and so on.

For instance, we can manage the page parameter using a PSR-7 middleware class
consuming the previous `Posts` adapter, that works as follow:

```php
use Psr\Http\Message\ResponseInterface;
use Psr\Http\Message\ServerRequestInterface;
use League\Plates\Engine;
use Zend\Paginator\Paginator;
use Zend\Paginator\ScrollingStyle\Sliding;
use Posts;

class PaginatorMiddleware
{
    /** @var Posts */
    protected $posts;

    /** @var Engine */
    protected $template;

    public function __construct(Posts $post, Engine $template = null)
    {
        $this->posts    = $post;
        $this->template = $template;
    }

    public function __invoke(
        ServerRequestInterface $request,
        ResponseInterface $response, callable $next = null
    ) {
        $paginator = new Paginator($this->posts);
        $page = $request->getAttribute('page', 1);

        Paginator::setDefaultScrollingStyle(new Sliding());
        $paginator->setCurrentPageNumber($page);
        $paginator->setDefaultItemCountPerPage(8);

        $pages = $paginator->getPages();
        $response->getBody()->write(
            $this->template->render('posts', [
                'paginator' => $paginator,
                'pages' => $pages,
            ])
        );
        return $response;
    }
}
```

We used a `posts.php` template, passing the paginator (`$paginator`) and
the pages (`$pages`) instances. That template could then look like the following:

```php
<?php $this->layout('template', ['title' => 'Blog Posts']) ?>

<div class="container">
  <h1>Blog Posts</h1>

  <?php foreach ($paginator as $post): ?>
    <div class="row">
      <?php // prints the post title, date, author, ... ?>
    </div>
  <?php endforeach; ?>

  <?php $this->insert('page-navigation', ['pages' => $pages]) ?>
</div>
```

The `page-navigation.php` template contains the HTML code for the page
navigation control, with button like previous, next, and page numbers.

```php
<nav aria-label="Page navigation">
  <ul class="pagination">
    <?php if (! isset($pages->previous)): ?>
      <li class="disabled"><a href="#" aria-label="Previous"><span aria-hidden="true">&laquo;</span></a></li>
    <?php else: ?>
      <li><a href="/blog/page/<?php echo $pages->previous ?>" aria-label="Previous"><span aria-hidden="true">&laquo;</span></a></li>
    <?php endif; ?>

    <?php foreach ($pages->pagesInRange as $num): ?>
      <?php if ($num === $pages->current): ?>
        <li class="active"><a href="/blog/page/<?php echo $num ?>"><?php echo $num ?> <span class="sr-only">(current)</span></a></li>
      <?php else: ?>
        <li><a href="/blog/page/<?php echo $num ?>"><?php echo $num ?></a></li>
      <?php endif; ?>
    <?php endforeach; ?>

    <?php if (! isset($pages->next)): ?>
      <li class="disabled"><a href="#" aria-label="Next"><span aria-hidden="true">&raquo;</span></a></li>
    <?php else: ?>
      <li><a href="/blog/page/<?php echo $pages->next ?>" aria-label="Next"><span aria-hidden="true">&raquo;</span></a></li>
    <?php endif; ?>
  </ul>
</nav>
```

## Summary

The zend-paginator component of Zend Framework is a powerful and easy to use
package that provides pagination of data. It can be used as standalone component
in many PHP projects using different frameworks and template engines. In this
article, I demonstrated how to use it in general purpose applications.
Moreover, I showed an example using Plates and Bootstrap, in a PSR-7 middleware
scenario.

Visit the [zend-paginator documentation](https://docs.zendframework.com/zend-paginator/)
to find out what else you might be able to do with this component!
