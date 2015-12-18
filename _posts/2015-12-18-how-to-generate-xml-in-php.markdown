---
layout: post
title:  "How to generate XML in PHP (in a painless way)"
date:   2015-12-18 12:23
---

I have always avoided working with XML. Parsing and writing it always felt too difficult. I thought
that way until I found an actually easy and manageable way of working with it. I will focus on the
process of generating XML documents using PHP.

As many developers out there, I started using fixed templates with placeholders. The following
example will look familiar to many of you:

{% highlight php %}
<?php
function generatePayload($username)
{
    return <<<XML
<?xml version="1.0" encoding="UTF-8"?>
<user xmlns="http://myns.org/">
    <username>$username</username>
    <permissions>
        <create />
        <delete />
        <update />
    </permission>
</user>
XML;
}
{% endhighlight %}

This approach works as long as your XML documents do not change too much.

What happens when the structure of the generated document has to change depending on certain
input and multiple namespaces are involved? How do you collect used namespaces? How do you add new
fragments programatically?

It starts getting complex, and using templates is not a solution anymore.

Using [DOM](http://php.net/manual/es/book.dom.php) is certainly an option:

{% highlight php %}
<?php
function generatePayload($username, array $permissions)
{
    $document = new \DOMDocument('1.0', 'UTF-8');
    // Should be turned off in production
    $document->formatOutput = true;
    $user = $document->createElementNS('http://myns.org', 'user');
    $username_elem = $document->createElementNS(
        'http://myns.org',
        'username',
        $username
    );
    $user->appendChild($username_elem);

    $permissions_elem = $document->createElementNS(
        'http://myns.org',
        'permissions'
    );

    foreach ($permissions as $permission) {
        $current_permission = $document->createElementNS(
            'http://myns.org',
            $permission
        );
        $permissions_elem->appendChild($current_permission);
    }

    $user->appendChild($permissions_elem);
    $document->appendChild($user);

    return $document->saveXML();
{% endhighlight %}

It's definitely an improvement over the first approach, but there are still some drawbacks:

1. Nested elements requires creating subelements and nesting them one by one
2. Adding elements with different namespaces requires working with namespaces and tag names
   separately
3. You have to be extra careful with namespace prefixes. `\DOMDocument` creates its own (ugly)
   prefixes by default, but it also accepts prefixing the tag name with your own custom prefix
   (danger zone!)

I came across multiple problems when generating XML documents for [AgenDAV](http://agendav.org/). Under
the hood, a lot of XML has to be generated (and parsed, but that's another story), and dealing with
multiple namespaces was starting to be difficult. Nesting elements required lots of code and started
to get confusing too.

Fortunately I found a solution to all my XML writing issues: the [sabre/xml](http://sabre.io/xml/)
package. Its author is a much respected PHP developer, [Evert Pot](https://evertpot.com/), who
produces really high quality packages that I use on some of my projects.

Using sabre/xml to write XML documents
--------------------------------------

[sabre/xml](http://sabre.io/xml/) XML generation is based on
[XMLWriter](http://php.net/manual/en/book.xmlwriter.php), which is part of PHP core since 5.1.2.
This package extends the native `XMLWriter` class so it does a better job.

Before starting with sabre/xml, let me introduce [Clark notation](http://sabre.io/xml/clark-notation/)
to you. In my opinion, sabre/xml is awesome thanks to supporting it.

Clark notation is a way of writing an XML element including both the namespace and the tag name in
just one string. The namespace goes before the tag name wrapped in braces. So the following string
in Clark notation:

`{http://myns.org/}/tag`

Can be understood as:

{% highlight xml %}
<tag xmlns="http://myns.org/" />
{% endhighlight %}

That means you have not worry anymore about namespace prefixes and their definitions.
sabre/xml will let you specify fully namespaced elements while doing the difficult parts by itself.
Isn't that great?

Installing sabre/xml is really simple using composer:

{% highlight console %}
% composer require sabre/xml '~1.2.0'
{% endhighlight %}

Let's rewrite our example:

{% highlight php %}
<?php
function generatePayload($username, array $permissions) {
    $writer = new Sabre\Xml\Writer();
    $writer->openMemory();
    $writer->startDocument('1.0', 'UTF-8');
    // Should be turned off in production
    $writer->setIndent(true);

    $writer->startElement('{http://myns.org/}/user');
    $writer->write([ '{http://myns.org/}/user' => $username ]);

    $writer->startElement('{http://myns.org/}/permissions');
    foreach ($permissions as $permission) {
        $writer->write([ $permission => []]);
    }
    $writer->endElement();

    $writer->endElement();

    return $writer->outputMemory();
}
{% endhighlight %}

Pretty self explaining, I hope. Writing XML like this lets you do it sequentially, opening and
closing XML elements as you go. It even handles namespaces and prefixes.

You can see we have used two different syntaxes:

1. `startElement($element)` and `endElement()`
2. `write()`

There [some other ways of doing it using sabre/xml](http://sabre.io/xml/writing/), but I found these
two to be more than enough for most cases.

As it handles Clark notation, we can now do call the `generatePayload()` like this:

{% highlight php %}
<?php
echo generatePayload('jorge', [
   '{http://myns.org/}/read',
   '{http://myns.org/}create',
   '{http://another-ns.com}list'
]);
{% endhighlight %}

Let's see the output:

{% highlight xml %}
<?xml version="1.0" encoding="UTF-8"?>
<x1:user xmlns:x1="http://myns.org">
 <x1:user xmlns:x1="http://myns.org">jorge</x1:user>
 <x1:permissions xmlns:x1="http://myns.org">
  <x1:read xmlns:x1="http://myns.org"/>
  <x1:create xmlns:x1="http://myns.org"/>
  <x2:list xmlns:x2="http://another-ns.com"/>
 </x1:permissions>
</x1:user>
{% endhighlight %}

While it is valid XML, we can improve it by providing `Sabre\Xml\Writer` with some frequently used
namespaces in our document and a custom prefix. We could just add the following after the
`openMemory()` call:

{% highlight php %}
<?php
// ...
$writer->namespaceMap = [
   'http://myns.org/' => 'm',
];
{% endhighlight %}

The XML document generated would be much nicer:

{% highlight xml %}
<?xml version="1.0" encoding="UTF-8"?>
<m:user xmlns:m="http://myns.org/">
 <m:user>jorge</m:user>
 <m:permissions>
  <m:read/>
  <m:create/>
  <x1:list xmlns:x1="http://another-ns.com"/>
 </m:permissions>
</m:user>
{% endhighlight %}

We can even decouple the generation of the `<permissions>` block from the original function:

{% highlight php %}
<?php

function generatePayload($username, array $permissions) {
   // ...
   addPermissions($writer, $permissions);
   // ...
}

function addPermissions(Sabre\Xml\Writer $writer, array $permissions)
{
    $writer->startElement('{http://myns.org/}permissions');
    foreach ($permissions as $permission) {
        $writer->write([ $permission => []]);
    }
    $writer->endElement();
}
{% endhighlight %}

After discovering sabre/xml, my relationship with XML has changed, and my code is now cleaner.

sabre/xml describes itself as _An XML library for PHP you may not hate_, and I agree. Not only do I
hate it, but I certainly love it!
