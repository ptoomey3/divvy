divvy - parallel script runner
==============================

This is a (forking) parallel task runner for Ruby designed to be run in the
foreground and to require no external infrastructure components (like redis or a
queue server).

Divvy provides a light system for defining parallelizable pieces of work and a
process based run environment for executing them. It's good for running coarse
grained tasks that are network or IO heavy. It's not good at crunching lots of
inputs quickly or parallelizing fine grained / CPU intense pieces of work.

GitHub uses divvy with [ModelIterator](https://github.com/technoweenie/model_iterator)
to perform one-off and regular maintenance tasks on different types of records
and their associated storage components.

## example

This is a simple and contrived example of a divvy job script. You must define a
class that includes the `Divvy::Parallelizable` module and implement the
`#dispatch` and `#process` methods. There are also hooks available for tapping
into the worker process lifecycle.

``` ruby
# This is a dance party. We're going to hand out tickets. We need to generate
# codes for each available ticket. Thing is, the ticket codes have to be
# generated by this external ticket code generator service (this part is
# just pretend) and there's a lot of latency involved. We can generate multiple
# ticket codes at the same time by making multiple connections.
require 'divvy'
require 'digest/sha1' # <-- your humble ticket code generator service

class DanceParty
  # The Parallelizable module provides default method implementations and marks
  # the object as following the interface defined below.
  include Divvy::Parallelizable

  # This is the main loop responsible for generating work items for worker
  # processes. It runs in the master process only. Each item yielded from this
  # method is marshalled over a pipe and distributed to the next available
  # worker process where it arrives at the #process method (see below).
  #
  # In this example we're just going to generate a series of numbers to pass
  # to the workers. The workers just write the number out with their pid and the
  # SHA1 hex digest of the number given.
  def dispatch
    tickets_available = ARGV[0] ? ARGV[0].to_i : 10
    puts "Generating #{tickets_available} ticket codes for the show..."
    (0...tickets_available).each do |ticket_number|
      yield ticket_number
    end
  end

  # The individual work item processing method. Each item produced by the
  # dispatch method is sent to this method in the worker processes. The
  # arguments to this method must match the arity of the work item yielded
  # from the #dispatch method.
  #
  # In this example we're given a Fixnum ticket number and asked to produce a
  # code. Pretend this is a network intense operation where you're mostly
  # sleeping waiting for a reply.
  def process(ticket_number)
    ticket_sha1 = Digest::SHA1.hexdigest(ticket_number.to_s)
    printf "%5d %6d %s\n" % [$$, ticket_number, ticket_sha1]
    sleep 0.150 # fake some latency
  end

  # Hook called after a worker process is forked off from the master process.
  # This runs in the worker process only. Typically used to re-establish
  # connections to external services or open files (logs and such).
  def after_fork(worker)
    # warn "In after_fork for worker #{worker.number}"
  end

  # Hook called before a worker process is forked off from the master process.
  # This runs in the master process only. This can be used to monitor the rate
  # at which workers are being created or to set a starting process state for
  # the newly forked process.
  def before_fork(worker)
    # warn "In before_fork for worker #{worker.number}"
  end
end
```

### divvy command

You can run the example script above with the `divvy` command, which includes
options for controlling concurrency and other cool stuff. Here we use five
worker processes to generate 10 dance party ticket codes:

```
$ divvy -n 5 danceparty.rb
51589        0 b6589fc6ab0dc82cf12099d1c2d40ab994e8410c
51590        1 356a192b7913b04c54574d18c28d46e6395428ab
51589        4 1b6453892473a467d07372d45eb05abc2031647a
51590        5 ac3478d69a3c81fa62e60f5c3696165a4e5e6ac4
51591        2 da4b9237bacccdf19c0760cab7aec4a8359010b0
51589        6 c1dfd96eea8cc2b62785275bca38ac261256e278
51592        3 77de68daecd823babbb58edb1c8e14d7106e83bb
51590        8 fe5dbbcea5ce7e2988b8c69bcfdfde8904aabc1f
51591        9 0ade7c2cf97f75d009975f4d720d1fa6c19f4897
51593        7 902ba3cda1883801594b6e1b452790cc53948fda
```

The columns of output are the worker pid, the ticket number input, and the
generated ticket code result. You can see items are distributed between
available workers evenly-ish and may not be processed in order.

### manual runner

You can also turn the current ruby process into a divvy master by creating a
`Divvy::Master` object, passing an instance of `Parallelizable` and the amount
of desired concurrency:

``` ruby
require 'danceparty'
task = DanceParty.new
master = Divvy::Master.new(task, 10)
master.run
```
