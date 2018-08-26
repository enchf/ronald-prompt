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

gem 'ronald_prompt', '~> x.x', '>= x.x.x'

Gemspec:

spec.add_runtime_dependency 'ronald_prompt', '~> x.x', '>= x.x.x'
```

Require the library in code:

```ruby
require 'ronald_prompt'
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
  import  'from/file'       # Or from file, the last one is set.
  prompt 'Type <enter>...'  # Override enter prompt.
  
  execute do
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
  accept 'Yes'
  
  execute do
    puts 'It was not skipped'
  end
end
```

In console, this will be shown as:

```
$ This one can be skipped! 
  Do you want to proceed? Yes
  It was not skipped

$ This one can be skipped! 
  Do you want to proceed? N
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
* Validations to be performed. Separate calls to 'validate' method will add a validation.
* Value prompt.
* Storage key, an identifier to enable getting the value to future steps.
* If no key is provided, value is basically not stored and not available in future steps.
* The type of the value, which by default is 'string'.
* Valid types are primitives in Ruby: integer, float, string, boolean.
* Also, the invalid type error message can be set within the type, and by default is 'Invalid type'. 

```ruby
class Value < RonaldPrompt::Value
  message 'This one requires a value!'
  prompt  'Enter an integer value between 1 and 10'   # Override value prompt.
  key     'one.value'                                 # Storage key.
  type    :integer, 'Value needs to be an integer'    # Override type and error message.
  
  validate('Value must be between 1 and 10') { |value| (1..10).include? value }
  
  execute do
    puts "In the step 'value' method provides you the input: #{value}."
    puts "In the whole application, 'RonaldPrompt::provide' does: #{RonaldPrompt.provide('one.value')}."
  end
end
```

In console, this will be shown as:

```
$ This one requires a value! 
  Enter an integer value between 1 and 10: No
  Value needs to be an integer.
  Enter an integer value between 1 and 10: 11
  Value must be between 1 and 10.
  Enter an integer value between 1 and 10: 5
  In the step 'value' method provides you the input: 5 
  In the whole application, 'RonaldPrompt::provide' does: 5.
```

#### Modify Step

This step can be seen as a mix of skip and value step.
The step provides a default value, which is taken if the step is skipped.
How the 'step is skipped'/'default value taken'? Leaving value prompt empty.
Otherwise it follows the value step flow.

```
- Show message and default value.
- Prompt asking for a value.
- Wait for user input to continue.
- Mark step as skip if input is empty, taking default value as step value.
- Otherwise, validate user input.
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
* If no key is provided, value is basically not stored and not available in future steps.
* Default value if step is skipped.

```ruby
class DefaultValue < RonaldPrompt::Modify
  message 'This has a default value!'
  prompt  'Enter an integer'
  key     'value.key'
  default 10                          # Automatically detects that type is :integer.
                                      # Validates that default value is one of the four allowed.
  
  execute do
    puts "The final value is: #{value}."
  end
end
```

In console, this will be shown as:

```
$ This has a default value! 
  Enter an integer:
  The final value is 10
```

#### Select Step

This step basically allows you to select an item from a list.
It is basically a Value step, with the possible values as the indexes of the options shown.
The flow is as follows:

```
- Show message.
- Show options.
- Prompt asking for the index of the option to be selected.
- Wait for user input to continue.
- Validate index.
- Show error message if index is invalid.
- If index is not valid, prompt again for an option (return to step 2).
- If index is valid, mark step as :done.
```

Code definitions:

* Welcome message (string or from file).
* Action to be performed.
* Index prompt. By default is 'Select an option (0 - <last index on the list>)'.
* Storage key, an identifier to enable getting the value to future steps.
* If no key is provided, value is basically not stored and not available in future steps.
* Invalid index message.
* The options we can select.

```ruby
class Options < RonaldPrompt::Select
  message 'This one selects a value!'
  prompt  'We are overriding the index prompt'
  key     'selected.value'
  options %w[Ruby Java Python Go]
  error   'Invalid index message here.'
  
  execute do
    puts "RonaldPrompt.provide or value methods return the selected value: #{RonaldPrompt.provide('selected.value')}."
  end
