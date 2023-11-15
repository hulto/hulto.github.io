---
layout: post
title: Linux Child Process Ownership
categories: [general]
tags: [redteam, linux]
description: A quick demo to look at what Linux child processes can do with background, double forking, disown, and nohup.
---

## Intro

Executing shell commands is generally something you want to avoid during red team ops since it opens a whole can of detections: suspicious process trees, auditd logging exec calls, or even bash history files.
In some situations though, running a shell command is unavoidable.

In this blog, I'll walk through some things you can do to help run shell commands a little more safely.

We'll go over four different activities and how you can combine them to improve your op safety.
- Background 
- Double forking
- Disown
- Nohup


For this blog, i'll be using the following script as a test process to demonstrate how these activities work.

```bash
#!/bin/bash

for i in `seq 1 100`; do
        echo "[PID: $$] hello from stdout $i"
        sleep 1
done
```

Here's what the process tree looks like when the script is run normally:

<figure class="aligncenter">
    <img src="/assets/png/linux-child-process-control.png" />
    <figcaption>Figure 1. - The default process tree for our control process execution.</figcaption>
</figure>

### Terms and ideasz

- Datastreams - STDIN `0` , STDOUT `1`, STDERR `2`.
	- STDIN - "Standard in" the input data stream for a process.
	- STDOUT - "Standard out" the output data stream where most `print` statements go.
	- STDERR - "Standard error" the output data stream where errors are logged.
	- These three data streams are the main way your interactive shells sends and receives information from interactive commands.
	- These streams can disconnected, redirected, or piped `|` into other streams.
- Signals
	- `SIGHUP` - Signal hangup
	- `SIGINT` - Signal interrupt sent when you press Crtl+C


### Summary

| Activity | Backgrounded | Parent process| Signal sent | STDOUT | STDERR | STDIN |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| Background | Yes | Same | N/a | Still attached | Still attached | Detached |
| Double forking | Not without backgrounding | 1 | N/a | Still attached | Still attached | Detached |
| nohup | No | Same | Protected from hangups | Redirected to `./nohup.out` | Redirected to `./nohup.out` | Still attached |
| Disown | Yes done before disown | Same | Protects from signals sent to parent | Still attached | Still attached | - |


### Background

Backgrounding a process allows you to continue using your current shell while the background task continues to run.
This can be done multiple times to allow many tasks to run concurrently.

Processes are usually backgrounded by appending the `&` character to the end of your command.

Backgrounding a process is just like running any other process except it disconnects your STDIN pipe. If the process needs to read from STDIN it will halt until input is provided.

#### Example

<figure class="aligncenter">
    <img src="/assets/png/linux-child-process-background.png" />
    <figcaption>Figure 2. - The process tree for a backgrounded process.</figcaption>
</figure>


Here we can see in the process tree that both the id command and our test script are running as children of a single bash shell.
In the bottom pane we see that the backgrounded process is running, and printing to the screen. In the red box we see that the bash shell is still receiving input and executing the `id` command.

to exit a backgrounded process run the command `fg` to reconnect the STDIN pipe to your current shell and then send the SIGINT signal by pressing `ctrl + c`.

If you have more than one backgrounded process you may wish to kill one that's not the most recent. This can be done using the jobs command.

<figure class="aligncenter">
    <img src="/assets/png/linux-child-process-background-jobs.png" />
    <figcaption>Figure 3. - The jobs list tracking backgrounded processses.</figcaption>
</figure>

#### Real world

I often use the background activity to monitor logs or network activity while performing a test.
This lets me generate a request and monitor the response in the same shell.

<figure class="aligncenter">
    <img src="/assets/png/linux-child-process-tailing.png" />
    <figcaption>Figure 4. - The jobs list tracking backgrounded processses.</figcaption>
</figure>

In the first command `tail -F /var/log/nginx/error.log &` we watch the error log for new entries "following" the content. Backgrounding it we're still able to use our current shell and run the `curl localhost/` command.

The output from curl is printed `<html>...</html>` as well as the output of the backgrounded process `2023/09/20 .... directory index.... is forbidden`.


### Double forking

When you background a process it's still attached to your parent process which if it's a c2 or exploited app will look very suspicious. To avoid this we can use a double fork. This creates an intermediate parent process that dies allowing it's orphaned process to be adopted by PID 1.

#### Example

Here is normal backgrounded execution.

<figure class="aligncenter">
    <img src="/assets/png/linux-child-process-background-control.png" />
    <figcaption>Figure 5. - Normal process backgrounding.</figcaption>
</figure>

Here is a double forked process.

<figure class="aligncenter">
    <img src="/assets/png/linux-child-process-double-fork.png" />
    <figcaption>Figure 6. - A background and double forked process.</figcaption>
</figure>

In this example we're creating an interemediate process using `(...)` this could be replaced with `/bin/bash -c '....'` to achieve the same result.

### nohup

As you can see in the above example even though we've successfully backgrounded and double forked our process it's still printing to our screen. The process printing to our screen is annoying but it can also be a greater problem if your processes doesn't handle the STDOUT or STDERR pipes closing unexpectedly.

One way to fix this is to use `nohup`. nohup does two things:
- Protects your child process from the HANGUP signal `SIGHUP` which is sent when the parent terminal sessions ends.
- Redirects STDOUT and STDERR. If unspecified nohup defaults to redirecting both to a local file `./nohup.out`

