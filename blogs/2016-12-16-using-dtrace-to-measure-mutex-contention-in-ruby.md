---
title: "Using DTrace to measure mutex contention in Ruby"
url: "http://workday.github.io/tutorial/2016/12/16/using-dtrace-to-measure-mutex-contention-in-ruby"
date: "2016-12-16T00:00:00+00:00"
author: ""
feed_url: "https://workday.github.io/atom.xml"
---
<p><em>by Tom Van Eyck, Workday Dublin</em></p>

<p>Working on Ruby code that contains a sizable number of threads can benefit from a tool that highlights which particular mutexes are the most heavily contended. After all, this type of information can be worth its weight in gold when trying to diagnose why a threaded program is running slower than expected.</p>

<p>This is where <a href="http://dtrace.org/guide/preface.html">DTrace</a> enters the picture. DTrace is a tracing framework that allows for the instrumentation of application and system internals, thereby enabling the measurement and collection of previously inaccessible metrics. DTrace makes it possible to gather a truly ridiculous amount of fine-grained information about what’s happening on a computer.</p>

<p>DTrace is available on Solaris, OS X, and FreeBSD. There is some Linux support as well, but it might be preferable to use one of the Linux specific alternatives instead. Info about these can be found <a href="http://www.brendangregg.com/dtrace.html">here</a> and <a href="http://www.brendangregg.com/blog/2015-07-08/choosing-a-linux-tracer.html">here</a>. At the time of writing this, it was announced that <a href="http://www.brendangregg.com/blog/2016-10-27/dtrace-for-linux-2016.html">DTrace will be making its way to Linux</a> as well. Please note that all work for this particular post was done on a MacBook running OS X El Capitan (version 10.11).</p>

<!--more-->

<h3 id="enabling-dtrace-for-ruby-on-os-x">Enabling DTrace for Ruby on OS X</h3>

<p>El Capitan comes with a new security feature called <a href="https://en.wikipedia.org/wiki/System_Integrity_Protection">System Integrity Protection</a> that provides additional protection against malicious software. Regrettably, it also prevents DTrace from working. Disabling SIP can be done by following <a href="http://stackoverflow.com/a/33584192/1420382">these instructions</a>. Note that it is possible to only partially disable SIP, although doing so will leave DTrace unable to attach to restricted processes.</p>

<p>Even though Ruby has DTrace support, there is a very good chance that any currently installed Ruby binary does not have this support enabled. This is especially likely to be true if the Ruby binary was compiled locally on an OS X system. Since OS X does not allow the Ruby compilation process to access the system DTrace binary, the resulting Ruby binary will lack DTrace functionality. More information about this can be found <a href="http://stackoverflow.com/a/29232051/1420382">here</a>.</p>

<p>The easiest way to obtain a DTrace compatible Ruby is to just go and download a precompiled Ruby binary. Some degree of carefulness is recommended here as downloading random binaries from the internet is not necessarily safe. Luckily, the good people over at <a href="https://rvm.io/">rvm.io</a> host some DTrace compatible Ruby binaries that are safe to download.</p>

<div class="language-bash highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="nv">$ </span>rvm mount <span class="nt">-r</span> https://rvm.io/binaries/osx/10.10/x86_64/ruby-2.2.3.tar.bz2
<span class="nv">$ </span>rvm use ruby-2.2.3
</code></pre></div></div>

<p>The snippet below provides an alternative approach that can be used when working with rvm (or other version managers) is not an option.</p>

<div class="language-bash highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="nv">$ </span>wget https://rvm.io/binaries/osx/10.11/x86_64/ruby-2.2.3.tar.bz2
<span class="nv">$ </span>bunzip2 ruby-2.2.3.tar.bz2
<span class="nv">$ </span><span class="nb">tar </span>xf ruby-2.2.3.tar
<span class="nv">$ </span>mv ruby-2.2.3/bin/ruby /usr/local/bin/druby
<span class="nv">$ </span>rm <span class="nt">-r</span> ruby-2.2.3 ruby-2.2.3.tar
</code></pre></div></div>

<p>Running the above snippet will install a DTrace compatible Ruby. Note that the binary is renamed to <code class="highlighter-rouge">druby</code> so as to prevent conflicts with existing Ruby installations. The above approach should really be treated as a last resort. Please make the effort to find a DTrace compatible Ruby binary through a version manager instead!</p>

<p>Having now ensured compatibility between DTrace and Ruby, let’s move on to the next section and learn some DTrace basics.</p>

<h3 id="dtrace-basics">DTrace basics</h3>

<p>DTrace is all about probes. A probe is a piece of code that fires when a specific condition is met. For example, DTrace can instrument a process with a probe that activates when this process returns from a specific system call. This particular type of probe is even capable of inspecting the value returned by the system call.</p>

<p>Interaction with DTrace is accomplished by writing scripts in the D scripting language (not related to the D programming language). This language is a mix of C and awk, and has a very low learning curve. An example of such a script written in D is shown below. This particular script will list all system calls initiated on a machine along with the name of the process that initiated them. Let’s save this script as <code class="highlighter-rouge">syscall_entry.d</code>.</p>

<pre><code class="language-C">/* syscall_entry.d */
syscall:*:*:entry
{
  printf("\tprocess_name: %s", execname);
}
</code></pre>

<p>The first line of this script tells DTrace which probes to use. The probe description <code class="highlighter-rouge">syscall:*:*:entry</code> is a pattern that matches every single probe associated with initiating a system call. DTrace has individual probes for every possible system call. If DTrace were to not have built-in functionality for matching multiple probes, it would have been necessary to manually specify every single system call probe, thereby making the above script a whole lot longer.</p>

<p>Before continuing on, it might be a good idea to briefly cover some DTrace terminology. Every DTrace probe adheres to the <code class="highlighter-rouge">&lt;provider&gt;:&lt;module&gt;:&lt;function&gt;:&lt;name&gt;</code> description format. The script above asks DTrace to match all probes of the <code class="highlighter-rouge">syscall</code> provider that have <code class="highlighter-rouge">entry</code> as their name. In this particular example, the <code class="highlighter-rouge">*</code> character was used as a wildcard so as to match multiple probes. However, keep in mind that the use of the <code class="highlighter-rouge">*</code> character is optional. Most DTrace documentation would have opted to write the equivalent <code class="highlighter-rouge">syscall:::entry</code> instead.</p>

<p>The rest of the script is rather straightforward. DTrace is being instructed to print the <code class="highlighter-rouge">execname</code> every time a probe fires. The <code class="highlighter-rouge">execname</code> is a built-in DTrace variable that holds the name of the process that caused the probe to be fired. Let’s go ahead and run this simple DTrace script.</p>

<div class="language-bash highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="nv">$ </span><span class="nb">sudo </span>dtrace <span class="nt">-s</span> syscall_entry.d

dtrace: script <span class="s1">'syscall_entry.d'</span> matched 500 probes
CPU     ID                    FUNCTION:NAME
  0    249                      ioctl:entry    process_name: dtrace
  0    373               gettimeofday:entry    process_name: java
  0    249                      ioctl:entry    process_name: dtrace
  0    751              psynch_cvwait:entry    process_name: java
  0    545                     sysctl:entry    process_name: dtrace
  0    545                     sysctl:entry    process_name: dtrace
  0    233                  sigaction:entry    process_name: dtrace
  0    233                  sigaction:entry    process_name: dtrace
  0    751              psynch_cvwait:entry    process_name: dtrace
  0    889                 kevent_qos:entry    process_name: Google Chrome Helper
  0    889                 kevent_qos:entry    process_name: Google Chrome Helper
  0    877           workq_kernreturn:entry    process_name: notifyd
  ...
</code></pre></div></div>

