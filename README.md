# Ronald Prompt

Ronald Prompt is a Wizard helper, chaining and linking prompts collecting data from Terminal.
This library simplifies the creation of a Terminal Wizard.
As a sample, see [Gemerator](https://github.com/enchf/gemerator) to see in code how to apply Ronald Prompt patterns.
Anyway, this document will try to explain it. 

## Getting Started

### Installing

Include the gem in your gemspec or Gemfile, check [RubyGems](http://rubygems.org/) for the latest version.

```
Gemfile:

gem 'ronald-prompt', '~> x.x', '>= x.x.x'

Gemspec:

spec.add_runtime_dependency 'ronald-prompt', '~> x.x', '>= x.x.x'
```

## Main Concepts

Ronald Prompt sees a Terminal Wizard as a sequence of `steps`.
Some steps can be skipped (`optional ones`), some other, depending on the answer, redirect to another step.
There are some fixed steps: `start`, `finish` and `fail`. At any step we can redirect to them.

Steps are also thought as independent steps. They don't know anything about other steps.
As they are sequentially, they can modify a global state of information, which in fact they are collecting.

So how the steps are interconnected? Through a configuration file in YAML format.
In this way we can decouple steps and flow from code.

### Step Types

There are below step types identified so far.

#### Message Step

This is a step which only shows a message, and waits for `<enter>` to continue.
The basic flow of this step is:

```
- Show message.
- Wait for user signal to continue.
- Execute an action, usually nothing.
- Mark step as :done 
``` 

So, what in code I need to define?
Assign a message (directly or from file), and (optionally) define an action.
Also you can override the 'continue' prompt (that which ask to press enter to continue).
By default, that message is `Press <enter> to continue`.

```ruby
class Welcome < RonaldPrompt::Message
  message 'Welcome!'        # Direct String
  load_message 'from/file' # Or from file, the last one is set.
  
  prompt 'Presiona <enter> para continuar...' # Override enter prompt.
  
  def action
    puts 'Do something'
  end
end
```

In console, this will be shown as:

```
$ Welcome! 
$ Presiona <enter> para continuar...
```

#### Skip Step

This is a step that can be skipped. A prompt will appear asking if you want to execute or skip the step.
Inherits the same methods as message, but also you can (optionally) define a skip prompt.

## Usage

## Authors

* **Ernesto** - *Gem Creator* - [enchf](https://github.com/enchf)

## License

This project is licensed under the MIT License - see the [LICENSE.md](LICENSE.md) file for details
