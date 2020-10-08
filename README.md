# Mcrain

Mcrain helps you to use docker container in test cases.
It supports redis, rabbitmq and riak (stand alone node or clustering) currently.

## Prerequisite

### With docker-machine

- [install docker into Mac](https://docs.docker.com/installation/mac/)
- [install docker into Windows](https://docs.docker.com/installation/windows/)


### Without docker-machine

The docker daemon must be started with tcp socket option like `-H tcp://0.0.0.0:2375`.
Because mcrain uses [Docker Remote API](https://docs.docker.com/reference/api/docker_remote_api/).

After [installing docker](https://docs.docker.com/installation/#installation),
edit the configuration file `/etc/default/docker` for Debian or Ubuntu,
or `/etc/sysconfig/docker` for CentOS. 

And add tcp option to DOCKER_OPTS like this:

```
DOCKER_OPTS="-H unix:///var/run/docker.sock -H tcp://0.0.0.0:2375"
```

Then restart the docker daemon.


Set `DOCKER_HOST` environment variable for mcrain.
```
export DOCKER_HOST='tcp://127.0.0.1:2375'
```

The port num must be equal to the port of tcp option in DOCKER_OPTS.

See the following documents for more information:
- https://docs.docker.com/reference/commandline/daemon/
- https://docs.docker.com/articles/networking/


## Installation

Add this line to your application's Gemfile:

```ruby
gem 'mcrain'
```

And then execute:

    $ bundle

Or install it yourself as:

    $ gem install mcrain

### with middleware clients

middleware | client gem (CRuby)  | client gem (JRuby)
-----------|---------------------|------------------
MySQL      | `gem 'mysql2'`      | N/A
Redis      | `gem 'redis'`       | (the same as CRuby)
RabbitMQ   | `gem 'rabbitmq_http_api_client', '>= 1.6.0'`       | (the same as CRuby)
Riak       | `gem 'docker-api', '~> 1.21.1'; gem 'riak-client'` | (the same as CRuby)
HBase      | N/A                 | `gem 'hbase-jruby'`


## Usage

### configure images

```ruby
Mcrain.configure do |config|
  config.images[:mysql] = "mysql:5.6"
  config.images[:redis] = "redis:3.2-alpine"
  # config.images[:rabbitmq] = ...
  # config.images[:riak] = ...
  # config.images[:hbase] = ...
end
```

Or put `.mcrain.yml` file with following content

```yaml
---
images:
  mysql: "mysql:5.6"
  redis: "redis:3.2-alpine"
  # rabbitmq: ...
  # riak: ...
  # hbase: ...
```

and load it by `Mcrain.load_config`:

```ruby
Mcrain.load_config "/path/to/.mcrain.yml"
```

`mcrain` command accepts `-c` (`--config`) option to configure images by yaml file, and its default is `.mcrain.yml`.

### redis in code

```ruby
Mcrain::Redis.new.start do |s|
  c = s.client # Redis::Client object
  c.ping
end
```

### rabbitmq in code

```ruby
Mcrain::Rabbitmq.new.start do |s|
  c = s.client # RabbitMQ::HTTP::Client object
  c.list_nodes
end
```

### riak in code

Mcrain::Riak uses [hectcastro/docker-riak](https://github.com/hectcastro/docker-riak).

```ruby
Mcrain::Riak.new.start do |s|
  c = s.client # Riak::Client object
  obj = c.bucket("bucket1").get_or_new("foo")
  obj.data = data
  obj.store
end
```

### hbase in code

Mcrain::Hbase uses [nerdammer/hbase](https://hub.docker.com/r/nerdammer/hbase).

Add a line like this to `/etc/hosts`

|      |       |
|------|-------|
| With docker toolbox    | `192.168.99.100 docker-host1` |
| Without docker toolbox | `127.0.0.1 docker-host1` |


```ruby
Mcrain::Hbase.new.start do |s|
  c = s.client # HBase object defined by hbase-jruby
  c.list
  c[:my_table].create! :f
  c[:my_table].put 100, 'f:a' => 1, 'f:b' => 'two', 'f:c' => 3.14
  c[:my_table].get(100).double('f:c') #=> 3.14
end
```


### redis in terminal

```
$ mcrain start redis
To connect:
require 'redis'
client = Redis.new({:host=>"192.168.59.103", :port=>50669})
OK

$ mcrain stop redis
OK
```

### rabbitmq in terminal

```
$ mcrain start rabbitmq
To connect:
require 'rabbitmq/http/client'
client = RabbitMQ::HTTP::Client.new(*["http://192.168.59.103:50684", {:username=>"guest", :password=>"guest"}])
OK

$ mcrain stop rabbitmq
OK
```

### riak in terminal

```
$ mcrain start riak
To connect:
require 'riak'
client = Riak::Client.new({:nodes=>[{:host=>"192.168.59.103", :pb_port=>33152}]})
OK

$ mcrain stop riak
OK
```


```
$ export DOCKER_RIAK_PATH=/path/to/docker-riak
$ mcrain start riak 5
To connect:
require 'riak'
client = Riak::Client.new({:nodes=>[{:host=>"192.168.59.103", :pb_port=>33162}, {:host=>"192.168.59.103", :pb_port=>33160}, {:host=>"192.168.59.103", :pb_port=>33158}, {:host=>"192.168.59.103", :pb_port=>33157}, {:host=>"192.168.59.103", :pb_port=>33155}]})
OK

$ mcrain stop riak 5
OK
```



### hbase in terminal

Add a line like this to `/etc/hosts`

|      |       |
|------|-------|
| With docker toolbox    | `192.168.99.100 docker-host1` |
| Without docker toolbox | `127.0.0.1 docker-host1` |


```
$ mcrain start hbase
(snip)
To connect:
$CLASSPATH << "/Users/akima/.mcrain/hbase/hbase-client-dep-1.0.jar"
$LOAD_PATH << "hbase-jruby/lib"
require 'hbase-jruby'
client = HBase.new(*[{"hbase.zookeeper.quorum"=>"192.168.99.100", "hbase.zookeeper.property.clientPort"=>54489, "hbase.master.port"=>60000, "hbase.master.info.port"=>54490, "hbase.regionserver.port"=>60020, "hbase.regionserver.info.port"=>54491}])
OK

$ mcrain stop riak
(snip)
86a8dd6c13cd2c346fe9111e16f97265cb4fdb67cc67873c495622a28f0c1062
OK
```

use irb or something in JRuby.



## Mcrain.before_setup

Use Mcrain.before_setup hook if you don't want your test or spec always works with mcrain.
Set block to Mcrain.before_setup like this:

```ruby
unless ENV['WITH_MCRAIN'] =~ /true|yes|on|1/i
  Mcrain.before_setup = ->(s){
    # RSpec::Core::Pending#skip
    # https://github.com/rspec/rspec-core/blob/5fc29a15b9af9dc1c9815e278caca869c4769767/lib/rspec/core/pending.rb#L118-L124
    message = "skip examples which uses mcrain"
    current_example = RSpec.current_example
    RSpec::Core::Pending.mark_skipped!(current_example, message) if current_example
    raise RSpec::Core::Pending::SkipDeclaredInExample.new(message)
  }
end
```




## Development

After checking out the repo, run `bin/setup` to install dependencies. Then, run `bin/console` for an interactive prompt that will allow you to experiment. Run `bundle exec mcrain` to use the code located in this directory, ignoring other installed copies of this gem.

To install this gem onto your local machine, run `bundle exec rake install`. To release a new version, update the version number in `version.rb`, and then run `bundle exec rake release` to create a git tag for the version, push git commits and tags, and push the `.gem` file to [rubygems.org](https://rubygems.org).

## Contributing

1. Fork it ( https://github.com/groovenauts/mcrain/fork )
2. Create your feature branch (`git checkout -b my-new-feature`)
3. Commit your changes (`git commit -am 'Add some feature'`)
4. Push to the branch (`git push origin my-new-feature`)
5. Create a new Pull Request