<p>The first thing to notice is that <code class="highlighter-rouge">syscall:*:*:entry</code> matches 500 different probes. At first glance this might seem like a lot, but on my machine there are well over 330,000 probes available. Listing all DTrace probes on your machine can be done by running <code class="highlighter-rouge">sudo dtrace -l</code>.</p>

<p>The second thing to notice is the insane amount of data returned by DTrace. The snippet above really doesn’t do justice to the many hundreds of lines of output. The next section will cover how to efficiently filter DTrace output.</p>

<p>Before moving on, it might be good to point out that the D scripting language is not Turing complete. It lacks such features as conditional branching and loops. DTrace is built around the ideas of minimal overhead and absolute safety. Giving people the ability to use DTrace to introduce arbitrary overhead on top of system calls does not fit with these ideas.</p>

<h3 id="ruby-and-dtrace">Ruby and DTrace</h3>

<p>DTrace probes have been supported by Ruby <a href="https://tenderlovemaking.com/2011/12/05/profiling-rails-startup-with-dtrace.html">since Ruby 2.0</a>. A list of supported Ruby probes can be found <a href="http://ruby-doc.org/core-2.2.3/doc/dtrace_probes_rdoc.html">here</a>. Now might be a good time to mention that DTrace probes come in two flavors: dynamic probes and static probes. Dynamic probes only appear in the <code class="highlighter-rouge">pid</code> and <code class="highlighter-rouge">fbt</code> probe providers. This means that the vast majority of available probes (including Ruby probes) is static.</p>

<p>So how exactly do dynamic and static probes differ? A closer look at DTrace’s inner workings is required before this difference can be explained. By invoking DTrace on a process, DTrace effectively gets given permission to patch additional DTrace instrumentation instructions into the address space of this process. Remember how the System Integrity Protection check had to be disabled in order to get DTrace to work on El Capitan? This is why.</p>

<p>In the case of dynamic probes, DTrace instrumentation instructions only get patched into a process when DTrace is invoked on this process. In other words, dynamic probes add zero overhead when not enabled. Static probes on the other hand have to be compiled into the binary that wants to make use of them. This is done through a <a href="https://github.com/ruby/ruby/blob/trunk/probes.d">probes.d file</a>.</p>

<p>However, even when probes have been compiled into the binary, this does not necessarily mean that they are getting triggered. When a process with static probes in its binary does not have DTrace invoked on it, any probe instructions get converted into NOP operations.
This usually introduces a negligible, but nevertheless non-zero, performance impact. More information about DTrace overhead can be found <a href="http://dtrace.org/blogs/brendan/2011/02/18/dtrace-pid-provider-overhead/">here</a>, <a href="http://www.solarisinternals.com/wiki/index.php/DTrace_Topics_Overhead#Dynamic_Probes">here</a>, and <a href="http://www.solarisinternals.com/wiki/index.php/DTrace_Topics_Overhead#Static_Probes">here</a>.</p>

<p>Having immersed ourselves in all things probe-related, let’s go ahead and list which DTrace probes are available for a Ruby process.</p>

<div class="language-bash highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="nv">$ </span><span class="nb">sudo </span>dtrace <span class="nt">-l</span> <span class="nt">-m</span> ruby <span class="nt">-c</span> <span class="s1">'ruby -v'</span>

ID         PROVIDER       MODULE              FUNCTION      NAME
114188    ruby86029         ruby       empty_ary_alloc      array-create
114189    ruby86029         ruby               ary_new      array-create
114190    ruby86029         ruby         vm_call_cfunc      cmethod-entry
114191    ruby86029         ruby         vm_call0_body      cmethod-entry
114192    ruby86029         ruby          vm_exec_core      cmethod-entry
114193    ruby86029         ruby         vm_call_cfunc      cmethod-return
114194    ruby86029         ruby         vm_call0_body      cmethod-return
114195    ruby86029         ruby            rb_iterate      cmethod-return
114196    ruby86029         ruby          vm_exec_core      cmethod-return
114197    ruby86029         ruby       rb_require_safe      find-require-entry
114198    ruby86029         ruby       rb_require_safe      find-require-return
114199    ruby86029         ruby              gc_marks      gc-mark-begin
...
</code></pre></div></div>

<p>Let’s start by taking a look at the command entered here. Once again, <code class="highlighter-rouge">-l</code> is used to have DTrace list its probes. The <code class="highlighter-rouge">-l</code> parameter is accompanied by <code class="highlighter-rouge">-m ruby</code> in order to limit the listing to probes from the ruby module. However, DTrace will not list any Ruby probes without being invoked on a Ruby process. This is why <code class="highlighter-rouge">-c 'ruby -v'</code> is needed here. The <code class="highlighter-rouge">-c</code> parameter allows specifying a command that will create a process for DTrace to run against. In this particular case, <code class="highlighter-rouge">ruby -v</code> causes a small Ruby process to be spawned in order to allow DTrace to list its Ruby probes.</p>

<p>The above snippet doesn’t actually list all Ruby probes, as the <code class="highlighter-rouge">sudo dtrace -l</code> command will omit any probes from the pid provider. This is because the pid provider actually defines a <a href="http://dtrace.org/guide/chp-pid.html">class of providers</a>, each of which gets its own set of probes depending on which process is being traced. Each pid probe corresponds to an internal C function that can be called by that particular process. The snippet below shows how to list the Ruby specific probes of this provider.</p>

<div class="language-bash highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="nv">$ </span><span class="nb">sudo </span>dtrace <span class="nt">-l</span> <span class="nt">-n</span> <span class="s1">'pid$target:::entry'</span> <span class="nt">-c</span> <span class="s1">'ruby -v'</span> | <span class="nb">grep</span> <span class="s1">'ruby'</span>

ID         PROVIDER       MODULE                     FUNCTION      NAME
1395302    pid86272         ruby                   rb_ary_eql      entry
1395303    pid86272         ruby                  rb_ary_hash      entry
1395304    pid86272         ruby                  rb_ary_aset      entry
1395305    pid86272         ruby                    rb_ary_at      entry
1395306    pid86272         ruby                 rb_ary_fetch      entry
1395307    pid86272         ruby                 rb_ary_first      entry
1395308    pid86272         ruby                rb_ary_push_m      entry
1395309    pid86272         ruby                 rb_ary_pop_m      entry
1395310    pid86272         ruby               rb_ary_shift_m      entry
1395311    pid86272         ruby                rb_ary_insert      entry
1395312    pid86272         ruby            rb_ary_each_index      entry
1395313    pid86272         ruby          rb_ary_reverse_each      entry
...
</code></pre></div></div>

<p>Only the pid entry probes are listed here, but keep in mind that every entry probe has a corresponding pid return probe. These probes are great as they allow for measuring which internal functions are getting called, the arguments passed to these, their return values, and even the offset in the function of the return instruction (useful for when a function has multiple return instructions). Additional information about the pid provider can be found <a href="http://dtrace.org/blogs/brendan/2011/02/09/dtrace-pid-provider">here</a>.</p>

<h3 id="a-first-dtrace-script-for-ruby">A first DTrace script for Ruby</h3>

<p>Let’s have a look at a first DTrace script for Ruby that measures when a Ruby method starts and stops executing along with the method’s execution time. This yet-to-be-written DTrace script will be getting invoked on the simple Ruby program shown below.</p>

<div class="language-ruby highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="c1"># sleepy.rb</span>
<span class="k">def</span> <span class="nf">even</span><span class="p">(</span><span class="n">rnd</span><span class="p">)</span>
  <span class="nb">sleep</span><span class="p">(</span><span class="n">rnd</span><span class="p">)</span>
<span class="k">end</span>

<span class="k">def</span> <span class="nf">odd</span><span class="p">(</span><span class="n">rnd</span><span class="p">)</span>
  <span class="nb">sleep</span><span class="p">(</span><span class="n">rnd</span><span class="p">)</span>
