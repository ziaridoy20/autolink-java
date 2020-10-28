autolink-java
=============

Java library to extract links such as URLs and email addresses from plain text.
Fast, small and smart about recognizing where links end.

Inspired by [Rinku](https://github.com/vmg/rinku). Similar to it, regular
expressions are not used. Instead, the input text is parsed in one pass with
limited backtracking.

This library requires at least Java 7 (tested up to Java 11). It works on Android (minimum API level 15). It has no external dependencies.

[Demo](https://onecompiler.com/java/3vjubz6sz)

Maven coordinates
(see
[here](https://search.maven.org/artifact/org.nibor.autolink/autolink/0.10.0/jar)
for other build systems):

```xml
<dependency>
    <groupId>org.nibor.autolink</groupId>
    <artifactId>autolink</artifactId>
    <version>0.10.0</version>
</dependency>
```

[![ci](https://github.com/robinst/autolink-java/workflows/ci/badge.svg)](https://github.com/robinst/autolink-java/actions?query=workflow%3Aci)
[![Coverage status](https://codecov.io/gh/robinst/autolink-java/branch/main/graph/badge.svg)](https://codecov.io/gh/robinst/autolink-java)
[![Maven Central status](https://img.shields.io/maven-central/v/org.nibor.autolink/autolink.svg)](https://search.maven.org/search?q=g:org.nibor.autolink%20AND%20a:autolink&core=gav)


Usage
-----

Extract links:

```java
import org.nibor.autolink.*;

String input = "wow, so example: http://test.com";
LinkExtractor linkExtractor = LinkExtractor.builder()
        .linkTypes(EnumSet.of(LinkType.URL, LinkType.WWW, LinkType.EMAIL))
        .build();
Iterable<LinkSpan> links = linkExtractor.extractLinks(input);
LinkSpan link = links.iterator().next();
link.getType();        // LinkType.URL
link.getBeginIndex();  // 17
link.getEndIndex();    // 32
input.substring(link.getBeginIndex(), link.getEndIndex());  // "http://test.com"
```

Note that by default all supported types of links are extracted. If
you're only interested in specific types, narrow it down using the
`linkTypes` method.

There's another method which is convenient for when you want to transform
all of the input text to something else. Here's an example of using that
to transform the text to HTML and wrapping URLs in an `<a>` tag (escaping
is done using owasp-java-encoder):

```java
import org.nibor.autolink.*;
import org.owasp.encoder.Encode;

String input = "wow http://test.com such linked";
LinkExtractor linkExtractor = LinkExtractor.builder()
        .linkTypes(EnumSet.of(LinkType.URL)) // limit to URLs
        .build();
Iterable<Span> spans = linkExtractor.extractSpans(input);

StringBuilder sb = new StringBuilder();
for (Span span : spans) {
    String text = input.substring(span.getBeginIndex(), span.getEndIndex());
    if (span instanceof LinkSpan) {
        // span is a URL
        sb.append("<a href=\"");
        sb.append(Encode.forHtmlAttribute(text));
        sb.append("\">");
        sb.append(Encode.forHtml(text));
        sb.append("</a>");
    } else {
        // span is plain text before/after link
        sb.append(Encode.forHtml(text));
    }
}

sb.toString();  // "wow <a href=\"http://test.com\">http://test.com</a> such linked"
```

Note that this assumes that the input is plain text, not HTML.
Also see the "What this is not" section below.

Features
--------

### URL extraction

Extracts URLs of the form `scheme://example` with any potentially valid scheme.
URIs such as `example:test` are not matched (may be added as an option in the
future). If only certain schemes should be allowed, the result can be filtered.
(Note that schemes can contain dots, so `foo.http://example` is recognized as
a single link.)

Includes heuristics for not including trailing delimiters such as punctuation
and unbalanced parentheses, see examples below.

Supports internationalized domain names (IDN). Note that they are not validated
and as a result, invalid URLs may be matched.

Example input and linked result:

* `http://example.com.` → [http://example.com]().
* `http://example.com,` → [http://example.com](),
* `(http://example.com)` → ([http://example.com]())
* `(... (see http://example.com))` → (... (see [http://example.com]()))
* `https://en.wikipedia.org/wiki/Link_(The_Legend_of_Zelda)` →
  [https://en.wikipedia.org/wiki/Link_(The_Legend_of_Zelda)]()
* `http://üñîçøðé.com/` → [http://üñîçøðé.com/]()

Use `LinkType.URL` for this, and see [test
cases here](src/test/java/org/nibor/autolink/AutolinkUrlTest.java).

### WWW link extraction

Extract links like `www.example.com`. They need to start with `www.` but
don't need a `scheme://`. For detecting the end of the link, the same
heuristics apply as for URLs.

Examples:

* `www.example.com.` → [www.example.com]().
* `(www.example.com)` → ([www.example.com]())
* `[..] link:www.example.com [..]` → \[..\] link:[www.example.com]() \[..\]

Not supported:

* Uppercase `www`'s, e.g. `WWW.example.com` and `wWw.example.com`
* Too many or too few `w`'s, e.g. `wwww.example.com`

The domain must have at least 3 parts, so `www.com` is not valid, but `www.something.co.uk` is.

Use `LinkType.WWW` for this, and see [test
cases here](src/test/java/org/nibor/autolink/AutolinkWwwTest.java).

### Email address extraction

Extracts emails such as `foo@example.com`. Matches international email
addresses, but doesn't verify the domain name (may match too much).

Examples:

* `foo@example.com` → [foo@example.com]()
* `foo@example.com.` → [foo@example.com]().
* `foo@example.com,` → [foo@example.com](),
* `üñîçøðé@üñîçøðé.com` → [üñîçøðé@üñîçøðé.com]()

Not supported:

* Quoted local parts, e.g. `"this is sparta"@example.com`
* Address literals, e.g. `foo@[127.0.0.1]`

Note that the domain must have at least one dot (e.g. `foo@com` isn't
matched), unless the `emailDomainMustHaveDot` option is disabled.

Use `LinkType.EMAIL` for this, and see [test cases
here](src/test/java/org/nibor/autolink/AutolinkEmailTest.java).

What this is not
----------------

This library is intentionally *not* aware of HTML. If it was, it would need to depend on an HTML parser and renderer.
Consider this input:

```
HTML that contains <a href="https://one.example">links</a> but also plain URLs like https://two.example.
```

If you want to turn the plain links into `a` elements but leave the already linked ones intact, I recommend:

1. Parse the HTML using an HTML parser library
2. Walk through the resulting DOM and use autolink-java to find links within *text* nodes only
3. Turn those into `a` elements
4. Render the DOM back to HTML

Contributing
------------

See CONTRIBUTING.md file.

License
-------

Copyright (c) 2015-2018 Robin Stocker and others, see Git history

MIT licensed, see LICENSE file.
