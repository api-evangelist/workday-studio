---
title: "Ruby Concurrency: In Praise of Condition Variables"
url: "http://workday.github.io/ruby/2017/08/14/ruby-concurrency-in-praise-of-condition-variables"
date: "2017-08-14T00:00:00+00:00"
author: ""
feed_url: "https://workday.github.io/atom.xml"
---
<p><em>by Tom Van Eyck, Workday Dublin</em></p>

<p>When it comes to articles about concurrency, much digital ink has been spilled about the topic of mutexes. While a programmer’s familiarity with mutexes is likely to depend on what kind of programs she usually writes, most developers tend to be at least somewhat familiar with these particular synchronization primitives. This article, however, is going to focus on a much lesser-known synchronization construct: the condition variable.</p>

<p>Condition variables are used for putting threads to sleep and waking them back up once a certain condition is met. Don’t worry if this sounds a bit vague; we’ll go into a lot more detail later. As condition variables always need to be used in conjunction with mutexes, we’ll lead with a quick mutex recap. Next, we’ll introduce consumer-producer problems and how to elegantly solve them with the aid of condition variables. Then, we’ll have a look at how to use these synchronization primitives for implementing blocking method calls. Finishing up, we’ll describe some curious condition variable behavior and how to safeguard against it.</p>

<!--more-->
<h3 id="a-mutex-recap">A mutex recap</h3>

<p>A mutex is a data structure for protecting shared state between multiple threads. When a piece of code is wrapped inside a mutex, the mutex guarantees that only one thread at a time can execute this code. If another thread wants to start executing this code, it’ll have to wait until our first thread is done with it. I realize this may all sound a bit abstract, so now is probably a good time to bring in some example code.</p>

<h4 id="writing-to-shared-state">Writing to shared state</h4>

<p>In this first example, we’ll have a look at what happens when two threads try to modify the same shared variable. The snippet below shows two methods: <code class="highlighter-rouge">counters_with_mutex</code> and <code class="highlighter-rouge">counters_without_mutex</code>. Both methods start by creating a zero-initialized <code class="highlighter-rouge">counters</code> array before spawning 5 threads. Each thread will perform 100,000 loops, with every iteration incrementing all elements of the <code class="highlighter-rouge">counters</code> array by one. Both methods are the same in every way except for one thing: only one of them uses a mutex.</p>

<div class="language-ruby highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="k">def</span> <span class="nf">counters_with_mutex</span>
  <span class="n">mutex</span> <span class="o">=</span> <span class="no">Mutex</span><span class="p">.</span><span class="nf">new</span>
  <span class="n">counters</span> <span class="o">=</span> <span class="p">[</span><span class="mi">0</span><span class="p">,</span> <span class="mi">0</span><span class="p">,</span> <span class="mi">0</span><span class="p">,</span> <span class="mi">0</span><span class="p">,</span> <span class="mi">0</span><span class="p">,</span> <span class="mi">0</span><span class="p">,</span> <span class="mi">0</span><span class="p">,</span> <span class="mi">0</span><span class="p">,</span> <span class="mi">0</span><span class="p">,</span> <span class="mi">0</span><span class="p">]</span>

  <span class="mi">5</span><span class="p">.</span><span class="nf">times</span><span class="p">.</span><span class="nf">map</span> <span class="k">do</span>
    <span class="no">Thread</span><span class="p">.</span><span class="nf">new</span> <span class="k">do</span>
      <span class="mi">100000</span><span class="p">.</span><span class="nf">times</span> <span class="k">do</span>
        <span class="n">mutex</span><span class="p">.</span><span class="nf">synchronize</span> <span class="k">do</span>
          <span class="n">counters</span><span class="p">.</span><span class="nf">map!</span> <span class="p">{</span> <span class="o">|</span><span class="n">counter</span><span class="o">|</span> <span class="n">counter</span> <span class="o">+</span> <span class="mi">1</span> <span class="p">}</span>
        <span class="k">end</span>
      <span class="k">end</span>
    <span class="k">end</span>
  <span class="k">end</span><span class="p">.</span><span class="nf">each</span><span class="p">(</span><span class="o">&amp;</span><span class="ss">:join</span><span class="p">)</span>

  <span class="n">counters</span><span class="p">.</span><span class="nf">inspect</span>
<span class="k">end</span>

<span class="k">def</span> <span class="nf">counters_without_mutex</span>
  <span class="n">counters</span> <span class="o">=</span> <span class="p">[</span><span class="mi">0</span><span class="p">,</span> <span class="mi">0</span><span class="p">,</span> <span class="mi">0</span><span class="p">,</span> <span class="mi">0</span><span class="p">,</span> <span class="mi">0</span><span class="p">,</span> <span class="mi">0</span><span class="p">,</span> <span class="mi">0</span><span class="p">,</span> <span class="mi">0</span><span class="p">,</span> <span class="mi">0</span><span class="p">,</span> <span class="mi">0</span><span class="p">]</span>

  <span class="mi">5</span><span class="p">.</span><span class="nf">times</span><span class="p">.</span><span class="nf">map</span> <span class="k">do</span>
    <span class="no">Thread</span><span class="p">.</span><span class="nf">new</span> <span class="k">do</span>
      <span class="mi">100000</span><span class="p">.</span><span class="nf">times</span> <span class="k">do</span>
        <span class="n">counters</span><span class="p">.</span><span class="nf">map!</span> <span class="p">{</span> <span class="o">|</span><span class="n">counter</span><span class="o">|</span> <span class="n">counter</span> <span class="o">+</span> <span class="mi">1</span> <span class="p">}</span>
      <span class="k">end</span>
    <span class="k">end</span>
  <span class="k">end</span><span class="p">.</span><span class="nf">each</span><span class="p">(</span><span class="o">&amp;</span><span class="ss">:join</span><span class="p">)</span>

  <span class="n">counters</span><span class="p">.</span><span class="nf">inspect</span>
