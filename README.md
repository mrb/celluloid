Celluloid
=========
[![Build Status](http://travis-ci.org/tarcieri/celluloid.png)](http://travis-ci.org/tarcieri/celluloid)

> "I thought of objects being like biological cells and/or individual 
> computers on a network, only able to communicate with messages"  
> _--Alan Kay, creator of Smalltalk, on the meaning of "object oriented programming"_

Celluloid is a concurrent object framework for Ruby inspired by Erlang and the
Actor model. Celluloid gives you thread-backed objects that run concurrently,
providing the simplicity of Ruby objects for the most common use cases, but
also the ability to call methods _asynchronously_, allowing the receiver to do
things in the background while the caller carries on with its business.
These concurrent objects are called "actors". Actors are somewhere in between
the kind of object you're typically used to working with and a network service.

Supported Platforms
-------------------

Celluloid works on Ruby 1.9.2, JRuby 1.6 (in 1.9 mode), and Rubinius 2.0. JRuby
or Rubinius are the preferred platforms as they support true concurrent threads.

To use JRuby in 1.9 mode, you'll need to pass the "--1.9" command line option
to the JRuby executable, or set the "JRUBY_OPTS=--1.9" environment variable.

Celluloid works on Rubinius in either 1.8 or 1.9 mode.

Usage
-----

To use Celluloid, define a normal Ruby class that includes Celluloid:

    class Sheen
      include Celluloid
  
      def initialize(name)
        @name = name
      end
  
      def set_status(status)
        @status = status
      end
      
      def report
        "#{@name} is #{@status}"
      end
    end
    
Now when you create new instances of this class, they're actually concurrent
objects, each running in their own thread:

    >> charlie = Sheen.new "Charlie Sheen"
     => #<Celluloid::Actor(Sheen:0x00000100a312d0) @name="Charlie Sheen">
    >> charlie.set_status "winning!"
     => "winning!" 
    >> charlie.report
     => "Charlie Sheen is winning!" 
    >> charlie.set_status! "asynchronously winning!"
     => nil 
    >> charlie.report
     => "Charlie Sheen is asynchronously winning!" 

You can call methods on this concurrent object just like you would any other 
Ruby object. The Sheen#set_status method works exactly like you'd expect,
returning the last expression evaluated.

However, Celluloid's secret sauce kicks in when you call banged predicate
methods (i.e. methods ending in !). Even though the Sheen class has no 
set_status! method, you can still call it. Why is this? Because bang methods
have a special meaning in Celluloid. (Note: this also means you can't define
bang methods on Celluloid classes and expect them to be callable from other
objects)

Adding a bang to the end of a method instructs Celluloid that you would like
for the given method to be called _asynchronously_. This means that rather
than the caller waiting for a response, the caller sends a message to the
concurrent object that you'd like the given method invoked, and then the
caller proceeds without waiting for a response. The concurrent object
receiving the message will then process the method call in the background.

Adding a bang to a method name is a convention in Ruby used to indicate that
the method is in some way "dangerous", and in Celluloid this is no exception.
You have no guarantees that just because you made an asynchronous call it was
ever actually invoked. Asynchronous calls will never raise an exception, even
if an exception occurs when the receiver is processing it. Worse, unhandled
exceptions will crash the receiver, and making an asynchronous call to a
crashed object will not raise an error.

However, you can still handle errors created by asynchronous calls using
two features of Celluloid called _supervisors_ and _linking_. See the
corresponding sections below for more information.

Futures
-------

Futures allow you to request a computation and get the result later. There are
two types of futures supported by Celluloid: method futures and block futures.
Method futures work by invoking the _future_ method on an actor. This method
is analogous to the typical _send_ method in that it takes a method name, 
followed by an arbitrary number of arguments, and a block. Let's invoke the
report method from the charlie object used in the above example using a future:

	>> future = charlie.future :report
	 => #<Celluloid::Future:0x000001009759b8>
	>> future.value
	 => "Charlie Sheen is winning!"

The call to charlie.future immediately returns a Celluloid::Future object,
regardless of how long it takes to execute the "report" method. To obtain
the result of the call to "report", we call the _value_ method of the
future object. This call will block until the value returned from the method 
call is available (i.e. the method has finished executing). If an exception
occured during the method call, the call to future.value will reraise the
same exception.

Futures also allow you to background the computation of any block:

	>> future = Celluloid::Future.new { 2 + 2 }
	 => #<Celluloid::Future:0x000001008425f0> 
	>> future.value
	 => 4 

One thing to be aware of when using futures: always make sure to obtain the
value of any future you make. Futures create a thread in the background which
will continue to run until the future's value is obtained. Failing to obtain
the value of futures you create will leak threads.

Supervisors
-----------

You may be familiar with tools like Monit or God which keep an eye on your
applications and restart them when they crash. Celluloid supervisors work in
a similar fashion, except instead of monitoring applications, they monitor
individual actors and restart them when they crash. Crashes occur whenever
an unhandled exception is raised anywhere within an actor.

To supervise an actor, start it with the _supervise_ method. Using the Sheen
class from the example above:

    >> supervisor = Sheen.supervise "Charlie Sheen"
     => #<Celluloid::Supervisor(Sheen) "Charlie Sheen">
 
This created a new Celluloid::Supervisor actor, and also created a new Sheen
actor, giving its initialize method the argument "Charlie Sheen". The 
_supervise_ method has the same method signature as _new_. However, rather
than returning the newly created actor, _supervise_ returns the supervisor.
To retrieve the actor that the supervisor is currently using, use the
Celluloid::Supervisor#actor method:

    >> supervisor = Sheen.supervise "Charlie Sheen"
     => #<Celluloid::Supervisor(Sheen) "Charlie Sheen">
    >> charlie = supervisor.actor
     => #<Celluloid::Actor(Sheen:0x00000100a312d0)>
     
Supervisors can also automatically put actors into the actor _registry_ using
the supervise_as method:

    >> Sheen.supervise_as :charlie, "Charlie Sheen"
     => #<Celluloid::Supervisor(Sheen) "Charlie Sheen">
    >> charlie = Celluloid::Actor[:charlie]
     => #<Celluloid::Actor(Sheen:0x00000100a312d0)>
 
In this case, the supervisor will ensure that an actor of the Sheen class,
created using the given arguments, is aways available by calling
Celluloid::Actor[:charlie]. The first argument to supervise_as is the name
you'd like the newly created actor to be registered under. The remaining
arguments are passed to initialize just like you called _new_.

See the "Registry" section below for more information on the actor registry

Linking
-------

Whenever any unhandled exceptions occur in any of the methods of an actor,
that actor crashes and dies. Let's start with an example:

    class JamesDean
      include Celluloid
      class CarInMyLaneError < StandardError; end
      
      def drive_little_bastard
        raise CarInMyLaneError, "that guy's gotta stop. he'll see us"
      end
    end
    
Now, let's have James drive Little Bastard and see what happens:

    >> james = JamesDean.new
     => #<Celluloid::Actor(JamesDean:0x1068)> 
    >> james.drive_little_bastard!
     => nil 
    >> james
     => #<Celluloid::Actor(JamesDean:0x1068) dead> 
    
When we told james asynchronously to drive Little Bastard, it killed him! If
we were Elizabeth Taylor, co-star in James' latest film at the time of his
death, we'd certainly want to know when he died. So how can we do that?

Actors can _link_ to other actors they're interested in and want to receive
crash notifications from. In order to receive these events, we need to use the
trap_exit method to be notified of them. Let's look at how a hypothetical
Elizabeth Taylor object could be notified that James Dean has crashed:

    class ElizabethTaylor
      include Celluloid
      trap_exit :actor_died
  
      def actor_died(actor, reason)
        puts "Oh no! #{actor.inspect} has died because of a #{reason.class}"
      end
    end

We've now used the trap_exit method to configure a callback which is invoked
whenever any linked actors crashed. Now we need to link Elizabeth to James so
James' crash notifications get sent to her:

    >> james = JamesDean.new
     => #<Celluloid::Actor(JamesDean:0x11b8)> 
    >> elizabeth = ElizabethTaylor.new
     => #<Celluloid::Actor(ElizabethTaylor:0x11f0)> 
    >> elizabeth.link james
     => #<Celluloid::Actor(JamesDean:0x11b8)> 
    >> james.drive_little_bastard!
     => nil 
    Oh no! #<Celluloid::Actor(JamesDean:0x11b8) dead> has died because of a JamesDean::CarInMyLaneError

Elizabeth called the _link_ method to receive crash events from James. Because
Elizabeth was linked to James, when James crashed, James' exit message was
sent to her. Because Elizabeth was trapping the exit messages she received
using the trap_exit method, the callback she specified was invoked, allowing
her to take action (in this case, printing the error). But what would happen
if she weren't trapping exits? Let's break James apart into two separate
objects, one for James himself and one for Little Bastard, his car:

    class PorscheSpider
      include Celluloid
      class CarInMyLaneError < StandardError; end
  
      def drive_on_route_466
        raise CarInMyLaneError, "head on collision :("
      end
    end
    
    class JamesDean
      include Celluloid
      
      def initialize
        @little_bastard = PorscheSpider.new_link
      end
      
      def drive_little_bastard
        @little_bastard.drive_on_route_466
      end
    end
    
If you take a look in JamesDean#initialize, you'll notice that to create an
instance of PorcheSpider, James is calling the new_link method.

This method works similarly to _new_, except it combines _new_ and _link_ 
into a single call.

Now what happens if we repeat the same scenario with Elizabeth Taylor watching
for James Dean's crash?

    >> james = JamesDean.new
     => #<Celluloid::Actor(JamesDean:0x1108) @little_bastard=#<Celluloid::Actor(PorscheSpider:0x10ec)>> 
    >> elizabeth = ElizabethTaylor.new
     => #<Celluloid::Actor(ElizabethTaylor:0x1144)> 
    >> elizabeth.link james
     => #<Celluloid::Actor(JamesDean:0x1108) @little_bastard=#<Celluloid::Actor(PorscheSpider:0x10ec)>> 
    >> james.drive_little_bastard!
     => nil 
    Oh no! #<Celluloid::Actor(JamesDean:0x1108) dead> has died because of a PorscheSpider::CarInMyLaneError

When Little Bastard crashed, it killed James as well. Little Bastard killed 
James, and because Elizabeth was trapping James' exit events, she received the 
notification of James' death.

Actors that are linked together propagate their error messages to all other 
actors that they're linked to. Unless those actors are trapping exit events,
those actors too will die, like James did in this case. If you have many, 
many actors linked together in a large object graph, killing one will kill them
all unless they are trapping exits.

This allows you to factor your problem into several actors. If an error occurs
in any of them, it will kill off all actors used in a particular system. In
general, you'll probably want to have a supervisor start a single actor which
is in charge of a particular part of your system, and have that actor 
new_link to other actors which are part of the same system. If any error
occurs in any of these actors, all of them will be killed off and the entire
subsystem will be restarted by the supervisor in a clean state.

If, for any reason, you've linked to an actor and want to sever the link,
there's a corresponding _unlink_ method to remove links between actors.

Registry
--------

Celluloid lets you register actors so you can refer to them symbolically.
You can register Actors using Celluloid::Actor[]:

    >> james = JamesDean.new
     => #<Celluloid::Actor(JamesDean:0x80c27ce0)> 
    >> Celluloid::Actor[:james] = james
     => #<Celluloid::Actor(JamesDean:0x80c27ce0)> 
    >> Celluloid::Actor[:james]
     => #<Celluloid::Actor(JamesDean:0x80c27ce0)> 

The Celluloid::Actor constant acts as a hash, allowing you to register actors
under the name of your choosing, and access actors by name rather than
reference. This is important because actors may crash. If you're attempting to
reference an actor explicitly by storing it in a variable, you may be holding
onto a reference to a crashed copy of that actor, rather than talking to a
working, freshly-restarted version.

The main use of the registry is for interfacing with actors that are
automatically restarted by supervisors when they crash.

Signaling
---------

Signaling is an advanced technique similar to condition variables in typical
multithreaded programming. One method within a concurrent object can suspend
itself waiting for a particular event, allowing other methods to run. Another
method can then signal all methods waiting for a particular event, and even
send them a value in the process:

	class SignalingExample
	  include Celluloid
	  attr_reader :signaled
  
	  def initialize
	    @signaled = false
	  end
  
	  def wait_for_signal
	    value = wait :ponycopter
	    @signaled = true
	    value
	  end
  
	  def send_signal(value)
	    signal :ponycopter, value
	  end
	end
	
The wait_for_signal method in turn calls a method called "wait". Wait suspends
the running method until another method of the same object calls the "signal"
method with the same label.

The send_signal method of this class does just that, signaling "ponycopter"
with the given value. This value is returned from the original wait call.

Logging
-------

By default, Celluloid will log any errors and backtraces from any crashing
actors to STDOUT. However, if you wish you can use any logger which is
compatible with the standard Ruby Logger API. For example, if you're using
Celluloid within a Rails application, you'll probably want to do:

    Celluloid.logger = Rails.logger
 
Contributing to Celluloid
-------------------------
 
* Fork Celluloid on github
* Make your changes and send me a pull request
* If I like them I'll merge them and give you commit access to my repository

Copyright
---------

Copyright (c) 2011 Tony Arcieri. See LICENSE.txt for further details.
