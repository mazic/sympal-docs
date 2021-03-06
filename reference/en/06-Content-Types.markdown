# What is a Content Type?

On the web, every page of a site contains a different set of information.
And while this information is unique, most sites can be divided into a finite
number of "page types". For example, a restaurant site may consist of basic
information pages, blog posts, and pages describing their menu.

<div style="text-align:center;">
  [asset:documentation-1-0-page-types]
</div>

Sympal's approach to this problem is no different than your normal dynamic
web application. The three different "page types" would be represented by
three different models in the database. For example, these page types could
be represented by the following models.

<div style="text-align:center;">
  [asset:documentation-1-0-content-types]
</div>

Each model would be setup to hold all of the content needed to render that
type of page. This is a common and age-old strategy that is given the
name "content types" inside sympal. The `sfSympalMenuPage`, for example,
might look like this:

    [yml]
    sfSympalMenuPage:
      actAs: [sfSympalContentTypeTemplate]
      columns:
        id:
        title:       string(255)
        hours:       string(255)
        description: clob
      relations:
        Dishes:
          class:    sfSympalMenuDish
          foreign:  menu_id
          onDelete: cascade

    sfSympalMenuDish:
      columns:
        id:
        name:        string(255)
        description: clob
        price:       double

Besides the `actAs: [sfSympalContentTypeTemplate]` line, which will be
explained below, content types are nothing more than normal models that
are used to render a page on your website.

In a later chapter, we'll talk about how to create one or more templates
to render each content (page) type. In these templates, you'll have access
to your content type model (e.g. `sfSympalMenuPage`) in the same way that
you would in a normal symfony application.

# Creating New Types

As sympal relies heavily on plugins, new content types are commonly placed
in their own plugin. In fact, sympal makes this very easy to do. To generate
a new plugin with schema ready for your content type, run the following task:

    $ php symfony sympal:plugin-generate Article --content-type=Article

> **NOTE**
> The above will generate a plugin named `sfSympalArticlePlugin` and will 
> contain a default schema for a new `Article` content type.

## Default Schema

A content type is simple. The only requirement is that you enable the model
behavior named, `sfSympalContentType`. If you take a look at the generated
code in the `config/doctrine/schema.yml` file of the new plugin, it will
look like the following:

    [yml]
    Article:
      actAs: [sfSympalContentType]
      columns:
        title: string(255)
        body: clob

The addition of the `sfSympalContentType` behavior adds an additional
column, `content_id` along with a one-to-one foreign relation with the
`sfSympalContent` model. This relationship forms the basis for how all
content is rendered inside sympal.

In fact, as you'll see below, these two objects will act almost as one
object. Any method called on `sfSympalContent` will be automatically
called on the related content model unless it exists on `sfSympalContent`.
For example, if you call `getHours()` on an `sfSympalContent` object of
type `sfSympalMenuPage`, the value of the `hours` field on `sfSympalMenuPage`
will be returned.

    [php]
    $content->getHours();

# Understanding `sfSympalContent` and Content Types

In a normal symfony application, each "page" is represented by a route,
which directs symfony to a module and action that will render the template
for that page. While this is still possible in sympal, most pages are
rendered through a slightly chain of events.

The root of all content in sympal is the `sfSympalContent` model. Unless
you've decided to create some normal symfony routes as described above,
every page on your site will be represented by one row in the `sfSympalContent`
model.

## The role of `sfSympalContent` 

Each `sfSympalContent` record in your database represents one page (unique
url) on your site. When a request comes in to your site, the following
chain takes place:

1. A unique `sfSympalContent` record is retrieved based on the incoming url
1. The content record is joined to the appropriate content type record
   (e.g. `sfSympalMenuPage`) via the one-to-one relationship established
   via the `sfSympalContentType` Doctrine behavior.
1. The correct template is rendered and the `sfSympalContent` is supplied
   to the template.
   
While the above is a simplification, the point is that the `sfSympalContent`
record does little more than help to determine the unique url for a page
and join over to the content type model.

When the template is rendered, a the `sfSympalContent` record is available
via a variable called `$content`.

1. Add a `content_id` foreign key
2. Relates `sfSympalContent` and `Article` with a one-to-one relationship.
3. Adds sharing of properties/methods between the related records.
4. Handles automatic creation of the relationship when creating new records.

It allows us to do things like this with our code:

    [php]
    $content = sfSympalContent::createNew('Article');
    $content->title = 'Test article';
    $content->body = 'Body of my test article!';
    $content->site_id = 1;
    $content->save();

