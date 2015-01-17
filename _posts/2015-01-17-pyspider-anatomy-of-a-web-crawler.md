---
layout: tech
title: pyspider
root: "../../../"
---


# Anatomy of a Web Crawler

In this first article we are going to analyze a web crawler.

A web crawler is a tool to scan the web and memorize informations, it opens a whole bunch of web pages, analyzes the page looking for any interesting data, stores the data in a database and starts again with other pages.

If in the page the crawler is analyzing there are some links, the cralwer will follow such links analyzing some more pages.

The search engines are based on this same idea.

I choose a healthy and young open source project in particular [pyspider][pyspider] wrote by our friend [binux][binux].

### A little side note

pyspider was thought to keep monitoring the web, it does not assume that a web page remain constant in time, so after a while it will revisit the same page.

## Overview

The crawler is made by mainly four component.

We have a **scheduler**, **fetcher**, **processor** and a component to monitor the whole.

The scheduler receive task and decide what to do.

There are several possibilities, it can discard a task (maybe the specific web page is been crawled a very little while ago) or it can assign different degree of priorities. 

When the priorities of the various task is been decided, then those tasks are feeded to the fetcher.

The fetcher retry the web page, it is fairly sophisticated, but logically does very little.

When a resource is been fetched from the web is responsibility of the processor to extract useful information.

The processor run a python script wrote by the user, such script is not sand boxed.

It is duty of the processor to capture any exception or log and manage those appropriately.

Finally there is the monitor component.

The web ui is extremely powerful, it let you to edit and debug your script, to manage the whole process, to monitor what task are going on and finally to export the result.

### projects and task

In pyspider we have the notion of **projects** and **tasks**.

A task is single page that need to be retrieved from the web and analyzed.

A project is a bigger entity that wraps all the pages touched by the crawler, the python script necessary to analyze the pages, the databases used to store the data and so on.

In pyspider we can have multiple projects running simultaneously.

## Code Structure Analysis

### Root

The folders we can find in the root are:

+ `data`, empty folder, it is where the data generate from the crawler itself are stored.

+ `docs`, it contains the markdown code for the documentation.

+ `pyspider`, it contains the actual code.

+ `test`, it contains a fairly good amount of tests.

Then there are some important files that I would like to highlight:

+ `.travis.yml`, wonderful, continuous integration! This is how you are sure that your project actually works, test in only your machine with only your version of libraries is not enough.

+ `Dockerfile`, again, wonderful! If I want to try the project in my machine I just run Docker, I don't need to install anything manually, this is a great way to involve developers to contribute in your project.

+ `LICENSE`, necessary for any Open Source project, do not forget yours.

+ `requirements.txt`, in the python world is how you write what packages need to be installed in your system in order to run the software, it is a must in any python project.

+ `run.py`, the main entry point of the software.

+ `setup.py`, this file is a python script that set the system ready to go.

Already from just the root of the project we can say that it is been developed in an extremely professional way, if you are writing any open source this level is what is expected.

### pyspider

Let's dive a little deeper and let's analyze the real code now.

In this folder we find yet other folders, the logical idea behind the whole software has been divide so that it is easier to manage and to grow.

The several folders are:

+ database

+ fetcher

+ libs

+ processor

+ result

+ scheduler

+ webui

In this folder we find also the main entry point of the whole project, `run.py`.

### run.py

This file makes all the necessary chores in order to run the crawler successfully.

Finally it spawns all the necessary computational unities.

Scrolling down we can see the entry point of the whole project, `cli()`.

#### cli()

This function seems complex but stay with me, it is not that bad.

The main goal of `cli()` is to create all the connection to the database and to the message system.

Mostly it parses command line arguments, and create a big dictionary with everything we need.

Finally we start the real work invoking the function `all()`.

#### all()

A web crawler does A LOT of IO, it is a good idea to spawn different thread/subprocess to manage all this work.

In this way when you are waiting for the network to get your html page, at the same time you are extracting useful information from the previous pages.