<span class="k">end</span>

<span class="kp">loop</span> <span class="k">do</span>
  <span class="n">rnd</span> <span class="o">=</span> <span class="nb">rand</span><span class="p">(</span><span class="mi">4</span><span class="p">)</span>
  <span class="p">(</span><span class="n">rnd</span> <span class="o">%</span> <span class="mi">2</span> <span class="o">==</span> <span class="mi">0</span><span class="p">)</span> <span class="p">?</span> <span class="n">even</span><span class="p">(</span><span class="n">rnd</span><span class="p">)</span> <span class="p">:</span> <span class="n">odd</span><span class="p">(</span><span class="n">rnd</span><span class="p">)</span>
<span class="k">end</span>
</code></pre></div></div>

<p>This simple Ruby program is clearly not going to win any awards. It is just one endless loop, each iteration of which calls a method depending on whether a random number was even or odd. While this is obviously a very contrived example, it is nevertheless very useful for illustrating the power of DTrace.</p>

<pre><code class="language-C">/* sleepy.d */
ruby$target:::method-entry
{
  self-&gt;start = timestamp;
  printf("Entering Method: class: %s, method: %s, file: %s, line: %d\n", copyinstr(arg0), copyinstr(arg1), copyinstr(arg2), arg3);
}

ruby$target:::method-return
{
  printf("Returning After: %d nanoseconds\n", (timestamp - self-&gt;start));
}
</code></pre>

<p>The above DTrace script makes use of two Ruby specific DTrace probes. The <code class="highlighter-rouge">method-entry</code> probe fires whenever a Ruby method is entered; the <code class="highlighter-rouge">method-return</code> probe fires whenever a Ruby method returns. Each probe can take multiple arguments. A probe’s arguments are available in the DTrace script through the <code class="highlighter-rouge">arg0</code>, <code class="highlighter-rouge">arg1</code>, <code class="highlighter-rouge">arg2</code> and <code class="highlighter-rouge">arg3</code> variables.</p>

<p>Figuring out what data is contained by a probe’s arguments is as easy as looking at <a href="http://ruby-doc.org/core-2.2.3/doc/dtrace_probes_rdoc.html#label-Declared+probes">its documentation</a>. In this particular case, the <code class="highlighter-rouge">method-entry</code> probe gets called by the Ruby process with exactly four arguments.</p>

<blockquote>
  <p>ruby:::method-entry(classname, methodname, filename, lineno);</p>

  <ul>
    <li>classname: name of the class (a string)</li>
    <li>methodname: name of the method about to be executed (a string)</li>
    <li>filename: the file name where the method is being called (a string)</li>
    <li>lineno: the line number where the method is being called (an int)</li>
  </ul>
</blockquote>

<p>The documentation shows that <code class="highlighter-rouge">arg0</code> holds the class name, <code class="highlighter-rouge">arg1</code> holds the method name, and so on. Equally important is that the first three arguments are strings, while the fourth one is an integer. This type information is needed for printing any of these arguments with <code class="highlighter-rouge">printf</code>.</p>

<p>One noticeable item in the above DTrace script is the wrapping of string variables inside the <code class="highlighter-rouge">copyinstr</code> method. The reason for this is a bit complex. When a string is passed as an argument to a DTrace probe, the probe doesn’t actually receive the entire string. Instead, it receives the memory address where the string begins. This memory address will be specific to the address space of the Ruby process. However, DTrace probes get executed in the kernel and thus make use of a different address space than the Ruby process. In order for a probe to read a string residing in user process data, the string contents first need to be copied into the kernel’s address space. The <code class="highlighter-rouge">copyinstr</code> method is a built-in DTrace function that takes care of this.</p>

<p>The <code class="highlighter-rouge">self-&gt;start</code> notation is interesting as well. DTrace variables starting with <code class="highlighter-rouge">self-&gt;</code> are thread-local variables. Thread-local variables make it possible to tag the thread that caused the probe to be fired with some data. In the above example <code class="highlighter-rouge">self-&gt;start = timestamp;</code> is used to tag every thread that triggers the <code class="highlighter-rouge">method-entry</code> probe with a thread-local <code class="highlighter-rouge">start</code> variable that contains the time in nanoseconds returned by the built-in <code class="highlighter-rouge">timestamp</code> method.</p>

<p>While it is impossible for one thread to access the thread-local variables of another thread, it is perfectly possible for a given probe to access the thread-local variables that were set on the current thread by another probe. Notice how the thread-local <code class="highlighter-rouge">self-&gt;start</code> variable is being shared between both the <code class="highlighter-rouge">method-entry</code> and <code class="highlighter-rouge">method-return</code> probes in the above DTrace script.</p>

<p>Let’s go ahead and invoke this DTrace script on the example Ruby program.</p>

<div class="language-bash highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="nv">$ </span><span class="nb">sudo </span>dtrace <span class="nt">-q</span> <span class="nt">-s</span> sleepy.d <span class="nt">-c</span> <span class="s1">'ruby sleepy.rb'</span>

Entering Method: class: RbConfig, method: expand, file: /Users/vaneyckt/.rvm/rubies/ruby-2.2.3/lib/ruby/2.2.0/x86_64-darwin14/rbconfig.rb, line: 241
Returning After: 39393 nanoseconds
Entering Method: class: RbConfig, method: expand, file: /Users/vaneyckt/.rvm/rubies/ruby-2.2.3/lib/ruby/2.2.0/x86_64-darwin14/rbconfig.rb, line: 241
Returning After: 12647 nanoseconds
Entering Method: class: RbConfig, method: expand, file: /Users/vaneyckt/.rvm/rubies/ruby-2.2.3/lib/ruby/2.2.0/x86_64-darwin14/rbconfig.rb, line: 241
Returning After: 11584 nanoseconds
...
...
...
Entering Method: class: Object, method: odd, file: sleepy.rb, line: 5
Returning After: 1003988894 nanoseconds
Entering Method: class: Object, method: odd, file: sleepy.rb, line: 5
Returning After: 1003887374 nanoseconds
Entering Method: class: Object, method: even, file: sleepy.rb, line: 1
Returning After: 15839 nanoseconds
</code></pre></div></div>

<p>It’s a bit hard to convey in the snippet above, but this DTrace invocation is generating well over a thousand lines of output. This output can be divided into two sections: a first section listing all the Ruby methods being called as part of the program getting ready to run, and a much smaller second section listing whether the Ruby program is calling the <code class="highlighter-rouge">even</code> or <code class="highlighter-rouge">odd</code> functions, along with the time spent in each of these function calls.</p>

<p>Such a large amount of output can quickly get overwhelming. It is oftentimes desirable to limit the output to just those lines pertaining to the <code class="highlighter-rouge">even</code> and <code class="highlighter-rouge">odd</code> methods getting called. DTrace uses predicates to make just this type of filtering possible. Predicates are <code class="highlighter-rouge">/</code> wrapped conditions that define whether a particular probe should be executed. The code below shows how to use predicates so as to only have the <code class="highlighter-rouge">method-entry</code> and <code class="highlighter-rouge">method-return</code> probes trigger when the <code class="highlighter-rouge">even</code> and <code class="highlighter-rouge">odd</code> methods get called.</p>

<pre><code class="language-C">/* predicates_sleepy.d */
ruby$target:::method-entry
/copyinstr(arg1) == "even" || copyinstr(arg1) == "odd"/
{
  self-&gt;start = timestamp;
  printf("Entering Method: class: %s, method: %s, file: %s, line: %d\n", copyinstr(arg0), copyinstr(arg1), copyinstr(arg2), arg3);
}

ruby$target:::method-return
/copyinstr(arg1) == "even" || copyinstr(arg1) == "odd"/
{
  printf("Returning After: %d nanoseconds\n", (timestamp - self-&gt;start));
}
</code></pre>

