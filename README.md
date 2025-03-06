# Learn-Rails-by-Reading-Source-Code
[![LICENSE](https://img.shields.io/badge/license-Anti%20996-blue.svg)](https://github.com/996icu/996.ICU/blob/master/LICENSE)
[![996.icu](https://img.shields.io/badge/link-996.icu-red.svg)](https://996.icu)
## Table of Contents

  * [Part 0: Before reading Rails 5 source code](#part-0-before-reading-rails-5-source-code)
      * [What will you learn from this tutorial?](#what-will-you-learn-from-this-tutorial)
  * [Part 1: Your app: an instance of YourProject::Application](#part-1-your-app-an-instance-of-yourprojectapplication)
  * [Part 2: config](#part-2-config)
  * [Part 3: Every request and response](#part-3-every-request-and-response)
      * [Puma](#puma)
      * [Rack apps](#rack-apps)
      * [The core app: ActionDispatch::Routing::RouteSet instance](#the-core-app-actiondispatchroutingrouteset-instance)
      * [Render view](#render-view)
      * [How can instance variables defined in Controller be accessed in view file?](#how-can-instance-variables-defined-in-controller-be-accessed-in-view-file)
  * [Part 4: What does `$ rails server` do?](#part-4-what-does--rails-server-do)
      * [Thor](#thor)
      * [Rails::Server#start](#railsserverstart)
      * [Starting Puma](#starting-puma)
      * [Conclusion](#conclusion)
      * [Exiting Puma](#exiting-puma)
          * [Process and Thread](#process-and-thread)
          * [Send `SIGTERM` to Puma](#send-sigterm-to-puma)
 

## Part 0: Before reading Rails 5 source code
1) I suggest you to learn Rack [http://rack.github.io/](http://rack.github.io/) first. 

In Rack, an object with `call` method is a Rack app.

So what is the object with `call` method in Rails? I will answer this question in Part 1.

2) You need a good IDE which can help for debugging. I use [RubyMine](https://www.jetbrains.com/).

### What will you learn from this tutorial?
* How does Rails start your application?

* How does Rails process every request?

* How does Rails combine ActionController, ActionView and Routes together?

* How does Puma, Rack, Rails work together?

* What's Puma's multiple threads?

I should start with the command `$ rails server`, but I put this to Part 4. Because it's a little bit complex.

## Part 1: Your app: an instance of YourProject::Application
Assume your Rails app's class name is `YourProject::Application` (defined in `./config/application.rb`).

First, I will give you a piece of important code.
```ruby
# ./gems/railties-5.2.2/lib/rails/commands/server/server_command.rb
module Rails
  module Command
    class ServerCommand < Base
      def perform
        # ...
        Rails::Server.new(server_options).tap do |server|
          # APP_PATH is '/path/to/your_project/config/application'.
          # require APP_PATH will create the 'Rails.application' object.
          # Actually, 'Rails.application' is an instance of `YourProject::Application`.
          # Rack server will start 'Rails.application'.
          require APP_PATH
          
          Dir.chdir(Rails.application.root)
          
          server.start
        end
      end
    end
  end
  
  class Server < ::Rack::Server
    def start
      #...
      # 'wrapped_app' will get an well prepared app from `./config.ru` file.
      # 'wrapped_app' will return an instance of `YourProject::Application`.
      # But the instance of `YourProject::Application` returned is not created in 'wrapped_app'.
      # It has been created when `require APP_PATH` in previous code: 
      # in Rails::Command::ServerCommand#perform
      wrapped_app   

      super # Will invoke ::Rack::Server#start. 
    end
  end
end
```
A Rack server need to start with an `App` object. The `App` object should have a `call` method.

`config.ru` is the conventional entry file for Rack app. So let's look at it.
```ruby
# ./config.ru
require_relative 'config/environment'

run Rails.application # It seems that this is the app.
```

Let's test it by `Rails.application.respond_to?(:call)`, it returns `true`.

Let's step into `Rails.application`.

```ruby
# ./gems/railties-5.2.2/lib/rails.rb
module Rails
  class << self
    @application = @app_class = nil

    attr_accessor :app_class
    
    # Oh, 'application' is a class method for module 'Rails'. It is not an object.
    # But it returns an object which is an instance of 'app_class'.
    # So it is important for us to know what class 'app_class' is.
    def application
      @application ||= (app_class.instance if app_class)  
    end
  end
end
```

Because `Rails.application.respond_to?(:call)` returns `true`, `app_class.instance` has a `call` method.

When was `app_class` set?
```ruby
module Rails
  class Application < Engine
    class << self
      def inherited(base) # This is a hooked method.
        Rails.app_class = base # This line set the 'app_class'.
      end
    end
  end
end
```

`Rails::Application` is inherited by `YourProject`,
```ruby
# ./config/application.rb
module YourProject
  # The hooked method `inherited` will be invoked here.
  class Application < Rails::Application
  end
end
```
So `YourProject::Application` is the `Rails.app_class` here.

You may have a question: When does Rails execute the code in `./config/application.rb`?

To answer this question, we need to look back to `config.ru`.
```ruby
# ./config.ru
require_relative 'config/environment' # Let's step into this line.

run Rails.application # It seems that this is the app.
```

```ruby
# ./config/environment.rb
# Load the Rails application.
require_relative 'application' # Let's step into this line.

# Initialize the Rails application.
Rails.application.initialize!
```

```ruby
# ./config/application.rb
require_relative 'boot'

require 'rails/all'

# Require the gems listed in Gemfile, including any gems
# you've limited to :test, :development, or :production.
Bundler.require(*Rails.groups)

module YourProject
  # The hooked method `inherited` will be invoked here.
  class Application < Rails::Application
    config.load_defaults 5.2
    config.i18n.default_locale = :zh
  end
end
```
Because `YourProject::Application` is `Rails.app_class`, `app_class.instance` is `YourProject::Application.instance`.

But where is the `call` method? 

`call` method should be a method of `YourProject::Application.instance`.

The `call` method processes every request. Here it is.
```ruby
# ./gems/railties/lib/rails/engine.rb
module Rails
  class Engine < Railtie
    def call(env) # This method will process every request. It is invoked by Rack. So it is very important. 
      req = build_request env
      app.call req.env # We will discuss the 'app' object later.
    end
  end
end

# ./gems/railties/lib/rails/application.rb
module Rails
  class Application < Engine
  end
end

# ./config/application.rb
module YourProject
  class Application < Rails::Application
  end
end
```

Ancestor's chain is `YourProject::Application < Rails::Application < Rails::Engine < Rails::Railtie`.

So `YourProject::Application.new.respond_to?(:call)` returns `true`.

But what does `app_class.instance` really do?

`instance` is just a method name. What we really expects is something like `app_class.new`.

Let's look at the definition of `instance`.
```ruby
# ./gems/railties/lib/rails/application.rb
module Rails
  class Application < Engine
    def instance
      super.run_load_hooks! # This line confused me at the beginning. 
    end
  end
end
```
After a deep research, I realized that this code is equal to:
```ruby
def instance
  return_value = super # Keyword 'super' means call the ancestor's same name method: 'instance'.
  return_value.run_load_hooks!
end
```

```ruby
# ./gems/railties/lib/rails/railtie.rb
module Rails
  class Railtie
    def instance
      # 'Rails::Railtie' is the top ancestor class.
      # Now 'app_class.instance' is 'YourProject::Application.new'.
      @instance ||= new
    end
  end
end
```

```ruby
module Rails
  def application
    @application ||= (app_class.instance if app_class)
  end
end
```

So `YourProject::Application.new` is `Rails.application`.

Rack server will start `Rails.application` in the end.

`Rails.application` is an important object in Rails.

And you'll only have one `Rails.application` in one Puma process. 

Multiple threads in a Puma process shares the `Rails.application`.

## Part 2: config
The first time we see the `config` is in `./config/application.rb`.
```ruby
# ./config/application.rb
#...
module YourProject
  class Application < Rails::Application
    # Actually, `config` is a method of `YourProject::Application`.
    # It is defined in it's grandfather's father: `Rails::Railtie`
    config.load_defaults 5.2  # Let's step into this line to see what is config.
    config.i18n.default_locale = :zh
  end
end
```

```ruby
module Rails
  class Railtie
    class << self
      # Method `:config` is defined here.
      # Actually, method `:config` is delegated to another object `:instance`.
      delegate :config, to: :instance 
      
      # Call `YourProject::Application.config` will actually call `YourProject::Application.instance.config`
      def instance
        # return an instance of `YourProject::Application`.
        # Call `YourProject::Application.config` will actually call `YourProject::Application.new.config`
        @instance ||= new 
      end
    end
  end
  
  class Engine < Railtie
  end
  
  class Application < Engine
    class << self
      def instance
        # 'super' will call `:instance` method in `Railtie`, 
        # which will return an instance of `YourProject::Application`.
        return_value = super  
        return_value.run_load_hooks!
      end
    end
    
    def run_load_hooks!
      return self if @ran_load_hooks
      @ran_load_hooks = true

      # ...
      self # `self` is an instance of `YourProject::Application`, and `self` is `Rails.application`.
    end
    
    # This is the method `config`.
    def config
      # It is an instance of class `Rails::Application::Configuration`. 
      # Please notice that `Rails::Application` is superclass of `YourProject::Application` (self's class).   
      @config ||= Application::Configuration.new(self.class.find_root(self.class.called_from))
    end
  end
end
```
In the end, `YourProject::Application.config === Rails.application.config` returns `true`.

Invoke Class's `config` method become invoke the class's instance's `config` method.

```ruby
module Rails
  class << self
    def configuration
      application.config
    end 
  end
end
```
So `Rails.configuration === Rails.application.config` returns `true`.

FYI:
```ruby
module Rails
  class Application
    class Configuration < ::Rails::Engine::Configuration
    end
  end
  
  class Engine
    class Configuration < ::Rails::Railtie::Configuration
      attr_accessor :middleware

      def initialize(root = nil)
        super()
        #...
        @middleware = Rails::Configuration::MiddlewareStackProxy.new
      end
    end
  end
  
  class Railtie
    class Configuration
    end
  end
end
```

## Part 3: Every request and response
Imagine we have this route for the home page. 
```ruby
# ./config/routes.rb
Rails.application.routes.draw do
  root 'home#index' # HomeController#index
end
```

### Puma
When a request is made from client, Puma will process the request in `Puma::Server#process_client`.

If you want to know how Puma enter the method `Puma::Server#process_client`, please read part 4 or just search 'process_client' in this document.

```ruby
# ./gems/puma-3.12.0/lib/puma/server.rb
require 'socket'

module Puma
  # The HTTP Server itself. Serves out a single Rack app.
  #
  # This class is used by the `Puma::Single` and `Puma::Cluster` classes
  # to generate one or more `Puma::Server` instances capable of handling requests.
  # Each Puma process will contain one `Puma::Server` instacne.
  #
  # The `Puma::Server` instance pulls requests from the socket, adds them to a
  # `Puma::Reactor` where they get eventually passed to a `Puma::ThreadPool`.
  #
  # Each `Puma::Server` will have one reactor and one thread pool.
  class Server
    def initialize(app, events=Events.stdio, options={})
      # app: #<Puma::Configuration::ConfigMiddleware:0x00007fcf1612c338 
      #        @app = #<YourProject::Application:0x00007fcf160fb120>
      #        @config = #<Puma::Configuration:0x00007fcf169a6c98>         
      #       >
      @app = app
      #...
    end
    
    # Given a connection on +client+, handle the incoming requests.
    #
    # This method support HTTP Keep-Alive so it may, depending on if the client
    # indicates that it supports keep alive, wait for another request before
    # returning.
    #
    def process_client(client, buffer)
      begin
        # ...
        while true
          # Let's step into this line.
          case handle_request(client, buffer) # Will return true in this example.
          when true
            return unless @queue_requests
            buffer.reset

            ThreadPool.clean_thread_locals if clean_thread_locals

            unless client.reset(@status == :run)
              close_socket = false
              client.set_timeout @persistent_timeout
              @reactor.add client
              return
            end
          end
        end
        # ...
      ensure
        buffer.reset
        client.close if close_socket
        #...
      end
    end
    
    # Given the request +env+ from +client+ and a partial request body
    # in +body+, finish reading the body if there is one and invoke
    # the Rack app. Then construct the response and write it back to
    # +client+
    #
    def handle_request(req, lines)
      env = req.env
      # ... 
      # app: #<Puma::Configuration::ConfigMiddleware:0x00007fcf1612c338 
      #        @app = #<YourProject::Application:0x00007fcf160fb120>
      #        @config = #<Puma::Configuration:0x00007fcf169a6c98>         
      #       >
      status, headers, res_body = @app.call(env) # Let's step into this line.

      # ...
      return keep_alive
    end    
  end
end
```
```ruby
# ./gems/puma-3.12.0/lib/puma/configuration.rb
module Puma
  class Configuration
    class ConfigMiddleware
      def initialize(config, app)
        @config = config
        @app = app
      end

      def call(env)
        env[Const::PUMA_CONFIG] = @config
        # @app: #<YourProject::Application:0x00007fb4b1b4bcf8>
        @app.call(env)
      end
    end
  end
end
```

### Rack apps
As we see when Ruby enter `Puma::Configuration::ConfigMiddleware#call`, the `@app` is `YourProject::Application` instance.

It is just the `Rails.application`.

Rack need a `call` method to process request. 

Rails defined this `call` method in `Rails::Engine#call`, so that `YourProject::Application` instance will have a `call` method.

```ruby
# ./gems/railties/lib/rails/engine.rb
module Rails
  class Engine < Railtie
    def call(env) # This method will process every request. It is invoked by Rack. 
      req = build_request env
      app.call req.env # The 'app' method is blow.
    end
    
    def app
      # FYI,
      # caller: [
      # "../gems/railties-5.2.2/lib/rails/application/finisher.rb:47:in `block in <module:Finisher>'", 
      # "../gems/railties-5.2.2/lib/rails/initializable.rb:32:in `instance_exec'", 
      # "../gems/railties-5.2.2/lib/rails/initializable.rb:32:in `run'", 
      # "../gems/railties-5.2.2/lib/rails/initializable.rb:63:in `block in run_initializers'", 
      # "../ruby-2.6.0/lib/ruby/2.6.0/tsort.rb:228:in `block in tsort_each'", 
      # "../ruby-2.6.0/lib/ruby/2.6.0/tsort.rb:350:in `block (2 levels) in each_strongly_connected_component'",
      # "../ruby-2.6.0/lib/ruby/2.6.0/tsort.rb:431:in `each_strongly_connected_component_from'", 
      # "../ruby-2.6.0/lib/ruby/2.6.0/tsort.rb:349:in `block in each_strongly_connected_component'", 
      # "../ruby-2.6.0/lib/ruby/2.6.0/tsort.rb:347:in `each'", 
      # "../ruby-2.6.0/lib/ruby/2.6.0/tsort.rb:347:in `call'", 
      # "../ruby-2.6.0/lib/ruby/2.6.0/tsort.rb:347:in `each_strongly_connected_component'", 
      # "../ruby-2.6.0/lib/ruby/2.6.0/tsort.rb:226:in `tsort_each'", 
      # "../ruby-2.6.0/lib/ruby/2.6.0/tsort.rb:205:in `tsort_each'", 
      # "../gems/railties-5.2.2/lib/rails/initializable.rb:61:in `run_initializers'", 
      # "../gems/railties-5.2.2/lib/rails/application.rb:361:in `initialize!'", 
      # "/Users/lanezhang/projects/mine/free-erp/config/environment.rb:5:in `<top (required)>'", 
      # "config.ru:2:in `require_relative'", "config.ru:2:in `block in <main>'", 
      # "../gems/rack-2.0.6/lib/rack/builder.rb:55:in `instance_eval'", 
      # "../gems/rack-2.0.6/lib/rack/builder.rb:55:in `initialize'", 
      # "config.ru:in `new'", "config.ru:in `<main>'", 
      # "../gems/rack-2.0.6/lib/rack/builder.rb:49:in `eval'", 
      # "../gems/rack-2.0.6/lib/rack/builder.rb:49:in `new_from_string'", 
      # "../gems/rack-2.0.6/lib/rack/builder.rb:40:in `parse_file'", 
      # "../gems/rack-2.0.6/lib/rack/server.rb:320:in `build_app_and_options_from_config'", 
      # "../gems/rack-2.0.6/lib/rack/server.rb:219:in `app'", 
      # "../gems/railties-5.2.2/lib/rails/commands/server/server_command.rb:27:in `app'", 
      # "../gems/rack-2.0.6/lib/rack/server.rb:357:in `wrapped_app'", 
      # "../gems/railties-5.2.2/lib/rails/commands/server/server_command.rb:92:in `log_to_stdout'", 
      # "../gems/railties-5.2.2/lib/rails/commands/server/server_command.rb:54:in `start'", 
      # "../gems/railties-5.2.2/lib/rails/commands/server/server_command.rb:149:in `block in perform'",
      # "../gems/railties-5.2.2/lib/rails/commands/server/server_command.rb:144:in `tap'", 
      # "../gems/railties-5.2.2/lib/rails/commands/server/server_command.rb:144:in `perform'",
      # "../gems/thor-0.20.3/lib/thor/command.rb:27:in `run'", 
      # "../gems/thor-0.20.3/lib/thor/invocation.rb:126:in `invoke_command'",
      # "../gems/thor-0.20.3/lib/thor.rb:391:in `dispatch'", 
      # "../gems/railties-5.2.2/lib/rails/command/base.rb:65:in `perform'",
      # "../gems/railties-5.2.2/lib/rails/command.rb:46:in `invoke'", 
      # "../gems/railties-5.2.2/lib/rails/commands.rb:18:in `<top (required)>'", 
      # "../path/to/your_project/bin/rails:5:in `require'",
      # "../path/to/your_project/bin/rails:5:in `<main>'"
      # ]
      puts "caller: #{caller.inspect}"
      
      # You may want to know when is the @app first time initialized.
      # It is initialized when 'config.ru' is load by Rack server.
      # Please search `Rack::Server#build_app_and_options_from_config` in this document for more information.
      # When `Rails.application.initialize!` (in ./config/environment.rb) executed, @app is initialized. 
      @app || @app_build_lock.synchronize { # '@app_build_lock = Mutex.new', so multiple threads share one '@app'.
        @app ||= begin
          # In the end, config.middleware will be an instance of ActionDispatch::MiddlewareStack with preset instance variable @middlewares (which is an Array). 
          stack = default_middleware_stack # Let's step into this line
          # 'middleware' is a 'middleware_stack'!
          config.middleware = build_middleware.merge_into(stack)
          # FYI, this line is the last line and the result of this line is the return value for @app.
          config.middleware.build(endpoint) # look at this endpoint below. We will enter method `build` later.
        end
      }
      
#  @app: #<Rack::Sendfile:0x00007ff14d905f60 
#          @app=#<ActionDispatch::Static:0x00007ff14d906168 
#                 @app=#<ActionDispatch::Executor:0x00007ff14d9061b8 
#                        ...
#                           @app=#<Rack::ETag:0x00007fa1e540c4f8  
#                                  @app=#<Rack::TempfileReaper:0x00007fa1e540c520 
#                                         @app=#<ActionDispatch::Routing::RouteSet:0x00007fa1e594cbe8>
#                                 >
#                         ...    
#                        >
#                             
#                 >
#         >
      @app
    end
    
    # Defaults to an ActionDispatch::Routing::RouteSet instance.
    def endpoint
      ActionDispatch::Routing::RouteSet.new_with_config(config)
    end
  end
end
```

```ruby
# ./gems/railties/lib/rails/application...
module Rails
  class Application < Engine
    def default_middleware_stack
      default_stack = DefaultMiddlewareStack.new(self, config, paths)
      default_stack.build_stack # Let's step into this line.
    end
    
    class DefaultMiddlewareStack
      attr_reader :config, :paths, :app

      def initialize(app, config, paths)
        @app = app
        @config = config
        @paths = paths
      end

      def build_stack
        ActionDispatch::MiddlewareStack.new do |middleware|
          if config.force_ssl
            middleware.use ::ActionDispatch::SSL, config.ssl_options
          end

          middleware.use ::Rack::Sendfile, config.action_dispatch.x_sendfile_header

          if config.public_file_server.enabled
            headers = config.public_file_server.headers || {}

            middleware.use ::ActionDispatch::Static, paths["public"].first, index: config.public_file_server.index_name, headers: headers
          end

          if rack_cache = load_rack_cache
            require "action_dispatch/http/rack_cache"
            middleware.use ::Rack::Cache, rack_cache
          end

          if config.allow_concurrency == false
            # User has explicitly opted out of concurrent request
            # handling: presumably their code is not threadsafe

            middleware.use ::Rack::Lock
          end

          middleware.use ::ActionDispatch::Executor, app.executor

          middleware.use ::Rack::Runtime
          middleware.use ::Rack::MethodOverride unless config.api_only
          middleware.use ::ActionDispatch::RequestId
          middleware.use ::ActionDispatch::RemoteIp, config.action_dispatch.ip_spoofing_check, config.action_dispatch.trusted_proxies

          middleware.use ::Rails::Rack::Logger, config.log_tags
          middleware.use ::ActionDispatch::ShowExceptions, show_exceptions_app
          middleware.use ::ActionDispatch::DebugExceptions, app, config.debug_exception_response_format

          unless config.cache_classes
            middleware.use ::ActionDispatch::Reloader, app.reloader
          end

          middleware.use ::ActionDispatch::Callbacks
          middleware.use ::ActionDispatch::Cookies unless config.api_only

          if !config.api_only && config.session_store
            if config.force_ssl && config.ssl_options.fetch(:secure_cookies, true) && !config.session_options.key?(:secure)
              config.session_options[:secure] = true
            end
            middleware.use config.session_store, config.session_options
            middleware.use ::ActionDispatch::Flash
          end

          unless config.api_only
            middleware.use ::ActionDispatch::ContentSecurityPolicy::Middleware
          end

          middleware.use ::Rack::Head
          middleware.use ::Rack::ConditionalGet
          middleware.use ::Rack::ETag, "no-cache"

          middleware.use ::Rack::TempfileReaper unless config.api_only
        end
       end
     end
  end
end
```
```ruby
# ./gems/actionpack-5.2.2/lib/action_dispatch/middleware/stack.rb
module ActionDispatch
  class MiddlewareStack
    def use(klass, *args, &block)
      middlewares.push(build_middleware(klass, args, block))
    end
    
    def build_middleware(klass, args, block)
      Middleware.new(klass, args, block)
    end    
    
    def build(app = Proc.new)
      # See Enumerable#inject for more information.
      return_val = middlewares.freeze.reverse.inject(app) do |a, middleware|
        # a: app, and will be changed when iterating
        # middleware: #<ActionDispatch::MiddlewareStack::Middleware:0x00007f8a4fada6e8>, 'middleware' will be switched to another instance of ActionDispatch::MiddlewareStack::Middleware when iterating
        middleware.build(a) # Let's step into this line.
      end

      return_val
    end    
    
    class Middleware
      def initialize(klass, args, block)
        @klass = klass
        @args  = args
        @block = block
      end

      def build(app)
        # klass is Rack middleware like : Rack::TempfileReaper, Rack::ETag, Rack::ConditionalGet or Rack::Head, etc.
        # It's typical Rack app to use these middlewares.
        # See https://github.com/rack/rack-contrib/blob/master/lib/rack/contrib for more information. 
        klass.new(app, *args, &block)
      end      
    end
  end
end
```
### The core app: ActionDispatch::Routing::RouteSet instance
```ruby
# Paste again FYI. 
#  @app: #<Rack::Sendfile:0x00007ff14d905f60 
#          @app=#<ActionDispatch::Static:0x00007ff14d906168 
#                 @app=#<ActionDispatch::Executor:0x00007ff14d9061b8 
#                        ...
#                           @app=#<Rack::ETag:0x00007fa1e540c4f8  
#                                  @app=#<Rack::TempfileReaper:0x00007fa1e540c520 
#                                         @app=#<ActionDispatch::Routing::RouteSet:0x00007fa1e594cbe8>
#                                 >
#                         ...    
#                        >
#                             
#                 >
#         >
``` 
As we see in the Rack middleware stack, the last @app is 

`@app=#<ActionDispatch::Routing::RouteSet:0x00007fa1e594cbe8>`
```ruby
# ./gems/actionpack-5.2.2/lib/action_dispatch/routing/route_set.rb
module ActionDispatch
  module Routing
    class RouteSet
      def initialize(config = DEFAULT_CONFIG)
        @set    = Journey::Routes.new
        @router = Journey::Router.new(@set)
      end
    
      def call(env)
        req = make_request(env) # return ActionDispatch::Request.new(env)
        req.path_info = Journey::Router::Utils.normalize_path(req.path_info)
        @router.serve(req) # Let's step into this line.
      end
    end
  end
  
# ./gems/actionpack5.2.2/lib/action_dispatch/journey/router.rb  
  module Journey
    class Router
      class RoutingError < ::StandardError
      end
    
      attr_accessor :routes
        
      def initialize(routes)
        @routes = routes
      end
        
      def serve(req)
        find_routes(req).each do |match, parameters, route| # Let's step into 'find_routes'
          set_params  = req.path_parameters
          path_info   = req.path_info
          script_name = req.script_name
        
          unless route.path.anchored
            req.script_name = (script_name.to_s + match.to_s).chomp("/")
            req.path_info = match.post_match
            req.path_info = "/" + req.path_info unless req.path_info.start_with? "/"
          end
        
          parameters = route.defaults.merge parameters.transform_values { |val|
            val.dup.force_encoding(::Encoding::UTF_8)
          }
        
          req.path_parameters = set_params.merge parameters
        
          # 'route' is an instance of ActionDispatch::Journey::Route. 
          # 'route.app' is an instance of ActionDispatch::Routing::RouteSet::Dispatcher.
          status, headers, body = route.app.serve(req) # Let's step into method 'serve'
        
          if "pass" == headers["X-Cascade"]
            req.script_name     = script_name
            req.path_info       = path_info
            req.path_parameters = set_params
            next
          end
        
          return [status, headers, body]
        end
    
        [404, { "X-Cascade" => "pass" }, ["Not Found"]]
      end
      
      def find_routes(req)
        routes = filter_routes(req.path_info).concat custom_routes.find_all { |r|
          r.path.match(req.path_info)
        }

        routes =
          if req.head?
            match_head_routes(routes, req)
          else
            match_routes(routes, req)
          end

        routes.sort_by!(&:precedence)

        routes.map! { |r|
          match_data = r.path.match(req.path_info)
          path_parameters = {}
          match_data.names.zip(match_data.captures) { |name, val|
            path_parameters[name.to_sym] = Utils.unescape_uri(val) if val
          }
          [match_data, path_parameters, r]
        }
      end

    end
  end
end

# ./gems/actionpack-5.2.2/lib/action_dispatch/routing/route_set.rb
module ActionDispatch
  module Routing
    class RouteSet
      class Dispatcher < Routing::Endpoint
        def serve(req)
          params     = req.path_parameters # params: { action: 'index', controller: 'home' }
          controller = controller(req) # controller: HomeController
          # The definition of 'make_response!' is 
          # ActionDispatch::Response.create.tap { |res| res.request = request; }
          res        = controller.make_response!(req) 
          dispatch(controller, params[:action], req, res) # Let's step into this line.
        rescue ActionController::RoutingError
          if @raise_on_name_error
            raise
          else
            return [404, { "X-Cascade" => "pass" }, []]
          end
        end
        
        private
        
        def controller(req)
          req.controller_class
        rescue NameError => e
          raise ActionController::RoutingError, e.message, e.backtrace
        end

        def dispatch(controller, action, req, res)
          controller.dispatch(action, req, res) # Let's step into this line.
        end
      end
    end
  end    
end

# ./gems/actionpack-5.2.2/lib/action_controller/metal.rb
module ActionController
  class Metal < AbstractController::Base
    abstract!
    
    def self.controller_name
      @controller_name ||= name.demodulize.sub(/Controller$/, "").underscore
    end

    def self.make_response!(request)
      ActionDispatch::Response.new.tap do |res|
        res.request = request
      end
    end
    
    class_attribute :middleware_stack, default: ActionController::MiddlewareStack.new
    
    def self.inherited(base)
      base.middleware_stack = middleware_stack.dup
      super
    end
    
    # Direct dispatch to the controller. Instantiates the controller, then
    # executes the action named +name+.
    def self.dispatch(name, req, res)
      if middleware_stack.any?
        middleware_stack.build(name) { |env| new.dispatch(name, req, res) }.call req.env
      else
        # 'self' is HomeController, so for this line Rails will new a HomeController instance.  
        # Invoke `HomeController.ancestors`, you can find many superclasses of HomeController.
        # These are some typical superclasses of HomeController. 
        # HomeController
        # < ApplicationController
        # < ActionController::Base
        # < ActiveRecord::Railties::ControllerRuntime (module included)
        # < ActionController::Instrumentation (module included)
        # < ActionController::Rescue (module included)
        # < AbstractController::Callbacks (module included)
        # < ActionController::ImplicitRender (module included)
        # < ActionController::BasicImplicitRender (module included)
        # < ActionController::Renderers (module included) 
        # < ActionController::Rendering (module included) 
        # < ActionView::Layouts (module included) 
        # < ActionView::Rendering (module included)
        # < ActionDispatch::Routing::UrlFor (module included) 
        # < AbstractController::Rendering (module included)
        # < ActionController::Metal
        # < AbstractController::Base
        new.dispatch(name, req, res) # Let's step into this line.
      end
    end
    
    def dispatch(name, request, response)
      set_request!(request)
      set_response!(response)
      process(name) # Let's step into this line.
      request.commit_flash
      to_a
    end
    
    def to_a
      response.to_a
    end
  end
end

# .gems/actionpack-5.2.2/lib/abstract_controller/base.rb
module AbstractController
  class Base
    def process(action, *args)
      @_action_name = action.to_s

      unless action_name = _find_action_name(@_action_name)
        raise ActionNotFound, "The action '#{action}' could not be found for #{self.class.name}"
      end

      @_response_body = nil

      # action_name: 'index'
      process_action(action_name, *args) # Let's step into this line.
    end
  end
end

# .gems/actionpack-5.2.2/lib/action_controller/metal/instrumentation.rb
module ActionController
  module Instrumentation
    def process_action(*args)
      raw_payload = {
        controller: self.class.name,
        action: action_name,
        params: request.filtered_parameters,
        headers: request.headers,
        format: request.format.ref,
        method: request.request_method,
        path: request.fullpath
      }
    
      ActiveSupport::Notifications.instrument("start_processing.action_controller", raw_payload.dup)
    
      ActiveSupport::Notifications.instrument("process_action.action_controller", raw_payload) do |payload|
        begin
          # self: #<HomeController:0x00007fcd3c5dfd48>
          result = super # Let's step into this line.
          payload[:status] = response.status
          result
        ensure
          append_info_to_payload(payload)
        end
      end
    end
  end
end

# .gems/actionpack-5.2.2/lib/action_controller/metal/rescue.rb
module ActionController
  module Rescue
    def process_action(*args)
      super # Let's step into this line.
    rescue Exception => exception
      request.env["action_dispatch.show_detailed_exceptions"] ||= show_detailed_exceptions?
      rescue_with_handler(exception) || raise
    end
  end  
end

# .gems/actionpack-5.2.2/lib/abstract_controller/callbacks.rb
module AbstractController
  # = Abstract Controller Callbacks
  #
  # Abstract Controller provides hooks during the life cycle of a controller action.
  # Callbacks allow you to trigger logic during this cycle. Available callbacks are:
  #
  # * <tt>after_action</tt>
  # * <tt>before_action</tt>
  # * <tt>skip_before_action</tt>
  # * ...
  module Callbacks
    def process_action(*args)
      run_callbacks(:process_action) do
        # self: #<HomeController:0x00007fcd3c5dfd48>
        super # Let's step into this line.
      end
    end
  end
end

# .gems/actionpack-5.2.2/lib/action_controller/metal/rendering.rb
module ActionController
  module Rendering
    def process_action(*)
      self.formats = request.formats.map(&:ref).compact
      super # Let's step into this line.
    end
  end
end

# .gems/actionpack-5.2.2/lib/abstract_controller/base.rb
module AbstractController
  class Base
    def process_action(method_name, *args)
      # self: #<HomeController:0x00007fcd3c5dfd48>, method_name: 'index'
      # In the end, method 'send_action' is method 'send' by `alias send_action send` 
      send_action(method_name, *args)
    end

    alias send_action send
  end
end

# .gems/actionpack-5.2.2/lib/action_controller/metal/basic_implicit_render.rb
module ActionController
  module BasicImplicitRender
    def send_action(method, *args)
      # self: #<HomeController:0x00007fcd3c5dfd48>, method_name: 'index'    
      # Because 'send_action' is an alias of 'send',  
      # self.send('index', *args) will goto HomeController#index.
      x = super
      # performed?: false (for this example)
      x.tap { default_render unless performed? } # Let's step into 'default_render' later.
    end
  end
end

# ./your_project/app/controllers/home_controller.rb
class HomeController < ApplicationController
  # Will go back to BasicImplicitRender#send_action when method 'index' is done.
  def index
    # Question: How does the instance variable '@users' defined in HomeController can be accessed in './app/views/home/index.html.erb' ?
    # I will answer this question later. 
    @users = User.all.pluck(:id, :name)
  end
end
```

```html
# ./app/views/home/index.html.erb
<div class="container">
  <h1 class="display-4 font-italic">
    <%= t('home.banner_title') %>
    <%= @users %>
  </h1>
</div>
```

### Render view
As we see in `ActionController::BasicImplicitRender::send_action`, the last line is `default_render`.

So after `HomeController#index` is done, Ruby will execute method `default_render`.

```ruby
# .gems/actionpack-5.2.2/lib/action_controller/metal/implicit_render.rb 
module ActionController
  # Handles implicit rendering for a controller action that does not
  # explicitly respond with +render+, +respond_to+, +redirect+, or +head+.
  module ImplicitRender
    def default_render(*args)
      # Let's step into template_exists?
      if template_exists?(action_name.to_s, _prefixes, variants: request.variant)
        # Rails has found the default template './app/views/home/index.html.erb', so render it.
        render(*args) # Let's step into this line later
      #...
      else
        logger.info "No template found for #{self.class.name}\##{action_name}, rendering head :no_content" if logger
        super
      end
    end
  end
end

# .gems/actionview-5.2.2/lib/action_view/lookup_context.rb
module ActionView
  class LookupContext
    module ViewPaths
      # Rails checks whether the default template exists.
      def exists?(name, prefixes = [], partial = false, keys = [], **options)
        @view_paths.exists?(*args_for_lookup(name, prefixes, partial, keys, options))
      end
      alias :template_exists? :exists?
    end
  end  
end

# .gems/actionpack-5.2.2/lib/action_controller/metal/instrumentation.rb
module ActionController
  module Instrumentation
    def render(*args)
      render_output = nil
      self.view_runtime = cleanup_view_runtime do
        Benchmark.ms {
          # self: #<HomeController:0x00007fa7e9c54278> 
          render_output = super # Let's step into super 
        }
      end
      render_output
    end
  end  
end

# .gems/actionpack-5.2.2/lib/action_controller/metal/rendering.rb
module ActionController
  module Rendering
    # Check for double render errors and set the content_type after rendering.
    def render(*args)
      raise ::AbstractController::DoubleRenderError if response_body
      super # Let's step into super
    end
  end
end

# .gems/actionpack-5.2.2/lib/abstract_controller/rendering.rb
module AbstractController
  module Rendering
    # Normalizes arguments, options and then delegates render_to_body and
    # sticks the result in <tt>self.response_body</tt>.
    def render(*args, &block)
      options = _normalize_render(*args, &block)
      
      rendered_body = render_to_body(options) # Let's step into this line.
      
      if options[:html]
        _set_html_content_type
      else
        _set_rendered_content_type rendered_format
      end
      
      self.response_body = rendered_body
    end
  end
end

# .gems/actionpack-5.2.2/lib/action_controller/metal/renderers.rb
module ActionController
  module Renderers
    def render_to_body(options)
      _render_to_body_with_renderer(options) || super # Let's step into this line and 'super' later.
    end
    
    # For this example, this method return nil in the end.
    def _render_to_body_with_renderer(options)
      # The '_renderers' is defined at line 31: `class_attribute :_renderers, default: Set.new.freeze.`
      # '_renderers' is an instance predicate method. For more information, 
      # see ./gems/activesupport/lib/active_support/core_ext/class/attribute.rb  
      _renderers.each do |name|
        if options.key?(name)
          _process_options(options)
          method_name = Renderers._render_with_renderer_method_name(name)
          return send(method_name, options.delete(name), options)
        end
      end
      nil
    end
  end  
end

# .gems/actionpack-5.2.2/lib/action_controller/metal/renderers.rb
module ActionController
  module Rendering
    def render_to_body(options = {})
      super || _render_in_priorities(options) || " " # Let's step into 'super'
    end
  end
end
```

```ruby
# .gems/actionview-5.2.2/lib/action_view/rendering.rb
module ActionView
  module Rendering
    def render_to_body(options = {})
      _process_options(options)
      
      _render_template(options) # Let's step into this line.
    end
    
    def _render_template(options)
      variant = options.delete(:variant)
      assigns = options.delete(:assigns)
      
      context = view_context # We will step into this line later.
        
      context.assign assigns if assigns
      lookup_context.rendered_format = nil if options[:formats]
      lookup_context.variants = variant if variant
        
      view_renderer.render(context, options) # Let's step into this line.
    end
  end
end

# .gems/actionview-5.2.2/lib/action_view/renderer/renderer.rb
module ActionView
  class Renderer
    def render(context, options)
      if options.key?(:partial)
        render_partial(context, options)
      else
        render_template(context, options) # Let's step into this line.
      end
    end
    
    # Direct access to template rendering.
    def render_template(context, options)
      TemplateRenderer.new(@lookup_context).render(context, options) # Let's step into this line.
    end
  end
end

# .gems/actionview-5.2.2/lib/action_view/renderer/template_renderer.rb
module ActionView
  class TemplateRenderer < AbstractRenderer
    def render(context, options)
      @view    = context
      @details = extract_details(options)
      template = determine_template(options)

      prepend_formats(template.formats)

      @lookup_context.rendered_format ||= (template.formats.first || formats.first)

      render_template(template, options[:layout], options[:locals]) # Let's step into this line.
    end
    
    def render_template(template, layout_name = nil, locals = nil)
      view, locals = @view, locals || {}

      render_with_layout(layout_name, locals) do |layout| # Let's step into this line
        instrument(:template, identifier: template.identifier, layout: layout.try(:virtual_path)) do
          # template: #<ActionView::Template:0x00007f822759cbc0>
          template.render(view, locals) { |*name| view._layout_for(*name) } # Let's step into this line
        end
      end
    end
    
    def render_with_layout(path, locals)
      layout  = path && find_layout(path, locals.keys, [formats.first])
      content = yield(layout)

      if layout
        view = @view
        view.view_flow.set(:layout, content)
        layout.render(view, locals) { |*name| view._layout_for(*name) }
      else
        content
      end
    end
  end
end

# .gems/actionview-5.2.2/lib/action_view/template.rb
module ActionView
  class Template
    def render(view, locals, buffer = nil, &block)
      instrument_render_template do
        # self: #<ActionView::Template:0x00007f89bab1efb8 
        #           @identifier="/path/to/your/project/app/views/home/index.html.erb" 
        #           @source="<div class='container'\n ..."
        #        >
        compile!(view)
        # method_name: "_app_views_home_index_html_erb___3699380246341444633_70336654511160" (This method is defined in 'def compile(mod)' below)
        # view: #<#<Class:0x00007ff10d6c9d18>:0x00007ff10ea050a8>, view is an instance of <a subclass of ActionView::Base> which has same instance variables defined in the instance of HomeController.
        # You will get the result html after invoking 'view.send'. 
        view.send(method_name, locals, buffer, &block)
      end
    rescue => e
      handle_render_error(view, e)
    end
    
    # Compile a template. This method ensures a template is compiled
    # just once and removes the source after it is compiled.
    def compile!(view)
      return if @compiled

      # Templates can be used concurrently in threaded environments
      # so compilation and any instance variable modification must
      # be synchronized
      @compile_mutex.synchronize do
        # Any thread holding this lock will be compiling the template needed
        # by the threads waiting. So re-check the @compiled flag to avoid
        # re-compilation
        return if @compiled

        if view.is_a?(ActionView::CompiledTemplates)
          mod = ActionView::CompiledTemplates
        else
          mod = view.singleton_class
        end

        instrument("!compile_template") do
          compile(mod) # Let's step into this line.
        end

        # Just discard the source if we have a virtual path. This
        # means we can get the template back.
        @source = nil if @virtual_path
        @compiled = true
      end
    end
    
    def compile(mod)
      encode!
      # @handler: #<ActionView::Template::Handlers::ERB:0x00007ff10e1be188>
      code = @handler.call(self) # Let's step into this line.

      # Make sure that the resulting String to be eval'd is in the
      # encoding of the code
      source = <<-end_src.dup
        def #{method_name}(local_assigns, output_buffer)
          _old_virtual_path, @virtual_path = @virtual_path, #{@virtual_path.inspect};_old_output_buffer = @output_buffer;#{locals_code};#{code}
        ensure
          @virtual_path, @output_buffer = _old_virtual_path, _old_output_buffer
        end
      end_src

      # ...

      #  source:   def _app_views_home_index_html_erb___1187260686135140546_70244801399180(local_assigns, output_buffer)
      #              _old_virtual_path, @virtual_path = @virtual_path, "home/index";_old_output_buffer = @output_buffer;;
      #              @output_buffer = output_buffer || ActionView::OutputBuffer.new;
      #              @output_buffer.safe_append='<div class="container">
      #                <h1 class="display-4 font-italic">
      #                  '.freeze;
      #                   @output_buffer.append=( t('home.banner_title') );
      #                   @output_buffer.append=( @users ); 
      #                   @output_buffer.safe_append='
      #                </h1>
      #              </div>
      #              '.freeze;
      #              @output_buffer.to_s
      #            ensure
      #              @virtual_path, @output_buffer = _old_virtual_path, _old_output_buffer
      #            end
      mod.module_eval(source, identifier, 0) # This line will actually define the method '_app_views_home_index_html_erb___1187260686135140546_70244801399180'
      # mod: ActionView::CompiledTemplates
      ObjectSpace.define_finalizer(self, Finalizer[method_name, mod])
    end

# .gems/actionview-5.2.2/lib/action_view/template/handler/erb.rb     
    module Handlers
      class ERB
        def call(template)
          template_source = template.source.dup.force_encoding(Encoding::ASCII_8BIT)

          erb = template_source.gsub(ENCODING_TAG, "")
          encoding = $2

          erb.force_encoding valid_encoding(template.source.dup, encoding)

          # Always make sure we return a String in the default_internal
          erb.encode!

          self.class.erb_implementation.new(
            erb,
            escape: (self.class.escape_whitelist.include? template.type),
            trim: (self.class.erb_trim_mode == "-")
          ).src
        end
      end
    end
  end
end
```

### How can instance variables defined in Controller be accessed in view file?
It's time to answer the question before: 

How can instance variables like `@users` defined in `HomeController` be accessed in `./app/views/home/index.html.erb`?

I will answer this question by showing the source code below.
```ruby
# ./gems/actionview-5.2.2/lib/action_view/rendering.rb
module ActionView
  module Rendering
    def view_context
      # view_context_class is a subclass of ActionView::Base.
      view_context_class.new( # Let's step into this line later.
        view_renderer,
        view_assigns, # This line will set the instance variables like '@users' in this example. Let's step into this line.  
        self
      )
    end
    
    def view_assigns
      # self: #<HomeController:0x00007f83ecfed310>
      protected_vars = _protected_ivars
      # instance_variables is an instance method of class `Object` and it will return an array. And the array contains @users.  
      variables      = instance_variables 

      variables.reject! { |s| protected_vars.include? s }
      ret = variables.each_with_object({}) { |name, hash|
        hash[name.slice(1, name.length)] = instance_variable_get(name)
      }
      
      # ret: {"marked_for_same_origin_verification"=>true, "users"=>[[1, "Lane"], [2, "John"], [4, "Frank"]]}
      ret
    end
    
    def view_context_class
      # Will return a subclass of ActionView::Base.
      @_view_context_class ||= self.class.view_context_class
    end
    
    # How this ClassMethods works? Please look at ActiveSupport::Concern in ./gems/activesupport-5.2.2/lib/active_support/concern.rb
    # FYI, the method 'append_features' will be executed automatically before method 'included' executed. 
    # https://apidock.com/ruby/v1_9_3_392/Module/append_features  
    module ClassMethods
      def view_context_class
        # self: HomeController
        @view_context_class ||= begin
          supports_path = supports_path?
          routes  = respond_to?(:_routes)  && _routes
          helpers = respond_to?(:_helpers) && _helpers

          Class.new(ActionView::Base) do
            if routes
              include routes.url_helpers(supports_path)
              include routes.mounted_helpers
            end

            if helpers
              include helpers
            end
          end
        end
      end
    end
  end
end

# ./gems/actionview-5.2.2/lib/action_view/base.rb
module ActionView
  class Base
    def initialize(context = nil, assigns = {}, controller = nil, formats = nil)
      @_config = ActiveSupport::InheritableOptions.new

      if context.is_a?(ActionView::Renderer)
        @view_renderer = context
      else
        lookup_context = context.is_a?(ActionView::LookupContext) ?
          context : ActionView::LookupContext.new(context)
        lookup_context.formats  = formats if formats
        lookup_context.prefixes = controller._prefixes if controller
        @view_renderer = ActionView::Renderer.new(lookup_context)
      end

      @cache_hit = {}
      
      assign(assigns) # Let's step into this line.
      
      assign_controller(controller)
      _prepare_context
    end
    
    def assign(new_assigns)
      @_assigns = 
        new_assigns.each do |key, value|
          # This line will set the instance variables (like '@users') in HomeController to itself.
          instance_variable_set("@#{key}", value)
        end
    end
  end
end

```
After all Rack apps called, user will get the response.  

## Part 4: What does `$ rails server` do?

If you start Rails by `$ rails server`. You may want to know what does this command do? 

The command `rails` locates at `./bin/`.
```ruby
#!/usr/bin/env ruby
APP_PATH = File.expand_path('../config/application', __dir__)

require_relative '../config/boot'
require 'rails/commands' # Let's look at this file.
```  

```ruby
# ./railties-5.2.2/lib/rails/commands.rb
require "rails/command"

aliases = {
  "g"  => "generate",
  "d"  => "destroy",
  "c"  => "console",
  "s"  => "server",
  "db" => "dbconsole",
  "r"  => "runner",
  "t"  => "test"
}

command = ARGV.shift
command = aliases[command] || command # command is 'server'

Rails::Command.invoke command, ARGV # Let's step into this line.
```

```ruby
# ./railties-5.2.2/lib/rails/command.rb
module Rails
  module Command
    class << self
      def invoke(full_namespace, args = [], **config)
        # ...
        # command_name: 'server'  
        # After calling `find_by_namespace`, we will get this result:
        # command: Rails::Command::ServerCommand
        command = find_by_namespace(namespace, command_name)
        
        # Equals to: Rails::Command::ServerCommand.perform('server', args, config)
        command.perform(command_name, args, config)
      end
    end
  end
end
```

```ruby
# ./gems/railties-5.2.2/lib/rails/commands/server/server_command.rb
module Rails 
  module Command
    # There is a class method 'perform' in the Base class.
    class ServerCommand < Base
    end
  end
end
```

### Thor
Thor is a toolkit for building powerful command-line interfaces.

[https://github.com/erikhuda/thor](https://github.com/erikhuda/thor)

Inheritance relationship: `Rails::Command::ServerCommand < Rails::Command::Base < Thor`

```ruby
# ./gems/railties-5.2.2/lib/rails/command/base.rb
module Rails
  module Command
    class Base < Thor
      class << self
        # command: 'server'
        def perform(command, args, config)
          #...
          dispatch(command, args.dup, nil, config) # Thor.dispatch
        end
      end
    end
  end
end
```

```ruby
# ./gems/thor-0.20.3/lib/thor.rb
class Thor
  class << self
    # meth is 'server'
    def dispatch(meth, given_args, given_opts, config)
      # ...
      # Will new a Rails::Command::ServerCommand instance here 
      # because 'self' is Rails::Command::ServerCommand.
      instance = new(args, opts, config)
      # ...
      # Method 'invoke_command' is defined in Thor::Invocation. 
      # command: {Thor::Command}#<struct Thor::Command name="server" ...> 
      instance.invoke_command(command, trailing || [])
    end
  end
end

# ./gems/thor-0.20.3/lib/thor/invocation.rb
class Thor
  # FYI, this module is included in Thor.
  # And Thor is grandfather of Rails::Command::ServerCommand
  module Invocation
    def invoke_command(command, *args) # 'invoke_command' is defined at here.
      # ...
      # self: #<Rails::Command::ServerCommand:0x00007fdcc49791b0> 
      # command: {Thor::Command}#<struct Thor::Command name="server" ...> 
      command.run(self, *args)
    end
  end
end

# ./gems/thor-0.20.3/lib/thor/command.rb
class Thor
  class Command < Struct.new(:name, :description, :long_description, :usage, :options, :ancestor_name)
    def run(instance, args = [])
      # ...
      # instance: #<Rails::Command::ServerCommand:0x00007fdcc49791b0>
      # name: "server"
      # This line will invoke Rails::Command::ServerCommand#server, 
      # the instance method 'server' is defined in Rails::Command::ServerCommand implicitly.
      # I will show you how the instance method 'server' is implicitly defined. 
      instance.__send__(name, *args) 
    end  
  end
end
```

```ruby
# ./gems/thor-0.20.3/lib/thor.rb
class Thor
  # ...
  include Thor::Base # Will invoke hooked method 'Thor::Base.included(self)'
end

# ./gems/thor-0.20.3/lib/thor/base.rb
module Thor
  module Base
    class << self
      # 'included' is a hooked method.
      # When module 'Thor::Base' is included, method 'included' is executed. 
      def included(base) 
        # base: Thor
        # this line will define `Thor.method_added`.  
        base.extend ClassMethods
        # Module 'Invocation' is included for class 'Thor' here. 
        # Because Thor is grandfather of Rails::Command::ServerCommand,
        # 'invoke_command' will be instance method of Rails::Command::ServerCommand       
        base.send :include, Invocation # 'invoke_command' is defined in module Invocation
        base.send :include, Shell
      end
    end
    
    module ClassMethods
      # 'method_added' is a hooked method.
      # When an instance method is created in Rails::Command::ServerCommand, 
      # `method_added` will be executed.
      # So, when method `perform` is defined in Rails::Command::ServerCommand,
      # `method_added` will be executed and create_command('perform') will be invoked. 
      # So in the end, method 'server' will be created by alias_method('server', 'perform').
      # And the method 'server' is for the 'server' in command `$ rails server`.
      def method_added(meth)
        # ...
        # self: {Class} Rails::Command::ServerCommand 
        create_command(meth) # meth is 'perform'. Let's step into this line.
      end
    end
  end
end

# ./gems/railties-5.2.2/lib/rails/command/base.rb
module Rails
  module Command
    # Rails::Command::Base is superclass of Rails::Command::ServerCommand
    module Base
      class << self
        def create_command(meth)
          if meth == "perform"
            # Calling instance method 'server' of Rails::Command::ServerCommand 
            # will be transferred to call instance method 'perform'.
            alias_method('server', meth) 
          end
        end  
      end
    end
  end
end

# ./gems/thor-0.20.3/lib/thor/command.rb
class Thor
  class Command < Struct.new(:name, :description, :long_description, :usage, :options, :ancestor_name)
    def run(instance, args = [])
      #...
      # instance: {Rails::Command::ServerCommand}#<Rails::Command::ServerCommand:0x00007fa5f319bf40>
      # name: 'server'.
      # Will actually invoke 'instance.perform(*args)'. 
      # Equals to invoke Rails::Command::ServerCommand#perform(*args).
      # Let's step into  Rails::Command::ServerCommand#perform.
      instance.__send__(name, *args) 
    end
  end
end

# ./gems/railties-5.2.2/lib/rails/commands/server/server_command.rb
module Rails
  module Command
    class ServerCommand < Base
      # This is the method will be executed when `$ rails server`.
      def perform
        # ...
        Rails::Server.new(server_options).tap do |server|
          # APP_PATH is '/path/to/your_project/config/application'.
          # require APP_PATH will create the 'Rails.application' object.
          # 'Rails.application' is 'YourProject::Application.new'.
          # Rack server will start 'Rails.application'.
          require APP_PATH
          
          Dir.chdir(Rails.application.root)
          
          server.start # Let's step into this line.
        end
      end
    end
  end
end
```

### Rails::Server#start
```ruby
# ./gems/railties-5.2.2/lib/rails/commands/server/server_command.rb
module Rails
  class Server < ::Rack::Server
    def start
      print_boot_information
      
      trap(:INT) do
        exit
      end
      
      create_tmp_directories
      setup_dev_caching
      
      # This line is important. Although the method name seems not.
      log_to_stdout# Let step into this line.

      super # Will invoke ::Rack::Server#start. I will show you later. 
    ensure
      puts "Exiting" unless @options && options[:daemonize]
    end
    
    def log_to_stdout
      # 'wrapped_app' will get an well prepared Rack app from './config.ru' file.
      # It's the first time invoke 'wrapped_app'.
      # The app is an instance of YourProject::Application.
      # But the app is not created in 'wrapped_app'.
      # It has been created when `require APP_PATH` in previous code,
      # just at the 'perform' method in Rails::Command::ServerCommand.   
      wrapped_app # Let's step into this line

      # ...
    end
  end
end

# ./gems/rack-2.0.6/lib/rack/server.rb
module Rack
  class Server
    def wrapped_app
      @wrapped_app ||= 
        build_app(
          app # Let's step into this line.
        )
    end
    
    def app
      @app ||= build_app_and_options_from_config # Let's step into this line.
      @app
    end
    
    def build_app_and_options_from_config
      # ...
      # self.options[:config]: 'config.ru'. Let's step into this line.
      app, options = Rack::Builder.parse_file(self.options[:config], opt_parser)
      # ...
      app
    end
    
    # This method is called in Rails::Server#start
    def start(&blk)
      #...
      wrapped_app
      #...

      # server: {Module} Rack::Handler::Puma
      # wrapped_app: {YourProject::Application} #<YourProject::Application:0x00007f7fe5523f98>
      server.run(wrapped_app, options, &blk) # We will step into this line (Rack::Handler::Puma.run) later.
    end
  end
end

# ./gems/rack/lib/rack/builder.rb
module Rack
  module Builder
    def self.parse_file(config, opts = Server::Options.new)
      # config: 'config.ru'
      cfgfile = ::File.read(config)
      
      app = new_from_string(cfgfile, config)
      
      return app, options
    end
  
    # Let's guess what does 'run Rails.application' do in config.ru?
    # You may guess that: 
    #   Run YourProject::Application instance.
    # But 'run' maybe not what you are thinking about.
    # Because the 'self' object in 'config.ru' is #<Rack::Builder:0x00007f8c861ec278 @warmup=nil, @run=nil, @map=nil, @use=[]>, 
    # 'run' is an instance method of Rack::Builder.
    # Let's look at the definition of the 'run' method: 
    # def run(app)
    #   @run = app # Just set an instance variable for Rack::Builder instance.
    # end
    def self.new_from_string(builder_script, file="(rackup)")
      # Rack::Builder implements a small DSL to iteratively construct Rack applications.
      eval "Rack::Builder.new {\n" + builder_script + "\n}.to_app",
        TOPLEVEL_BINDING, file, 0
    end
  end
end
```

### Starting Puma
As we see in `Rack::Server#start`, there is `Rack::Handler::Puma.run(wrapped_app, options, &blk)`.

```ruby
# ./gems/puma-3.12.0/lib/rack/handler/puma.rb
module Rack
  module Handler
    module Puma
      # This method is invoked in `Rack::Server#start`:
      # Rack::Handler::Puma.run(wrapped_app, options, &blk) 
      def self.run(app, options = {})
        conf   = self.config(app, options)

        # ...
        launcher = ::Puma::Launcher.new(conf, :events => events)

        begin
          # Puma will run your app (instance of YourProject::Application)
          launcher.run # Let's step into this line.
        rescue Interrupt 
          puts "* Gracefully stopping, waiting for requests to finish"
          launcher.stop
          puts "* Goodbye!"
        end
      end
    end  
  end
end

# .gems/puma-3.12.0/lib/puma/launcher.rb
module Puma
  # Puma::Launcher is the single entry point for starting a Puma server based on user
  # configuration. It is responsible for taking user supplied arguments and resolving them
  # with configuration in `config/puma.rb` or `config/puma/<env>.rb`.
  #
  # It is responsible for either launching a cluster of Puma workers or a single
  # Puma server.
  class Launcher
    def initialize(conf, launcher_args={})
      @runner        = nil
      @config        = conf

      # ...
      if clustered?
        # ...
        @runner = Cluster.new(self, @events)
      else
        # For this example, it is Single.new.
        @runner = Single.new(self, @events)
      end
      
      # ...
    end
        
    def run
      #...

      # Set the behaviors for signals like `$ kill -s SIGTERM puma_process_id` received. 
      setup_signals # We will discuss this line later.
      
      set_process_title
      
      @runner.run # We will enter `Single.new(self, @events).run` here.

      case @status
      when :halt
        log "* Stopping immediately!"
      when :run, :stop
        graceful_stop
      when :restart
        log "* Restarting..."
        ENV.replace(previous_env)
        @runner.before_restart
        restart!
      when :exit
        # nothing
      end
    end
  end
end
```

```ruby
# .gems/puma-3.12.0/lib/puma/single.rb
module Puma
  # This class is instantiated by the `Puma::Launcher` and used
  # to boot and serve a Ruby application when no puma "workers" are needed
  # i.e. only using "threaded" mode. For example `$ puma -t 1:5`
  #
  # At the core of this class is running an instance of `Puma::Server` which
  # gets created via the `start_server` method from the `Puma::Runner` class
  # that this inherits from.
  class Single < Runner
    def run
      # ...

      # @server: Puma::Server.new(app, @launcher.events, @options)
      @server = server = start_server # Let's step into this line.

      # ...
      thread = server.run # Let's step into this line later.
      
      # This line will suspend the main thread execution.
      # And the `thread`'s block (which is method `handle_servers`) will be executed.
      # See `Thread#join` for more information.
      # I will show you a simple example for using `thread.join`.
      # Please search `test_thread_join.rb` in this document.
      thread.join
      
      # The below line will never be executed because `thread` is always running and `thread` has joined.
      # When `$ kill -s SIGTERM puma_process_id`, the below line will still not be executed
      # because the block of `Signal.trap "SIGTERM"` in `Puma::Launcher#setup_signals` will be executed.
      # If you remove the line `thread.join`, the below line will be executed, 
      # but the main thread will exit after all code executed and all the threads not joined will be killed.
      puts "anything which will never be executed..."
    end
  end
end
```
```ruby
# .gems/puma-3.12.0/lib/puma/runner.rb
module Puma
  # Generic class that is used by `Puma::Cluster` and `Puma::Single` to
  # serve requests. This class spawns a new instance of `Puma::Server` via
  # a call to `start_server`.
  class Runner
    def app
      @app ||= @launcher.config.app
    end
    
    def start_server
      min_t = @options[:min_threads]
      max_t = @options[:max_threads]

      server = Puma::Server.new(app, @launcher.events, @options)
      server.min_threads = min_t
      server.max_threads = max_t
      # ...

      server
    end
  end
end
```

```ruby
# .gems/puma-3.12.0/lib/puma/server.rb
module Puma
  class Server
    def run(background=true)
      #...
      @status = :run
      queue_requests = @queue_requests

      # This part is important.
      # Remember the block of ThreadPool.new will be called when a request added to the ThreadPool instance.
      # And the block will process the request by calling method `process_client`.
      # Let's step into this line later to see how Puma call the block.
      @thread_pool = ThreadPool.new(@min_threads,
                                    @max_threads,
                                    IOBuffer) do |client, buffer|

        # Advertise this server into the thread
        Thread.current[ThreadLocalKey] = self

        process_now = false

        if queue_requests
          process_now = client.eagerly_finish
        end  
        
        # ...
        if process_now
          # Process the request. You can look upon `client` as request.
          # If you want to know more about 'process_client', please read part 3 
          # or search 'process_client' in this document.  
          process_client(client, buffer)  
        else
          client.set_timeout @first_data_timeout
          @reactor.add client
        end
      end
      
      # ...

      if background # background: true (for this example)
        # This part is important.
        # Remember Puma created a thread here!
        # We will know that the newly created thread's job is waiting for requests.
        # When a request comes, the thread will transfer the request processing work to a thread in ThreadPool.
        # The method `handle_servers` in thread's block will be executed immediately 
        # (executed in the newly created thread, not in the main thread).
        @thread = Thread.new { handle_servers } # Let's step into this line to see what I said.
        return @thread
      else
        handle_servers
      end
    end
    
    def handle_servers      
      sockets = [check] + @binder.ios
      pool = @thread_pool
      queue_requests = @queue_requests

      # ...
      
      # The thread is always running, because @status has been set to :run in Puma::Server#run.
      # Yes, it should always be running to transfer the incoming requests.
      while @status == :run
        begin
          # This line will cause current thread waiting until a request arrives.
          # So it will be the entry of every request!
          # sockets: [#<IO:fd 23>, #<TCPServer:fd 22, AF_INET, 0.0.0.0, 3000>] 
          ios = IO.select sockets
        
          ios.first.each do |sock|
            if sock == check
              break if handle_check
            else
              if io = sock.accept_nonblock
                # You can simply look upon a Puma::Client instance as a request.  
                client = Client.new(io, @binder.env(sock))
                
                # ...
                
                # FYI, the method '<<' is redefined.
                # Add the request (client) to thread pool means 
                # a thread in the pool will process this request (client).   
                pool << client # Let's step into this line.
                
                pool.wait_until_not_full # Let's step into this line later.
              end
            end
          end
        rescue Object => e
          @events.unknown_error self, e, "Listen loop"
        end
      end
    end    
  end
end
```

```ruby
# .gems/puma-3.12.0/lib/puma/thread_pool.rb
module Puma
  class ThreadPool
    # Maintain a minimum of +min+ and maximum of +max+ threads
    # in the pool.
    #
    # The block passed is the work that will be performed in each
    # thread.
    #
    def initialize(min, max, *extra, &block)
      #..
      @mutex = Mutex.new
      @todo = [] # @todo is requests (in Puma, they are Puma::Client instances) which need to be processed.  
      @spawned = 0 # the count of @spawned threads
      @min = Integer(min) # @min threads count
      @max = Integer(max) # @max threads count
      @block = block # block will be called in method `spawn_thread` to processed a request.    
      @workers = []
      @reaper = nil

      @mutex.synchronize do
        @min.times { spawn_thread } # Puma spawns @min count threads.
      end
    end
    
    def spawn_thread
      @spawned += 1

      # Create a new Thread now.
      # The block of the thread will be executed immediately and separately from the calling thread (main thread).
      th = Thread.new(@spawned) do |spawned|
        # Thread name is new in Ruby 2.3
        Thread.current.name = 'puma %03i' % spawned if Thread.current.respond_to?(:name=)
        block = @block
        mutex = @mutex
        #...

        extra = @extra.map { |i| i.new }

        # Pay attention to here: 
        # 'while true' means this part will always be running.
        # And there will be @min count threads always running!
        # Puma uses these threads to process requests. 
        # The line: 'not_empty.wait(mutex)' will make current thread waiting.  
        while true
          work = nil

          continue = true

          mutex.synchronize do
            while todo.empty?
              if @trim_requested > 0
                @trim_requested -= 1
                continue = false
                not_full.signal
                break
              end

              if @shutdown
                continue = false
                break
              end

              @waiting += 1 # `@waiting` is the waiting threads count.
              not_full.signal
              
              # This line will cause current thread waiting
              # until `not_empty.signal` executed in some other place to wake it up .
              # Actually, `not_empty.signal` is located at `def <<(work)` in the same file.
              # You can search `def <<(work)` in this document.
              # Method `<<` is used in method `handle_servers`: `pool << client` in Puma::Server#run.
              # `pool << client` means add a request to the thread pool, 
              # and then the waked up thread will process the request. 
              not_empty.wait mutex
              
              @waiting -= 1
            end

            # `work` is the request (in Puma, it's Puma::Client instance) which need to be processed.
            work = todo.shift if continue  
          end

          break unless continue

          if @clean_thread_locals
            ThreadPool.clean_thread_locals
          end

          begin
            # `block.call` will switch program to the block definition part.
            # The block definition part is in `Puma::Server#run`:
            # @thread_pool = ThreadPool.new(@min_threads,
            #                               @max_threads,
            #                               IOBuffer) do |client, buffer| #...; end
            # So please search `ThreadPool.new` in this document to look back.
            block.call(work, *extra)
          rescue Exception => e
            STDERR.puts "Error reached top of thread-pool: #{e.message} (#{e.class})"
          end
        end

        mutex.synchronize do
          @spawned -= 1
          @workers.delete th
        end
      end # end of the Thread.new.

      @workers << th

      th
    end
    
    def wait_until_not_full
      @mutex.synchronize do
        while true
          return if @shutdown

          # If we can still spin up new threads and there
          # is work queued that cannot be handled by waiting
          # threads, then accept more work until we would
          # spin up the max number of threads.
          return if @todo.size - @waiting < @max - @spawned

          @not_full.wait @mutex 
        end
      end
    end
    
    # Add +work+ to the todo list for a Thread to pickup and process.
    def <<(work)
      @mutex.synchronize do
        if @shutdown
          raise "Unable to add work while shutting down"
        end
        
        # work: #<Puma::Client:0x00007ff114ece6b0>
        # You can look upon Puma::Client instance as a request.
        @todo << work

        if @waiting < @todo.size and @spawned < @max
          spawn_thread # Create one more thread to process request. 
        end

        # Wake up the waiting thread to process the request.
        # The waiting thread is defined in the same file: Puma::ThreadPool#spawn_thread.
        # This code is in `spawn_thread`:
        # while true 
        #   # ...
        #   not_empty.wait mutex
        #   # ... 
        #   block.call(work, *extra) # This line will process the request.
        # end 
        @not_empty.signal  
      end
    end
  end
end
```

### Conclusion
In conclusion, `$ rails server` will execute `Rails::Command::ServerCommand#perform`. 

In `#perform`, call `Rails::Server#start`. Then call `Rack::Server#start`.

Then call `Rack::Handler::Puma.run(YourProject::Application.new)`.

In `.run`, Puma will new a always running Thread for `ios = IO.select(#<TCPServer:fd 22, AF_INET, 0.0.0.0, 3000>)`.

Request is created from `ios` object.

A thread in Puma threadPool will process the request.

The thread will invoke Rack apps' `call` to get the response for the request.

### Exiting Puma
#### Process and Thread
Because Puma is using multiple threads, we need to have some basic concepts about Process and Thread.

This link is good for you to obtain the concepts: [Process and Thread](https://stackoverflow.com/questions/4894609/will-a-cpu-process-have-at-least-one-thread)  

In the next part, you will often see `thread.join`.

I will use two simple example to tell what does `thread.join` do.

##### Example one
Try to run `test_thread_join.rb`. 

```ruby
# ./test_thread_join.rb
thread = Thread.new() do
  3.times do |n|
    puts "~~~~ " + n.to_s
  end
end

# sleep 1
puts "==== I am the main thread."

# thread.join # Try to uncomment these two lines to see the differences.
# puts "==== after thread.join"
```
You will find that if there is no `thread.join`, you can see 
```log
==== I am the main thread.
==== after thread.join
~~~~ 0
~~~~ 1
~~~~ 2
```
in console.

After you added `thread.join`, you can see:
```log
==== I am the main thread.
~~~~ 0
~~~~ 1
~~~~ 2
==== after thread.join
````
in console.

##### Example two
Try to run `test_thread_join2.rb`. 
```ruby
# ./test_thread_join2.rb
arr = [
  Thread.new do
    puts 'I am arr[0]'
    sleep 1
    puts 'After arr[0]'
  end,
  Thread.new do
    puts 'I am arr[1]'
    sleep 5
    puts 'After arr[1]'
  end,
  Thread.new do
    puts 'I am arr[2]'
    sleep 8
    puts 'After arr[2]'
  end
]

puts "Thread.list.size:  #{Thread.list.size}" # returns 4 (including the main thread)

sleep 2

arr.each { |thread| puts "~~~~~ #{thread}" }

puts "Thread.list.size:  #{Thread.list.size}" # returns 3 (because arr[0] is dead)

arr[1].join # uncomment to see differences

arr.each { |thread| puts "~~~~~ #{thread}" }

sleep 7
puts "Exit main thread"
```

#### Send `SIGTERM` to Puma
When you stop Puma by running `$ kill -s SIGTERM puma_process_id`, you will enter `setup_signals` in `Puma::Launcher#run`.
```ruby
# .gems/puma-3.12.0/lib/puma/launcher.rb
module Puma
  # Puma::Launcher is the single entry point for starting a Puma server based on user
  # configuration.
  class Launcher
    def run
      #...

      # Set the behaviors for signals like `$ kill -s SIGTERM puma_process_id`. 
      setup_signals # Let's step into this line.
      
      set_process_title
      
      @runner.run

      # ...
    end
    
    # Set the behaviors for signals like `$ kill -s SIGTERM puma_process_id`.
    # Signal.list #=> {"EXIT"=>0, "HUP"=>1, "INT"=>2, "QUIT"=>3, "ILL"=>4, "TRAP"=>5, "IOT"=>6, "ABRT"=>6, "FPE"=>8, "KILL"=>9, "BUS"=>7, "SEGV"=>11, "SYS"=>31, "PIPE"=>13, "ALRM"=>14, "TERM"=>15, "URG"=>23, "STOP"=>19, "TSTP"=>20, "CONT"=>18, "CHLD"=>17, "CLD"=>17, "TTIN"=>21, "TTOU"=>22, "IO"=>29, "XCPU"=>24, "XFSZ"=>25, "VTALRM"=>26, "PROF"=>27, "WINCH"=>28, "USR1"=>10, "USR2"=>12, "PWR"=>30, "POLL"=>29}
    # Press `Control + C` to quit means 'SIGINT'.
    def setup_signals
      begin
        # After running `$ kill -s SIGTERM puma_process_id`, Ruby will execute the block of `Signal.trap "SIGTERM"`.
        Signal.trap "SIGTERM" do
          graceful_stop # Let's step into this line.

          raise SignalException, "SIGTERM"
        end
      rescue Exception
        log "*** SIGTERM not implemented, signal based gracefully stopping unavailable!"
      end
            
      begin
        Signal.trap "SIGUSR2" do
          restart
        end
      rescue Exception
        log "*** SIGUSR2 not implemented, signal based restart unavailable!"
      end

      begin
        Signal.trap "SIGUSR1" do
          phased_restart
        end
      rescue Exception
        log "*** SIGUSR1 not implemented, signal based restart unavailable!"
      end

      begin
        Signal.trap "SIGINT" do
          if Puma.jruby?
            @status = :exit
            graceful_stop
            exit
          end

          stop
        end
      rescue Exception
        log "*** SIGINT not implemented, signal based gracefully stopping unavailable!"
      end

      begin
        Signal.trap "SIGHUP" do
          if @runner.redirected_io?
            @runner.redirect_io
          else
            stop
          end
        end
      rescue Exception
        log "*** SIGHUP not implemented, signal based logs reopening unavailable!"
      end
    end
    
    def graceful_stop
      # @runner: instance of Puma::Single (for this example)
      @runner.stop_blocked # Let's step into this line.
      log "=== puma shutdown: #{Time.now} ==="
      log "- Goodbye!"
    end
  end
end

# .gems/puma-3.12.0/lib/puma/launcher.rb
module Puma
  class Single < Runner
    def run
      # ...

      # @server: Puma::Server.new(app, @launcher.events, @options)
      @server = server = start_server # Let's step into this line.

      # ...
      thread = server.run
      
      # This line will suspend the main thread execution.
      # And the `thread`'s block (which is the method `handle_servers`) will be executed.  
      thread.join
    end
      
    def stop_blocked
      log "- Gracefully stopping, waiting for requests to finish"
      @control.stop(true) if @control
      # @server: instance of Puma::Server
      @server.stop(true) # Let's step into this line
    end
  end
end

# .gems/puma-3.12.0/lib/puma/server.rb
module Puma
  class Server
    def initialize(app, events=Events.stdio, options={})
      # 'Puma::Util.pipe' returns `IO.pipe`.
      @check, @notify = Puma::Util.pipe # @check, @notify is a pair.

      @status = :stop
    end
      
    def run(background=true)
      # ...
      @thread_pool = ThreadPool.new(@min_threads,
                                    @max_threads,
                                    IOBuffer) do |client, buffer|

        #...
        # Process the request.
        process_client(client, buffer)  
        #...
      end
      
      # 'Thread.current.object_id' returns '70144214949920', 
      # which is the same as the 'Thread.current.object_id' in Puma::Server#stop.
      # Current thread is the main thread here.
      puts "#{Thread.current.object_id}"
      
      # The created @thread is the @thread in `stop` method below.
      @thread = Thread.new {
        # FYI, this is in the Puma starting process.      
        # 'Thread.current.object_id' returns '70144220123860', 
        # which is the same as the 'Thread.current.object_id' in 'handle_servers' in Puma::Server#run
        # def handle_servers
        #   begin
        #     # ...
        #   ensure
        #     # FYI, the 'ensure' part is in the Puma stopping process.
        #     puts "#{Thread.current.object_id}" # returns '70144220123860' too.
        #   end
        # end
        puts "#{Thread.current.object_id}" # returns '70144220123860'
        
        handle_servers
      }
      return @thread
    end
      
    # Stops the acceptor thread and then causes the worker threads to finish
    # off the request queue before finally exiting.
    def stop(sync=false)
      # This line will set '@status = :stop',
      # and cause `ios = IO.select sockets` (in method `handle_servers`) to return result.
      # So that the code after `ios = IO.select sockets` will be executed.   
      notify_safely(STOP_COMMAND) # Let's step into this line.
      
      # 'Thread.current.object_id' returns '70144214949920', 
      # which is the same as the 'Thread.current.object_id' in Puma::Server#run.
      # Current thread is exactly the main thread here.
      puts "#{Thread.current.object_id}"
      
      # The @thread is just the always running Thread created in `Puma::Server#run`.
      # Please look at method `Puma::Server#run`.
      # `@thread.join` will suspend the main thread execution.
      # And the code in @thread will continue be executed.
      @thread.join if @thread && sync
    end
    
    def notify_safely(message)
      @notify << message
    end
    
    def handle_servers
      begin
        check = @check
        # sockets: [#<IO:fd 23>, #<TCPServer:fd 22, AF_INET, 0.0.0.0, 3000>]
        sockets = [check] + @binder.ios
        pool = @thread_pool
        #...
 
        while @status == :run
          # After `notify_safely(STOP_COMMAND)` in main thread, `ios = IO.select sockets` will return result.
          # FYI, `@check, @notify = IO.pipe`.
          # def notify_safely(message)
          #   @notify << message
          # end
          # sockets: [#<IO:fd 23>, #<TCPServer:fd 22, AF_INET, 0.0.0.0, 3000>]
          ios = IO.select sockets
            
          ios.first.each do |sock|
            if sock == check
              # The @status is updated to :stop for this example in `handle_check`.
              break if handle_check # Let's step into this line.
            else
              if io = sock.accept_nonblock
                client = Client.new(io, @binder.env(sock))
                
                # ...
                pool << client
                pool.wait_until_not_full
              end
            end
          end
        end
        
        # Let's step into `graceful_shutdown`.
        graceful_shutdown if @status == :stop || @status == :restart
        
      # ...
      ensure
        # FYI, the 'ensure' part is in the Puma stopping process.
        # 'Thread.current.object_id' returns '70144220123860', 
        # which is the same as the 'Thread.current.object_id' in 'Thread.new block' in Puma::Server#run
        # @thread = Thread.new do
        #   # FYI, this is in the Puma starting process.
        #   puts "#{Thread.current.object_id}" # returns '70144220123860'
        #   handle_servers
        # end
        puts "#{Thread.current.object_id}"
          
        @check.close
        @notify.close

        # ...
      end
    end
    
    def handle_check
      cmd = @check.read(1)

      case cmd
      when STOP_COMMAND
        @status = :stop # The @status is updated to :stop for this example. 
        return true
      when HALT_COMMAND
        @status = :halt
        return true
      when RESTART_COMMAND
        @status = :restart
        return true
      end

      return false
    end
    
    def graceful_shutdown
      if @thread_pool
        @thread_pool.shutdown # Let's step into this line.
      end
    end    
  end
end
```

```ruby
module Puma
  class ThreadPool
    # Tell all threads in the pool to exit and wait for them to finish.
    def shutdown(timeout=-1)
      threads = @mutex.synchronize do
        @shutdown = true
        # `broadcast` will wakes up all threads waiting for this lock.
        @not_empty.broadcast
        @not_full.broadcast

        # ...
        
        # dup workers so that we join them all safely
        # @workers is an array.   
        # @workers.dup will not create new thread.
        # @workers is an instance variable and will be changed when shutdown (by `@workers.delete th`). 
        # So ues @workers.dup here.
        @workers.dup
      end

      # Wait for threads to finish without force shutdown.
      threads.each do |thread|
        thread.join
      end

      @spawned = 0
      @workers = []
    end

    def initialize(min, max, *extra, &block)
      #..
      @mutex = Mutex.new
      @spawned = 0 # The count of @spawned threads.
      @todo = [] # @todo is requests (in Puma, it's Puma::Client instance) which need to be processed.  
      @min = Integer(min) # @min threads count
      @block = block # block will be called in method `spawn_thread` to process a request.    
      @workers = []

      @mutex.synchronize do
        @min.times { spawn_thread } # Puma spawns @min count threads.
      end
    end
    
    def spawn_thread
      @spawned += 1

      # Run a new Thread now.
      # The block of the thread will be executed separately from the calling thread. 
      th = Thread.new(@spawned) do |spawned|
        block = @block
        mutex = @mutex
        #...
 
        while true
          work = nil

          continue = true

          mutex.synchronize do
            while todo.empty?
              # ...
 
              if @shutdown
                continue = false
                break
              end

              # ...
              # After `@not_empty.broadcast` is executed in '#shutdown', `not_empty` is waked up.
              # Ruby will continue to execute the next line here.
              not_empty.wait mutex
              
              @waiting -= 1
            end

            # ...
          end

          break unless continue

          # ...
        end

        mutex.synchronize do
          @spawned -= 1
          @workers.delete th
        end
      end # end of the Thread.new.

      @workers << th

      th
    end
  end
end
```

So all the threads in the ThreadPool joined and finished.

Let's inspect the caller in block of `Signal.trap "SIGTERM"` below.

```ruby
# .gems/puma-3.12.0/lib/puma/launcher.rb
module Puma
  # Puma::Launcher is the single entry point for starting a Puma server based on user
  # configuration.
  class Launcher
    def run
      #...

      # Set the behaviors for signals like `$ kill -s SIGTERM puma_process_id`. 
      setup_signals # Let's step into this line.
      
      set_process_title
      
      # Process.pid: 42264 
      puts "Process.pid: #{Process.pid}"
      
      @runner.run

      # ...
    end
    
    def setup_signals
      # ...
      begin
        # After running `$ kill -s SIGTERM puma_process_id`, Ruby will execute the block of `Signal.trap "SIGTERM"`.
        Signal.trap "SIGTERM" do
          # I inspect `caller` to see the caller stack.
          # caller: [
          #   "../gems/puma-3.12.0/lib/puma/single.rb:118:in `join'", 
          #   "../gems/puma-3.12.0/lib/puma/single.rb:118:in `run'",
          #   "../gems/puma-3.12.0/lib/puma/launcher.rb:186:in `run'",
          #   "../gems/puma-3.12.0/lib/rack/handler/puma.rb:70:in `run'",
          #   "../gems/rack-2.0.6/lib/rack/server.rb:298:in `start'",
          #   "../gems/railties-5.2.2/lib/rails/commands/server/server_command.rb:55:in `start'", 
          #   "../gems/railties-5.2.2/lib/rails/commands/server/server_command.rb:149:in `block in perform'", 
          #   "../gems/railties-5.2.2/lib/rails/commands/server/server_command.rb:144:in `tap'", 
          #   "../gems/railties-5.2.2/lib/rails/commands/server/server_command.rb:144:in `perform'", 
          #   "../gems/thor-0.20.3/lib/thor/command.rb:27:in `run'", 
          #   "../gems/thor-0.20.3/lib/thor/invocation.rb:126:in `invoke_command'", 
          #   "../gems/thor-0.20.3/lib/thor.rb:391:in `dispatch'", 
          #   "../gems/railties-5.2.2/lib/rails/command/base.rb:65:in `perform'", 
          #   "../gems/railties-5.2.2/lib/rails/command.rb:46:in `invoke'", 
          #   "../gems/railties-5.2.2/lib/rails/commands.rb:18:in `<top (required)>'", 
          #   "../path/to/your_project/bin/rails:5:in `require'", 
          #   "../path/to/your_project/bin/rails:5:in `<main>'"
          # ]
          puts "caller: #{caller.inspect}"
          
          # Process.pid: 42264 which is the same as the `Process.pid` in the Puma::Launcher#run.
          puts "Process.pid: #{Process.pid}"
          
          graceful_stop
          
          # This SignalException is not rescued in the caller stack.
          # So in the the caller stack, Ruby will goto the `ensure` part in
          # "../gems/railties-5.2.2/lib/rails/commands/server/server_command.rb:55:in `start'".
          # So the last code executed is `puts "Exiting" unless @options && options[:daemonize]`
          # when running `$ kill -s SIGTERM puma_process_id`.
          # You can search `puts "Exiting"` in this document to see it.
          raise SignalException, "SIGTERM"
        end
      rescue Exception
        # This `rescue` is only for `Signal.trap "SIGTERM"`, not for `raise SignalException, "SIGTERM"`.
        log "*** SIGTERM not implemented, signal based gracefully stopping unavailable!"
      end
    end
  end
end
```

Welcome to point out the mistakes in this article :)

