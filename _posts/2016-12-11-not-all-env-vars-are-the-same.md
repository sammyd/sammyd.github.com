---
layout: post
title: "Not all env variables are born the same"
date: 2016-12-11 16:05:04 +0700
comments: true
tags: [docker, linux, rails]
cover: /assets/covers/boat_from_the_water.jpg
navigation: true
---
### _or how I found my rails problem in the Linux source_

Whilst building the latest web application in the raywenderlich.com web empire, I stumbled across a problem I expected was a simple error on my part. It wouldn't take long to track down and fix. Oh, the naïvety of  the pre-debugging ignorance.

Codenamed *kerching*, the app in question is a relatively small ruby-on-rails app, running on [AWS](https://aws.amazon.com/) in three separate docker containers. Maybe I'll write about the dockerisation architecture one day—it's probably sufficiently interesting—but the focus of this post is on just one of those containers—the worker.

The worker container is responsible for running all kinds of background jobs, and uses AWS's Simple Queue Service (SQS) for task queuing via the [shoryuken](https://github.com/phstc/shoryuken) gem (probably worthy of another post). As such, it requires full access to the rails stack, and in practice is actually running the same docker image as the app container. In addition to running jobs, it also schedules them via cron jobs, assisted by the [whenever](https://github.com/javan/whenever) gem.

With me so  far? Don't worry—I've nearly finished setting the scene.

The cron jobs are incredibly simple—they just request a particular [ActiveJob](http://guides.rubyonrails.org/active_job_basics.html) to run. This then gets chucked onto the SQS queue, and handled by the primary worker process. To some extent, this is working around a limitation of SQS, but that's not important right now.

This means that both the worker process (shoryuken) and the cron jobs have to fire up the rails stack. But that's fine—the container is based on the app image so it'll all be gravy. Won't it?

## Enter environment variables stage right
I promised you something about environment variables. You've been forced to read (well, let’s face it, skim) a lot of text, and I haven't even mentioned them. What are they all about?

When building  web apps, especially those that are built into docker containers, you don't want to include any secrets in source code. Therefore a popular (and potentially "best") practice is to provide them via environment variables. Docker supports this—allowing you to easily inject variables into a container at runtime via an env file.

We use this a lot in our development, staging and production stacks—allowing us to use an identical container image for all three environments, but connecting to different infrastructure, with different configurations in each instance.

## A problem surfaces
You finally have enough background to understand the first problem: the cron jobs were unable to load the rails stack.

Remember that the cron job needs to load the rails stack to schedule a background job? And that the rails stack is configured almost exclusively via environment variables? And that docker injects the environment variables at runtime?

Well, environment variables are only available to the process that docker starts. If that's a shell (like bash) then that has access to the variables you define outside docker, but any other process has no such knowledge.

This is the case for cron. It runs as a background service, and so doesn't get provided the environment variables injected by docker. However, it does load a list of environment variables before it starts—from the `/etc/environment` file.

Therefore as part of the startup process for the worker, we populate this file with a copy of the current environment:

{% highlight bash %}
# Make a copy of the environment variables
env > /etc/environment

# Start the cron service
service cron start

# Start the shoryuken worker process 
bundle exec shoryuken -C config/shoryuken.yml --rails
{% endhighlight %}

Now, when the cron job starts, it reads the contents of the `/etc/environment` file and has access to all the environment variables it needs. Super.

## and that's the end of the story
Balls is it.

This worked splendidly for a while in staging. But on a recent deploy, we started getting notifications that the worker process was no longer, well, working. On investigation I discovered that the job worker was fine, but that the cron jobs were no longer firing.

So began a process of trying to debug cron.

First I needed to find where the hell stuff was logged. Turns out, it wasn't.

In my minimal docker image (based on Debian jessie), I had neither a mail transport agent, nor a system logging framework. That meant that the two possible places cron would be sending error logs didn't exist.

Adding `rsyslog` to the docker image was pretty easy—it's just a new package to install at build time. I could then see that the cron job was running at the correct time in `/var/log/syslog`, although still not the errors. But I still couldn't see the output of the cron job itself:

```
Dec 11 03:47:01 7d0126f97a33 CRON[552]: (root) CMD (echo "HELLO FROM CRON")
```

Adding `> /var/log/cron.log 2>&1` to the end of the command in the cron tab means that error and standard output will all be piped into a new file, which was helpful for debugging, but not standard practice for docker:

```
HELLO FROM CRON
```

## aside: logging in docker

It's standard practice within docker containers to log everything to standard out. That way the logging is picked up by docker and can be piped to a multitude of services. In our stack, we pipe all docker logs to AWS CloudWatch, providing a single location for logs for our entire infrastructure.

Both rsyslog and our cron tasks are now logging to files—how can I get them to standard out?

Well, first the cron job itself.

You might think using `> /dev/stdout 2>&1` would do the trick. But you'd be both unnecessarily verbose and incorrect. But uou wouldn't be alone. That's precisely what I thought would work.

It doesn't.

Why doesn't it work? Well, `/dev/stdout` is linked to the standard out _of the current process_:

```
0 lrwxrwxrwx 1 root root     15 Dec 11 02:29 stderr -> /proc/self/fd/2
0 lrwxrwxrwx 1 root root     15 Dec 11 02:29 stdin -> /proc/self/fd/0
0 lrwxrwxrwx 1 root root     15 Dec 11 02:29 stdout -> /proc/self/fd/1
```

We only see (and hence docker only logs) the stdout of the launch process. Cron runs jobs in a different process, and hence their stdout gets lost in the ether (or emailed to you if you have an MTA) configured.

To get to the stdout of the launch process, you need `/proc/1/fd/1`. The first `1` refers to the PID of the process. Provided you haven't told docker to do anything different (like specified it launches using the host PIDs) then the process it launches will have a PID of `1`.

The second `1` refers to stdout. `0` would be stdin and `2` is stderr/.

That means updating the cron command to the following will log out as expected:

```
* * * * * echo "HELLO FROM CRON" >> /proc/1/fd/1 2>&1
```

We use exactly the same principle for syslog too, this time creating a symlink between the file output and this stdout:

```
$ ln -sf /proc/1/fd/1 /var/log/syslog
```

Now the output from both the cron daemon and the jobs themselves gets piped directly to the stdout of the primary process of the container. The can be a little distracting whilst working in the container, but it means everything will be logged by docker.

## Back on track: the environment is broken

Now we can see the output from the cron job, we can have a crack at fixing it.

The output is from a component we use to sign CloudFront URLs. This uses OpenSSL to sign using RSA, and therefore requires a public key. The error suggests that this key is malformed:

```
/usr/local/bundle/gems/cloudfront-signer-3.0.0/lib/cloudfront-signer.rb:54:in `initialize': Neither PUB key nor PRIV key: nested asn1 error (OpenSSL::PKey::RSAError)
  from /usr/local/bundle/gems/cloudfront-signer-3.0.0/lib/cloudfront-signer.rb:54:in `new'
  from /usr/local/bundle/gems/cloudfront-signer-3.0.0/lib/cloudfront-signer.rb:54:in `key='
  from /var/www/kerching/config/initializers/cloudfront_signer.rb:5:in `block in <top (required)>'
```

But how can that be? The key is provided by an environment variable, and the worker process has no such problem.

I can see the correct key in `/etc/environment`, which is where the cron job gets its environment variables, so what's going on?

Adding some logging to the rails initialiser shows that the key variable is being truncated.

A check to see that the cron job does indeed have access to the `/etc/environment` file, adding the following cron job:

```
* * * * * cat /etc/environment > /proc/1/fd/1 2>&1
```

This yields the following results:

```
HOSTNAME=9017b7e8da1a
TERM=xterm
MY_KEY=01........|.........|.........|.........|.........02........|.........|.........|.........|.........03........|.........|.........|.........|.........04........|.........|.........|.........|.........05........|.........|.........|.........|.........06........|.........|.........|.........|.........07........|.........|.........|.........|.........08........|.........|.........|.........|.........09........|.........|.........|.........|.........10........|.........|.........|.........|.........11........|.........|.........|.........|.........12........|.........|.........|.........|.........13........|.........|.........|.........|.........14........|.........|.........|.........|.........15........|.........|.........|.........|.........16........|.........|.........|.........|.........17........|.........|.........|.........|.........18........|.........|.........|.........|.........19........|.........|.........|.........|.........20........|.........|.........|.........|.........21........|.........|.........|.........|.........22........|.........|.........|.........|.........23........|.........|.........|.........|.........24........|.........|.........|.........|.........25........|.........|.........|.........|.........26........|.........|.........|.........|.........27........|.........|.........|.........|.........28........|.........|.........|.........|.........29........|.........|.........|.........|.........30........|.........|.........|.........|.........
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
PWD=/
SHLVL=1
HOME=/root
no_proxy=*.local, 169.254/16
_=/usr/bin/env
```

Yep. So what about the loaded environment, with:

```
* * * * * env > /proc/1/fd/1 2>&1
```

Well, that produces the following:

```
no_proxy=*.local, 169.254/16
HOSTNAME=9017b7e8da1a
SHLVL=1
HOME=/root
MY_KEY=01........|.........|.........|.........|.........02........|.........|.........|.........|.........03........|.........|.........|.........|.........04........|.........|.........|.........|.........05........|.........|.........|.........|.........06........|.........|.........|.........|.........07........|.........|.........|.........|.........08........|.........|.........|.........|.........09........|.........|.........|.........|.........10........|.........|.........|.........|.........11........|.........|.........|.........|.........12........|.........|.........|.........|.........13........|.........|.........|.........|.........14........|.........|.........|.........|.........15........|.........|.........|.........|.........16........|.........|.........|.........|.........17........|.........|.........|.........|.........18........|.........|.........|.........|.........19........|.........|.........|.........|.........20........|.........|.........|.........|.........21........|.....
LOGNAME=root
_=/usr/bin/env
TERM=xterm
PATH=/usr/bin:/bin
SHELL=/bin/sh
PWD=/root
```

Aha—`MY_KEY` seems to stop just after `21`, instead of continuing to past `30`. The private key is being truncated, which explains why it is now malformed.

But why?

> __Note:__ We're now into the depths of Linux and my knowledge is somewhat patchy. Please feel free to correct my mistakes and I'll update this post.

Pluggable Authentication Modules ([PAM](http://www.linux-pam.org)) is  mechanism within Linux whose responsibility is authentication. Part of its remit includes preparing the environment for different processes. Services have their environment configured as per the **pam_env** module, with the configuration appearing in the `/etc/pam.d` directory.

The cron file within there shows that it should be loading the environment from the `/etc/environment` file as expected:

```
# The PAM configuration file for the cron daemon

@include common-auth

# Sets the loginuid process attribute
session    required     pam_loginuid.so

# Read environment variables from pam_env's default files, /etc/environment
# and /etc/security/pam_env.conf.
session       required   pam_env.so

# In addition, read system locale information
session       required   pam_env.so envfile=/etc/default/locale
```

That narrows it down then—something about this **pam_env** module appears to be truncating the environment variables it imports.

## Off we trundle into C
Knowing this wasn't enough for me. I wanted to confirm that my suspicions were correct, and to find exactly *why* it is being truncated. For that I needed to take a trip off into the PAM source code.

> That'll never work.


The **pam_env** part of [libpam](https://git.fedorahosted.org/cgit/linux-pam.git/tree/libpam/pam_env.c) itself details how environment variables are set, parsed, returned and stored. But nothing about reading them from a file.

For that we need to look at the **pam_env** [module](https://git.fedorahosted.org/cgit/linux-pam.git/tree/modules/pam_env/pam_env.c).

Within here there is a function whose job it is to read env variables from a file. Perfect.

{% highlight c %}
static int
_parse_config_file(pam_handle_t *pamh, int ctrl, const char *file)
{
    int retval;
    char buffer[BUF_SIZE];
    FILE *conf;
    VAR Var, *var=&Var;

    D(("Called."));

    var->name=NULL; var->defval=NULL; var->override=NULL;

    D(("Config file name is: %s", file));

    /*
     * Lets try to open the config file, parse it and process
     * any variables found.
     */

    if ((conf = fopen(file,"r")) == NULL) {
      pam_syslog(pamh, LOG_ERR, "Unable to open config file: %s: %m", file);
      return PAM_IGNORE;
    }

    /* _pam_assemble_line will provide a complete line from the config file,
     * with all comments removed and any escaped newlines fixed up
     */

    while (( retval = _assemble_line(conf, buffer, BUF_SIZE)) > 0) {
      D(("Read line: %s", buffer));
...
{% endhighlight %}

This isn't overly complicated C code, in fact the problem is in the short snippet above.

In C, you're responsible for all the memory management—including allocating the correct amount of space to read a line from a file in. That's exactly what's happening here: a buffer is allocated, and then populated using the `_assemble_line` function.

The buffer is defined right at the top of the above snippet:

{% highlight c %}
char buffer[BUF_SIZE];
{% endhighlight %}

It has a size of MAX_BUFFER characters long. Which means that's the maximum length line that can be read from an environment file.

Definitely getting somewhere now—"but what's that value?" I hear you ask. Well it's defined as a constant on line 55 of this file:

{% highlight c %}
#define BUF_SIZE 1024
{% endhighlight %}

And finally I've reached the nub of the problem: although environment variables can be pretty much any size (I think they’re limited by memory constraints), the longest line that can be read from `/etc/environment` is 1024 characters.

Our private key is 1700 characters.

That'll never work.

## Solution

> has the advantage that it'll, ya know, actually work.

My solution to this problem is probably a little unsatisfying: we don't actually need that environment variable for the cron job. The cron job will never be asked to sign a URL. Therefore I can rescue the exception and continue on with my day.

But what if I actually _needed_ that key.  Well, here's a couple of options:

1. Split the env var up into 2, and then rejoin them in code. The length of the entire line (including variable name) has to be less than 1024, but you could conceivably split your variables up into pieces and reassemble them later. To an extent, we already do this to cope with newlines in the private key, which the docker env injection doesn't cope with. However, this feels pretty unsatisfactory.
2. Store the environment variables somewhere else. You could create a script that exports each of them in turn, and then run that before any cron job:

{% highlight bash %}
set -a
export MYLONGVAR="hello"
{% endhighlight %}

This is a bit more fiddly to write as you can't just use the output from `env`,  but it's not that difficult, and has the advantage that it'll, ya know, actually work.

## and I'm done

It's been a while since I wrote a post on here, and there have been lots of things that have made me think "that'd make a good post".

This is the first of those that have actually taken form. A lot of the work I've been doing lately is in a similar space. If you're interested in reading more about docker, rails, Linux, AWS etc then let me know on Twitter. Otherwise this might be a short lived reanimation. 

sam