<span class="k">end</span>

<span class="nb">puts</span> <span class="n">counters_with_mutex</span>
<span class="c1"># =&gt; [500000, 500000, 500000, 500000, 500000, 500000, 500000, 500000, 500000, 500000]</span>

<span class="nb">puts</span> <span class="n">counters_without_mutex</span>
<span class="c1"># =&gt; [500000, 447205, 500000, 500000, 500000, 500000, 203656, 500000, 500000, 500000]</span>
<span class="c1"># note that we seem to have lost some increments here due to not using a mutex</span>
</code></pre></div></div>

<p>As you can see, only the method that uses a mutex ends up producing the correct result. The method without a mutex seems to have lost some increments. This is because the lack of a mutex makes it possible for our second thread to interrupt our first thread at any point during its execution. This can lead to some serious problems.</p>

<p>For example, imagine that our first thread has just read the first entry of the <code class="highlighter-rouge">counters</code> array, incremented it by one, and is now getting ready to write this incremented value back to our array. However, before our first thread can write this incremented value, it gets interrupted by the second thread. This second thread then goes on to read the current value of the first entry, increments it by one, and succeeds in writing the result back to our <code class="highlighter-rouge">counters</code> array. Now we have a problem!</p>

<p>We have a problem because the first thread got interrupted before it had a chance to write its incremented value to the array. When the first thread resumes, it will end up overwriting the value that the second thread just placed in the array. This will cause us to essentially lose an increment operation, which explains why our program output has entries in it that are less than 500,000.</p>

<p>All these problems can be avoided by using a mutex. Remember that a thread executing code wrapped by a mutex cannot be interleaved with another thread wanting to execute this same code. Therefore, our second thread would never have gotten interleaved with the first thread, thereby avoiding the possibility of results getting overwritten.</p>

<h4 id="reading-from-shared-state">Reading from shared state</h4>

<p>There’s a common misconception that a mutex is only required when writing to a shared variable, and not when reading from it. The snippet below shows 50 threads flipping the boolean values in the <code class="highlighter-rouge">flags</code> array over and over again. Many developers think this snippet is without error as the code responsible for changing these values was wrapped inside a mutex. If that were true, then every line of the output of <code class="highlighter-rouge">puts flags.to_s</code> should consist of 10 repetitions of either <code class="highlighter-rouge">true</code> or <code class="highlighter-rouge">false</code>. As we can see below, this is not the case.</p>

<div class="language-ruby highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="n">mutex</span> <span class="o">=</span> <span class="no">Mutex</span><span class="p">.</span><span class="nf">new</span>
<span class="n">flags</span> <span class="o">=</span> <span class="p">[</span><span class="kp">false</span><span class="p">,</span> <span class="kp">false</span><span class="p">,</span> <span class="kp">false</span><span class="p">,</span> <span class="kp">false</span><span class="p">,</span> <span class="kp">false</span><span class="p">,</span> <span class="kp">false</span><span class="p">,</span> <span class="kp">false</span><span class="p">,</span> <span class="kp">false</span><span class="p">,</span> <span class="kp">false</span><span class="p">,</span> <span class="kp">false</span><span class="p">]</span>

<span class="n">threads</span> <span class="o">=</span> <span class="mi">50</span><span class="p">.</span><span class="nf">times</span><span class="p">.</span><span class="nf">map</span> <span class="k">do</span>
  <span class="no">Thread</span><span class="p">.</span><span class="nf">new</span> <span class="k">do</span>
    <span class="mi">100000</span><span class="p">.</span><span class="nf">times</span> <span class="k">do</span>
      <span class="c1"># don't do this! Reading from shared state requires a mutex!</span>
      <span class="nb">puts</span> <span class="n">flags</span><span class="p">.</span><span class="nf">to_s</span>

      <span class="n">mutex</span><span class="p">.</span><span class="nf">synchronize</span> <span class="k">do</span>
        <span class="n">flags</span><span class="p">.</span><span class="nf">map!</span> <span class="p">{</span> <span class="o">|</span><span class="n">f</span><span class="o">|</span> <span class="o">!</span><span class="n">f</span> <span class="p">}</span>
      <span class="k">end</span>
    <span class="k">end</span>
  <span class="k">end</span>
<span class="k">end</span>
<span class="n">threads</span><span class="p">.</span><span class="nf">each</span><span class="p">(</span><span class="o">&amp;</span><span class="ss">:join</span><span class="p">)</span>
</code></pre></div></div>
<div class="language-bash highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="nv">$ </span>ruby flags.rb <span class="o">&gt;</span> output.log
<span class="nv">$ </span><span class="nb">grep</span> <span class="s1">'true, false'</span> output.log | wc <span class="nt">-l</span>
    30
</code></pre></div></div>

<p>What’s happening here is that our mutex only guarantees that no two threads can modify the <code class="highlighter-rouge">flags</code> array at the same time. However, it is perfectly possible for one thread to start reading from this array while another thread is busy modifying it, thereby causing the first thread to read an array that contains both <code class="highlighter-rouge">true</code> and <code class="highlighter-rouge">false</code> entries. Luckily, all of this can be easily avoided by wrapping <code class="highlighter-rouge">puts flags.to_s</code> inside our mutex. This will guarantee that only one thread at a time can read from or write to the <code class="highlighter-rouge">flags</code> array.</p>

<p>Before moving on, I would just like to mention that even very experienced people have gotten tripped up by not using a mutex when accessing shared state. In fact, at one point there even was a <a href="http://www.javaworld.com/article/2074979/java-concurrency/double-checked-locking--clever--but-broken.html">Java design pattern</a> that assumed it was safe to not always use a mutex to do so. Needless to say, this pattern has since been amended.</p>

<h3 id="consumer-producer-problems">Consumer-producer problems</h3>