<div class="language-bash highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="nv">$ </span><span class="nb">sudo </span>dtrace <span class="nt">-q</span> <span class="nt">-s</span> predicates_sleepy.d <span class="nt">-c</span> <span class="s1">'ruby sleepy.rb'</span>

Entering Method: class: Object, method: odd, file: sleepy.rb, line: 5
Returning After: 3005086754 nanoseconds
Entering Method: class: Object, method: even, file: sleepy.rb, line: 1
Returning After: 2004313007 nanoseconds
Entering Method: class: Object, method: even, file: sleepy.rb, line: 1
Returning After: 2005076442 nanoseconds
Entering Method: class: Object, method: even, file: sleepy.rb, line: 1
Returning After: 21304 nanoseconds
...
</code></pre></div></div>

<p>This modified DTrace script now only has its probes triggered when the Ruby program enters into or returns from an <code class="highlighter-rouge">even</code> or <code class="highlighter-rouge">odd</code> method. This causes the DTrace output to be much more relevant and succinct. Having now covered a fair few DTrace basics, let’s continue onwards to the topic of writing a DTrace script for measuring mutex contention in Ruby programs.</p>

<h3 id="monitoring-mutex-contention-with-dtrace">Monitoring mutex contention with DTrace</h3>

<p>The goal of this section is to come up with a DTrace script that measures mutex contention in a multi-threaded Ruby program. This is far from a trivial undertaking and will require some investigation of the Ruby Interpreter source code. But first, let’s begin by taking a look at this section’s example Ruby program..</p>

<div class="language-ruby highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="c1"># mutex.rb</span>
<span class="n">mutex</span> <span class="o">=</span> <span class="no">Mutex</span><span class="p">.</span><span class="nf">new</span>
<span class="n">threads</span> <span class="o">=</span> <span class="p">[]</span>

<span class="n">threads</span> <span class="o">&lt;&lt;</span> <span class="no">Thread</span><span class="p">.</span><span class="nf">new</span> <span class="k">do</span>
  <span class="kp">loop</span> <span class="k">do</span>
    <span class="n">mutex</span><span class="p">.</span><span class="nf">synchronize</span> <span class="k">do</span>
      <span class="nb">sleep</span> <span class="mi">2</span>
    <span class="k">end</span>
  <span class="k">end</span>
<span class="k">end</span>

<span class="n">threads</span> <span class="o">&lt;&lt;</span> <span class="no">Thread</span><span class="p">.</span><span class="nf">new</span> <span class="k">do</span>
  <span class="kp">loop</span> <span class="k">do</span>
    <span class="n">mutex</span><span class="p">.</span><span class="nf">synchronize</span> <span class="k">do</span>
      <span class="nb">sleep</span> <span class="mi">4</span>
    <span class="k">end</span>
  <span class="k">end</span>
<span class="k">end</span>

<span class="n">threads</span><span class="p">.</span><span class="nf">each</span><span class="p">(</span><span class="o">&amp;</span><span class="ss">:join</span><span class="p">)</span>
</code></pre></div></div>

<p>The above Ruby code starts by creating a mutex object, after which it kicks off two threads. Each thread runs an infinite loop that causes the thread to grab the mutex for a bit before releasing it again. Since the second thread holds onto the mutex for longer than the first thread, it is intuitively obvious that the first thread will spend a fair amount of time waiting for the second thread to release the mutex.</p>

<p>The goal of this post is to write a DTrace script that tracks when a given thread has to wait for a mutex to become available, as well as which particular thread is holding the mutex at that point in time. To the best of my knowledge, it is impossible to obtain this contention information by monkey patching the Mutex object, which makes this a great showcase for DTrace. Please get in touch if you think I am wrong on this.</p>

<p>Writing such a DTrace script is no easy task; it requires a fair bit of knowledge about what exactly happens when a thread calls <code class="highlighter-rouge">synchronize</code> on a Mutex object. The Mutex object and its methods are implemented as part of the <a href="https://github.com/ruby/ruby/tree/ruby_2_2">Ruby MRI</a>, which is written in C. Invoking <code class="highlighter-rouge">synchronize</code> on a Mutex object causes the following methods to be called:</p>

<ol>
  <li><code class="highlighter-rouge">synchronize</code> (<a href="https://github.com/ruby/ruby/blob/325587ee7f76cbcabbc1e6d181cfacb976c39b52/thread_sync.c#L1254">source</a>) calls <code class="highlighter-rouge">rb_mutex_synchronize_m</code></li>
  <li><code class="highlighter-rouge">rb_mutex_synchronize_m</code> (<a href="https://github.com/ruby/ruby/blob/325587ee7f76cbcabbc1e6d181cfacb976c39b52/thread_sync.c#L502">source</a>) checks if <code class="highlighter-rouge">synchronize</code> was called with a block and then goes on to call <code class="highlighter-rouge">rb_mutex_synchronize</code></li>
  <li><code class="highlighter-rouge">rb_mutex_synchronize</code> (<a href="https://github.com/ruby/ruby/blob/325587ee7f76cbcabbc1e6d181cfacb976c39b52/thread_sync.c#L488">source</a>) calls <code class="highlighter-rouge">rb_mutex_lock</code></li>
  <li><code class="highlighter-rouge">rb_mutex_lock</code> (<a href="https://github.com/ruby/ruby/blob/325587ee7f76cbcabbc1e6d181cfacb976c39b52/thread_sync.c#L241">source</a>) is where the currently active Ruby thread that executed the <code class="highlighter-rouge">mutex.synchronize</code> code will try to grab the mutex</li>
</ol>

<p>There’s a lot going on in <code class="highlighter-rouge">rb_mutex_lock</code>. The most interesting thing is the call to <code class="highlighter-rouge">rb_mutex_trylock</code> (<a href="https://github.com/ruby/ruby/blob/325587ee7f76cbcabbc1e6d181cfacb976c39b52/thread_sync.c#L157">source</a>) on <a href="https://github.com/ruby/ruby/blob/325587ee7f76cbcabbc1e6d181cfacb976c39b52/thread_sync.c#L252">line 252</a>. This method immediately returns <code class="highlighter-rouge">true</code> or <code class="highlighter-rouge">false</code> depending on whether the Ruby thread managed to grab the mutex. Following the code from line 252 onwards, notice how <code class="highlighter-rouge">rb_mutex_trylock</code> returning <code class="highlighter-rouge">true</code> causes <code class="highlighter-rouge">rb_mutex_lock</code> to immediately return as well. On the other hand, <code class="highlighter-rouge">rb_mutex_lock</code> returning <code class="highlighter-rouge">false</code> causes <code class="highlighter-rouge">rb_mutex_lock</code> to keep executing (and occasionally blocking) until the Ruby thread has managed to get a hold of the mutex.</p>

<p>The behavior of <code class="highlighter-rouge">rb_mutex_lock</code> gives a vital clue about how to write our DTrace script. When a thread starts executing <code class="highlighter-rouge">rb_mutex_lock</code>, this means it wants to acquire a mutex; when a thread returns from <code class="highlighter-rouge">rb_mutex_lock</code>, this means it has managed to successfully obtain a mutex. It is therefore possible to create a DTrace script for measuring mutex contention by using DTrace pid probes that fire upon a thread entering into or returning from the <code class="highlighter-rouge">rb_mutex_lock</code> method.</p>

<p>Let’s go over what exactly such a DTrace script should do:</p>