The function `all()` decides if run subprocess or thread and then invokes all the necessary functions inside a different thread/subprocess.

At this point the required number of threads are spawned for any of the logical piece of the crawler, including the webui.

```python
finally:
    # exit components run in subprocess
	for each in threads:
        if not each.is_alive():
            continue
		if hasattr(each, 'terminate'):
            each.terminate()
		each.join()
```

When we finish and close the webui we will close each thread in a nice and clean way.

Now our crawler is live, let's explore it a little more deeply.

### scheduler

The scheduler receive tasks from two different queues (`newtask_queue` and `status_queue`) and put tasks in another queue (`out_queue`) that will be later consumed by the fetcher.

The very first thing that the scheduler does is to load from the database all the tasks that need to be done.

```python
def run(self):
	while not self._quit:
        try:
            time.sleep(self.LOOP_INTERVAL)
			self._update_projects()
			self._check_task_done()
            self._check_request()
            while self._check_cronjob():
                pass
            self._check_select()
            self._check_delete()
            self._try_dump_cnt()
            self._exceptions = 0
        except KeyboardInterrupt:
            break
        except Exception as e:
            logger.exception(e)
			self._exceptions += 1
			if self._exceptions > self.EXCEPTION_LIMIT:
                break
			continue

```

After that it starts an infinite loop.

In the loop several methods are called:

1. `_update_projects()`: tries to update various settings of the scheduler, eg, if we want to change the crawl speed while the crawler is working.

2. `_check_task_done()`: analyzes the completed task and save those on the database, it get its task from the `status_queue`.

3. `_check_request()`: if the processor asks to analyze more pages, put such pages in the queue, it get the new task from the `newtask_queue`.

4. `_check_select()`: adds new web page to the queue of the fetcher.

5. `_check_delete()`: deletes tasks and projects that have been marked by the user.

6. `_try_dump_cnt()`: writes how many tasks have been done in a file, it is necessary to prevent data loss if the program exits abnormally.

The loop also check for exception or if we ask python to stop the process
### fetcher

The goal of the fetcher is to retrieve web resource.

pyspider is able to manage either plain html text pages and Ajax based pages.

It is important to know that only the fetcher is aware of this difference.

We are going to focus only on the plain html text fetcher, however most of the ideas can be easily ported to the Ajax fetcher.

The idea here is somehow similar to the scheduler's one, we have two queues, one for the input and one for the output and a big loop.

For any element in the input queue, the fetcher makes a request and puts the result in the output queue.

It sound easy but there is a big problem.

The net usually is extremely slow, if you block all the computation whenever you are waiting for a web page the whole is going to run extremely slow.

The solution is simple, do not block the whole computation while you are waiting for the network.

The idea is to send a lot of messages, pretty much all together, over the network and asynchronously wait for the responses to come back.

As soon as we got back one response we call another function, a callback that will take care of manage, in the most appropriate way, such response.

All the heavy asynchronous lift in pyspider is made by [tornado][tornado] another great open source project.

Now that we have the big idea in mind let's explore a little more deeply how this is implemented.

#### run()
```python
def run(self):
	def queue_loop():
        if not self.outqueue or not self.inqueue:
            return
		while not self._quit:
            try:
                if self.outqueue.full():
                    break
				task = self.inqueue.get_nowait()
				task = utils.decode_unicode_obj(task)
				self.fetch(task)
			except queue.Empty:
                break
	tornado.ioloop.PeriodicCallback(queue_loop, 100, io_loop=self.ioloop).start()
	self._running = True
    self.ioloop.start()

```

The method `run()` is the big loop of our fetcher.

Inside run another function `queue_loop()` is defined, such function take all the tasks in the input queue and fetches them, also it listen for an interruption signal.

`queue_loop()` is passed as argument to a tornado class `PeriodicCallback` that, how you can guess, will call it every specific amount of time.

`queue_loop()` just calls another function that brings us a step closer to actually retrieve the web resource: `fetch()`.

#### fetch(self, task, callback=None)

