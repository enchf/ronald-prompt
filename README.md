# Ronald Prompt

Ronald Prompt is a Wizard helper, chaining Prompts who collect data from Terminal. 
Ronald simplifies the creation of a Terminal Wizard.
As a sample, see [Gemerator](https://github.com/enchf/gemerator) to see in code how to apply Ronald Prompt patterns.
Or, read this document who tries to explain it. 

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

Steps are also thought as independent beings. They don't know anything about other steps.
As they are sequentially, they can modify without concurrent worries the state of the information they are collecting.

So how the steps are interconnected? Through a configuration file in YAML format.
In this way we can decouple steps and flow from code.

### Step Types

All the steps have some actions and a result of execution.
Actions include show message information, prompts and executing custom actions related to the step.
By default, every step is considered `done` after `action` method is executed.
The possible step results are `done`, `skip` and `fail`.

Now, see the step patterns Ronald Prompt has identified.

#### Message Step

This is a step which only shows a message, and waits for `<enter>` to continue.
The flow of this step is:

```
- Show message.
- Prompt message.
- Wait for user signal to continue.
- Execute an action, usually nothing.
- Mark step as :done 
``` 

So, what in code I need to define?

* Assign a message (directly or from file)
* Optionally define an action.
* Override the 'continue' prompt (that which ask to press enter to continue).
By default, that message is `Press <enter> to continue`.

```ruby
class Welcome < RonaldPrompt::Message
  message 'Welcome!'        # Direct String
  load_message 'from/file'  # Or from file, the last one is set.
  
  prompt 'Type <enter>...'  # Override enter prompt.
  
  def action
    puts 'Do something'
  end
end
```

In console, this will be shown as:

```
$ Welcome! 
  Type <enter>...
  Do something
```

#### Skip Step

This is a step that can be skipped. A prompt will appear asking if you want to execute or skip the step.
Inherits the same methods as message, but also you can (optionally) define a skip prompt.
To keep it simple, we can see the Skip Step as a Message Step that can be skipped.

The flow of this step is:

```
- Show message.
- Prompt asking whether to skip or not. 
- Wait for user signal to continue.
- Execute an action if step is not skipped.
- Mark step as :done or :skip.
``` 

Code definitions:

* Welcome message (string or from file).
* Action to be performed.
* Skip prompt. The default is: `Do you want to execute the step? (Y/N)`
* Yes response by default is `Y`. It can be overriden using `accept_value`.
* No response is always anything other than the positive response, to keep it simple.

```ruby
class CanBeSkipped < RonaldPrompt::Skip
  message 'This one can be skipped!'  # Direct String
  prompt 'Do you want to proceed?'    # Override enter prompt.
  accept_value `Yes`
  
  def action
    puts 'It was not skipped'
  end
end
```

In console, this will be shown as:

```
$ This one can be skipped! 
  Do you want to proceed? Yes
  It was not skipped
```

#### Value Step

This is a step that can requires input value. A prompt will appear asking for a valid value, and cannot be skipped.
But it can be seen as a skip step that instead of a Yes/No answer expects a value.
Inherits the same methods as message, but also you can (optionally) define a value prompt.
Also, you can define validations using a `validate block`, passing a `predicate` as argument.

The flow of this step is:

```
- Show message.
- Prompt asking for a value.
- Wait for user input to continue.
- Validate user input.
- Show validation failures if any.
- If input is not valid, prompt again for a value (return to substep 2).
- If input is valid, mark step as :done.
``` 

Code definitions:

* Welcome message (string or from file).
* Action to be performed.
* Validations to be performed.
* Value prompt.
* Storage key, an identifier to enable getting the value to future steps.

```ruby
class Value < RonaldPrompt::Value
  message 'This one requires a value!'             # Direct String
  prompt 'Enter an integer value between 1 and 10' # Override enter prompt.
  storage_key 'one.value'
  
  validate('Value needs to be an integer') { |value| value.to_s == value.to_i.to_s }
  validate('Value must be between 1 and 10') { |value| (1..10).include? value }
  
  def action
    puts "In the step 'value' method provides you the input: #{value}."
    puts "In the whole application, 'RonaldPrompt::provide' does: #{RonaldPrompt.provide('one.value')}."
  end
end
```

In console, this will be shown as:

```
$ This one requires a value! 
  Enter an integer value between 1 and 10: No
  Value needs to be an integer. Enter an integer value between 1 and 10: 11
  Value must be between 1 and 10. Enter an integer value between 1 and 10: 5
  In the step 'value' method provides you the input: 5 
  In the whole application, 'RonaldPrompt::provide' does: 5.
```

## Usage

## Authors

* **Ernesto** - Gem Creator - [enchf](https://github.com/enchf)

## License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details