<ol>
  <li>when the Ruby program calls <code class="highlighter-rouge">mutex.synchronize</code>, it should make a note of which particular file and line these instructions appear on. This will allow the DTrace script to report the exact location of any code causing mutex contention.</li>
  <li>when <code class="highlighter-rouge">rb_mutex_lock</code> starts executing, the script should write down the current timestamp, as this is when the thread starts trying to acquire the mutex</li>
  <li>when <code class="highlighter-rouge">rb_mutex_lock</code> returns, the script should compare the current timestamp with the one written down earlier, as this shows how long the thread had to wait trying to acquire the mutex. This duration, along with some information about the location of the <code class="highlighter-rouge">mutex.synchronize</code> call, should be printed to the terminal.</li>
</ol>

<p>Putting it all together results in the DTrace script shown below.</p>

<pre><code class="language-C">/* mutex.d */
ruby$target:::cmethod-entry
/copyinstr(arg0) == "Mutex" &amp;&amp; copyinstr(arg1) == "synchronize"/
{
  self-&gt;file = copyinstr(arg2);
  self-&gt;line = arg3;
}

pid$target:ruby:rb_mutex_lock:entry
/self-&gt;file != NULL &amp;&amp; self-&gt;line != NULL/
{
  self-&gt;mutex_wait_start = timestamp;
}

pid$target:ruby:rb_mutex_lock:return
/self-&gt;file != NULL &amp;&amp; self-&gt;line != NULL/
{
  mutex_wait_ms = (timestamp - self-&gt;mutex_wait_start) / 1000;
  printf("Thread %d acquires mutex %d after %d ms - %s:%d\n", tid, arg1, mutex_wait_ms, self-&gt;file, self-&gt;line);
  self-&gt;file = NULL;
  self-&gt;line = NULL;
}
</code></pre>

<p>The snippet above contains three different probes, the first of which is a Ruby probe that fires whenever a C method is entered. Since the Mutex class and its methods have been implemented in C as <a href="https://github.com/ruby/ruby/blob/325587ee7f76cbcabbc1e6d181cfacb976c39b52/thread_sync.c#L1246-L1255">part of the Ruby MRI</a>, it makes sense to use a <code class="highlighter-rouge">cmethod-entry</code> probe. Note the use of a predicate to ensure this probe only gets triggered when its first two arguments are “Mutex” and “synchronize”. Remember how these arguments <a href="https://ruby-doc.org/core-2.1.0/doc/dtrace_probes_rdoc.html">correspond to the class and method name</a> of the Ruby code that triggered the probe. So this predicate guarantees that this particular probe will only fire when the Ruby code calls the <code class="highlighter-rouge">synchronize</code> method on a Mutex object.</p>

<p>The rest of this probe is rather straightforward. It just stores the file and line number of the Ruby code that triggered the probe into thread-local variables. Thread-local variables are used here for two reasons. Firstly, thread-local variables make it trivial to share data with other probes. Secondly, Ruby programs that make use of mutexes will generally be running multiple threads. Using thread-local variables ensures that each Ruby thread will get its own set of probe-specific variables.</p>

<p>The second probe comes from the pid provider. This provider provides a probe for every internal method of a process. It is used here to create a probe that gets triggered whenever <code class="highlighter-rouge">rb_mutex_lock</code> starts executing. Remember that a thread will invoke this particular method when starting to acquire a mutex! The probe itself is pretty simple in that it just stores the current time in a thread-local variable, so as to keep track of when a thread started trying to obtain a mutex. A simple predicate is also used to ensure this probe can only be triggered after the previous probe has fired.</p>

<p>The final probe fires whenever <code class="highlighter-rouge">rb_mutex_lock</code> finishes executing. It has a similar predicate as the second probe so as to ensure it can only be triggered after the first probe has fired. Knowing that <code class="highlighter-rouge">rb_mutex_lock</code> returns whenever a thread has successfully obtained a lock, the time spent waiting on this lock can be easily calculated by comparing the current time with the previously stored <code class="highlighter-rouge">self-&gt;mutex_wait_start</code> variable. The time spent waiting, along with the IDs of the current thread and mutex, as well as the location of where the call to <code class="highlighter-rouge">mutex.synchronize</code> took place are then printed to the DTrace output. This probe then concludes by assigning <code class="highlighter-rouge">NULL</code> to the <code class="highlighter-rouge">self-&gt;file</code> and <code class="highlighter-rouge">self-&gt;line</code> variables, so as to ensure that the second and third probe can only be triggered after the first one has fired again.</p>

<p>Let’s have a quick chat about how exactly the thread and mutex IDs are obtained. The thread ID is accessible through <code class="highlighter-rouge">tid</code>; this is a built-in DTrace variable that identifies the current thread. The mutex ID is a bit more complex. A <code class="highlighter-rouge">pid:::return</code> probe stores the return value of the method that triggered it inside its <code class="highlighter-rouge">arg1</code> variable. The <code class="highlighter-rouge">rb_mutex_lock</code> method just happens to <a href="https://github.com/ruby/ruby/blob/325587ee7f76cbcabbc1e6d181cfacb976c39b52/thread_sync.c#L306">return an identifier for the mutex that was passed to it</a>, so the <code class="highlighter-rouge">arg1</code> variable of this probe does indeed contain the mutex ID.</p>

<p>The final result looks a lot like this.</p>

<div class="language-bash highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="nv">$ </span><span class="nb">sudo </span>dtrace <span class="nt">-q</span> <span class="nt">-s</span> mutex.d <span class="nt">-c</span> <span class="s1">'ruby mutex.rb'</span>

Thread 286592 acquires mutex 4313316240 after 2 ms - mutex.rb:14
Thread 286591 acquires mutex 4313316240 after 4004183 ms - mutex.rb:6
Thread 286592 acquires mutex 4313316240 after 2004170 ms - mutex.rb:14
Thread 286592 acquires mutex 4313316240 after 6 ms - mutex.rb:14
Thread 286592 acquires mutex 4313316240 after 4 ms - mutex.rb:14
Thread 286592 acquires mutex 4313316240 after 4 ms - mutex.rb:14
Thread 286591 acquires mutex 4313316240 after 16012158 ms - mutex.rb:6
Thread 286592 acquires mutex 4313316240 after 2002593 ms - mutex.rb:14
Thread 286591 acquires mutex 4313316240 after 4001983 ms - mutex.rb:6
Thread 286592 acquires mutex 4313316240 after 2004418 ms - mutex.rb:14
Thread 286591 acquires mutex 4313316240 after 4000407 ms - mutex.rb:6
Thread 286592 acquires mutex 4313316240 after 2004163 ms - mutex.rb:14
Thread 286591 acquires mutex 4313316240 after 4003191 ms - mutex.rb:6
Thread 286591 acquires mutex 4313316240 after 2 ms - mutex.rb:6
Thread 286592 acquires mutex 4313316240 after 4005587 ms - mutex.rb:14
...
</code></pre></div></div>

<p>The above output actually provides some interesting info about our program:</p>

<ol>
  <li>there are two threads: 286591 and 286592</li>
  <li>both threads try to acquire mutex 4313316240</li>
  <li>the mutex acquisition code of the fist thread lives at line 6 of the mutex.rb file</li>
  <li>the acquisition code of the second thread is located at line 14 of the same file</li>
  <li>there is a lot of mutex contention, with threads having to wait several seconds for the mutex to become available</li>
</ol>

<p>None of the above should come as a surprise due to our familiarity with the Ruby program’s source code. The real power of DTrace lies in making it possible to run this <code class="highlighter-rouge">mutex.d</code> script against any Ruby program, no matter how complex, and obtain this level of information without having to read any source code at all. Taking this one step further, DTrace even allows running this mutex contention script against an already running Ruby process with <code class="highlighter-rouge">sudo dtrace -q -s mutex.d -p &lt;pid&gt;</code>. This can be helpful for diagnosing issues in active production code.</p>

