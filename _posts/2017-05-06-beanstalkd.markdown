---
layout: post
title:  "Asychronous Tasks with Beanstalkd"
date:   2017-05-06 03:00:00 -0600
categories: beanstalkd
---

[Beanstalkd](https://kr.github.io/beanstalkd/) is a simple job-queueing server, first brought to my attention by the excellent [lornajane blog](https://lornajane.net/posts/2014/working-with-php-and-beanstalkd). Queues allow tasks to be run asynchronously: jobs are loaded onto the queue by user-driven scripts and removed for processing by worker scripts. This is useful for anything you don't want users waiting around on&mdash;in my case, sending out updates to integrations at Sycamore.

Sending requests to outside APIs leaves us at the mercy of someone else's response time. We handle this in most cases by running updates on a fixed schedule, instead of tying them directly to user-interactions. This isn't terribly efficient, but in the past, none of our exports have been very large or time-sensitive. When the need arose to push a substantial volume of real-time data, it was time to try something new.

Beanstalkd is a simple first-in, first-out stack of generic text strings. When a worker script requests a job, it receives the text exactly as it was sent in, so the format is entirely up to you. In our workflow, we create an array of data which contains, among other things, the type of the job. We encode this array as JSON and send it off to the Beanstalkd server with [Pheanstalk](https://github.com/pda/pheanstalk).
```
//jobs go in
$queue = new Pheanstalk\Pheanstalk('127.0.0.1');
$data = [
    'type' => 'foo',
    'bar' => 'baz'
];
$queue->put(json_encode($data));
```
We keep a processing script alive with [Supervisord](http://supervisord.org/), which listens continuously for new tasks. When it receives one, it decodes the JSON, checks the type to determine which worker process to load and passes the rest of the data to that process.
```
//jobs come out
$queue = new Pheanstalk\Pheanstalk('127.0.0.1');
while($job = $queue->reserve()) {
    $data = json_decode($job->getData(), true);
    $queue->bury($job);
    if($data['type'] === 'foo') {
        echo $data['bar'];
        $queue->delete($job);
```

Before beginning any job, we put it into a buried state, as suggested by the [FAQ section](https://github.com/kr/beanstalkd/wiki/faq). This is a sort of limbo where jobs can live outside of the active queue. Deleting jobs is the last step of our worker process, so encountering a fatal error will leave the job buried until a real person comes along to decide what to do with it. As JSON is human-readable, this is easy for us to do.

Under the hood, Beanstalkd is a telnet server. This makes it possible to interact with it directly on the command line, which is useful for testing and monitoring, as well as handling buried jobs. See the full [protocol](https://github.com/beanstalkd/beanstalk4py/wiki/Beanstalk-Protocol) for the available commands.
```
telnet 127.0.0.1 11300
```
One downside of Beanstalkd's simplicity is that security is completely up to you. If your whole application lives on one server, you can just listen on localhost and forget about the outside world. In our case, we make use of firewalls, VPNs and voodoo to ensure that connections are limited to our own machines.