<p>With that mutex refresher out of the way, we can now start looking at condition variables. Condition variables are best explained by trying to come up with a practical solution to the <a href="https://en.wikipedia.org/wiki/Producer-consumer_problem">consumer-producer problem</a>. In fact, consumer-producer problems are so common that Ruby already has a data structure aimed at solving these: <a href="https://ruby-doc.org/core-2.3.1/Queue.html">the Queue class</a>. This class uses a condition variable to implement the blocking variant of its <code class="highlighter-rouge">shift()</code> method. In this article, we made a conscious decision not to use the <code class="highlighter-rouge">Queue</code> class. Instead, we’re going to write everything from scratch with the help of condition variables.</p>

<p>Let’s have a look at the problem that we’re going to solve. Imagine that we have a website where users can generate tasks of varying complexity, e.g. a service that allows users to convert uploaded jpg images to pdf. We can think of these users as producers of a steady stream of tasks of random complexity. These tasks will get stored on a backend server that has several worker processes running on it. Each worker process will grab a task, process it, and then grab the next one. These workers are our task consumers.</p>

<p>With what we know about mutexes, it shouldn’t be too hard to write a piece of code that mimics the above scenario. It’ll probably end up looking something like this.</p>

<div class="language-ruby highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="n">tasks</span> <span class="o">=</span> <span class="p">[]</span>
<span class="n">mutex</span> <span class="o">=</span> <span class="no">Mutex</span><span class="p">.</span><span class="nf">new</span>
<span class="n">threads</span> <span class="o">=</span> <span class="p">[]</span>

<span class="k">class</span> <span class="nc">Task</span>
  <span class="k">def</span> <span class="nf">initialize</span>
    <span class="vi">@duration</span> <span class="o">=</span> <span class="nb">rand</span><span class="p">()</span>
  <span class="k">end</span>

  <span class="k">def</span> <span class="nf">execute</span>
    <span class="nb">sleep</span> <span class="vi">@duration</span>
  <span class="k">end</span>
<span class="k">end</span>

<span class="c1"># producer threads</span>
<span class="n">threads</span> <span class="o">+=</span> <span class="mi">2</span><span class="p">.</span><span class="nf">times</span><span class="p">.</span><span class="nf">map</span> <span class="k">do</span>
  <span class="no">Thread</span><span class="p">.</span><span class="nf">new</span> <span class="k">do</span>
    <span class="k">while</span> <span class="kp">true</span>
      <span class="n">mutex</span><span class="p">.</span><span class="nf">synchronize</span> <span class="k">do</span>
        <span class="n">tasks</span> <span class="o">&lt;&lt;</span> <span class="no">Task</span><span class="p">.</span><span class="nf">new</span>
        <span class="nb">puts</span> <span class="s2">"Added task: </span><span class="si">#{</span><span class="n">tasks</span><span class="p">.</span><span class="nf">last</span><span class="p">.</span><span class="nf">inspect</span><span class="si">}</span><span class="s2">"</span>
      <span class="k">end</span>
      <span class="c1"># limit task production speed</span>
      <span class="nb">sleep</span> <span class="mf">0.5</span>
    <span class="k">end</span>
  <span class="k">end</span>
<span class="k">end</span>

<span class="c1"># consumer threads</span>
<span class="n">threads</span> <span class="o">+=</span> <span class="mi">5</span><span class="p">.</span><span class="nf">times</span><span class="p">.</span><span class="nf">map</span> <span class="k">do</span>
  <span class="no">Thread</span><span class="p">.</span><span class="nf">new</span> <span class="k">do</span>
    <span class="k">while</span> <span class="kp">true</span>
      <span class="n">task</span> <span class="o">=</span> <span class="kp">nil</span>
      <span class="n">mutex</span><span class="p">.</span><span class="nf">synchronize</span> <span class="k">do</span>
        <span class="k">if</span> <span class="n">tasks</span><span class="p">.</span><span class="nf">count</span> <span class="o">&gt;</span> <span class="mi">0</span>
          <span class="n">task</span> <span class="o">=</span> <span class="n">tasks</span><span class="p">.</span><span class="nf">shift</span>
          <span class="nb">puts</span> <span class="s2">"Removed task: </span><span class="si">#{</span><span class="n">task</span><span class="p">.</span><span class="nf">inspect</span><span class="si">}</span><span class="s2">"</span>
        <span class="k">end</span>
      <span class="k">end</span>
      <span class="c1"># execute task outside of mutex so we don't unnecessarily</span>
      <span class="c1"># block other consumer threads</span>
      <span class="n">task</span><span class="p">.</span><span class="nf">execute</span> <span class="k">unless</span> <span class="n">task</span><span class="p">.</span><span class="nf">nil?</span>
    <span class="k">end</span>
  <span class="k">end</span>
<span class="k">end</span>

<span class="n">threads</span><span class="p">.</span><span class="nf">each</span><span class="p">(</span><span class="o">&amp;</span><span class="ss">:join</span><span class="p">)</span>
</code></pre></div></div>

<p>The above code should be fairly straightforward. There is a <code class="highlighter-rouge">Task</code> class for creating tasks that take between 0 and 1 seconds to run. We have 2 producer threads, each running an endless <code class="highlighter-rouge">while</code> loop that safely appends a new task to the <code class="highlighter-rouge">tasks</code> array every 0.5 seconds with the help of a mutex. Our 5 consumer threads are also running an endless <code class="highlighter-rouge">while</code> loop, each iteration grabbing the mutex so as to safely check the <code class="highlighter-rouge">tasks</code> array for available tasks. If a consumer thread finds an available task, it removes the task from the array and starts processing it. Once the task had been processed, the thread moves on to its next iteration, thereby repeating the cycle anew.</p>

