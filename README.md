TranslationBundle
=================

A Symfony2 bundle for translating Doctrine2 entities ![Travis build status](https://secure.travis-ci.org/matteosister/TranslationBundle.png)

[How to install](https://github.com/matteosister/TranslationBundle/blob/master/Resources/doc/Installation.md)

WYSIWYG
-------

```php
<?php
$book = new Cypress\MyBundle\Entity\Book();
// setters
$book->setTitle('the lord of the rings');
$book->setTitleEs('el señor de los anillos');
$book->setTitleIt('il signore degli anelli');
// getters
$book->getTitle();
$book->getTitleEs();
// etc...
```

In your twig templates

```html+jinja
<h1>{{ book|translate('title') }}</h1>
<h1>{{ book|translate('title', 'es') }}</h1>
```

Configuration
-------------

Let's assume you have a **Book** entity with a title property

```php
<?php
namespace Cypress\MyBundle\Entity;

use Doctrine\ORM\Mapping as ORM;

/**
 * Books
 *
 * @ORM\Entity
 * @ORM\Table(name="book")
 */
class Book
{
    /**
     * @var integer
     *
     * @ORM\Id
     * @ORM\Column(type="integer")
     * @ORM\GeneratedValue(strategy="AUTO")
     */
    private $id;

    /**
     * @var string
     *
     * @ORM\Column
     */
    private $title;

    // constructor, getter, setter and others amenities...
}
```

In order to translate it:

* create the BookTranslations class (pick the name you want), and make it extends the TranslationEntity superclass. You have to define the $object property, which has a ManyToOne relation with your main book class

```php
<?php
namespace Cypress\MyBundle\Entity;

use Cypress\TranslationBundle\Entity\Base\TranslationEntity;
use Doctrine\ORM\Mapping as ORM;

/**
 * @ORM\Entity
 * @ORM\Table(name="book_translations",
 *     uniqueConstraints={@ORM\UniqueConstraint(name="lookup_unique_idx", columns={
 *         "locale", "object_id", "field"
 *     })}
 * )
 */
class BookTranslations extends TranslationEntity
{
    /**
     * @var Book
     *
     * @ORM\ManyToOne(targetEntity="Cypress\MyBundle\Entity\Book", inversedBy="translations")
     * @ORM\JoinColumn(onDelete="CASCADE")
     */
    protected $object;
}

```

the sensible parts that you'll probably want to change is: the **namespace**, the **table name**, the **index name** and the **target entity**.

Do not change the inversedBy attribute! And, yes, this is your class, but do not add properties here, do it in the main class!

* add the **TranslatableEntity** superclass to your Book entity, define a **translations** property with a OneToMany relation with the translations entity, and implement the three abstract methods:

```php
<?php
namespace Cypress\MyBundle\Entity;

use Doctrine\ORM\Mapping as ORM;
use Cypress\TranslationBundle\Entity\Base\TranslatableEntity;

/**
 * Books
 *
 * @ORM\Entity
 * @ORM\Table(name="book")
 */
class Book extends TranslatableEntity
{
    /**
     * @var integer
     *
     * @ORM\Id
     * @ORM\Column(type="integer")
     * @ORM\GeneratedValue(strategy="AUTO")
     */
    private $id;

    /**
     * @var string
     *
     * @ORM\Column
     */
    protected $title;

    /**
     * @var string
     *
     * @ORM\OneToMany(targetEntity="Cypress\MyBundle\Entity\BookTranslations", mappedBy="object", cascade={"all"})
     */
    protected $translations;

    // constructor, getter, setter and others amenities...

    /**
     * get the name of the TranslationEntity
     *
     * @return mixed
     */
    public function getTranslationEntity()
    {
        return 'Cypress\MyBundle\Entity\BookTranslations';
    }

    /**
     * get the default language
     *
     * @return string
     */
    public function getDefaultLanguage()
    {
        return 'en';
    }

    /**
     * get an array of supported languages
     *
     * @return array
     */
    public function getOtherLanguages()
    {
        return array('it', 'es');
    }
}
```

**getTranslationEntity**: return a string with the fully qualified name of the translation entity

**getDefaultLanguage**: return a string with the two digit code of the main language

**getOtherLanguages**: return an array with the two digit codes of the other languages

**CAREFUL!!**: you need to set the properties that has a translation as *protected* and not *private*, or no party.

* Rebuild you model

```sh
$ ./app/console doctrine:schema:update --force
```

**Important**

If your translatable entity contains a constructor remember to call the parent constructor or set the translations property to an empty ArrayCollection. For example:

```php
<?php
class Book extends TranslatableEntity
{
    // properties

    public function __contruct()
    {
        // call the parent constructor
        parent::__construct();
        // or alternatively, set the translations property
        $this->translations = new ArrayCollection();
        // your logic ...
    }
}
```

**You're done!**

MongoDB
-------

read the docs [about MongoDB odm here](https://github.com/matteosister/TranslationBundle/blob/master/Resources/doc/mongo.md)

Usage
-----

```php
<?php
$book = new Cypress\MyBundle\Entity\Book();
$book->setTitle('the lord of the rings'); // default language defined in getDefaultLanguage()
$book->setTitleEn('the lord of the rings'); // same as before
$book->setTitleEs('el señor de los anillos'); // set the title in spanish
$book->setTitleIt('il signore degli anelli'); // guess?
$book->setTitleRu('some weird letters here'); // throws an exception!

$em->persist($book); // $em is a doctrine entity manager
$em->flush(); // if you WTF on this go read the doctrine docs... :)

// now retrieve
echo $book->getTitle(); // the lord of the rings
echo $book->getTitleEn(); // the lord of the rings
echo $book->getTitleIt(); // il signore degli anelli...
// and so on...
```

You can use any naming convention for your properties, underscore and camelCase, as long as you define a getter/setter for the property

Twig
----

In your twig templates you can use a nice filter

```html+jinja
{% for book in books %}
    <h1>{{ book|translate('title') }}</h1>
    <p>{{ book|translate('description') }}</p>
{% endfor %}
```

Remember to apply the filter directly to the TranslatableEntity instance, and to set the property name as the filter argument

By default, twig gets the language from the actual environment (from the session in sf 2.0 and from the request in sf 2.1), but you can also force the language with twig, just pass the two digit code as the second argument of the filter

```html+jinja
<h1>{{ book|translate('title') }}</h1>
<h2>spanish translation: {{ book|translate('title', 'es') }}</h2>
<h2>italian translation: {{ book|translate('title', 'it') }}</h2>
```

In some case, you don't know what is the method to call on the object. For example a menu where the voices are different kind of objects with "title" and "name" as the main translated property. In this cases you could also pass an array as the property name. The first that match wins.

```html+jinja
<ul>
    {% for menu_voice in menu_voices %}
    <li>{{ menu_voice|translate(['title', 'name'], 'es') }}</li>
    {% endfor %}
</ul>
```

If you don't use twig add this to your configuration file:

```yml
cypress_translation:
    twig: false
```

Sonata
------

**this bundle works great with sonata admin bundle** Just name the properties in your admin class

```php
<?php
namespace Sonata\NewsBundle\Admin;
use Sonata\AdminBundle\Admin\Admin;

class TagAdmin extends Admin
{
    protected function configureFormFields(FormMapper $formMapper)
    {
        $formMapper
            ->add('title')
            ->add('title_it', 'text')
            ->add('title_es', 'text')
            ->add('myAwesomeDescription_es', 'text')
        ;
    }
}
```

You only need to define the field type, as sonata is not able to guess the type on a non-existent property

Careful
-------

Use a 2 digit code for your languages. Like "en", "it" or "es".

"en_US" **DO NOT WORK!**

Testing
-------

This bundle is unit tested with phpunit. Here is the [travis build page](http://travis-ci.org/#!/matteosister/TranslationBundle)
