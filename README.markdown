# Spawner

## Dan's Note
------------------------------

I forked this from Tra's spawn project because it felt like a rename was a better
approach to dealing with the problems of Ruby 1.9 name collision. Also, "spawner"
as a name made more sense. A spawner spawns... like a driver drives.

### _NOTE: I've also added an alias so Spawner.run will work the same as Spawner.spawn_

I also incorporated some of the cleanup that rfc2822 included: https://github.com/rfc2822/spawn
I just didn't like "spawn_block". I liked the idea of calling it a Spawner.

This was also an exercise in learning how to set up a github repo, how to set it
as a gem, and how to incorporate it into a Rails project.

------------------------------

This plugin provides a *Spawner.spawn* method to easily fork OR thread long-running sections of
code so that your application can return results to your users more quickly.
This plugin works by creating new database connections in ActiveRecord::Base for the
spawned block.

_(lib/patches.rb has been merged functionally and thus removed. D#)_

## Installation

To install this gem (Rails 3+) from the master branch (recommended), add
the following line to your Gemfile:

     gem "spawner", :git => 'git://github.com/drsharp/spawner.git'

If you want to install the gem from the 'edge' branch (latest development):

    gem "spawner", :git => 'git://github.com/drsharp/spawner.git', :branch => :edge

## Usage

_You now call it with explicit Module prefixing for all methods!_
(this keeps it clear, explicit, and avoids method space clutter/collision)

Here's a simple example of how to demonstrate the spawner plugin.
In one of your controllers, insert this code (after installing the plugin of course):

     Spawner.spawn do
       logger.info("I feel sleepy...")
       sleep 11
       logger.info("Time to wake up!")
     end

You can also use the aliased method run():

     Spawner.run do
       ...
     end


If everything is working correctly, your controller should finish quickly then you'll see
the last log message several seconds later.

If you need to wait for the spawned processes/threads, then pass the objects returned by
spawner to wait(), like this:

    spawner_ids = []
    10.times do |i|
      # spawn 10 blocks of code
      spawner_ids[i] = Spawner.spawn do
        something(i)
      end
    end
    # wait for all N blocks of code to finish running
    Spawner.wait(spawner_ids)

## Options

The options you can pass to spawner are:

<table>
  <tr><th>Option</th><th>Values</th></tr>
  <tr><td>:method</td><td>:fork, :thread, :yield</td></tr>
  <tr><td>:nice</td><td>integer value 0-19, 19 = really nice</td></tr>
  <tr><td>:kill</td><td>boolean value indicating whether the parent should kill the spawned process
   when it exits (only valid when :method => :fork)</td></tr>
  <tr><td>:argv</td><td>string to override the process name</td></tr>
</table>

Any option to spawner can be set as a default so that you don't have to pass them in
to every call of spawn.   To configure the spawner default options, add a line to
your configuration file(s) like this:

    Spawner::default_options {:method => :thread}

If you don't set any default options, the :method will default to :fork.  To
specify different values for different environments, add the default_options call to
he appropriate environment file (development.rb, test.rb).   For testing you can set
the default :method to :yield so that the code is run inline.

    # in environment.rb
    Spawner::method :method => :fork, :nice => 7
    # in test.rb, will override the environment.rb setting
    Spawner::method :method => :yield

This allows you to set your production and development environments to use different
methods according to your needs.

### be nice

If you want your forked child to run at a lower priority than the parent process, pass in
the :nice option like this:

    Spawner.spawn(:nice => 7) do
      do_something_nicely
    end

### fork me

By default, spawner will use the fork to spawn child processes.  You can configure it to
do threading either by telling the spawn method when you call it or by configuring your
environment.
For example, this is how you can tell spawn to use threading on the call,

    Spawner.spawn(:method => :thread) do
      something
    end

For older versions of Rails (1.x), when using the :thread setting, spawner will check to
make sure that you have set allow_concurrency=true in your configuration.   If you
want this setting then put this line in one of your environment config files:

    config.active_record.allow_concurrency = true

If it is not set, then spawn will raise an exception.

### kill or be killed

Depending on your application, you may want the children processes to go away when
the parent  process exits.   By default spawner lets the children live after the
parent dies.   But you can tell it to kill the children by setting the :kill option
to true.

### a process by any other name

If you'd like to be able to identify which processes are spawned by looking at the
output of ps then set the :argv option with a string of your choice.
You should then be able to see this string as the process name when
listing the running processes (ps).

For example, if you do something like this,

    3.times do |i|
      Spawner.spawn(:argv => "spawner -#{i}-") do
        something(i)
      end
    end

then in the shell,

    $ ps -ef | grep spawner
    502  2645  2642   0   0:00.01 ttys002    0:00.02 spawner -0-
    502  2646  2642   0   0:00.02 ttys002    0:00.02 spawner -1-
    502  2647  2642   0   0:00.02 ttys002    0:00.03 spawner -2-

The length of the process name may be limited by your OS so you might want to experiment
to see how long it can be (it may be limited by the length of the original process name).

## Running the tests

Make sure you install *rspec 2* or greater. Then, it's simply a matter of a command like the following:

    rspec spec/spawner_integration_spec.rb  --color --format=documentation

_Of course, this assumes you've git cloned this repo and are in the *spawner/* directory._

## Forking vs. Threading

There are several tradeoffs for using threading vs. forking.   Forking was chosen as the
default primarily because it requires no configuration to get it working out of the box.

Forking advantages:

- more reliable? - the ActiveRecord code is generally not deemed to be thread-safe.
  Even though spawn attempts to patch known problems with the threaded implementation,
  there are no guarantees.  Forking is heavier but should be fairly reliable.
- keep running - this could also be a disadvantage, but you may find you want to fork
  off a process that could have a life longer than its parent.  For example, maybe you
  want to restart your server without killing the spawned processes.
  We don't necessarily condone this (i.e. haven't tried it) but it's technically possible.
- easier - forking works out of the box with spawn, threading requires you set
  allow_concurrency=true (for older versions of Rails).
  Also, beware of automatic reloading of classes in development
  mode (config.cache_classes = false).

Threading advantages:
- less filling - threads take less resources... how much less?  it depends.   Some
  flavors of Unix are pretty efficient at forking so the threading advantage may not
  be as big as you think... but then again, maybe it's more than you think.  ;-)
- debugging - you can set breakpoints in your threads

## Acknowledgements

This plugin was initially inspired by Scott Persinger's blog post on how to use fork
in rails for background processing.
    http://geekblog.vodpod.com/?p=26

Further inspiration for the threading implementation came from Jonathon Rochkind's
blog post on threading in rails.
    http://bibwild.wordpress.com/2007/08/28/threading-in-rails/

Also thanks to all who have helped debug problems and suggest improvements
including:

-  Ahmed Adam, Tristan Schneiter, Scott Haug, Andrew Garfield, Eugene Otto, Dan Sharp,
  Olivier Ruffin, Adrian Duyzer, Cyrille Labesse

-  Garry Tan, Matt Jankowski (Rails 2.2.x fixes), Mina Naguib (Rails 2.3.6 fix)

-  Tim Kadom, Mauricio Marcon Zaffari, Danial Pearce, Hongli Lai, Scott Wadden
  (passenger fixes)

-  &lt;your name here&gt;

Copyright (c) 2007-present Tom Anderson (tom@squeat.com), see LICENSE