<p>While the above implementation seems to work just fine, it is not optimal as it requires all consumer threads to constantly poll the <code class="highlighter-rouge">tasks</code> array for available work. This polling does not come for free. The Ruby interpreter has to constantly schedule the consumer threads to run, thereby preempting threads that may have actual important work to do. To give an example, the above code will interleave consumer threads that are executing a task with consumer threads that just want to check for newly available tasks. This can become a real problem when there is a large number of consumer threads and only a few tasks.</p>

<p>If you want to see for yourself just how inefficient this approach is, you only need to modify the original code for consumer threads with the code shown below. This modified program prints well over a thousand lines of <code class="highlighter-rouge">This thread has nothing to do</code> for every single line of <code class="highlighter-rouge">Removed task</code>. Hopefully, this gives you an indication of the general wastefulness of having consumer threads constantly poll the <code class="highlighter-rouge">tasks</code> array.</p>

<div class="language-ruby highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="c1"># modified consumer threads code</span>
<span class="n">threads</span> <span class="o">+=</span> <span class="mi">5</span><span class="p">.</span><span class="nf">times</span><span class="p">.</span><span class="nf">map</span> <span class="k">do</span>
  <span class="no">Thread</span><span class="p">.</span><span class="nf">new</span> <span class="k">do</span>
    <span class="k">while</span> <span class="kp">true</span>
      <span class="n">task</span> <span class="o">=</span> <span class="kp">nil</span>
      <span class="n">mutex</span><span class="p">.</span><span class="nf">synchronize</span> <span class="k">do</span>
        <span class="k">if</span> <span class="n">tasks</span><span class="p">.</span><span class="nf">count</span> <span class="o">&gt;</span> <span class="mi">0</span>
          <span class="n">task</span> <span class="o">=</span> <span class="n">tasks</span><span class="p">.</span><span class="nf">shift</span>
          <span class="nb">puts</span> <span class="s2">"Removed task: </span><span class="si">#{</span><span class="n">task</span><span class="p">.</span><span class="nf">inspect</span><span class="si">}</span><span class="s2">"</span>
        <span class="k">else</span>
          <span class="nb">puts</span> <span class="s1">'This thread has nothing to do'</span>
        <span class="k">end</span>
      <span class="k">end</span>
      <span class="c1"># execute task outside of mutex so we don't unnecessarily</span>
      <span class="c1"># block other consumer threads</span>
      <span class="n">task</span><span class="p">.</span><span class="nf">execute</span> <span class="k">unless</span> <span class="n">task</span><span class="p">.</span><span class="nf">nil?</span>
    <span class="k">end</span>
  <span class="k">end</span>
<span class="k">end</span>
</code></pre></div></div>

<h3 id="condition-variables-to-the-rescue">Condition variables to the rescue</h3>

<p>So how we can create a more efficient solution to the consumer-producer problem? That is where condition variables come into play. Condition variables are used for putting threads to sleep and waking them only once a certain condition is met. Remember that our current solution to the producer-consumer problem is far from ideal because consumer threads need to constantly poll for new tasks to arrive. Things would be much more efficient if our consumer threads could go to sleep and be woken up only when a new task has arrived.</p>

<p>Shown below is a solution to the consumer-producer problem that makes use of condition variables. We’ll talk about how this works in a second. For now though, just have a look at the code and perhaps have a go at running it. If you were to run it, you would probably see that <code class="highlighter-rouge">This thread has nothing to do</code> does not show up anymore. Our new approach has completely gotten rid of consumer threads busy polling the <code class="highlighter-rouge">tasks</code> array.</p>

<p>The use of a condition variable will now cause our consumer threads to wait for a task to be available in the <code class="highlighter-rouge">tasks</code> array before proceeding. As a result of this, we can now remove some of the checks we had to have in place in our original consumer code. I’ve added some comments to the code below to help highlight these removals.</p>

<div class="language-ruby highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="n">tasks</span> <span class="o">=</span> <span class="p">[]</span>
<span class="n">mutex</span> <span class="o">=</span> <span class="no">Mutex</span><span class="p">.</span><span class="nf">new</span>
<span class="n">cond_var</span> <span class="o">=</span> <span class="no">ConditionVariable</span><span class="p">.</span><span class="nf">new</span>
<span class="n">threads</span> <span class="o">=</span> <span class="p">[]</span>

<span class="k">class</span> <span class="nc">Task</span>
  <span class="k">def</span> <span class="nf">initialize</span>
    <span class="vi">@duration</span> <span class="o">=</span> <span class="nb">rand</span><span class="p">()</span>
  <span class="k">end</span>

  <span class="k">def</span> <span class="nf">execute</span>
    <span class="nb">sleep</span> <span class="vi">@duration</span>
  <span class="k">end</span>
<span class="k">end</span>

<span class="c1"># producer threads</span>
<span class="n">threads</span> <span class="o">+=</span> <span class="mi">2</span><span class="p">.</span><span class="nf">times</span><span class="p">.</span><span class="nf">map</span> <span class="k">do</span>
  <span class="no">Thread</span><span class="p">.</span><span class="nf">new</span> <span class="k">do</span>
    <span class="k">while</span> <span class="kp">true</span>
      <span class="n">mutex</span><span class="p">.</span><span class="nf">synchronize</span> <span class="k">do</span>
        <span class="n">tasks</span> <span class="o">&lt;&lt;</span> <span class="no">Task</span><span class="p">.</span><span class="nf">new</span>
        <span class="n">cond_var</span><span class="p">.</span><span class="nf">signal</span>
        <span class="nb">puts</span> <span class="s2">"Added task: </span><span class="si">#{</span><span class="n">tasks</span><span class="p">.</span><span class="nf">last</span><span class="p">.</span><span class="nf">inspect</span><span class="si">}</span><span class="s2">"</span>
      <span class="k">end</span>
      <span class="c1"># limit task production speed</span>
      <span class="nb">sleep</span> <span class="mf">0.5</span>
    <span class="k">end</span>
  <span class="k">end</span>
