---
layout: post
title:  "shell scripting with clojure (using nailgun)"
date:   2013-10-13 22:00:00
categories: productivity
---

This is an really nice way to bridge the gap between your interactive shell (ie bash) and a clojure runtime.  The basic idea is that we want to make it as easy to run a clojure program as a shell script, as it is to run a bash script.  The pipeline for processing the clojure script is more complicated:

    shell -> nailgun -> JVM -> clojure

but this is just a one-time setup cost.  After this, you can script in clojure as easily as with something like bash.  And since we're using nailgun, we don't have to worry about JVM startup times.  Execution times are lightning fast.

### The script

It will be necessary to start every script with a pound bang line to tell the OS how it should process the contents after line 1.  

> The #! is actually a two-byte magic number, a special marker that designates a file type, or in this case an executable shell script (type man magic for more details on this fascinating topic). Immediately following the sha-bang is a path name. This is the path to the program that interprets the commands in the script, whether it be a shell, a programming language, or a utility. This command interpreter then executes the commands in the script, starting at the top (the line following the sha-bang line), and ignoring comments

This means that you can ask a `program` to interpret a set of lines of code using a very generic syntax like:

    #!/usr/bin/env program
    start code
    ....
    end code 

This tells the shell to do two things.  First, it will fork an environment and run a program named `program`.   But more interestingly, it will pass the remaining text in the file to the program to be interpreted.  The stdout and stderr streams for the shell will now be attached to this running program.

For the case of interpreting clojure programs, we would like to set up something where the code is clojure, and the interpreter is a clojure environment running in the JVM.  Something like:

     #!/usr/bin/env ng-clojure
     
     (println "in clojure")

`ng-clojure` is still pretty arbitrary here.  I'm calling it ng-clojure because that's what they called it in [this post][install].  This could technially be any program that starts up a JVM with a clojure interpreter.  However, it would be more interesting if we could send the clojure code to an already running JVM for, as close as possible, _immediate_ execution.

[NailGun][nailgun] is a very handy tool that can create a JVM capable of standing _ready_ to execute tasks.  Think of it as a JVM that you can start up, load it's classpath up with jars, and then have it wait for a client to tell it what to do.  One of those tasks could be to run some clojure code.  The easiest way to make this happen is to create a re-usable script called ng-clojure someone on your PATH containing one single line:

    ng clojure.main "$@"

This line uses the nailgun client, called `ng`, to request that the main entry point to clojure `clojure.main` be passed all of the arguments to `ng-clojure`.  The only thing to do now is to install and startup nailgun:

### Installing and starting up on OS X

There is a [link][install] on using nailgun with clojure [here][install].  But if you use `brew` with OS X, the easiest way to get nailgun up and running is with:

     bash> brew install nailgun
     bash> ng-server
     bash> ng ng-cp ~/.m2/repository/org/clojure/clojure/1.5.1/clojure-1.5.1.jar

Obviously, the above does 3 things:

1.  installs nailgun (client and server)
2.  starts the server
3.  tells nailgun server to load a new jar onto it's classpath

There are two other assumptions here.  It has loaded clojure 1.5.1 and it has done so using a place that would exist if you have _ever_ used 'leiningen' on this machine before.  So, if you have a project using leiningen and you've used version 1.5.1 on this machine, then that directory should.  At the end of this, you have a running nailgun server, and you have a file named ng-clojure on your PATH with the line above.  If that is true, you should be able to make an executable (ie `chmod 755 script`) with the lines:

    #!/usr/bin/env ng-clojure
    
    (println "run in clojure")

and run it.  You will not see any of the characteristic start up time for the JVM because it's already running.

At this point, you also have a good way of writing your shell scripts directly in clojure.  

[nailgun]: http://www.martiansoftware.com/nailgun/
[install]: http://prsm.tc/0LVwDi
