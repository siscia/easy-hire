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

Liquid is been developed Tobias Luetke by and it is extensively used in Shopify.

Liquid is also been adopted as standard template engine for Jekyll, a stateless blog engine that is used in the github's page, this same blog use Liquid.

Before to dive into the detail of Liquid itself let's explore what this wonderfully small piece of software does and why it is successfully.

Liquid provide a safe environment, this means that I can render on Liquid templates that I don't trust.

Liquid is stateless, once I parse the template the rendering process is a completely different deal.

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

The scope of `Liquid::Template.parse(source)` is pretty limited, it takes a (long) string, a template, as input and create an in-memory representation of such templates.

Now we are focus on understand how this is accomplished.

### Template.parse(source, options)

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

The second argument is self explaining, it is a simple way to pass common and default options to the parse.

The first argument however is a little more complex and we are going to analyze this one first.

### tokenize(source)

```ruby
def tokenize(source)
      source = source.source if source.respond_to?(:source)
      return [] if source.to_s.empty?
      tokens = calculate_line_numbers(source.split(TemplateParser))
      tokens
    end
```

Also this function is simple to analyze.

At first the source is split by a regular expression into smaller string, the regular expression will catch variable and tag initialization.

You can see that the regular expression is quite complex, however is made out of smaller expression, so it is easier to write and to understand.

{% raw %}
```ruby
TagStart                    = /\{\%/
TagEnd                      = /\%\}/
VariableStart               = /\{\{/
VariableEnd                 = /\}\}/
VariableIncompleteEnd       = /\}\}?/
AnyStartingTag              = /\{\{|\{\%/
PartialTemplateParser = /#{TagStart}.*?#{TagEnd}|#{VariableStart}.*?#{VariableIncompleteEnd}/om
TemplateParser = /(#{PartialTemplateParser}|#{AnyStartingTag})/om
irb> a = "foo {{bar}} {%start%} something {%end%}"
irb> a.split(Liquid::TemplateParser)
=> ["foo ", "{{bar}}", " ", "{%start%}", " something ", "{%end%}"]
```
{% endraw %}

Now that we have splitted the source in more manageable little string those little string are used to create `Token` in `calculate_line_numbers(["source"])`

{% raw %}
<div class="hidden-br">
  <br> <br> <br> <br> <br> <br>
</div>
{% endraw %}

### calculate\_line\_numbers(["tokens"])

``` ruby
def calculate_line_numbers(raw_tokens)
  return raw_tokens unless @line_numbers
  current_line = 1
  raw_tokens.map do |token|
    Token.new(token, current_line).tap do
	  current_line += token.count("\n")
	end
  end
end
```
Also this function is extremely small, the function is nothing more that a loop in the input vector, for any element into the vector in creates a `Token` and return a vector of those tokens.

While the function loop it also keep count of what line we are in the source file, such information will be helpful in case of debug.

Now, finally we have understand what is the arguments passed to `Document.parse(tokens, options)`.

Let's see what this method does.

### Document.parse(tokens, options)

```ruby
class Document < BlockBody
  def self.parse(tokens, options)
    doc = new
	doc.parse(tokens, options)
	doc
  end
  def parse(tokens, options)
	super do |end_tag_name, end_tag_params|
	  unknown_tag(end_tag_name, options) if end_tag_name
	end
  end
```

You can see that this method does not do anything interesting, however we can note that Document is a subclass of BlockBody, and we also see that in the parse method we call super, so we are expecting to see the real parse implementation in the BlockBody class.

`BockBody.parse` will return a couple of string (`end_tag_name` and `end_tag_params`) if it cannot parse a particular tag, the function `unknown_tag` will simply raise an exception if such event occur.

### BlockBody.parse(token, option)
``` ruby
def parse(tokens, options)
  while token = tokens.shift
    begin
	  unless token.empty?
	    case
		  when token.start_with?(TAGSTART)
            ...
		  end
		  when token.start_with?(VARSTART)
		    ...
		  else
		    ...
		  end
        end
	  rescue SyntaxError => e
	    e.set_line_number_from_token(token)
		raise
      end
    end
    yield nil, nil
  end
end
```

The code of this function is pretty long, but it is pretty easy to follow.

We have one big loop that scan every single token.

It analyzes the token, and if the token is not empty there are three different path that can be taken.

When the token starts like a tag, when the token starts like a variable and when the token is simple text.

The loop also listen for errors, if an error occur information about the line of the error are added and finally the error is raised.

If in the loop everything is fine, the loop yield a couple of nil.

<div class="hidden-br">
  <br> <br> 
</div>

#### parse of tags

```ruby
when token.start_with?(TAGSTART)
  if token =~ FullToken
    tag_name = $1
	markup = $2
	  if tag = Template.tags[tag_name]
	    markup = token.child(markup) if token.is_a?(Token)
		new_tag = tag.parse(tag_name, markup, tokens, options)
		new_tag.line_number = token.line_number if token.is_a?(Token)
		@blank &&= new_tag.blank?
		@nodelist << new_tag
	  else
	    return yield tag_name, markup
	  end
  else
	raise_missing_tag_terminator(token, options)
  end
```

The most difficult parse is the parse of a tag.

The very first thing is to be sure that we are parsing a valid token, if, the token is not valid an exception is raised immediately.

If the token is valid we extract the name of token and the body/markup inside the tag itself.

We try to recognize the tag of the token, if we cannot recognize the tag we return from the function and an error will be raised.

If we are able to recognize the tag we analyze the whole token and we save the result.

Looking carefully thought the code you will notice that the parsing of the body inside a tag is recursive, basically we will repeat this same procedure.

#### parse of vars

``` ruby
when token.start_with?(VARSTART)
  new_var = create_variable(token, options)
  new_var.line_number = token.line_number if token.is_a?(Token)
  @nodelist << new_var
  @blank = false
else
```

The parse of a variable is a little simpler, `create_variable` simply create a new variable.

The variable will also analyze the filters that are applied, all this is done with an extensive use of regular expression.

#### simple text tokens

```ruby
else
  @nodelist << token
  @blank &&= !!(token =~ /\A\s*\z/)
end
```

This is the simplest possible way, the token is simply push into the list of the nodes.

[liquid-github]: https://github.com/Shopify/liquid