<span class="k">end</span>

<span class="c1"># consumer threads</span>
<span class="n">threads</span> <span class="o">+=</span> <span class="mi">5</span><span class="p">.</span><span class="nf">times</span><span class="p">.</span><span class="nf">map</span> <span class="k">do</span>
  <span class="no">Thread</span><span class="p">.</span><span class="nf">new</span> <span class="k">do</span>
    <span class="k">while</span> <span class="kp">true</span>
      <span class="n">task</span> <span class="o">=</span> <span class="kp">nil</span>
      <span class="n">mutex</span><span class="p">.</span><span class="nf">synchronize</span> <span class="k">do</span>
        <span class="k">while</span> <span class="n">tasks</span><span class="p">.</span><span class="nf">empty?</span>
          <span class="n">cond_var</span><span class="p">.</span><span class="nf">wait</span><span class="p">(</span><span class="n">mutex</span><span class="p">)</span>
        <span class="k">end</span>

        <span class="c1"># the `if tasks.count == 0` statement will never be true as the thread</span>
        <span class="c1"># will now only reach this line if the tasks array is not empty</span>
        <span class="nb">puts</span> <span class="s1">'This thread has nothing to do'</span> <span class="k">if</span> <span class="n">tasks</span><span class="p">.</span><span class="nf">count</span> <span class="o">==</span> <span class="mi">0</span>

        <span class="c1"># similarly, we can now remove the `if tasks.count &gt; 0` check that</span>
        <span class="c1"># used to surround this code. We no longer need it as this code will</span>
        <span class="c1"># now only get executed if the tasks array is not empty.</span>
        <span class="n">task</span> <span class="o">=</span> <span class="n">tasks</span><span class="p">.</span><span class="nf">shift</span>
        <span class="nb">puts</span> <span class="s2">"Removed task: </span><span class="si">#{</span><span class="n">task</span><span class="p">.</span><span class="nf">inspect</span><span class="si">}</span><span class="s2">"</span>
      <span class="k">end</span>
      <span class="c1"># Note that we have now removed `unless task.nil?` from this line as</span>
      <span class="c1"># our thread can only arrive here if there is indeed a task available.</span>
      <span class="n">task</span><span class="p">.</span><span class="nf">execute</span>
    <span class="k">end</span>
  <span class="k">end</span>
<span class="k">end</span>

<span class="n">threads</span><span class="p">.</span><span class="nf">each</span><span class="p">(</span><span class="o">&amp;</span><span class="ss">:join</span><span class="p">)</span>
</code></pre></div></div>

<p>Aside from us removing some <code class="highlighter-rouge">if</code> statements, our new code is essentially identical to our previous solution. The only exception to this are the five new lines shown below. Don’t worry if some of the accompanying comments don’t quite make sense yet. Now is also a good time to point out that the new code for both the producer and consumer threads was added inside the existing mutex synchronization blocks. Condition variables are not thread-safe and therefore always need to be used in conjunction with a mutex!</p>

<div class="language-ruby highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="c1"># declaring the condition variable</span>
<span class="n">cond_var</span> <span class="o">=</span> <span class="no">ConditionVariable</span><span class="p">.</span><span class="nf">new</span>
</code></pre></div></div>

<div class="language-ruby highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="c1"># a producer thread now signals the condition variable</span>
<span class="c1"># after adding a new task to the tasks array</span>
<span class="n">cond_var</span><span class="p">.</span><span class="nf">signal</span>
</code></pre></div></div>

<div class="language-ruby highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="c1"># a consumer thread now goes to sleep when it sees that</span>
<span class="c1"># the tasks array is empty. It can get woken up again</span>
<span class="c1"># when a producer thread signals the condition variable.</span>
<span class="k">while</span> <span class="n">tasks</span><span class="p">.</span><span class="nf">empty?</span>
  <span class="n">cond_var</span><span class="p">.</span><span class="nf">wait</span><span class="p">(</span><span class="n">mutex</span><span class="p">)</span>
<span class="k">end</span>
</code></pre></div></div>

<p>Let’s talk about the new code now. We’ll start with the consumer threads snippet. There’s actually so much going on in these three lines that we’ll limit ourselves to covering what <code class="highlighter-rouge">cond_var.wait(mutex)</code> does for now. We’ll explain the need for the <code class="highlighter-rouge">while tasks.empty?</code> loop later. The first thing to notice about the <code class="highlighter-rouge">wait</code> method is the parameter that’s being passed to it. Remember how a condition variable is not thread-safe and therefore should only have its methods called inside a mutex synchronization block? It is that mutex that needs to be passed as a parameter to the <code class="highlighter-rouge">wait</code> method.</p>

<p>Calling <code class="highlighter-rouge">wait</code> on a condition variable causes two things to happen. First of all, it causes the thread that calls <code class="highlighter-rouge">wait</code> to go to sleep. That is to say, the thread will tell the interpreter that it no longer wants to be scheduled. However, this thread still has ownership of the mutex as it’s going to sleep. We need to ensure that the thread relinquishes this mutex because otherwise all other threads waiting for this mutex will be blocked. By passing this mutex to the <code class="highlighter-rouge">wait</code> method, the <code class="highlighter-rouge">wait</code> method internals will ensure that the mutex gets released as the thread goes to sleep.</p>

<p>Let’s move on to the producer threads. These threads are now calling <code class="highlighter-rouge">cond_var.signal</code>. The <code class="highlighter-rouge">signal</code> method is pretty straightforward in that it wakes up exactly one of the threads that were put to sleep by the <code class="highlighter-rouge">wait</code> method. This newly awoken thread will indicate to the interpreter that it is ready to start getting scheduled again and then wait for its turn.</p>