<p>Before moving on to the next section, it’d be good to point out that the above DTrace output actually shows some cool stuff about how the Ruby MRI schedules threads. Lines 3-6 of the output show that the second thread gets scheduled four times in a row. This indicates that when multiple threads are competing for a mutex, the Ruby MRI does not care if any of these threads was recently held by that mutex.</p>

<h3 id="advanced-mutex-contention-monitoring">Advanced mutex contention monitoring</h3>

<p>The above DTrace script can be taken one step further by adding an additional probe that triggers whenever a thread releases a mutex. Another improvement can be made by changing the script to print timestamps instead of durations. While this makes it possible to construct a chronological sequence of the goings-on of a program’s mutexes, it does so at the cost of making the script’s output less suitable for human consumption.</p>

<p>Luckily, the problem of human-friendly output can be easily solved by using a Ruby program to aggregate this new output into something more suited for human consumption. As an aside, DTrace actually has built-in logic for aggregating data, but I personally prefer to focus my DTrace usage on obtaining data that would otherwise be hard to get, while having my aggregation logic live somewhere else.</p>

<p>Let’s start by having a look at how to add a probe that can detect a mutex being released. Luckily, it turns out there is a C method called <code class="highlighter-rouge">rb_mutex_unlock</code> (<a href="https://github.com/ruby/ruby/blob/325587ee7f76cbcabbc1e6d181cfacb976c39b52/thread_sync.c#L371">source</a>) that releases mutexes. Similarly to <code class="highlighter-rouge">rb_mutex_lock</code>, this method returns an identifier to the mutex it acted on. The only thing that needs to be done is adding a probe that fires whenever <code class="highlighter-rouge">rb_mutex_unlock</code> returns. The final script is shown here.</p>

<pre><code class="language-C">/* mutex.d */
ruby$target:::cmethod-entry
/copyinstr(arg0) == "Mutex" &amp;&amp; copyinstr(arg1) == "synchronize"/
{
  self-&gt;file = copyinstr(arg2);
  self-&gt;line = arg3;
}

pid$target:ruby:rb_mutex_lock:entry
/self-&gt;file != NULL &amp;&amp; self-&gt;line != NULL/
{
  printf("Thread %d wants to acquire mutex %d at %d - %s:%d\n", tid, arg1, timestamp, self-&gt;file, self-&gt;line);
}

pid$target:ruby:rb_mutex_lock:return
/self-&gt;file != NULL &amp;&amp; self-&gt;line != NULL/
{
  printf("Thread %d has acquired mutex %d at %d - %s:%d\n", tid, arg1, timestamp, self-&gt;file, self-&gt;line);
  self-&gt;file = NULL;
  self-&gt;line = NULL;
}

pid$target:ruby:rb_mutex_unlock:return
{
  printf("Thread %d has released mutex %d at %d\n", tid, arg1, timestamp);
}
</code></pre>

<div class="language-bash highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="nv">$ </span><span class="nb">sudo </span>dtrace <span class="nt">-q</span> <span class="nt">-s</span> mutex.d <span class="nt">-c</span> <span class="s1">'ruby mutex.rb'</span>

Thread 500152 wants to acquire mutex 4330240800 at 53341356615492 - mutex.rb:6
Thread 500152 has acquired mutex 4330240800 at 53341356625449 - mutex.rb:6
Thread 500153 wants to acquire mutex 4330240800 at 53341356937292 - mutex.rb:14
Thread 500152 has released mutex 4330240800 at 53343360214311
Thread 500152 wants to acquire mutex 4330240800 at 53343360266121 - mutex.rb:6
Thread 500153 has acquired mutex 4330240800 at 53343360301928 - mutex.rb:14
Thread 500153 has released mutex 4330240800 at 53347365475537
Thread 500153 wants to acquire mutex 4330240800 at 53347365545277 - mutex.rb:14
Thread 500152 has acquired mutex 4330240800 at 53347365661847 - mutex.rb:6
Thread 500152 has released mutex 4330240800 at 53349370397555
Thread 500152 wants to acquire mutex 4330240800 at 53349370426972 - mutex.rb:6
Thread 500153 has acquired mutex 4330240800 at 53349370453489 - mutex.rb:14
Thread 500153 has released mutex 4330240800 at 53353374785751
Thread 500153 wants to acquire mutex 4330240800 at 53353374834184 - mutex.rb:14
Thread 500152 has acquired mutex 4330240800 at 53353374868435 - mutex.rb:6
...
</code></pre></div></div>

<p>The above output is pretty hard to parse for human readers. The snippet below shows a Ruby program that aggregates this data into a more readable format. This aggregation program just does some string filtering and bookkeeping in order to keep track of how each thread interacts with the mutexes and the contention caused by this.</p>

<div class="language-ruby highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="c1"># aggregate.rb</span>
<span class="n">mutex_owners</span>     <span class="o">=</span> <span class="no">Hash</span><span class="p">.</span><span class="nf">new</span>
<span class="n">mutex_queuers</span>    <span class="o">=</span> <span class="no">Hash</span><span class="p">.</span><span class="nf">new</span> <span class="p">{</span> <span class="o">|</span><span class="n">h</span><span class="p">,</span><span class="n">k</span><span class="o">|</span> <span class="n">h</span><span class="p">[</span><span class="n">k</span><span class="p">]</span> <span class="o">=</span> <span class="no">Array</span><span class="p">.</span><span class="nf">new</span> <span class="p">}</span>
<span class="n">mutex_contention</span> <span class="o">=</span> <span class="no">Hash</span><span class="p">.</span><span class="nf">new</span> <span class="p">{</span> <span class="o">|</span><span class="n">h</span><span class="p">,</span><span class="n">k</span><span class="o">|</span> <span class="n">h</span><span class="p">[</span><span class="n">k</span><span class="p">]</span> <span class="o">=</span> <span class="no">Hash</span><span class="p">.</span><span class="nf">new</span><span class="p">(</span><span class="mi">0</span><span class="p">)</span> <span class="p">}</span>

<span class="n">time_of_last_update</span> <span class="o">=</span> <span class="no">Time</span><span class="p">.</span><span class="nf">now</span>
<span class="n">update_interval_sec</span> <span class="o">=</span> <span class="mi">1</span>

