---
layout: post
title: Resque Signals
category: resque
---
[Resque](http://github.com/defunkt/resque) is a small redis backed queuing library written in Ruby by [Chris Wanstrath](http://chriswanstrath.com/). Using Resque, you can create background jobs that can be placed in multiple queues to be processed later. It also allows you to monitor your queues, workers, and job failures using a [Sinatra](http://www.sinatrarb.com/) web app. Chris does an excellent job explaining the history and reasoning for the project in his [original blog post](https://github.com/blog/542-introducing-resque).

In January 2012, I took over maintenance of the project. Over the last few months, we've been working on ironing out the last major issues in [Resque 1.x](http://github.com/defunkt/resque/tree/1-x-stable). One of the last holdouts is the way [POSIX signals](http://unixhelp.ed.ac.uk/CGI/man-cgi?signal+7) are handled. For those unfamiliar, signals are a form of interprocess communication to indicate a certain event or state change. Using signals, you can pause a running process, tell it to shutdown, or notify it that a certain time limit has passed.

## Problem

For signal handling, Resque borrows from nginx (from the [README](https://github.com/defunkt/resque/blob/1-x-stable/README.markdown#signals)):

* [QUIT](http://en.wikipedia.org/wiki/SIGQUIT) - Wait for child to finish processing then exit
* [TERM](http://en.wikipedia.org/wiki/SIGTERM) / INT - Immediately kill child then exit

The current implementation differs from standard UNIX philosophy on this. Traditionally, you use SIGQUIT to exit immediately and core dump. SIGTERM is then used to ask a process to terimnate nicely giving it a chance to clean itself up.

If you treat the resque worker process like any other unix process, you'd probably send a SIGTERM to the worker to ask it to shutdown. Historically, this will [immediately kill the child using SIGKILL](https://github.com/defunkt/resque/blob/0286e695402179661552897392c7221e23350181/lib/resque/worker.rb#L308). Just a note, the child process is where the job work is being performed. If you're in the middle of a long running job, you'll have no way to run any clean up code. At Heroku, we use resque to handle background work for dealing with [database backups](https://devcenter.heroku.com/articles/pgbackups). This signal handling won't work since we need to mark this job as failed in our databases, so the user knows the backup never finished.

## Solution

In Resque 1.22.0, we've addressed this issue by adding [a new signal handling path](https://github.com/defunkt/resque/blob/de26a891253be9f9642685918cb0e81f16ff992c/lib/resque/worker.rb#L349-366) that will be the default in Resque 2. The old way is deprecated as of this release. Since, we're held to semantic versioning, the old way is still the default. We don't want people upgrading minor versions to have their apps break all of sudden in case they're dependent on the old functionality. In order to opt-in you just need to set the environment variable `TERM_CHILD`.

{% highlight sh %}
$ TERM_CHILD=1 QUEUES=* rake resque:work
{% endhighlight %}

The changes contain two parts:
1. reverting the traps from the parent in the child
2. a new signal flow for SIGTERM

Reverting the traps from the parent allows the child to receive signals properly. This means that sending SIGQUIT will actually cause the child process to quit immediately.

With these changes the new signal flow is the following:

* you send SIGTERM to the worker
* the parent worker sends SIGTERM to the child process
* the parent worker waits for a period of time for the job to exit successfully
* if the child is still running, the parent worker sends SIGKILL to the child
* the parent worker then exits

The advantage of this is it allows the job to be notified of being killed. In the database example above, we can now write our job this way.

{% highlight ruby %}
require 'resque/errors'

class DatabaseBackupJob
  def self.perform
    # do backup work
  rescue Resque::TermException
    # write failure to database
  end
end
{% endhighlight %}

Rescuing `Resque::TermException` allows us to have a block of code that executes during that "wait period" for cleanup purposes. If you don't want to `require 'resque/errors'` in your job, you can just rescue the traditional `SignalException` that ruby throws.

{% highlight ruby %}
class DatabaseBackupJob
  def self.perform
    # do backup work
  rescue SignalException
    # write failure to database
  end
end
{% endhighlight %}

The "wait period" is customizable with the environment variable `RESQUE_TERM_TIMEOUT` and its unit is in seconds. By default, we use 4 seconds.

{% highlight sh %}
$ TERM_CHILD=1 RESQUE_TERM_TIMEOUT=10 QUEUES=* rake resque:work
{% endhighlight %}

## Example

So let's a look at how this works. Here's a really simple job I'm using:

{% highlight ruby %}
require 'resque/errors'

class SleepJob
  @queue = :test

  def self.perform(time = 100)
    sleep(time)
  rescue Resque::TermException
    sleep(2)
    puts "omg job cleaned up!!!!"
  end
end
{% endhighlight %}

### Deprecated Signal Handling

This is the old way we're handling signals. Here's the command to run this on Resque 1.22.0.

{% highlight sh %}
env VVERBOSE=1 QUEUE=* bundle exec rake resque:work
{% endhighlight %}

With the log output we can see that when we send SIGTERM, the parent worker just kills the child immediately with no remorse.

    20:30:08 worker.1     | started with pid 18903
    20:30:13 worker.1     | ** [20:30:13 2012-08-15] 18903: got: (Job{test} | SleepJob | [30])
    > Process.kill("TERM", 18903)
    20:30:23 worker.1     | ** [20:30:23 2012-08-15] 18903: Exiting...
    20:30:23 worker.1     | ** [20:30:23 2012-08-15] 18903: Killing child at 18946
    20:30:23 worker.1     |   PID S
    20:30:23 worker.1     | 18946 S
    20:30:23 worker.1     | process terminated

### New Signal Handling Cleaning Up

This is the case where we use the new signal code.

{% highlight sh %}
env TERM_CHILD=1 VVERBOSE=1 QUEUE=* bundle exec rake resque:work
{% endhighlight %}

In the logs we can see it actually executes the cleanup code and prints "omg job cleaned up!!!!".

    20:36:10 worker.1     | started with pid 21301
    20:36:11 worker.1     | ** [20:36:11 2012-08-15] 21301: got: (Job{test} | SleepJob | [30])
    > Process.kill("TERM", 21301)
    20:36:20 worker.1     | ** [20:36:20 2012-08-15] 21301: Exiting...
    20:36:20 worker.1     | ** [20:36:20 2012-08-15] 21301: Sending TERM signal to child 21318
    20:36:25 worker.1     | omg job cleaned up!!!!
    20:36:25 worker.1     | ** [20:36:25 2012-08-15] 21318: done: (Job{test} | SleepJob | [30])
    20:36:25 worker.1     | process terminated

### New Signal Handling Timing Out

This case is where we use the new logic, but the cleanup code takes longer than the time allowed. We simulate this by sleeping longer than the allotted time.

{% highlight sh %}
env RESQUE_TERM_TIMEOUT=1 TERM_CHILD=1 VVERBOSE=1 QUEUE=* bundle exec rake resque:work
{% endhighlight %}

In the logs we can see the "omg job cleaned up!!!!" isn't printed.

    20:26:26 worker.1     | started with pid 17395
    20:26:26 worker.1     | ** [20:26:26 2012-08-15] 17395: got: (Job{test} | SleepJob | [30])
    > Process.kill("TERM", 17395)
    20:26:49 worker.1     | ** [20:26:49 2012-08-15] 17395: Exiting...
    20:26:49 worker.1     | ** [20:26:49 2012-08-15] 17395: Sending TERM signal to child 17412
    20:26:53 worker.1     | ** [20:26:53 2012-08-15] 17395: Sending KILL signal to child 17412
    20:26:53 worker.1     | process terminated

I believe this new way of signal handling is superior to the past implementation. It fixes both the child process not handling signals properly and provides a clean way to add clean up code to your job. Again, this is the path forward on how processes will be handled in Resque. I would like to call out a special thanks to [Jonathan Dance](http://github.com/wuputah), [Ryan Biesemeyer](https://github.com/yaauie), and [Aaron Patterson](http://github.com/tenderlove) for design discussion and work on this feature.