<p>So what code does our newly awoken thread start executing once it gets scheduled again? It starts executing from where it left off. Essentially, a newly awoken thread will return from its call to <code class="highlighter-rouge">cond_var.wait(mutex)</code> and resume from there. Personally, I like to think of calling <code class="highlighter-rouge">wait</code> as creating a save point inside a thread from which work can resume once the thread gets woken up and rescheduled again. Please note that since the thread wants to resume from where it originally left off, it’ll need to reacquire the mutex in order to get scheduled. This mutex reacquisition is very important, so be sure to remember it.</p>

<p>This segues nicely into why we need to use <code class="highlighter-rouge">while tasks.empty?</code> when calling <code class="highlighter-rouge">wait</code> in a consumer thread. When our newly awoken thread resumes execution by returning from <code class="highlighter-rouge">cond_var.wait</code>, the first thing it’ll do is complete its previously interrupted iteration through the <code class="highlighter-rouge">while</code> loop, thereby evaluating <code class="highlighter-rouge">while tasks.empty?</code> again. This actually causes us to neatly avoid a possible race condition.</p>

<p>Let’s say we don’t use a <code class="highlighter-rouge">while</code> loop and use an <code class="highlighter-rouge">if</code> statement instead. The resulting code would then look like shown below. Unfortunately, there is a very hard to find problem with this code. Note how we now need to re-add the previously removed <code class="highlighter-rouge">if tasks.count &gt; 0</code> and <code class="highlighter-rouge">unless task.nil?</code> statements to our code below in order to ensure its safe execution.</p>

<div class="language-ruby highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="c1"># consumer threads</span>
<span class="n">threads</span> <span class="o">+=</span> <span class="mi">5</span><span class="p">.</span><span class="nf">times</span><span class="p">.</span><span class="nf">map</span> <span class="k">do</span>
  <span class="no">Thread</span><span class="p">.</span><span class="nf">new</span> <span class="k">do</span>
    <span class="k">while</span> <span class="kp">true</span>
      <span class="n">task</span> <span class="o">=</span> <span class="kp">nil</span>
      <span class="n">mutex</span><span class="p">.</span><span class="nf">synchronize</span> <span class="k">do</span>
        <span class="n">cond_var</span><span class="p">.</span><span class="nf">wait</span><span class="p">(</span><span class="n">mutex</span><span class="p">)</span> <span class="k">if</span> <span class="n">tasks</span><span class="p">.</span><span class="nf">empty?</span>

        <span class="c1"># using `if tasks.empty?` forces us to once again add this</span>
        <span class="c1"># `if tasks.count &gt; 0` check. We need this check to protect</span>
        <span class="c1"># ourselves against a nasty race condition.</span>
        <span class="k">if</span> <span class="n">tasks</span><span class="p">.</span><span class="nf">count</span> <span class="o">&gt;</span> <span class="mi">0</span>
          <span class="n">task</span> <span class="o">=</span> <span class="n">tasks</span><span class="p">.</span><span class="nf">shift</span>
          <span class="nb">puts</span> <span class="s2">"Removed task: </span><span class="si">#{</span><span class="n">task</span><span class="p">.</span><span class="nf">inspect</span><span class="si">}</span><span class="s2">"</span>
        <span class="k">else</span>
          <span class="nb">puts</span> <span class="s1">'This thread has nothing to do'</span>
        <span class="k">end</span>
      <span class="k">end</span>
      <span class="c1"># using `if tasks.empty?` forces us to re-add `unless task.nil?`</span>
      <span class="c1"># in order to safeguard ourselves against a now newly introduced</span>
      <span class="c1"># race condition</span>
      <span class="n">task</span><span class="p">.</span><span class="nf">execute</span> <span class="k">unless</span> <span class="n">task</span><span class="p">.</span><span class="nf">nil?</span>
    <span class="k">end</span>
  <span class="k">end</span>
<span class="k">end</span>
</code></pre></div></div>

<p>Imagine a scenario where we have:</p>

<ul>
  <li>two producer threads</li>
  <li>one consumer thread that’s awake</li>
  <li>four consumer threads that are asleep</li>
</ul>

<p>A consumer thread that’s awake will go back to sleep only when there are no more tasks in the <code class="highlighter-rouge">tasks</code> array. That is to say, a single consumer thread will keep processing tasks until no more tasks are available. Now, let’s say one of our producer threads adds a new task to the currently empty <code class="highlighter-rouge">tasks</code> array before calling <code class="highlighter-rouge">cond_var.signal</code> at roughly the same time as our active consumer thread is finishing its current task. This <code class="highlighter-rouge">signal</code> call will awaken one of our sleeping consumer threads, which will then try to get itself scheduled. This is where a race condition is likely to happen!</p>

<p>We’re now in a position where two consumer threads are competing for ownership of the mutex in order to get scheduled. Let’s say our first consumer thread wins this competition. This thread will now go and grab the task from the <code class="highlighter-rouge">tasks</code> array before relinquishing the mutex. Our second consumer thread then grabs the mutex and gets to run. However, as the <code class="highlighter-rouge">tasks</code> array is empty now, there is nothing for this second consumer thread to work on. So this second consumer thread now has to do an entire iteration of its <code class="highlighter-rouge">while true</code> loop for no real purpose at all.</p>

<p>We now find ourselves in a situation where a complete iteration of the <code class="highlighter-rouge">while true</code> loop can occur even when the <code class="highlighter-rouge">tasks</code> array is empty. This is a not unlike the position we were in when our program was just busy polling the <code class="highlighter-rouge">tasks</code> array. Sure, our current program will be more efficient than busy polling, but we will still need to safeguard our code against the possibility of an iteration occurring when there is no task available. This is why we needed to re-add the <code class="highlighter-rouge">if tasks.count &gt; 0</code> and <code class="highlighter-rouge">unless task.nil?</code> statements. Especially the latter of these two is important, as otherwise our program might crash with a <code class="highlighter-rouge">NilException</code>.</p>