<span class="no">ARGF</span><span class="p">.</span><span class="nf">each</span> <span class="k">do</span> <span class="o">|</span><span class="n">line</span><span class="o">|</span>
  <span class="c1"># when a thread wants to acquire a mutex</span>
  <span class="k">if</span> <span class="n">matches</span> <span class="o">=</span> <span class="n">line</span><span class="p">.</span><span class="nf">match</span><span class="p">(</span><span class="sr">/^Thread (\d+) wants to acquire mutex (\d+) at (\d+) - (.+)$/</span><span class="p">)</span>
    <span class="n">captures</span>  <span class="o">=</span> <span class="n">matches</span><span class="p">.</span><span class="nf">captures</span>
    <span class="n">thread_id</span> <span class="o">=</span> <span class="n">captures</span><span class="p">[</span><span class="mi">0</span><span class="p">]</span>
    <span class="n">mutex_id</span>  <span class="o">=</span> <span class="n">captures</span><span class="p">[</span><span class="mi">1</span><span class="p">]</span>
    <span class="n">timestamp</span> <span class="o">=</span> <span class="n">captures</span><span class="p">[</span><span class="mi">2</span><span class="p">].</span><span class="nf">to_i</span>
    <span class="n">location</span>  <span class="o">=</span> <span class="n">captures</span><span class="p">[</span><span class="mi">3</span><span class="p">]</span>

    <span class="n">mutex_queuers</span><span class="p">[</span><span class="n">mutex_id</span><span class="p">]</span> <span class="o">&lt;&lt;</span> <span class="p">{</span>
      <span class="ss">thread_id: </span><span class="n">thread_id</span><span class="p">,</span>
      <span class="ss">location:  </span><span class="n">location</span><span class="p">,</span>
      <span class="ss">timestamp: </span><span class="n">timestamp</span>
    <span class="p">}</span>
  <span class="k">end</span>

  <span class="c1"># when a thread has acquired a mutex</span>
  <span class="k">if</span> <span class="n">matches</span> <span class="o">=</span> <span class="n">line</span><span class="p">.</span><span class="nf">match</span><span class="p">(</span><span class="sr">/^Thread (\d+) has acquired mutex (\d+) at (\d+) - (.+)$/</span><span class="p">)</span>
    <span class="n">captures</span>  <span class="o">=</span> <span class="n">matches</span><span class="p">.</span><span class="nf">captures</span>
    <span class="n">thread_id</span> <span class="o">=</span> <span class="n">captures</span><span class="p">[</span><span class="mi">0</span><span class="p">]</span>
    <span class="n">mutex_id</span>  <span class="o">=</span> <span class="n">captures</span><span class="p">[</span><span class="mi">1</span><span class="p">]</span>
    <span class="n">timestamp</span> <span class="o">=</span> <span class="n">captures</span><span class="p">[</span><span class="mi">2</span><span class="p">].</span><span class="nf">to_i</span>
    <span class="n">location</span>  <span class="o">=</span> <span class="n">captures</span><span class="p">[</span><span class="mi">3</span><span class="p">]</span>

    <span class="c1"># set new owner</span>
    <span class="n">mutex_owners</span><span class="p">[</span><span class="n">mutex_id</span><span class="p">]</span> <span class="o">=</span> <span class="p">{</span>
      <span class="ss">thread_id: </span><span class="n">thread_id</span><span class="p">,</span>
      <span class="ss">location: </span><span class="n">location</span>
    <span class="p">}</span>

    <span class="c1"># remove new owner from list of queuers</span>
    <span class="n">mutex_queuers</span><span class="p">[</span><span class="n">mutex_id</span><span class="p">].</span><span class="nf">delete_if</span> <span class="k">do</span> <span class="o">|</span><span class="n">queuer</span><span class="o">|</span>
      <span class="n">queuer</span><span class="p">[</span><span class="ss">:thread_id</span><span class="p">]</span> <span class="o">==</span> <span class="n">thread_id</span> <span class="o">&amp;&amp;</span>
      <span class="n">queuer</span><span class="p">[</span><span class="ss">:location</span><span class="p">]</span> <span class="o">==</span> <span class="n">location</span>
    <span class="k">end</span>
  <span class="k">end</span>

  <span class="c1"># when a thread has released a mutex</span>
  <span class="k">if</span> <span class="n">matches</span> <span class="o">=</span> <span class="n">line</span><span class="p">.</span><span class="nf">match</span><span class="p">(</span><span class="sr">/^Thread (\d+) has released mutex (\d+) at (\d+)$/</span><span class="p">)</span>
    <span class="n">captures</span>  <span class="o">=</span> <span class="n">matches</span><span class="p">.</span><span class="nf">captures</span>
    <span class="n">thread_id</span> <span class="o">=</span> <span class="n">captures</span><span class="p">[</span><span class="mi">0</span><span class="p">]</span>
    <span class="n">mutex_id</span>  <span class="o">=</span> <span class="n">captures</span><span class="p">[</span><span class="mi">1</span><span class="p">]</span>
    <span class="n">timestamp</span> <span class="o">=</span> <span class="n">captures</span><span class="p">[</span><span class="mi">2</span><span class="p">].</span><span class="nf">to_i</span>

    <span class="n">owner_location</span> <span class="o">=</span> <span class="n">mutex_owners</span><span class="p">[</span><span class="n">mutex_id</span><span class="p">][</span><span class="ss">:location</span><span class="p">]</span>

    <span class="c1"># calculate how long the owner caused each queuer to wait</span>
    <span class="c1"># and change queuer timestamp to the current timestamp in preparation</span>
    <span class="c1"># for the next round of queueing</span>
    <span class="n">mutex_queuers</span><span class="p">[</span><span class="n">mutex_id</span><span class="p">].</span><span class="nf">each</span> <span class="k">do</span> <span class="o">|</span><span class="n">queuer</span><span class="o">|</span>
      <span class="n">mutex_contention</span><span class="p">[</span><span class="n">owner_location</span><span class="p">][</span><span class="n">queuer</span><span class="p">[</span><span class="ss">:location</span><span class="p">]]</span> <span class="o">+=</span> <span class="p">(</span><span class="n">timestamp</span> <span class="o">-</span> <span class="n">queuer</span><span class="p">[</span><span class="ss">:timestamp</span><span class="p">])</span>
      <span class="n">queuer</span><span class="p">[</span><span class="ss">:timestamp</span><span class="p">]</span> <span class="o">=</span> <span class="n">timestamp</span>
    <span class="k">end</span>
  <span class="k">end</span>

  <span class="c1"># print mutex contention information</span>
  <span class="k">if</span> <span class="no">Time</span><span class="p">.</span><span class="nf">now</span> <span class="o">-</span> <span class="n">time_of_last_update</span> <span class="o">&gt;</span> <span class="n">update_interval_sec</span>
    <span class="nb">system</span><span class="p">(</span><span class="s1">'clear'</span><span class="p">)</span>
    <span class="n">time_of_last_update</span> <span class="o">=</span> <span class="no">Time</span><span class="p">.</span><span class="nf">now</span>

    <span class="nb">puts</span> <span class="s1">'Mutex Contention'</span>
    <span class="nb">puts</span> <span class="s2">"================</span><span class="se">\n\n</span><span class="s2">"</span>

    <span class="n">mutex_contention</span><span class="p">.</span><span class="nf">each</span> <span class="k">do</span> <span class="o">|</span><span class="n">owner_location</span><span class="p">,</span> <span class="n">contention</span><span class="o">|</span>
      <span class="nb">puts</span> <span class="n">owner_location</span>
      <span class="n">owner_location</span><span class="p">.</span><span class="nf">length</span><span class="p">.</span><span class="nf">times</span> <span class="p">{</span> <span class="nb">print</span> <span class="s1">'-'</span> <span class="p">}</span>
      <span class="nb">puts</span> <span class="s2">"</span><span class="se">\n</span><span class="s2">"</span>

      <span class="n">total_duration_sec</span> <span class="o">=</span> <span class="mf">0.0</span>
      <span class="n">contention</span><span class="p">.</span><span class="nf">sort</span><span class="p">.</span><span class="nf">each</span> <span class="k">do</span> <span class="o">|</span><span class="n">queuer_location</span><span class="p">,</span> <span class="n">queueing_duration</span><span class="o">|</span>
        <span class="n">duration_sec</span> <span class="o">=</span> <span class="n">queueing_duration</span> <span class="o">/</span> <span class="mf">1000000000.0</span>
        <span class="n">total_duration_sec</span> <span class="o">+=</span> <span class="n">duration_sec</span>
        <span class="nb">puts</span> <span class="s2">"</span><span class="si">#{</span><span class="n">queuer_location</span><span class="si">}</span><span class="se">\t</span><span class="si">#{</span><span class="n">duration_sec</span><span class="si">}</span><span class="s2">s"</span>
      <span class="k">end</span>
      <span class="nb">puts</span> <span class="s2">"total</span><span class="se">\t\t</span><span class="si">#{</span><span class="n">total_duration_sec</span><span class="si">}</span><span class="s2">s</span><span class="se">\n\n</span><span class="s2">"</span>
    <span class="k">end</span>
  <span class="k">end</span>
<span class="k">end</span>
</code></pre></div></div>

<div class="language-bash highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="nv">$ </span><span class="nb">sudo </span>dtrace <span class="nt">-q</span> <span class="nt">-s</span> mutex.d <span class="nt">-c</span> <span class="s1">'ruby mutex.rb'</span> | ruby aggregate.rb


