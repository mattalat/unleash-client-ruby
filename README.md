# Unleash::Client

[![Build Status](https://travis-ci.org/Unleash/unleash-client-ruby.svg?branch=master)](https://travis-ci.org/Unleash/unleash-client-ruby)
[![Coverage Status](https://coveralls.io/repos/github/Unleash/unleash-client-ruby/badge.svg?branch=master)](https://coveralls.io/github/Unleash/unleash-client-ruby?branch=master)
[![Gem Version](https://badge.fury.io/rb/unleash.svg)](https://badge.fury.io/rb/unleash)

Unleash client so you can roll out your features with confidence.

Leverage the [Unleash Server](https://github.com/Unleash/unleash) for powerful feature toggling in your ruby/rails applications.

## Supported Ruby Interpreters

  * MRI 3.0
  * MRI 2.7
  * MRI 2.6
  * MRI 2.5
  * jruby

## Installation

Add this line to your application's Gemfile:

```ruby
gem 'unleash', '~> 3.2.3'
```

And then execute:

    $ bundle

Or install it yourself as:

    $ gem install unleash

## Configure

It is **required** to configure the `url` of the unleash server and `app_name` with the name of the runninng application. Please substitute the sample `'http://unleash.herokuapp.com/api'` for the url of your own instance.

It is **highly recommended** to configure the `instance_id` parameter as well.


```ruby
Unleash.configure do |config|
  config.url         = 'http://unleash.herokuapp.com/api'
  config.app_name    = 'my_ruby_app'
end
```

or instantiate the client with the valid configuration:

```ruby
UNLEASH = Unleash::Client.new(url: 'http://unleash.herokuapp.com/api', app_name: 'my_ruby_app')
```

#### List of Arguments

Argument | Description | Required? |  Type |  Default Value|
---------|-------------|-----------|-------|---------------|
`url`      | Unleash server URL. | Y | String | N/A |
`app_name` | Name of your program. | Y | String | N/A |
`instance_id` | Identifier for the running instance of program. Important so you can trace back to where metrics are being collected from. **Highly recommended be be set.** | N | String | random UUID |
`environment` | Environment the program is running on. Could be for example `prod` or `dev`. Not yet in use. | N | String | `default` |
`refresh_interval` | How often the unleash client should check with the server for configuration changes. | N | Integer |  15 |
`metrics_interval` | How often the unleash client should send metrics to server. | N | Integer | 10 |
`disable_client` | Disables all communication with the Unleash server, effectively taking it *offline*. If set, `is_enabled?` will always answer with the `default_value` and configuration validation is skipped. Defeats the entire purpose of using unleash, but can be useful in when running tests. | N | Boolean | `false` |
`disable_metrics` | Disables sending metrics to Unleash server. | N | Boolean | `false` |
`custom_http_headers` | Custom headers to send to Unleash. | N | Hash | {} |
`timeout` | How long to wait for the connection to be established or wait in reading state (open_timeout/read_timeout) | N | Integer | 30 |
`retry_limit` | How many consecutive failures in connecting to the Unleash server are allowed before giving up. Use `Float::INFINITY` if you would like it to never give up. | N | Numeric | 5 |
`backup_file` | Filename to store the last known state from the Unleash server. Best to not change this from the default. | N | String | `Dir.tmpdir + "/unleash-#{app_name}-repo.json` |
`logger` | Specify a custom `Logger` class to handle logs for the Unleash client. | N | Class | `Logger.new(STDOUT)` |
`log_level` | Change the log level for the `Logger` class. Constant from `Logger::Severity`. | N | Constant | `Logger::ERROR` |

For in a more in depth look, please see `lib/unleash/configuration.rb`.


## Usage in a plain Ruby Application

```ruby
require 'unleash'
require 'unleash/context'

@unleash = Unleash::Client.new(url: 'http://unleash.herokuapp.com/api', app_name: 'my_ruby_app')

feature_name = "AwesomeFeature"
unleash_context = Unleash::Context.new
unleash_context.user_id = 123

if @unleash.is_enabled?(feature_name, unleash_context)
  puts " #{feature_name} is enabled according to unleash"
else
  puts " #{feature_name} is disabled according to unleash"
end
```

## Usage in a Rails Application

#### Add Initializer

Put in `config/initializers/unleash.rb`:

```ruby
Unleash.configure do |config|
  config.url      = 'http://unleash.herokuapp.com/api'
  config.app_name = Rails.application.class.parent.to_s
  # config.instance_id = "#{Socket.gethostname}"
  config.logger   = Rails.logger
  config.environment = Rails.env
end

UNLEASH = Unleash::Client.new
```
For `config.instance_id` use a string with a unique identification for the running instance. For example: it could be the hostname, if you only run one App per host. Or the docker container id, if you are running in docker. If it is not set the client will generate an unique UUID for each execution.


#### Add Initializer if using [Puma](https://github.com/puma/puma)

In `puma.rb` ensure that the unleash client is configured and instantiated as below, inside the `on_worker_boot` code block:

```ruby
on_worker_boot do
  # ...

  Unleash.configure do |config|
    config.url      = 'http://unleash.herokuapp.com/api'
    config.app_name = Rails.application.class.parent.to_s
    config.environment = Rails.env
  end
  Rails.configuration.unleash = Unleash::Client.new
end
```

Instead of the configuration in `config/initializers/unleash.rb`.

#### Add Initializer if using [Phusion Passenger](https://github.com/phusion/passenger)

The unleash client needs to be configured and instantiated inside the `PhusionPassenger.on_event(:starting_worker_process)` code block due to [smart spawning](https://www.phusionpassenger.com/library/indepth/ruby/spawn_methods/#smart-spawning-caveats):

The initializer in `config/initializers/unleash.rb` should look like:

```ruby
PhusionPassenger.on_event(:starting_worker_process) do |forked|
  if forked
    Unleash.configure do |config|
      config.url      = 'http://unleash.herokuapp.com/api'
      config.app_name = Rails.application.class.parent.to_s
      # config.instance_id = "#{Socket.gethostname}"
      config.logger   = Rails.logger
      config.environment = Rails.env
    end

    UNLEASH = Unleash::Client.new
  end
end
```

#### Set Unleash::Context

Be sure to add the following method and callback in the application controller to have `@unleash_context` set for all requests:

Add in `app/controllers/application_controller.rb`:

```ruby
  before_action :set_unleash_context

  private
  def set_unleash_context
    @unleash_context = Unleash::Context.new(
      session_id: session.id,
      remote_address: request.remote_ip,
      user_id: session[:user_id]
    )
  end
```

Or if you see better fit, only in the controllers that you will be using unleash.

#### Sample usage

Then wherever in your application that you need a feature toggle, you can use:

```ruby
if UNLEASH.is_enabled? "AwesomeFeature", @unleash_context
  puts "AwesomeFeature is enabled"
end
```

or if client is set in `Rails.configuration.unleash`:

```ruby
if Rails.configuration.unleash.is_enabled? "AwesomeFeature", @unleash_context
  puts "AwesomeFeature is enabled"
end
```

If the feature is not found in the server, it will by default return false. However you can override that by setting the default return value to `true`:

```ruby
if UNLEASH.is_enabled? "AwesomeFeature", @unleash_context, true
  puts "AwesomeFeature is enabled by default"
end
```

Another possibility is to send a block, [Lambda](https://ruby-doc.org/core-3.0.1/Kernel.html#method-i-lambda) or [Proc](https://ruby-doc.org/core-3.0.1/Proc.html#method-i-yield)
to evaluate the default value:

```ruby
net_check_proc = proc do |feature_name, context|
  context.remote_address.starts_with?("10.0.0.")
end

if UNLEASH.is_enabled?("AwesomeFeature", @unleash_context, &net_check_proc)
  puts "AwesomeFeature is enabled by default if you are in the 10.0.0.* network."
end
```

or

```ruby
awesomeness = 10
@unleash_context.properties[:coolness] = 10

if UNLEASH.is_enabled?("AwesomeFeature", @unleash_context) { |feat, ctx| awesomeness >= 6 && ctx.properties[:coolness] >= 8 }
  puts "AwesomeFeature is enabled by default if both the user has a high enought coolness and the application has a high enough awesomeness"
end
```

Note:
- The block/lambda/proc can use feature name and context as an arguments.
- The client will evaluate the fallback function once per call of `is_enabled()`.
  Please keep this in mind when creating your fallback function!
- The returned value of the block should be a boolean.
  However the client will coerce the result to boolean via `!!`.
- If both a `default_value` and `fallback_function` are supplied,
  the client will define the default value by `OR`ing the default value and the output of the fallback function.


Alternatively by using `if_enabled` you can send a code block to be executed as a parameter:

```ruby
UNLEASH.if_enabled "AwesomeFeature", @unleash_context, true do
  puts "AwesomeFeature is enabled by default"
end
```

Note: `if_enabled` only supports `default_value`, but not `fallback_function`.

##### Variations

If no variant is found in the server, use the fallback variant.

```ruby
fallback_variant = Unleash::Variant.new(name: 'default', enabled: true, payload: {"color" => "blue"})
variant = UNLEASH.get_variant "ColorVariants", @unleash_context, fallback_variant

puts "variant color is: #{variant.payload.fetch('color')}"
```


#### Client methods

Method Name | Description | Return Type |
---------|-------------|-------------|
`is_enabled?` | Check if feature toggle is to be enabled or not. | Boolean |
`enabled?` | Alias to the `is_enabled?` method. But more ruby idiomatic. | Boolean |
`if_enabled` | Run a code block, if a feature is enabled. | `yield` |
`get_variant` | Get variant for a given feature | `Unleash::Variant` |
`shutdown` | Save metrics to disk, flush metrics to server, and then kill ToggleFetcher and MetricsReporter threads. A safe shutdown. Not really useful in long running applications, like web applications. | nil |
`shutdown!` | Kill ToggleFetcher and MetricsReporter threads immediately. | nil |

For the full method signatures, please see [client.rb](lib/unleash/client.rb)

## Local test client

```
# cli unleash client:
bundle exec bin/unleash-client --help

# or a simple sample implementation (with values hardcoded):
bundle exec examples/simple.rb
```

## Available Strategies

This client comes with the all the required strategies out of the box:

 * ApplicationHostnameStrategy
 * DefaultStrategy
 * GradualRolloutRandomStrategy
 * GradualRolloutSessionIdStrategy
 * GradualRolloutUserIdStrategy
 * RemoteAddressStrategy
 * UnknownStrategy
 * UserWithIdStrategy


## Development

After checking out the repo, run `bin/setup` to install dependencies. Then, run `rake spec` to run the tests. You can also run `bin/console` for an interactive prompt that will allow you to experiment.

To install this gem onto your local machine, run `bundle exec rake install`. To release a new version, update the version number in `version.rb`, and then run `bundle exec rake release`, which will create a git tag for the version, push git commits and tags, and push the `.gem` file to [rubygems.org](https://rubygems.org).


## Contributing

Bug reports and pull requests are welcome on GitHub at https://github.com/unleash/unleash-client-ruby.

Please include tests with any pull requests, to avoid regressions.
