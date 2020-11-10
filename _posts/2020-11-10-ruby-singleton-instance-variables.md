---
layout: post
title: Singletons and instance varibles
---

Understanding how `class << self` works and its interaction with the `@instance_var`
and the big trouble it can cause.

<!--more-->

## Introduction

Sometimes we find ourselves needing to use **class methods** in Ruby, and sometimes,
we end up with needing to split said code into smaller methods - thus ending up
requiring _private_ class methods. The widely used pattern for doing that is using
`class << self` - however the way this works **is not** as how most people would
expect, and this creates a very big opportunity for _race conditions_.

## How _class << self_ syntax works

Say we have the below snippet of code:

```ruby
class MyTestClass
  class << self
    def run(jid, object)
      @object = object
      puts @object
      # do complicated computations with @object
    end
  end
end
```

The way the `class << self` syntax works is by essentially creating a singleton of your class.
The code with no syntactic sugar would look something like this:

```ruby
class MyTestClass
  include Singleton

  def self.run(jid, object)
    MyTestClasss.instance.run(jid, object)
  end

  def run
    @object = object
    puts @object
    # do complicated computations with @object
  end
end
```

Notice the presence of **singleton** - this is how the `class << self` pattern works. This
means that our instance variable `@object` will actually be shared by _all_ callers of the
method `run`.

## So why is this a problem?

If you run `MyTestClass` in a multi-threaded context - serving requests, background workers,
etc - you are essentially leaking state between one job/request to another. We noticed
this problem on one of our very high frequency jobs - but it took us a while to isolate,
since the functionality of the code works, and tests were passing!

## Example test case

We came up with the following test to prove that this was the problem code:

```ruby
require 'spec_helper'

class Logger
  def self.info(str)
    puts "INFO: #{str}"
  end
end

class MyTestClass
  class << self
    def run(jid, object)
      Logger.info("JID: #{jid} - I'm running")
      @object = object
      Logger.info("JID: #{jid} - my object #{object} - instance var #{@object}")
      sleep(1)
      Logger.info("JID: #{jid} - my object #{object} - instance var #{@object}")
    end
  end
end

RSpec.describe do
  it "Should produce inconsistent values" do
    allow(Logger).to receive(:info)
    t = []
    t << Thread.new { MyTestClass.run("job-1", value: 123) }
    t << Thread.new { MyTestClass.run("job-2", value: 456) }
    t.each(&:join)
    expect(Logger).to have_received(:info).with("JID: job-1 - I'm running").once
    expect(Logger).to have_received(:info).with("JID: job-2 - I'm running").once
    expect(Logger).to have_received(:info).with(
      "JID: job-1 - my object {:value=>123} - instance var {:value=>123}",
    ).twice

    # Correct statement
    expect(Logger).to have_received(:info).with(
      "JID: job-2 - my object {:value=>456} - instance var {:value=>456}",
    ).once

    # Incorrect statement - due to the state of job1 leaking into job2.
    expect(Logger).to have_received(:info).with(
      "JID: job-2 - my object {:value=>456} - instance var {:value=>123}",
    ).once
  end
end
```


## Conclusion

If you need to use class methods - do **think twice**. At most you should use class
methods to spawn an instance and call methods on the instance - this way you avoid
a whole lot of pain with having to manage `self.` methods and private state.

What I've learnt is that **you can never have too much instrumentation/logging for your
jobs**. This was also very hard to decipher as all of the code was printing the instance
variable value, so it looked fairly consistent - it took us digging into specific
thread level job execution logs and isolating them to figure out the issue.

Another thing we've learnt is that the bug was much more present when we had more jobs
running, and thus more parallelisation - and since peak times were usually after office
hours, it meant that when we were live tailing logs we were a lot more unlikely to see
the problem.

Overall, this was a bug that was eating at me for more than 2 weeks and I'm so happy
to have it put behind me!


