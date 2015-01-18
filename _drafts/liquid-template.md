---
layout: tech
title: Liquid
root: "../"
---


# Liquid template engine

## Abstract

In this series we are going to explore the [liquid][liquid-github] template engine.

I will start with a little background about Liquid and then we will analyze how the engine works.

We are going to analyze how the parsing system works and how filters and tags are implemented.

## Liquid Overview

``` ruby
irb> require 'liquid'
irb> greets = Liquid::Template.parse("Ciao {{ name }}")
irb> greets.render('name' => 'OpenHire')
=> "Ciao OpenHire"
```

Liquid is been developed Tobias Luetke by and it is extensivily used in Shoppify.

Liquid is also been adopted as stadard template engine for Jekyll, a stateless blog engine that is used in the github's page, this same blog use Liquid.

Before to dive into the detail of Liquid itself let's explore what this wonderfully small piece of software does and why it is sucesfull.

Liquid provide a safe environmet, this means that I can render on Liquid templates that I don't trust.

Liquid is stateless, once I parse the template the rendering process is a completelly different deal.

## Simple analysis

``` ruby
irb> greets = Liquid::Template.parse("Ciao {{ name | capitalize}} # filter
                                      {% for item in {{array}} %} # tags
                                        Item: {{item}}
                                      {% endfor %}")
irb> greets.render('name' => 'simone', 'array' => ['Open', 'Hire'])
=> "Ciao Simone # the name here start with capital S
    Item: Open
    Item: Hire"
irb> greets.render('name' => 'simone', 'array' => [1, 2])
=> "Ciao Simone
    Item: 1
    Item: 2"
```

Liquid is extremely powerful, other than simple output substitution it provides filters and tags to make the designer job a little easier.

`tags` are the logic of the markup, so we would find loops (for) and branches (if and cases).

`filters` modify the text passed as argument.

Let's try to analyze what the template engines need to do in order to perform all the operation you can see.

The very first step is to parse the text, this operation is farily expensive and need to be done only once, so Liquid will remember the result of the parse.

After the text is been parsed, it is the moment to render the final string.

In order to render the final string we need to:

+ substitute every simple output.

+ run the filters

+ run the tags



[liquid-github]: https://github.com/Shopify/liquid
