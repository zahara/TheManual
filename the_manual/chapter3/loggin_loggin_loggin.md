# Chapter 3 - Deep dive

## Loggin, loggin, loggin

To mess around with the bit of Sonic Pi that handles synths you will need to download the source and compile your own instance of Sonic Pi.

The various markdown files at the root level in the [source code](https://github.com/sonic-pi-net/sonic-pi) have instructions on how to build for many platforms.

For this exploration we will be looking at the `ruby` part of Sonic Pi.

Luckily once Sonic Pi is built this is very straightforward. `ruby` is an interpreted and not a compiled language and by simply editing `ruby` source code and stopping and restarting Sonic Pi we can see the changes.

Once we have compiled a built Sonic Pi we can start and run it by invoking the binary `sonic-pi` which is created in the directory `app/build/gui/qt`.

Lets look at 5 techniques for understanding what is going on:

* existing log messages
* built in messaging inside the runtime
* logging during boot
* (some) logging of OSC messages between Sonic Pi and SuperCollider
* native Ruby Logging

but before we do, beware false friends!

## False friends

The Sonic Pi language has a couple of ***false friend*** functions - things that look like they will be helpful in this context, but they mostly aren't.

They are the commands `use_debug` and `with_debug` in the language reference. They only affect logging of synth triggers to the front end.

If we run the following code in the Sonic PI gui:

```ruby
use_synth :bass_foundation

play 60
```

we see the following log message in the log window of the GUI:

```
=> Starting run 6

{run: 6, time: 0.0}
 └─ synth :bass_foundation, {note: 60.0}
```

If we now add the `use_debug` command the log message goes away:
 
```ruby
use_debug false
use_synth :bass_foundation

play 60
```

This is just a convenience function for front end users and not a proper debugging tool.

## Existing log messages

Sonic Pi writes its log to the directory `~/.sonic-pi/log`. If we pop in there we can see a useful set of logs:

```
gordon@raspberrypi:~/.sonic-pi/log $ ls
daemon.log  gui.log  jackd.log    spider.log    tau.log
debug.log   history  scsynth.log  tau_boot.log
```
You can get a lot more info if you go into the [util module](https://github.com/sonic-pi-net/sonic-pi/blob/dev/app/server/ruby/lib/sonicpi/util.rb#L335) and set the debug mode to `true` tho:

```ruby
    def debug_mode
      false
    end
```

***BEWARE***: there is more than one module called `util.lib` you want the one in `/app/server/ruby/lib/sonicpi/`


## Built in messaging inside the runtime

When we run code in the Sonic Pi like:

```ruby
load_synthdefs "/home/gordon/.synthdefs"

use_synth :myfirstsynth

play 60
```

we see messages on the logging tab like this


```
=> Starting run 3

=> Loaded synthdefs in path: /home/gordon/.synthdefs
   - /home/gordon/.synthdefs/myfirstsynth.scsyndef

=> Completed run 3
```

If we grep the string `Loaded synthdefs` we can find the origin - in the module [sound.rb](https://github.com/sonic-pi-net/sonic-pi/blob/58164cad453458ce0795b01696987e4a2946a451/app/server/ruby/lib/sonicpi/lang/sound.rb#L3357):

```ruby
      def load_synthdefs(path=Paths.synthdef_path)
        raise "load_synthdefs argument must be a valid path to a synth design. Got an empty string." if path.empty?
        path = File.expand_path(path)
        raise "No directory or file exists called #{path.inspect}" unless File.exist? path
        if File.file?(path)
          load_synthdef(path)
        else
          @mod_sound_studio.load_synthdefs(path)
          sep = "   - "
          synthdefs = Dir.glob(path + "/*.scsyndef").join("#{sep}\n")
          __info "Loaded synthdefs in path: #{path}
#{sep}#{synthdefs}"
        end
      end
      doc name:          :load_synthdefs,
          introduced:    Version.new(2,0,0),
          summary:       "Load external synthdefs",
          doc:           "Load all pre-compiled synth designs in the specified directory. This is useful if you wish to use your own SuperCollider synthesiser designs within Sonic Pi.

...
``` 

The function `__info` that is being called to write the msg to the front end is found in the module [runtime.rb](https://github.com/sonic-pi-net/sonic-pi/blob/067a9c7ee2ec2dd839dff054a81112e50326532a/app/server/ruby/lib/sonicpi/runtime.rb#L349):

```ruby
	def __info(s, style=0)
      __msg_queue.push({:type => :info, :style => style, :val => s.to_s}) unless __system_thread_locals.get :sonic_pi_spider_silent
    end
```
We can prove it is this definition by adding another message push as so:

```ruby
	def __info(s, style=0)
      __msg_queue.push({:type => :info, :style => style, :val => "banjo"}) unless __system_thread_locals.get :sonic_pi_spider_silent
      __msg_queue.push({:type => :info, :style => style, :val => s.to_s}) unless __system_thread_locals.get :sonic_pi_spider_silent
    end
```
So now when we stop and start Sonic Pi and run the same code we see that every msg we get in the front end, we get a `banjo` before it.

```
=> Starting run 3

=> banjo

=> Loaded synthdefs in path: /home/gordon/.synthdefs
   - /home/gordon/.synthdefs/myfirstsynth.scsyndef

=> banjo

=> Completed run 3
```
So by simply adding lines to our ruby that calls this `__info` function we can see what's going on when we do stuff.

## Logging during boot

The runtime logging is great but what happens when you want to figure out what is happening during boot before the GUI is available to show your messages?

Well it turns out that Sonic Pi has that sorted too. You can write a message to a buffer and when the boot is completed the buffer is dumped into the log window. Lets see that in action in the [studio module](https://github.com/sonic-pi-net/sonic-pi/blob/dev/app/server/ruby/lib/sonicpi/studio.rb#L67) which handles the boot process and the creation of the GUI.

If I invoke the function `message` with a string, as I do here, it will appear in the log screen on boot.

```ruby
    def init_scsynth
      message "bingo bongo dandy dongo"
      @server = Server.new(@scsynth_port, @msg_queue, @state, @register_cue_event_lambda, @current_spider_time_lambda)
      message "Initialised SuperCollider Audio Server #{@server.version}"
    end
```

and as expected printing my message in the GUI's log window:

```
ome to Sonic Pi v5.0.0-Tech Preview 2

=> Running on Ruby v2.7.4

=> Initialised Erlang OSC Scheduler

=> Initialised SuperCollider Audio Server v3.11.2

=> bingo bongo dandy dongo

=> Remember, when live coding music
   there are no mistakes
   only opportunities to learn
   and improve.

=> Let the Live Coding begin...

=> Has Sonic Pi made you smile?

   We need *your* help to fund further development!

   Sonic Pi is not financially supported by
   any organisation.

   We are therefore crowdsourcing funds from kind
   people like you using Patreon.

   We need at least 1000 supporters to continue.
   Currently we have 733 generous individuals.

   Please consider becoming a Patreon supporter too,
   and help us keep Sonic Pi alive:

   https://patreon.com/samaaron
```

## (Some) logging of OSC messages between Sonic Pi and SuperCollider

We can use this command to get some OSC logging:

```ruby
use_osc_logging true
```

It will dump error messages into the Sonic Pi log `scsynth.log in `~/.sonic_pi/logs`

```
*** ERROR: SynthDef sonic-pi-mysecondsynth not found
FAILURE IN SERVER /s_new SynthDef not found
FAILURE IN SERVER /n_set Node 10 not found
FAILURE IN SERVER /n_set Node 10 not found
FAILURE IN SERVER /n_set Node 10 not found
FAILURE IN SERVER /n_set Node 10 not found
```

## Native ruby logging

Sometimes, maybe, the front end logging might not be enough.
 
In that case we can use the built in `ruby` [Logger](https://ruby-doc.org/stdlib-2.4.0/libdoc/logger/rdoc/Logger.html).

We can now sprinkle the code with log calls and try and figure out how the server works.

Using [Logger](https://ruby-doc.org/stdlib-2.4.0/libdoc/logger/rdoc/Logger.html) is pretty straightforward, you need to load the library into the module, create a new logger with a fully qualified filename to log to and write a log statement:

```ruby
require 'logger'
...
logger = Logger.new("/home/gordon/Dev/tmp/sonic_pi.log")
logger.debug("normalising synth args")
```

I am telling [Logger](https://ruby-doc.org/stdlib-2.4.0/libdoc/logger/rdoc/Logger.html) to use a file in my home directory, you need to get it write it to wherever suits you. The file must already exist and the path must be fully qualified so no `../..` or `~`s.