> **NOTE**
> Notice above how we are able to access and manipulate the properties of 
> Article. When we call `save()` it will automatically take of saving and 
> relating the `Content` and `Article` models together through the `content_id`
> foreign key.

## Default Installation

You can install a Sympal plugin and the content type it bundles by running the
following command.

    $ php symfony sympal:plugin-install Article
  
When you run the installation process for a Sympal plugin and it contains a 
content type then a few things occur.

1. New content type record is created.
2. A sample content type record is created.
3. A sample content type list is created.
4. A menu item is added pointing to the content list.

The items performed above make the plugin instantly available and useable in 
your site. You can begin adding new content of that type through the normal
Sympal content interface.

## Customizing Schema

When we generated the plugin above and told it to generate a new content type 
named `Article`, we left the schema as it was with only a `title` and `body`.

This works for our demonstration purposes but often you will want to customize 
this schema and make it fit the needs of your content type. So lets modify the 
`Article` schema and customize it a bit.

    [yml]
    Article:
      actAs: [sfSympalContentType]
      columns:
        category:
          type: enum
          values: [Tutorial, Rant, News]
        title: string(255)
        excerpt: string(500)
        body: clob

Now if you re-run the installation process for the plugin it will re-install
the plugin with the changes you made.

    $ php symfony sympal:plugin-install Article

## Customizing Installation

When you create a new content type, we try and automate the installation 
initially but this is just to help you get started fast. You can easily 
customize the installation by creating a custom install class named after
your plugin.

Creating a new file in `sfSympalArticlePlugin/lib` named
`sfSympalArticlePluginInstall` that extends `sfSympalPluginManagerInstall`.

> **NOTE**
> We are extending the base installation class that gets used if you run the
> install for a plugin that does not have a custom install class.

    [php]
    class sfSympalArticlePluginInstall extends sfSympalPluginManagerInstall
    {
      public function customInstall($installVars)
      {
        
      }
    }

You will notice above how the `customInstall()` method accepts a variable
that is an array containing the unsaved objects that would have been inserted
in to the database had you not created the `customInstall()` method. Now
it is up to  the developer of the plugin what to insert and what other code
to perform when a plugin is installed.

First lets customize the default `Article` content record that is inserted
when  installed. We want to set the `category` and `excerpt` properties.

> **NOTE**
> The `customInstall()` method can also be placed in the configuration
> for the plugin. In this example you could place it in the
> `sfSympalArticlePluginConfiguration` class in `sfSympalArticlePlugin/config`.

    [php]
    class sfSympalArticlePluginInstall extends sfSympalPluginManagerInstall
    {
      public function customInstall($installVars)
      {
        // Customize the sample content record then save
        $installVars['content']->category = 'Tutorial';
        $installVars['content']->excerpt = 'This is the excerpt from the 
          sample article.';
        $installVars['content']->save();
      }
    }

Now we'll add the code to save some of the default records that we don't
need to modify before saving.

    [php]
    class sfSympalArticlePluginInstall extends sfSympalPluginManagerInstall
    {
      public function customInstall($installVars)
      {
        // ...

        // Save menu item that points to the content list
        $this->saveMenuItem($installVars['menuItem']);

        // Save the default content type record
        $installVars['contentType']->save();
      }
    }

Now we can re-run our install for the `Article` plugin and our `customInstall()`
method will be invoked.

## Default Uninstall

When you run the uninstall process for a Sympal plugin and it contains a
content type then all related records in the database will be deleted. This
means the content type record, templates, menu items, and any actual content
records. After all data has been deleted then any tables the plugin introduce
will be dropped from the database.

    php symfony sympal:plugin-uninstall Article

You can also pass the `--delete` option to also delete all files and remove
the plugin from your project entirely.

## Customizing Uninstall

Customizing the uninstall process of a Sympal plugin is the same as the
install. You just need to create a class named `sfSympalArticlePluginUninstall`
in the same location as the install class.

    [php]
    class sfSympalArticlePluginUninstall extends sfSympalPluginManagerUninstall
    {
      public function customUninstall()
      {
        // perform some custom uninstall code
      }
    }

> **NOTE**
> Just like the `customInstall()`, the `customUninstall()` method can be
> placed in your `ProjectConfiguration` class.

Now if you run the uninstall process for the `Article` plugin, the
`customUninstall()` method will be invoked.

    $ php symfony sympal:plugin-uninstall Article