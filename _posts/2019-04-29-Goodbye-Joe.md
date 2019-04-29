---
layout: post
title:  "Goodbye Joe Armstrong"
date:   2019-04-29 08:00:00 -0400
categories: aws elixir
---
![Erlang logo](/images/erlang.jpg)

[Joe Armstrong](https://en.wikipedia.org/wiki/Joe_Armstrong_(programmer)), one of the creators of the Erlang language, died a little over a week ago on April 20, 2019. The Erlang language, is now over 30 years old, is second most popular virtual machine (i.e. BEAM) after the JVM. It is the first production grade language to focus on concurrency at scale.  It is common for Erlang programs to spin up tens to hundredes of thousands of processes.

Although Erlang was originally written to power voice switches at Ericcson, Cisco now ships two million devices a year running Erlang.  It is estimated 90% of all internet traffic goes through [Erlang](https://twitter.com/guieevc/status/1002494428748140544) controlled nodes.

## The rise of Elixir

Although I do not write much code in Erlang, I am writing more and more code in [Elixir](https://elixir-lang.org/).  Elixir runs on the Erlang virtual machine, and is fully bytecode compatible with Erlang.  However, its syntax is more like _Ruby_, and therefore more digestible and comprehensible than Erlang, even while maintaining its functional roots.

With the rise of IoT, web-sockets and the need to more efficiently drive our multi-core processors I see languages like Erlang and Elixir becoming more popular over time. In particular, one should look at [Phoenix](https://phoenixframework.org/), the Elixir web framework which can handle millions of connections without a sweat.

Finally, if you have an opportunity to go to an Elixir conference -> **Go**.  I had the priviledge of meeting Jose Valim, the creator of [Elixir](https://en.wikipedia.org/wiki/Elixir_(programming_language)).  Jose is brilliant and he could not be more approachable and down to earth. Jose is interested in solving real-world problems with his language. He is also a great speaker and evangelist for the language and ecosystem.

## Elixir and AWS

It is common when working on the AWS platform, that you most poll different regions to find out information.  For example, how many instances am I running world-wide?  Surprisingly, this is a challenging question to answer.  Each region is built independent of all the others (in order to isolate regional failures) and it therefore requires independent calls into the nearly 20 regions. **NOTE:** I would also recommend that one examines "AWS Config" (a terribly named service), which does have the ability to aggregate data from multiple regions. The AWS "Tag Editor" can also be helpful when working in the Console for tracking down resources.

Now, 20 API calls is not a big thing.  It can easily be done sequentially.  However, why should you?  Most of your time is waiting for network I/O. You can easily speed up your query by using [concurrency](https://blog.golang.org/concurrency-is-not-parallelism).

I wrote a series of a small programs that did just that.  This is what I found.

---

| Language        | Time (seconds) |
|-----------------|----------------|
| Bash AWS CLI    | 18             |
| Python          | 11             |
| Go (serial)     | 10             |
| Elixir          | 1.8            |
| Go (concurrent) | 1.5            |

---

Now, the Go program turned out to be the fastest, which is not entirely surprising.  Go has an officially supported [SDK](https://github.com/aws/aws-sdk-go-v2) on AWS. It compiles down to a tiny binary, and does not have the overhead of a virtual machine, which can add a great deal of latency for short-lived programs.

Yet, the Elixir program is less than 1/2 the length of the Go program, and has a certain functional elegance that enhances its readability by a huge margin. Any fraction of a second Elixir loses in performance, it gains in developer productivity and maintainability IMHO.

### Let me see the code!

**Elixir code**
{% highlight elixir %}
def poll_all_instances() do
    instance_data =
      pmap(get_good_regions(), fn (region) ->
        EC2.describe_instances() |> ExAws.request(region: region) |> parse_instance_info() end)
      |> List.flatten

# .. Print out data
  end
{% endhighlight %}

This code uses `pmap` to parallelizes the same request over all regions. The request is to ask for all instances in the region.  The rest of the function relies upon the [ExAws](https://github.com/ex-aws/ex_aws) library to generate the appropriately signed HTTPS REST-ful call into the AWS platform. I then parse out the data I am most interested in.  When the data is all collected, I flatten out the result into a list that I then pretty-print out to the screen.

If you are not familiar with the pipe operator (|>), you do not know whare you are missing. In case you are curious, `pmap` is a user-defined function, building upon Elixir primitives that create async processes, and then waits for all processes to be completed.

Doing the same thing in _Go_ takes up many more LOC using `wg.Add(1)/wg.Wait()` and go routines, although logically identical.

{% highlight elixir %}
  def pmap(collection, func) do
    collection
    |> Enum.map(&(Task.async(fn -> func.(&1) end))) |> Enum.map(&Task.await/1)
    end
{% endhighlight %}

The output of my program...
```bash
+---------------------+---------+------------+
| Instance ID         | State   | AZ         |
+---------------------+---------+------------+
| i-0710e838f877a5c3f | running | us-east-1c |
| i-01c2952c2e8987f86 | running | us-west-2b |
+---------------------+---------+------------+
```

## Doing this in the Real World

In case you are curious, a company recently issued a [blog](https://runbook.cloud/blog/posts/how-we-massively-reduced-our-aws-lambda-bill-with-go/) post, where they talked about how they used _Go_ to increase the efficiency of their lambda functions.  They ended up saving millions of dollars in execution costs, while essentially doing the same thing that we accomplished above.

I see a bright future for Elixir, _Go_ and a variety of other highly concurrent languages. I am investigating [Pony](https://www.ponylang.io/) this week.

**RIP** Joe.

___