<p>Luckily, we can safely get rid of these easily overlooked safeguards by forcing each newly awakened consumer thread to check for available tasks and having it put itself to sleep again if no tasks are available. This behavior can be accomplished by replacing the <code class="highlighter-rouge">if tasks.empty?</code> statement with a <code class="highlighter-rouge">while tasks.empty?</code> loop. If tasks are available, a newly awoken thread will exit the loop and execute the rest of its code. However, if no tasks are found, then the loop is repeated, thereby causing the thread to put itself to sleep again by executing <code class="highlighter-rouge">cond_var.wait</code>. We’ll see in a later section that there is yet another benefit to using this <code class="highlighter-rouge">while</code> loop.</p>

<h3 id="building-our-own-queue-class">Building our own Queue class</h3>

<p>At the beginning of a previous section, we touched on how condition variables are used by the <code class="highlighter-rouge">Queue</code> class to implement blocking behavior. The previous section taught us enough about condition variables for us to go and implement a basic <code class="highlighter-rouge">Queue</code> class ourselves. We’re going to create a thread-safe <code class="highlighter-rouge">SimpleQueue</code> class that is capable of:</p>

<ul>
  <li>having data appended to it with the <code class="highlighter-rouge">&lt;&lt;</code> operator</li>
  <li>having data retrieved from it with a non-blocking <code class="highlighter-rouge">shift</code> method</li>
  <li>having data retrieved from it with a blocking <code class="highlighter-rouge">shift</code> method</li>
</ul>

<p>It’s easy enough to write code that meets these first two criteria. It will probably end up looking something like the code shown below. Note that our <code class="highlighter-rouge">SimpleQueue</code> class is using a mutex as we want this class to be thread-safe, just like the original <code class="highlighter-rouge">Queue</code> class.</p>

<div class="language-ruby highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="k">class</span> <span class="nc">SimpleQueue</span>
  <span class="k">def</span> <span class="nf">initialize</span>
    <span class="vi">@elems</span> <span class="o">=</span> <span class="p">[]</span>
    <span class="vi">@mutex</span> <span class="o">=</span> <span class="no">Mutex</span><span class="p">.</span><span class="nf">new</span>
  <span class="k">end</span>

  <span class="k">def</span> <span class="nf">&lt;&lt;</span><span class="p">(</span><span class="n">elem</span><span class="p">)</span>
    <span class="vi">@mutex</span><span class="p">.</span><span class="nf">synchronize</span> <span class="k">do</span>
      <span class="vi">@elems</span> <span class="o">&lt;&lt;</span> <span class="n">elem</span>
    <span class="k">end</span>
  <span class="k">end</span>

  <span class="k">def</span> <span class="nf">shift</span><span class="p">(</span><span class="n">blocking</span> <span class="o">=</span> <span class="kp">true</span><span class="p">)</span>
    <span class="vi">@mutex</span><span class="p">.</span><span class="nf">synchronize</span> <span class="k">do</span>
      <span class="k">if</span> <span class="n">blocking</span>
        <span class="k">raise</span> <span class="s1">'yet to be implemented'</span>
      <span class="k">end</span>
      <span class="vi">@elems</span><span class="p">.</span><span class="nf">shift</span>
    <span class="k">end</span>
  <span class="k">end</span>
<span class="k">end</span>

<span class="n">simple_queue</span> <span class="o">=</span> <span class="no">SimpleQueue</span><span class="p">.</span><span class="nf">new</span>
<span class="n">simple_queue</span> <span class="o">&lt;&lt;</span> <span class="s1">'foo'</span>

<span class="n">simple_queue</span><span class="p">.</span><span class="nf">shift</span><span class="p">(</span><span class="kp">false</span><span class="p">)</span>
<span class="c1"># =&gt; "foo"</span>

<span class="n">simple_queue</span><span class="p">.</span><span class="nf">shift</span><span class="p">(</span><span class="kp">false</span><span class="p">)</span>
<span class="c1"># =&gt; nil</span>
</code></pre></div></div>

<p>Now let’s have a look at what’s needed to implement the blocking <code class="highlighter-rouge">shift</code> behavior. As it turns out, this is actually very easy. We only want the thread to block if the <code class="highlighter-rouge">shift</code> method is called when the <code class="highlighter-rouge">@elems</code> array is empty. This is all the information we need to determine where we need to place our condition variable’s call to <code class="highlighter-rouge">wait</code>. Similarly, we want the thread to stop blocking once the <code class="highlighter-rouge">&lt;&lt;</code> operator appends a new element, thereby causing <code class="highlighter-rouge">@elems</code> to no longer be empty. This tells us exactly where we need to place our call to <code class="highlighter-rouge">signal</code>.</p>

<p>In the end, we just need to create a condition variable that makes the thread go to sleep when a blocking <code class="highlighter-rouge">shift</code> is called on an empty <code class="highlighter-rouge">SimpleQueue</code>. Likewise, the <code class="highlighter-rouge">&lt;&lt;</code> operator just needs to signal the condition variable when a new element is added, thereby causing the sleeping thread to be woken up. The takeaway from this is that blocking methods work by causing their calling thread to fall asleep. Also, please note that the call to <code class="highlighter-rouge">@cond_var.wait</code> takes place inside a <code class="highlighter-rouge">while @elems.empty?</code> loop. Always use a <code class="highlighter-rouge">while</code> loop when calling <code class="highlighter-rouge">wait</code> on a condition variable! Never use an <code class="highlighter-rouge">if</code> statement!</p>