Mutex Contention
<span class="o">================</span>

mutex.rb:6
<span class="nt">----------</span>
mutex.rb:14	  10.016301065s
total		  10.016301065s

mutex.rb:14
<span class="nt">-----------</span>
mutex.rb:6	  16.019252339s
total		  16.019252339s
</code></pre></div></div>

<p>The final result looks like shown above. Note that the above Ruby program will clear the terminal every second before printing summarized contention information. The above output shows that, after having run the program for a bit, <code class="highlighter-rouge">mutex.rb:6</code> caused <code class="highlighter-rouge">mutex.rb:14</code> to spend about 10 seconds waiting for the mutex to become available. The <code class="highlighter-rouge">total</code> field indicates the total amount of waiting across all other threads caused by <code class="highlighter-rouge">mutex.rb:6</code>. This number becomes more useful when there are more than two threads competing for a single mutex.</p>

<p>While the example shown here was kept very simple on purpose, the DTrace script and Ruby aggregation code written in this post is in fact more than capable of handling more complex scenarios. For example, let’s have a look at some Ruby code that uses multiple mutexes, some of which are nested.</p>

<div class="language-ruby highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="c1"># mutex.rb</span>
<span class="n">mutexes</span> <span class="o">=</span><span class="p">[</span><span class="no">Mutex</span><span class="p">.</span><span class="nf">new</span><span class="p">,</span> <span class="no">Mutex</span><span class="p">.</span><span class="nf">new</span><span class="p">]</span>
<span class="n">threads</span> <span class="o">=</span> <span class="p">[]</span>

<span class="n">threads</span> <span class="o">&lt;&lt;</span> <span class="no">Thread</span><span class="p">.</span><span class="nf">new</span> <span class="k">do</span>
  <span class="kp">loop</span> <span class="k">do</span>
    <span class="n">mutexes</span><span class="p">[</span><span class="mi">0</span><span class="p">].</span><span class="nf">synchronize</span> <span class="k">do</span>
      <span class="nb">sleep</span> <span class="mi">2</span>
    <span class="k">end</span>
  <span class="k">end</span>
<span class="k">end</span>

<span class="n">threads</span> <span class="o">&lt;&lt;</span> <span class="no">Thread</span><span class="p">.</span><span class="nf">new</span> <span class="k">do</span>
  <span class="kp">loop</span> <span class="k">do</span>
    <span class="n">mutexes</span><span class="p">[</span><span class="mi">1</span><span class="p">].</span><span class="nf">synchronize</span> <span class="k">do</span>
      <span class="nb">sleep</span> <span class="mi">2</span>
    <span class="k">end</span>
  <span class="k">end</span>
<span class="k">end</span>

<span class="n">threads</span> <span class="o">&lt;&lt;</span> <span class="no">Thread</span><span class="p">.</span><span class="nf">new</span> <span class="k">do</span>
  <span class="kp">loop</span> <span class="k">do</span>
    <span class="n">mutexes</span><span class="p">[</span><span class="mi">1</span><span class="p">].</span><span class="nf">synchronize</span> <span class="k">do</span>
      <span class="nb">sleep</span> <span class="mi">1</span>
    <span class="k">end</span>
  <span class="k">end</span>
<span class="k">end</span>

<span class="n">threads</span> <span class="o">&lt;&lt;</span> <span class="no">Thread</span><span class="p">.</span><span class="nf">new</span> <span class="k">do</span>
  <span class="kp">loop</span> <span class="k">do</span>
    <span class="n">mutexes</span><span class="p">[</span><span class="mi">0</span><span class="p">].</span><span class="nf">synchronize</span> <span class="k">do</span>
      <span class="nb">sleep</span> <span class="mi">1</span>
      <span class="n">mutexes</span><span class="p">[</span><span class="mi">1</span><span class="p">].</span><span class="nf">synchronize</span> <span class="k">do</span>
        <span class="nb">sleep</span> <span class="mi">1</span>
      <span class="k">end</span>
    <span class="k">end</span>
  <span class="k">end</span>
<span class="k">end</span>

<span class="n">threads</span><span class="p">.</span><span class="nf">each</span><span class="p">(</span><span class="o">&amp;</span><span class="ss">:join</span><span class="p">)</span>
</code></pre></div></div>

<div class="language-bash highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="nv">$ </span><span class="nb">sudo </span>dtrace <span class="nt">-q</span> <span class="nt">-s</span> mutex.d <span class="nt">-c</span> <span class="s1">'ruby mutex.rb'</span> | ruby aggregate.rb


Mutex Contention
<span class="o">================</span>

mutex.rb:6
<span class="nt">----------</span>
mutex.rb:30	  36.0513079s
total		  36.0513079s

mutex.rb:14
<span class="nt">-----------</span>
mutex.rb:22	  78.123187353s
mutex.rb:32	  36.062005125s
total		  114.185192478s

mutex.rb:22
<span class="nt">-----------</span>
mutex.rb:14	  38.127435904s
mutex.rb:32	  19.060814411s
total		  57.188250315000005s

mutex.rb:32
<span class="nt">-----------</span>
mutex.rb:14	  24.073966949s
mutex.rb:22	  24.073383955s
total		  48.147350904s

mutex.rb:30
<span class="nt">-----------</span>
mutex.rb:6	  103.274153073s
total		  103.274153073s
</code></pre></div></div>

<p>The above output very clearly shows that any attempt at making this Ruby program faster should focus its efforts on lines 14 and 30. The really nice thing is that this approach will work regardless of a program’s complexity and requires absolutely no familiarity with the source code. It is literally possible to run this DTrace script against completely unknown code and end up obtaining a decent idea of where the mutex bottlenecks are located. And on top of that, since this is DTrace, there is not even any need to add instrumentation code to the program under investigation. In fact, DTrace can just run against an already active process without even having to interrupt it.</p>

<h3 id="conclusion">Conclusion</h3>

<p>DTrace is a pretty amazing tool that can open up whole new ways of trying to approach a problem. There is so much this post haven’t even touched on yet. The topic is just too big to cover in a single post. Some very good resources for learning more about DTrace can be found here:</p>

<ul>
  <li>the <a href="https://github.com/opendtrace/toolkit">DTrace Toolkit</a> is a curated collection of DTrace scripts for various systems</li>
  <li>the <a href="http://www.tablespace.net/quicksheet/dtrace-quickstart.html">DTrace QuickStart</a> and the <a href="http://www.brendangregg.com/DTrace/DTrace-cheatsheet.pdf">DTrace Cheatsheet</a> are great for quick DTrace refreshers</li>
  <li>the first chapters of <a href="https://www.amazon.com/DTrace-Dynamic-Tracing-Solaris-FreeBSD/dp/0132091518">DTrace: Dynamic Tracing in Oracle Solaris, Mac OS X and FreeBSD</a> act as a DTrace tutorial. The rest of the book is all about how to use DTrace to solve real-life scenarios with tons and tons of examples.</li>
  <li><a href="https://awesome-dtrace.com/">Awesome DTrace</a> is a curated list about various DTrace topics, some of which can be very hard to find information about elsewhere</li>
</ul>

<p>Just one more thing. DTrace might complain about not being able to control executables signed with restricted entitlements on OS X. Using the <code class="highlighter-rouge">-p</code> parameter to directly specify the pid of the process you want DTrace to run against provides an easy workaround for this. Please contact me if you manage to find the proper fix for this.</p>

<h3 id="about-tom">About Tom</h3>

<p><em>Tom Van Eyck is a Software Engineer on the Production Engineering Team at Workday Dublin. The Production Engineering Team builds tools to manage the lifecycle of the Workday Stack.</em></p>