end
```

In console, this will be shown as:

```
$ This one selects a value!
  |---|--------|
  | 0 | Ruby   |
  | 1 | Java   |
  | 2 | Python |
  | 3 | Go     |
  |---|--------|
  
  We are overriding the index prompt: 4
  Invalid index message here.

  |---|--------|
  | 0 | Ruby   |
  | 1 | Java   |
  | 2 | Python |
  | 3 | Go     |
  |---|--------|
  
  We are overriding the index prompt: 0
  RonaldPrompt.provide or value methods return the selected value: Ruby.
```

#### List Step

This step is basically the flow when you add/remove elements on a list.
It is a composition of two Select and one Value steps.
Select step is used twice, one to select the add/remove/finish option and another one to remove a value.
Value step is used when adding values to the list.
After an add/remove action is executed, it returns to the 'action menu'. 
The list step flow is as follows:

```
- Show message.
- Show list status.
- Prompt menu options, by default is 'add (1), remove (2), finish (3)'.
- If 'add', prompt for value.
- Validate value.
- If valid, execute 'add action', otherwise show validation error messages.
- Return to step 2.
- If 'remove', prompt an index of the element to remove. By default is 'Give element index to remove'.
- Validate index, if valid, remove, otherwise, show index not found error message.
- Return to step 2.
- If 'finish', execute action and mark step as :done. 
```

Code definitions:

* Welcome message (string or from file).
* Menu prompt, this means, the way we ask for action (add/remove/finish).
* Invalid action message, by default is 'Select a valid option'.
* Add prompt, prompt requesting a value to be added.
* Value validations.
* Action to be (optionally) executed after add.
* The way in which the values added are displayed, by default uses `.to_s` method.
* Storage key for the list. If not provided the list is not available in future steps.
* Remove Prompt, prompt requesting index of the element to be removed.
* Action to be (optionally) executed after remove.
* The 'index not found' message.
* Final action to be performed after the step is finished.

```ruby
class NameList < RonaldPrompt::List
  message 'This will fill a list!'
  key     'Key for the list'
  
  options.prompt 'This is the menu prompt (add (1), remove(2), finish(3))'
  options.error  'This is shown if an invalid option is selected.'
  
  add.prompt  'Enter a name less or equal than 10 characters'
  add.display { |name| name.upcase }   # Show value uppercase.
  add.validate('Name must be less or equal than 10 characters.') { |name| name.length <= 10 }
  
  add.execute do
    puts "The value added was #{last_value}."    # Last item is accessed using last_value method.
  end
  
  remove.prompt 'Indexes to remove items starts from 0'
  remove.error  'Indexes must be valid, as shown in values table.'
  remove.execute do
    puts "Here also works last_value: #{last_value}"
  end
  
  execute do
    puts "The final value is: [ #{value.join(',')} ]."
  end
end
```

In console, this will be shown as:

```
$ This will fill a list!
  |---|-----------------------|
  |   |                       |
  |---|-----------------------|
  
  This is the menu prompt (add (1), remove(2), finish(3)): 4
  This is shown if an invalid option is selected.
  This is the menu prompt (add (1), remove(2), finish(3)): 1
  Enter a name less than 10 characters: Longeeeername
  Name must be less or equal than 10 characters.
  Enter a name less than 10 characters: Jose
  The value added was Jose.
  
  |---|-----------------------|
  | 0 | JOSE                  |
  |---|-----------------------|
  
  This is the menu prompt (add (1), remove(2), finish(3)): 1
  Enter a name less than 10 characters: Maria
  The value added was Maria.
    
  |---|-----------------------|
  | 0 | JOSE                  |
  | 1 | MARIA                 |
  |---|-----------------------|
  
  This is the menu prompt (add (1), remove(2), finish(3)): 2
  Indexes to remove items starts from 0: 2
  Indexes must be valid, as shown in values table.
  
  |---|-----------------------|
  | 0 | JOSE                  |
  | 1 | MARIA                 |
  |---|-----------------------|
    
  This is the menu prompt (add (1), remove(2), finish(3)): 2
  Indexes to remove items starts from 0: 0
  Here also works last_value: Jose
  
  |---|-----------------------|
  | 0 | MARIA                 |
  |---|-----------------------|
    
  This is the menu prompt (add (1), remove(2), finish(3)): 3
  The final value is [ Maria ].
```

#### Group Step

## Usage

## Authors

* **Ernesto** - Gem Creator - [enchf](https://github.com/enchf)

## License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details
