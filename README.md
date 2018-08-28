# Ronald Prompt

Ronald Prompt is a Wizard helper, chaining Prompts who collect data from Terminal. 
Ronald simplifies the creation of a Terminal Wizard.
As a sample, see [Gemerator](https://github.com/enchf/gemerator) to see in code how to apply Ronald Prompt patterns.
Or, read this document who tries to explain it. 

## Table of Contents

* [Getting Started](#getting-started)
  * [Installing](#installing)
* [Main Concepts](#main-concepts)
  * [Step Types](#step-types)
    * [Message Step](#message-step)
    * [Skip Step](#skip-step)
    * [Value Step](#value-step)
    * [Modify Step](#modify-step)
    * [Select Step](#select-step)
    * [List Step](#list-step)
    * [Group Step](#group-step)
  * [Configuration](#configuration)
* [Authors](#authors)
* [License](#license)

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

So, what in code I can define?

| Method  | Description                                      | Default Value  |
| ------- | ------------------------------------------------ | -------------- |
| message | Display message when launching step.             | (empty string) |
| import  | Import display message from a file.              | (empty string) |
| execute | Action to perform previous to finish the step.   | (empty method) |
| prompt  | Prompt message when asking to continue the step. | `Press <enter> to continue` |

Notes:

* All definitions are optional.
* Each call to message or import overrides previously assigned message. Last value is taken.

This is how it is seen in code:

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

| Method  | Description                                  | Default Value  |
| ------- | -------------------------------------------- | -------------- |
| message | Display message when launching step.         | (empty string) |
| import  | Import display message from a file.          | (empty string) |
| accept  | Value to avoid skipping the step.            | `Y`            |  
| execute | Action to perform if step is not skipped.    | (empty method) |
| prompt  | Prompt message when asking to skip the step. | `Do you want to execute the step? (Y/N)` |

Notes:

* All definitions are optional.
* Each call to message or import overrides previously assigned message. Last value is taken.

This is how it is seen in code:

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

| Method   | Description                                       | Default Value   |
| -------- | ------------------------------------------------- | --------------- |
| message  | Display message when launching step.              | (empty string)  |
| import   | Import display message from a file.               | (empty string)  |
| key      | Key to enable getting step value in future steps. | (no key)        |
| execute  | Action to perform after entering a valid value.   | (empty method)  |
| validate | Validations on input value, receives a block.     | (no validation) |
| prompt   | Prompt message when asking for a value.           | `Enter value`   |
| type     | Type and error message of the input value.        | `:string, 'Invalid type'` |

Notes:

* All definitions are optional.
* Validation for type is implicit. For example, entering NaN value for an :integer will show type error message.
* Valid types are primitives in Ruby: `:integer, :float, :string, :boolean`.
* Extra validations are executed sequentially as they are defined. Can have more than 1 validation.
* Each call to message or import overrides previously assigned message. Last value is taken.
* We can recover the input value in `execute` block using `value` function.
* In future steps, when providing a key, value can be retrieved using `RonaldPrompt.provide` method.
* If no key is provided, step value will not be available in future steps.

This is how it is seen in code:

```ruby
class Value < RonaldPrompt::Value
  message 'This one requires a value!'
  prompt  'Enter an integer value between 1 and 10'   # Override value prompt.
  key     :key_to_value                               # Storage key.
  type    :integer, 'Value needs to be an integer'    # Override type and error message.
  
  validate('Value must be between 1 and 10') { |value| (1..10).include? value }
  
  execute do
    puts "In the step 'value' method provides you the input: #{value}."
    puts "In the whole application, 'RonaldPrompt::provide' does: #{RonaldPrompt.provide(:key_to_value)}."
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

| Method   | Description                                       | Default Value   |
| -------- | ------------------------------------------------- | --------------- |
| message  | Display message when launching step.              | (empty string)  |
| import   | Import display message from a file.               | (empty string)  |
| key      | Key to enable getting step value in future steps. | (no key)        |
| execute  | Action to perform after entering a valid value.   | (empty method)  |
| validate | Validations on input value, receives a block.     | (no validation) |
| default  | Establish a default value. *This is required*     | None            |
| error    | Message error when input type validation fails.   | `Invalid type`  |
| prompt   | Prompt message when asking for a value.           | `Enter value (default = #default)` |

Notes:

* A default value is required for this step.
* Validates that default value type is one of the four allowed: `:integer, :float, :string, :boolean`.
* This step infers the input value type as per the default one.
* That means that input value is validated against default value type.
* In default prompt, the legend `(default = #default)` is always appended to enable user to know the default one.
* Extra validations are executed sequentially as they are defined. Can have more than 1 validation.
* Each call to message or import overrides previously assigned message. Last value is taken.
* We can recover the input value in `execute` block using `value` function.
* In future steps, when providing a key, value can be retrieved using `RonaldPrompt.provide` method.
* If no key is provided, step value will not be available in future steps.

This is how it is seen in code:

```ruby
class DefaultValue < RonaldPrompt::Modify
  message 'This has a default value!'
  prompt  'Enter an integer'
  error   'This appears if your input is a different type than the default value'
  default 10                          # Automatically detects that type is :integer.
  
  execute do
    puts "The final value is: #{value}."
  end
end
```

In console, this will be shown as:

```
$ This has a default value! 
  Enter an integer:
  The final value is 10.

$ This has a default value! 
  Enter an integer: 400
  The final value is 400.

$ This has a default value! 
  Enter an integer: NaN 
  This appears if your input is a different type than the default value
  Enter an integer: 42
  The final value is 42.
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

| Method   | Description                                       | Default Value   |
| -------- | ------------------------------------------------- | --------------- |
| message  | Display message when launching step.              | (empty string)  |
| import   | Import display message from a file.               | (empty string)  |
| key      | Key to enable getting step value in future steps. | (no key)        |
| execute  | Action to perform after selecting a value.        | (empty method)  |
| error    | Message error when index type validation fails.   | `Invalid index` |
| options  | Options we can select. *This is required*         | None            |
| prompt   | Prompt message when asking for an index.          | `Select an option (0 - <last index>)` |

Notes:

* The options to choose are required for this step.
* Input is validated to be a valid index on the list.
* Each call to message or import overrides previously assigned message. Last value is taken.
* We can recover the input value in `execute` block using `value` function.
* In future steps, when providing a key, value can be retrieved using `RonaldPrompt.provide` method.
* If no key is provided, step value will not be available in future steps.

This is how it is seen in code:

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

Code definitions here are separated in four:

* Definitions for the step per-se.

| Method   | Description                                       | Default Value   |
| -------- | ------------------------------------------------- | --------------- |
| message  | Display message when launching step.              | (empty string)  |
| import   | Import display message from a file.               | (empty string)  |
| key      | Key to enable getting the list in future steps.   | (no key)        |
| execute  | Action to perform after finishing the list.       | (empty method)  |

Also, it has 3 methods to enable the setup of the sub-steps.
See code sample to see them in action.

| Method   | Description       |
| -------- | ----------------- |
| select   | Object reference to set definitions when selecting an action.         | 
| add      | Object reference to set definitions when input values.                |
| remove   | Object reference to set definitions when removing values of the list. |

* Definitions for the options. It is seen as a Select Step with 3 options: `:add, :remove, :finish`.
Each of the options chain other steps: `:add` chains a Value step, and `:remove` a Select Step.
The option `finish` 'returns' to the outer step and finish it. So the definitions for this are:

| Method | Description                          | Default Value    |
| ------ | ------------------------------------ | ---------------- |
| prompt | Prompt message for an action option. | `Select an option: add (0), remove(1), finish (2)` |
| error  | When the option selected is invalid. | `Invalid option` |

* Definitions on the add action. The add action is basically a Value step but instead of loop until
a valid value is input, it returns to the main menu. It has all Value step methods except of `key`,
`input` and `message`, with the addition of `unique` setting which disables repeated values on the list.
If set to true, it validates input and shows an error message if a repeated value is tried to be added.
Also, the way in which the elements are shown in the list can be overriden with the `display` option.

| Method   | Description                                                | Default Value    |
| -------- | ---------------------------------------------------------- | ---------------- |
| execute  | Action to execute after an element is added.               | (empty method)   |
| validate | Validations on the added element.                          | (no validations) |
| prompt   | Prompt shown when requesting a value.                      | `Enter value`    |
| display  | The way in which the added elements are shown in Terminal. | Uses `.to_s`     |
| type     | Type of the elements on the list and invalid type message. | `:string, 'Invalid type'` | 
| unique   | True if the list accepts only unique elements.             | `false, 'Repeated element'` |

* Definitions on the remove action. This option is a Select step, having all current list elements as options.
The action performed on the Select step is to remove the element.
Also, a post-remove action can be set with `execute` method.

| Method   | Description                                       | Default Value    |
| -------- | ------------------------------------------------- | ---------------- |
| execute  | Action to execute after an element is removed.    | (empty method)   |
| error    | Error message shown if an invalid index is input. | `Invalid index`  |
| prompt   | Prompt shown when requesting an element index.    | `Select element` |

```ruby
class NameList < RonaldPrompt::List
  message 'This will fill a list!'
  key     'Key for the list'
  
  execute do
    puts "The final value is: [ #{value.join(',')} ]."
  end
  
  select.prompt 'This is the menu prompt (add (0), remove (1), finish (2))'
  select.error  'This is shown if an invalid option is selected.'
  
  add.prompt  'Enter a name less or equal than 10 characters'
  add.display { |name| name.upcase }   # Show value uppercase.
  add.validate('Name must be less or equal than 10 characters.') { |name| name.length <= 10 }
  add.unique true, 'Overriding error message if a repeated element is added.'
  
  add.execute do
    puts "The value added was #{last_value}."    # Last item is accessed using last_value method.
  end
  
  remove.prompt 'Indexes to remove items starts from 0'
  remove.error  'Indexes must be valid, as shown in values table.'
  
  remove.execute do
    puts "Here also works last_value: #{last_value}"
  end
end
```

In console, this will be shown as:

```
$ This will fill a list!
  |---|-----------------------|
  |   |                       |
  |---|-----------------------|
  
  This is the menu prompt (add (0), remove (1), finish (2)): 4
  This is shown if an invalid option is selected.
  This is the menu prompt (add (0), remove (1), finish (2)): 0
  Enter a name less than 10 characters: Longeeeername
  Name must be less or equal than 10 characters.
  Enter a name less than 10 characters: Jose
  The value added was Jose.
  
  |---|-----------------------|
  | 0 | JOSE                  |
  |---|-----------------------|
  
  This is the menu prompt (add (0), remove (1), finish (2)): 0
  Enter a name less than 10 characters: Maria
  The value added was Maria.
    
  |---|-----------------------|
  | 0 | JOSE                  |
  | 1 | MARIA                 |
  |---|-----------------------|
  
  This is the menu prompt (add (0), remove (1), finish (2)): 1
  Indexes to remove items starts from 0: 2
  Indexes must be valid, as shown in values table.
  
  |---|-----------------------|
  | 0 | JOSE                  |
  | 1 | MARIA                 |
  |---|-----------------------|
    
  This is the menu prompt (add (0), remove (1), finish (2)): 1
  Indexes to remove items starts from 0: 0
  Here also works last_value: Jose
  
  |---|-----------------------|
  | 0 | MARIA                 |
  |---|-----------------------|
    
  This is the menu prompt (add (0), remove (1), finish (2)): 2
  The final value is [ Maria ].
```

#### Group Step

Group step is basically a hybrid Skip step, which can have configured a set of steps in YAML configuration.
In code, it works exactly as a Skip step, but in configuration, a set of sub-steps can be assigned.
See [Usage](#usage) section for configuration examples. 

### Configuration

## Authors

* **Ernesto** - Gem Creator - [enchf](https://github.com/enchf)

## License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details
