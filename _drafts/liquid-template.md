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

## The parser

### Overview

The scope of `Liquid::Template.parse(source)` is pretty limited, it takes a (long) string, a template, as input and create an in-memory rappresentation of such templates.

Now we are focus on understand how this is accomplished.

### .parse(source, options)

``` ruby
def parse(source, options = {})
      @options = options
      @profiling = options[:profile]
      @line_numbers = options[:line_numbers] || @profiling
      @root = Document.parse(tokenize(source), DEFAULT_OPTIONS.merge(options))
      @warnings = nil
      self
    end
```

The function simply get the options and some other parameter, finally it sets `@root = Document.parse(tokenize(source), DEFAULT_OPTIONS.merge(options))` here is were the real job is done.

The method return self so it will be easy to chain other calls.

Let's focus a little bit more on the input of `Document.parse()`.

The second argument is self explaning, it is a simple way to pass common and default options to the parse.

The first argument however is a little more comple and we are going to analyze this one first.

### tokenize(source)

```ruby
def tokenize(source)
      source = source.source if source.respond_to?(:source)
      return [] if source.to_s.empty?
      tokens = calculate_line_numbers(source.split(TemplateParser))
      # removes the rogue empty element at the beginning of the array
      tokens.shift if tokens[0] and tokens[0].empty?
      tokens
    end
```

Also this function is pretty simple to analyze

[liquid-github]: https://github.com/Shopify/liquid
