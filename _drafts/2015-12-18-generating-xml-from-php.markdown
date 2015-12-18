---
layout: post
title:  "How to generate XML from PHP (in a painless way)"
date:   2015-12-18 12:23
---

I have always avoided working with XML. Reading and writing it always felt complex to me. As many
developers out there, I started using fixed templates with placeholders, as in the following
example:

{% highlight php %}
<?php
function generatePayload($username)
{
    return <<<XML
<?xml version="1.0" encoding="UTF-8"?>
<user xmlns="http://myns.org">
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

This approach works as long your XML documents do not change too much.

What happens when the structure of the generated document has to change depending on certain
input and multiple namespaces are involved? This is getting complex, and using templates is not
a solution anymore. How do you collect used namespaces? How do you add new fragments
programatically?

Using [DOMDocument](http://php.net/manual/es/class.domdocument.php) is certainly an option:

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

1. Nesting elements requires creating subelements and nesting them, one by one
2. Adding elements with different namespaces requires working with namespaces and tag names
   separately
3. You have to be extra careful with namespace prefixes. `\DOMDocument` creates its own (ugly)
   prefixes by default, but it also accepts prefixing the tag name with your own custom prefix

I came across multiple problems when generating XML documents for [AgenDAV](http://agendav.org/). Under
the hood, a lot of XML has to be generated (and parsed, but that's another story), and dealing with
multiple namespaces was starting to be difficult.

Fortunately I found a solution to all my XML writing issues: the [sabre/xml](http://sabre.io/xml/)
package. Its author is a much respected PHP developer, [Evert Pot](https://evertpot.com/), who
produces really high quality code that I use a lot on some of my projects.

Using sabre/xml to write XML documents
--------------------------------------