#### Examples

In this example we look at the default behavior of `nohup`.
STDOUT and STDERR have been redirected to the `./nouhp.out` file and when we try to kill it with the hangup signal `SIGHUP` it's doesn't die.

<figure class="aligncenter">
    <img src="/assets/png/linux-child-process-nohup.png" />
    <figcaption>Figure 7. - Default nohup process tree and output</figcaption>
</figure>


### Disown

Disowning a process allows it to continue running even in the parent process dies or is killed. This means any kill signals sent to the parent process will not be passed down to the child process. When the parent process dies Linux will reclaim it as a child of PID 1.

The disowned process is still attached to stdout and stderr so even though your current shell doesn't control the task you'll still receive the output this is generally unideal and you'll want to pipe STDOUT and STDERR somewhere else.

#### Example

In this example we start our script backgrounded and disown it.
We can see that once it's been disowned we can no longer control it through the `jobs` or `fg` command but are still receiving output from the STDOUT pipe.

We can also see in the process tree our script `my-test-script.sh` is no longer a child of our bash shell.
See **Double forking**

<figure class="aligncenter">
    <img src="/assets/png/linux-child-process-disown-jobs.png" />
    <figcaption>Figure 8. - Disowned jobs listed before and after.</figcaption>
</figure>

What would happen to our disowned process if our shells STDOUT and STDERR pipes closed (as they do when exiting the parent shell)?

The program would fail as it tries to write to the output pipe.

We can test this by running our program like we did before then closing it's stdout pipe while it's running with GDB.


<figure class="aligncenter">
    <img src="/assets/png/linux-child-process-gdb-demo.png" />
    <figcaption>Figure 9. - Disowned jobs listed before and after.</figcaption>
</figure>

In this example the program mostly harmlessly errors writing to stderr that it cannot write to stdout using the echo command anymore. If your program is unable to handle the failure though this could cause the entire process to crash.

## Combining these activities

What happens when we double fork, nohup, disown, and background a process?
If you do all four:
- The child process will be backgrounded.
- The child process will be protected from hangup signals.
- The child process will be protected from signals sent to the parent process.
- The child process will have it's STDOUT and STDERR redirected to `./nohup.out`.
- The child process will be removed from the parent shells job control.
- When we use a double fork the intermediate parent process will exit allowing the child to be inherited by PID 1.

<figure class="aligncenter">
    <img src="/assets/png/linux-child-process-disown-ppid-1.png" />
    <figcaption>Figure 10. - Combining these activites.</figcaption>
</figure>

If we were able to capture the process tree before the intermediate parent process died it would look like this.

<figure class="aligncenter">
    <img src="/assets/png/linux-child-process-figure-11.png" />
    <figcaption>Figure 11.</figcaption>
</figure>


## Getting fancy
### Output redirectionz

It's nice that nohup reconnects our processes output pipes but often we don't want them logged to the `nohup.out` file. If we want to run multiple nohup-ed commands having all of them append to `nohup.out` would be gross.
To fix this we'll use output redirection.

Here we redirect stdout (and stderr which nohup has already redirected to stdout) to a log file in `/tmp/`

<figure class="aligncenter">
    <img src="/assets/png/linux-child-process-figure-12.png" />
    <figcaption>Figure 12.</figcaption>
</figure>

If we wanted to be sneaky or just ignore the output we could also redirect nohups output to `/dev/null`

<figure class="aligncenter">
    <img src="/assets/png/linux-child-process-figure-13.png" />
    <figcaption>Figure 13.</figcaption>
</figure>


### Backgrounding our current process

If you just kicked off a long running process and realized that you want to background it you can stop the process with `crtl + z` and then background it (which also resumes it.)

<figure class="aligncenter">
    <img src="/assets/png/linux-child-process-figure-14.png" />
    <figcaption>Figure 14.</figcaption>
</figure>

If you want to bring the job back to the foreground you can use the fg command.

<figure class="aligncenter">
    <img src="/assets/png/linux-child-process-figure-15.png" />
    <figcaption>Figure 15.</figcaption>
</figure>


# Additional reading
- Detailed explanation of the disown arguments [https://phoenixnap.com/kb/disown-command-linux](https://phoenixnap.com/kb/disown-command-linux)
- The difference between nohup and disown [https://unix.stackexchange.com/a/148698](https://unix.stackexchange.com/a/148698)
- Closing file descriptors [https://www.baeldung.com/linux/bash-close-file-descriptors#:~:text=%3E%26%2D%20is%20the%20syntax%20to%20close%20the%20specified%20file%20descriptor](https://www.baeldung.com/linux/bash-close-file-descriptors#:~:text=%3E%26%2D%20is%20the%20syntax%20to%20close%20the%20specified%20file%20descriptor)
- More detail on disown SIGHUP behavior [http://wresch.github.io/2014/02/27/bash-nohup-disown-child.html](http://wresch.github.io/2014/02/27/bash-nohup-disown-child.html)
- Difference between nohup and disown -h [https://unix.stackexchange.com/a/484283](https://unix.stackexchange.com/a/484283)
- Termination signal reference [https://www.gnu.org/software/libc/manual/html_node/Termination-Signals.html](https://www.gnu.org/software/libc/manual/html_node/Termination-Signals.html)