`fetch()` just decides what is the proper way to retrieve a resource, if the resource must be retrieved using `phantomjs_fetch()` or the simpler `http_fetch()`.

We are going to follow `http_fetch()`.

#### http_fetch(self, url, task, callback)

```python
def http_fetch(self, url, task, callback):
    '''HTTP fetcher'''
    fetch = copy.deepcopy(self.default_options)
	fetch['url'] = url
	fetch['headers']['User-Agent'] = self.user_agent

	def handle_response(response):
		...
		return task, result

	try:
        request = tornado.httpclient.HTTPRequest(header_callback=header_callback, **fetch)            
		if self.async:
			self.http_client.fetch(request, handle_response)
		else:
            return handle_response(self.http_client.fetch(request))
        
```

Finally, here is where the real work is done.

This method is a little long, but it is well structured and easy to follow.

In the first part of the method it sets the header, so stuff like the User-Agent, the timeout etc.

Then a function to handle the response is defined, `handle_response()`, we are going to analyze this function later.

Finally we make a tornado request and we fired up such request.

Please note how the same function is used in either case, asynchronous and not asynchronous, to handle the response.

Let's go a little back and analyze what `handle_response()` does.

#### handle_response(response)

```python
def handle_response(response):
	result = {}
	result['orig_url'] = url
    result['content'] = response.body or ''
	callback('http', task, result)
	return task, result
```


The function saves in a dictionary all the relevant informations about a response, stuff like the url, the status code and the actual response, then it calls the callback.

The callback is a little method, `send_result()`.

#### send_result(self, type, task, result)
```python
def send_result(self, type, task, result):
    if self.outqueue:
		self.outqueue.put((task, result))
```

This final method puts the result in the output queue, ready to be consumed by the processor.
	
### processor

The processor's goal is to analyze the pages that have been crawled.

Also the processor is a big loop, but it has three queues in output (`status_queue`, `newtask_queue` and `result_queue`) and only one queue in input (`inqueue`).

Let's analyze a little more deeply the loop in `run()`.

#### run(self)

```python
def run(self):
	try:
		task, response = self.inqueue.get(timeout=1)
		self.on_task(task, response)
		self._exceptions = 0
	except KeyboardInterrupt:
        break
	except Exception as e:
		self._exceptions += 1
		if self._exceptions > self.EXCEPTION_LIMIT:
			break
		continue
```

This method is small and easy to follow, it simply gets the next task that needs to be analyzed from the queue and analyzes it with `on_taks(task, response)`.

The loop listens for the interruption signal, whenever we send such signal to python the loop will break.

Finaly the loops counts the number of exceptions it incurs to, too many exceptions will break the loop.

#### on_task(self, task, response)
```python
def on_task(self, task, response):
	response = rebuild_response(response)
	project = task['project']
	project_data = self.project_manager.get(project, updatetime)
	ret = project_data['instance'].run(
		
	status_pack = {
		'taskid': task['taskid'],
		'project': task['project'],
		'url': task.get('url'),
		...
		}
    self.status_queue.put(utils.unicode_obj(status_pack))
	if ret.follows:
		self.newtask_queue.put(
			[utils.unicode_obj(newtask) for newtask in ret.follows])

	for project, msg, url in ret.messages:
        self.inqueue.put(({...},{...}))

	return True

```
`on_task()` is the method that does the real work.

From the task in input it tries to obtain the project such task belongs to.

The it runs the custom script from the project.

Finally it analyzes the responses of the custom script.

If everything went good a nice dictionary with all the informations we gathered from the page is created and finally puts on the `status_queue` that will be later re-used by the scheduler.

If there are some new links to follow in the page just analyzed such links are pushed into the `newtask_queue` that will be later consumed by the scheduler.

Now, pyspider sends tasks to other projects if this apply.

Finally if some errors happened, stuff like a page returned error, is add to the log.

### end


[pyspider]: https://github.com/binux/pyspider
[binux]: https://github.com/binux
[tornado]: http://tornadoweb.org/
