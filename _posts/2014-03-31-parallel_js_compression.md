---
layout: post
title: "Parallel JS compression with Rust"
date: 2014-03-31
categories: rust javascript
---

This is an experiment made with Rust and UglifyJS. My idea comes from
the possibility of minifying Javascript in parallel so that is finishes
faster. So far the results with 4 files have been showing ~300% speed
increase. Altough this is not a definitive and good measurement, it makes me
think I'm pointed in the right direction.


### How it works

The code will walk the `js/` directory looking for JS files. Right now I
spin up a process for each file. Since there are only 4 files, It fits
one uglifyJS process per core. Since we're missing UglifyJS Rust
bindings, using processes made sense instead of tasks or threads.

Each process needs a temporal file to write the compressed JS to,
otherwise it writes to `STDOUT` and I can't ensure they will do it in
order. I would need to use a more complex IPC mechanism and that felt
like an overkill.  Using files as IPC makes a lot of sense for this
small experiment.

Once the process writes on a temporal file, I need to read it back to
put it in the final file. Retrieving the file in which a
process wrote to, was really hard (I didn't actually make it work this
way) and that lead me to write the `UProc` struct.

**UProc** is a struct that contains the process its running and a
reference to the file its using to write the output of the process.
This made it so easy now! I was full with joy coding this thing.

I also created the **TempFile** struct which only hold a reference to a
file. My intention with this struct is to wrap the temp file creation
mess within a method. It works well, but I can smell something really
rotten in there.

Once I spun up some `Uproc`s I needed to be able to interact with them
again, so I needed to store them somehow and the first storing mechanism
that came to mind was (drumbroll...) a Vector!. Everytime I create a new
UProc, I put it inside  of my vector so that later I itrate over it and
have control over each process.

On each iteration I `wait()` for the process to finish, read the temp
file and dump it into the final one.

### What doesn't work

The main bummer I got from this experiment is that I couldn't figure out
how to split the files into buckets that will match the number cpu cores
the system has :(. For instance, if you have 4 cores and 5 files, one
bucket needs to contain 2. If you had 6 files, I could split it like 2,
2, 1, 1.

It sounds dead simple and I'm sure It will come to me while I'm on BART or
the Ferry. If you know how, please let me know :D

Another thing that doesn't work are dependencies. (Oh god,
dependencies). At the very first I tried to use jquery, backbone,
underscore and d3 as examples and of course the thing worked but once in
the browser, jquery had to be at the beginning of everything since is a
dependency for backbone. At the same time, underscore is a dependency of
backbone, so yes, it was a mess. To make it simpler, I got rid of
backbone and went with Raphael. Now everything works __perfectly__ ¬¬

I'm interested in the dependencies part because I **feel** it won't be
too hard to implement a small system that controls which file goes in
which processing bucket. The worst case scenario would happen when you
want to compile a bunch of files that depend on 1 lib. For example,
jQuery, undercore and backbone. My options to solve this problem would
depend on the workload split mechanism I decide to use, but once
that's decided, It could either group them in 1 unit of work or uglify
them arbitrarly, but write them to the final file with the dependency
order.

Probably I should just hardcode that jQuery goes always first ;p

### Speed

I believe I'm hitting a good speed increase. Overall its 3x faster with
parallelism. Even though this is not a confident measurement since I'm
missing dependencies ordering, this gave a good feeling. Here's a picture.

![Uglify time](/images/uglify.png)

### Source code

You can find it [here](https://github.com/alan-andrade/Doodles/tree/master/rust/uglify)

Or here an abstract:

{% highlight rust %}
use std::io::fs::{walk_dir, File};
use std::io::{Process,TempDir};

fn main () {
    let js_dir = Path::new("js");
    let mut paths = walk_dir(&js_dir).unwrap();
    let mut uglies = Vec::new();

    for path in paths {
        uglies.push(UProc::new([path.as_str().unwrap().to_owned()]));
    }

    let mut file = File::create(&Path::new("mylib.js"));

    for ugly in uglies.mut_iter() {
        ugly.ps.wait();
        file.write(ugly.read()).unwrap()
    }
}
{% endhighlight %}

---

### Conclusion

This experiment was FUN! I learned about ways of controlling processes,
vectors and refined concepts of Rust. I think I will continue with
really splitting up the workload on even chunks of workload per core and
also the mini dependency system.

**Special thanks** to the people on the IRC channel. They are truly
amazing. `strcat` , `mcpherrin`, `Havvy`, `cmr`, `sfackler`
