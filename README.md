[![Gem](https://img.shields.io/gem/v/u-service.svg?style=flat-square)](https://rubygems.org/gems/u-service)
[![Build Status](https://travis-ci.com/serradura/u-service.svg?branch=master)](https://travis-ci.com/serradura/u-service)
[![Maintainability](https://api.codeclimate.com/v1/badges/a30b18528a317435c2ee/maintainability)](https://codeclimate.com/github/serradura/u-service/maintainability)
[![Test Coverage](https://api.codeclimate.com/v1/badges/a30b18528a317435c2ee/test_coverage)](https://codeclimate.com/github/serradura/u-service/test_coverage)

μ-service (Micro::Service)
==========================

Create simple and powerful service objects.

The main goals of this project are:
1. The smallest possible learning curve.
2. Referential transparency and data integrity.
3. No callbacks, compose a pipeline of service objects to represents complex business logic. (input >> process/transform >> output)

- [μ-service (Micro::Service)](#%ce%bc-service-microservice)
  - [Required Ruby version](#required-ruby-version)
  - [Installation](#installation)
  - [Usage](#usage)
    - [How to create a Service Object?](#how-to-create-a-service-object)
    - [How to use the result hooks?](#how-to-use-the-result-hooks)
    - [How to create a pipeline of Service Objects?](#how-to-create-a-pipeline-of-service-objects)
    - [What is a strict Service Object?](#what-is-a-strict-service-object)
    - [How to validate Service Object attributes?](#how-to-validate-service-object-attributes)
    - [It's possible to compose pipelines with other pipelines?](#its-possible-to-compose-pipelines-with-other-pipelines)
  - [Examples](#examples)
  - [Comparisons](#comparisons)
  - [Benchmarks](#benchmarks)
  - [Development](#development)
  - [Contributing](#contributing)
  - [License](#license)
  - [Code of Conduct](#code-of-conduct)

## Required Ruby version

> \>= 2.2.0

## Installation

Add this line to your application's Gemfile:

```ruby
gem 'u-service'
```

And then execute:

    $ bundle

Or install it yourself as:

    $ gem install u-service

## Usage

### How to create a Service Object?

```ruby
class Multiply < Micro::Service::Base
  attributes :a, :b

  def call!
    if a.is_a?(Numeric) && b.is_a?(Numeric)
      Success(a * b)
    else
      Failure(:invalid_data)
    end
  end
end

#====================#
# Calling a service  #
#====================#

result = Multiply.call(a: 2, b: 2)

p result.success? # true
p result.value    # 4

# Note:
# The result of a Micro::Service#call
# is an instance of Micro::Service::Result

#----------------------------#
# Calling a service instance #
#----------------------------#

result = Multiply.new(a: 2, b: 3).call

p result.success? # true
p result.value    # 6

#===========================#
# Verify the result failure #
#===========================#

result = Multiply.call(a: '2', b: 2)

p result.success? # false
p result.failure? # true
p result.value    # :invalid_data
```

### How to use the result hooks?

```ruby
class Double < Micro::Service::Base
  attributes :number

  def call!
    return Failure(:invalid) { 'the number must be a numeric value' } unless number.is_a?(Numeric)
    return Failure(:lte_zero) { 'the number must be greater than 0' } if number <= 0

    Success(number * 2)
  end
end

#================================#
# Printing the output if success #
#================================#

Double
  .call(number: 3)
  .on_success { |number| p number }
  .on_failure(:invalid) { |msg| raise TypeError, msg }
  .on_failure(:lte_zero) { |msg| raise ArgumentError, msg }

# The output when is a success:
# 6

#=============================#
# Raising an error if failure #
#=============================#

Double
  .call(number: -1)
  .on_success { |number| p number }
  .on_failure(:invalid) { |msg| raise TypeError, msg }
  .on_failure(:lte_zero) { |msg| raise ArgumentError, msg }

# The output (raised an error) when is a failure:
# ArgumentError (the number must be greater than 0)
```

### How to create a pipeline of Service Objects?

```ruby
module Steps
  class ConvertToNumbers < Micro::Service::Base
    attribute :numbers

    def call!
      if numbers.all? { |value| String(value) =~ /\d+/ }
        Success(numbers: numbers.map(&:to_i))
      else
        Failure('numbers must contain only numeric types')
      end
    end
  end

  class Add2 < Micro::Service::Strict
    attribute :numbers

    def call!
      Success(numbers: numbers.map { |number| number + 2 })
    end
  end

  class Double < Micro::Service::Strict
    attribute :numbers

    def call!
      Success(numbers: numbers.map { |number| number * 2 })
    end
  end

  class Square < Micro::Service::Strict
    attribute :numbers

    def call!
      Success(numbers: numbers.map { |number| number * number })
    end
  end
end

#-------------------------------------------------#
# Creating a pipeline using the collection syntax #
#-------------------------------------------------#

Add2ToAllNumbers = Micro::Service::Pipeline[
  Steps::ConvertToNumbers,
  Steps::Add2
]

result = Add2ToAllNumbers.call(numbers: %w[1 1 2 2 3 4])

p result.success? # true
p result.value    # {:numbers => [3, 3, 4, 4, 5, 6]}

#-------------------------------------------------------#
# An alternative way to create a pipeline using classes #
#-------------------------------------------------------#

class DoubleAllNumbers
  include Micro::Service::Pipeline

  pipeline Steps::ConvertToNumbers, Steps::Double
end

DoubleAllNumbers
  .call(numbers: %w[1 1 b 2 3 4])
  .on_failure { |message| p message } # "numbers must contain only numeric types"

#-----------------------------------------------------------------#
# Another way to create a pipeline using the composition operator #
#-----------------------------------------------------------------#

SquareAllNumbers =
  Steps::ConvertToNumbers >> Steps::Square

SquareAllNumbers
  .call(numbers: %w[1 1 2 2 3 4])
  .on_success { |value| p value[:numbers] } # [1, 1, 4, 4, 9, 16]

#=================================================================#
# Attention:                                                      #
# When happening a failure, the service object responsible for it #
# will be accessible in the result                                #
#=================================================================#

result = SquareAllNumbers.call(numbers: %w[1 1 b 2 3 4])

result.failure?                               # true
result.service.is_a?(Steps::ConvertToNumbers) # true

result.on_failure do |_message, service|
  puts "#{service.class.name} was the service responsible for the failure" } # Steps::ConvertToNumbers was the service responsible for the failure
end
```

### What is a strict Service Object?

A: Is a service object which will require all keywords (attributes) on its initialization.

```ruby
class Double < Micro::Service::Strict
  attribute :numbers

  def call!
    Success(numbers.map { |number| number * 2 })
  end
end

Double.call({})

# The output (raised an error):
# ArgumentError (missing keyword: :numbers)
```

### How to validate Service Object attributes?

Note: To do this your application must have the [activemodel >= 3.2](https://rubygems.org/gems/activemodel) as a dependency.

```ruby
#
# By default, if your project has the activemodel
# any kind of service attribute can be validated.
#
class Multiply < Micro::Service::Base
  attributes :a, :b

  validates :a, :b, presence: true, numericality: true

  def call!
    return Failure(errors: self.errors) unless valid?

    Success(number: a * b)
  end
end

#
# But if do you want an automatic way to fail
# your services if there is some invalid data.
# You can use:

# In some file. e.g: A Rails initializer
require 'micro/service/with_validation' # or require 'u-service/with_validation'

# In the Gemfile
gem 'u-service', '~> 0.12.0', require: 'u-service/with_validation'

# Using this approach, you can rewrite the previous sample with fewer lines of code.

class Multiply < Micro::Service::Base
  attributes :a, :b

  validates :a, :b, presence: true, numericality: true

  def call!
    Success(number: a * b)
  end
end

# Note:
# After requiring the validation mode, the
# Micro::Service::Strict classes will inherit this new behavior.
```

### It's possible to compose pipelines with other pipelines?

Answer: Yes

```ruby
module Steps
  class ConvertToNumbers < Micro::Service::Base
    attribute :numbers

    def call!
      if numbers.all? { |value| String(value) =~ /\d+/ }
        Success(numbers: numbers.map(&:to_i))
      else
        Failure('numbers must contain only numeric types')
      end
    end
  end

  class Add2 < Micro::Service::Strict
    attribute :numbers

    def call!
      Success(numbers: numbers.map { |number| number + 2 })
    end
  end

  class Double < Micro::Service::Strict
    attribute :numbers

    def call!
      Success(numbers: numbers.map { |number| number * 2 })
    end
  end

  class Square < Micro::Service::Strict
    attribute :numbers

    def call!
      Success(numbers: numbers.map { |number| number * number })
    end
  end
end

Add2ToAllNumbers = Steps::ConvertToNumbers >> Steps::Add2
DoubleAllNumbers = Steps::ConvertToNumbers >> Steps::Double
SquareAllNumbers = Steps::ConvertToNumbers >> Steps::Square
DoubleAllNumbersAndAdd2 = DoubleAllNumbers >> Steps::Add2
SquareAllNumbersAndAdd2 = SquareAllNumbers >> Steps::Add2
SquareAllNumbersAndDouble = SquareAllNumbersAndAdd2 >> DoubleAllNumbers
DoubleAllNumbersAndSquareAndAdd2 = DoubleAllNumbers >> SquareAllNumbersAndAdd2

SquareAllNumbersAndDouble
  .call(numbers: %w[1 1 2 2 3 4])
  .on_success { |value| p value[:numbers] } # [6, 6, 12, 12, 22, 36]

DoubleAllNumbersAndSquareAndAdd2
  .call(numbers: %w[1 1 2 2 3 4])
  .on_success { |value| p value[:numbers] } # [6, 6, 18, 18, 38, 66]
```

Note: You can blend any of the [syntaxes/approaches to create the pipelines](#how-to-create-a-pipeline-of-service-objects)) - [examples](https://github.com/serradura/u-service/blob/master/test/micro/service/pipeline/blend_test.rb#L7-L34).

## Examples

1. [Rescuing an exception inside of service objects](https://github.com/serradura/u-service/blob/master/examples/rescuing_exceptions.rb)
2. [Users creation](https://github.com/serradura/u-service/blob/master/examples/users_creation.rb)

    An example of how to use services pipelines to sanitize and validate the input data, and how to represents a common use case, like: create an user.
3. [CLI calculator](https://github.com/serradura/u-service/tree/master/examples/calculator)

    A more complex example which use rake tasks to demonstrate how to handle user data, and how to use different failures type to control the app flow.

## Comparisons

Check it out implementations of the same use case with different libs (abstractions).

* [interactor](https://github.com/serradura/u-service/blob/master/comparisons/interactor.rb)
* [u-service](https://github.com/serradura/u-service/blob/master/comparisons/u-service.rb)

## Benchmarks

**[interactor](https://github.com/collectiveidea/interactor)** VS **[u-service](https://github.com/serradura/u-service)**

https://github.com/serradura/u-service/tree/master/benchmarks/interactor

![interactor VS u-service](https://github.com/serradura/u-service/blob/master/assets/u-service_benchmarks.png?raw=true)

## Development

After checking out the repo, run `bin/setup` to install dependencies. Then, run `./test.sh` to run the tests. You can also run `bin/console` for an interactive prompt that will allow you to experiment.

To install this gem onto your local machine, run `bundle exec rake install`. To release a new version, update the version number in `version.rb`, and then run `bundle exec rake release`, which will create a git tag for the version, push git commits and tags, and push the `.gem` file to [rubygems.org](https://rubygems.org).

## Contributing

Bug reports and pull requests are welcome on GitHub at https://github.com/serradura/u-service. This project is intended to be a safe, welcoming space for collaboration, and contributors are expected to adhere to the [Contributor Covenant](http://contributor-covenant.org) code of conduct.

## License

The gem is available as open source under the terms of the [MIT License](https://opensource.org/licenses/MIT).

## Code of Conduct

Everyone interacting in the Micro::Service project’s codebases, issue trackers, chat rooms and mailing lists is expected to follow the [code of conduct](https://github.com/serradura/u-service/blob/master/CODE_OF_CONDUCT.md).