<div class="language-ruby highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="k">class</span> <span class="nc">SimpleQueue</span>
  <span class="k">def</span> <span class="nf">initialize</span>
    <span class="vi">@elems</span> <span class="o">=</span> <span class="p">[]</span>
    <span class="vi">@mutex</span> <span class="o">=</span> <span class="no">Mutex</span><span class="p">.</span><span class="nf">new</span>
    <span class="vi">@cond_var</span> <span class="o">=</span> <span class="no">ConditionVariable</span><span class="p">.</span><span class="nf">new</span>
  <span class="k">end</span>

  <span class="k">def</span> <span class="nf">&lt;&lt;</span><span class="p">(</span><span class="n">elem</span><span class="p">)</span>
    <span class="vi">@mutex</span><span class="p">.</span><span class="nf">synchronize</span> <span class="k">do</span>
      <span class="vi">@elems</span> <span class="o">&lt;&lt;</span> <span class="n">elem</span>
      <span class="vi">@cond_var</span><span class="p">.</span><span class="nf">signal</span>
    <span class="k">end</span>
  <span class="k">end</span>

  <span class="k">def</span> <span class="nf">shift</span><span class="p">(</span><span class="n">blocking</span> <span class="o">=</span> <span class="kp">true</span><span class="p">)</span>
    <span class="vi">@mutex</span><span class="p">.</span><span class="nf">synchronize</span> <span class="k">do</span>
      <span class="k">if</span> <span class="n">blocking</span>
        <span class="k">while</span> <span class="vi">@elems</span><span class="p">.</span><span class="nf">empty?</span>
          <span class="vi">@cond_var</span><span class="p">.</span><span class="nf">wait</span><span class="p">(</span><span class="vi">@mutex</span><span class="p">)</span>
        <span class="k">end</span>
      <span class="k">end</span>
      <span class="vi">@elems</span><span class="p">.</span><span class="nf">shift</span>
    <span class="k">end</span>
  <span class="k">end</span>
<span class="k">end</span>

<span class="n">simple_queue</span> <span class="o">=</span> <span class="no">SimpleQueue</span><span class="p">.</span><span class="nf">new</span>

<span class="c1"># this will print "blocking shift returned with: foo" after 5 seconds</span>
<span class="c1"># that is to say, the first thread will go to sleep until the second</span>
<span class="c1"># thread adds an element to the queue, thereby causing the first thread</span>
<span class="c1"># to be woken up again</span>
<span class="n">threads</span> <span class="o">=</span> <span class="p">[]</span>
<span class="n">threads</span> <span class="o">&lt;&lt;</span> <span class="no">Thread</span><span class="p">.</span><span class="nf">new</span> <span class="p">{</span> <span class="nb">puts</span> <span class="s2">"blocking shift returned with: </span><span class="si">#{</span><span class="n">simple_queue</span><span class="p">.</span><span class="nf">shift</span><span class="si">}</span><span class="s2">"</span> <span class="p">}</span>
<span class="n">threads</span> <span class="o">&lt;&lt;</span> <span class="no">Thread</span><span class="p">.</span><span class="nf">new</span> <span class="p">{</span> <span class="nb">sleep</span> <span class="mi">5</span><span class="p">;</span> <span class="n">simple_queue</span> <span class="o">&lt;&lt;</span> <span class="s1">'foo'</span> <span class="p">}</span>
<span class="n">threads</span><span class="p">.</span><span class="nf">each</span><span class="p">(</span><span class="o">&amp;</span><span class="ss">:join</span><span class="p">)</span>
</code></pre></div></div>

<p>One thing to point out in the above code is that <code class="highlighter-rouge">@cond_var.signal</code> can get called even when there are no sleeping threads around. This is a perfectly okay thing to do. In these types of scenarios calling <code class="highlighter-rouge">@cond_var.signal</code> will just do nothing.</p>

<h3 id="spurious-wakeups">Spurious wakeups</h3>

<p>A “spurious wakeup” refers to a sleeping thread getting woken up without any <code class="highlighter-rouge">signal</code> call having been made. This is an impossible to avoid edge-case in condition variables. It’s important to point out that this is not being caused by a bug in the Ruby interpreter or anything like that. Instead, the designers of the threading libraries used by your OS found that allowing for the occasional spurious wakeup <a href="https://stackoverflow.com/a/8594644/1420382">greatly improves the speed of condition variable operations</a>. As such, any code that uses condition variables needs to take spurious wakeups into account.</p>

<p>So does this mean that we need to rewrite all the code that we’ve written in this article in an attempt to make it resistant to possible bugs introduced by spurious wakeups? You’ll be glad to know that this isn’t the case as all code snippets in this article have always wrapped the <code class="highlighter-rouge">cond_var.wait</code> statement inside a <code class="highlighter-rouge">while</code> loop!</p>

<p>We covered earlier how using a <code class="highlighter-rouge">while</code> loop makes our code more efficient when dealing with certain race conditions as it causes a newly awakened thread to check whether there is actually anything to do for it, and if not, the thread goes back to sleep. This same <code class="highlighter-rouge">while</code> loop helps us deal with spurious wakeups as well.</p>

<p>When a thread gets woken up by a spurious wakeup and there is nothing for it to do, our usage of a <code class="highlighter-rouge">while</code> loop will cause the thread to detect this and go back to sleep. From the thread’s point of view, being awakened by a spurious wakeup isn’t any different than being woken up with no available tasks to do. So the same mechanism that helps us deal with race conditions solves our spurious wakeup problem as well. It should be obvious by now that <code class="highlighter-rouge">while</code> loops play a very important role when working with condition variables.</p>

<h3 id="conclusion">Conclusion</h3>

<p>Ruby’s condition variables are somewhat notorious for their poor documentation. That’s a shame, because they are wonderful data structures for efficiently solving a very specific set of problems. Although, as we’ve seen, using them isn’t without pitfalls. I hope that this post will go some way towards making them (and their pitfalls) a bit better understood in the wider Ruby community.</p>

<p>I also feel like I should point out that while everything mentioned above is correct to the best of my knowledge, I’m unable to guarantee that absolutely no mistakes snuck in while writing this. As always, please feel free to contact me if you think I got anything wrong, or even if you just want to say hello.</p>
