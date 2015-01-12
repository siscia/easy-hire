
# Anatomy of a Web Crawler

In this first article we are going to analyze a web crawler.

I choose an healthy and young open source project in particular [pyspider][pyspider] wrote by our friend [binux][binux].

### A little side note

pyspider was thought to keep monitoring the web, it does not assume that a web page remain constant in time, so after a while it will revisit the same page.

## Overview

The crawler is made by mainly four component.

We have a **scheduler**, **fetcher**, **processor** and a component to monitor the whole.

The scheduler receive task and decide what to do.

There are several possibilities, it can discard a task (maybe the specific web page is been crawled a very little while ago) or it can assign different degree of priorities (a new page will have an higher priorities of a page already crawled 5 day ago.) 

When the priorities of the various task is been decided, then those tasks are feeded to the fetcher.

The fetcher retry the web page, it is fairly sophisticated, but logically does very little.

When a resource is been fetched from the web is responsibility of the processor to extract useful information.

The processor run a python script wrote by the user, such script is not sand boxed.

It is duty of the processor to capture any exception or log and manage those appropriately.

Finally there is the monitor component.

The web ui is extremely powerful, it let you to edit and debug your script, to manage the whole process, to monitor what task are going on and finally to export the result.

## Code Structure Analysis

### Root

The folder we can find in the root are:

+ `data`, empty folder, I suppose it is where the data generate from the crawler itself are stored.

+ `docs`, it contains the markdown code for the documentation.

+ `pyspider`, it contains the actual code.

+ `test`, it contains a fairly good amount of test.

Then there are some important file that I would like to highlight:

+ `.travis.yml`, wonderful continuous integration, this is how you are sure that your project actually works, test in only your machine with only your version of the code is not enough.

+ `Dockerfile`, again, wonderful, if I want to try the project in my machine I just run Docker, I don't need to install anything manually, this is a great way to involve developer to contribute in your project.

+ `LICENSE`, necessary for any Open Source project, do not forget your.

+ `requirements.txt`, in the python world is how you write what package need to be installed in your system in order to run the software, it is a must in any python project.

+ `run.py`, the main entry point of the software.

+ `setup.py`, this file is a python script that set the system ready to go.

Already from just the root of the project we can say that it is been developed in an extremely professional way, if you are writing any open source this level is what is expected.

### pyspider

Let's dive a little deeper and let's analyze the real code now.

In this folder we find yet other folders, the logical idea behind the whole software is been divide so that it is easier to manage and grow.

The several folder are:

+ database

+ fetcher

+ libs

+ processor

+ result

+ scheduler

+ webui

In this folder we find also the main entry point of the whole project, `run.py`.

### run.py

Scrolling down we can see the entry point of the whole project, `cli()`.

#### cli()

This function seems complex but stay with me, it is not that bad.

Firstly it set the logging system.

Then it set up all the database infrastructure necessary, you can see that it support MySQL, MongoDB and SQLite (why don't you add PostgreSQL ?) and also manage the special case of bench marking.

After the database we take care of set up the Queue system, you can guess that it will support RabbitMQ and the normal multiprocessing queue offered by python.

It set phantomjs a way to manage Ajax web page.

It add a list to the dictionary that keep all the information it has build until now.

Up until now this function have simply build the context where the software is going to run.

Finally, if we just want to run the application another function is invoked.

The function `all`.

#### all()

A web crawler does A LOT of IO, it is a good idea to spawn different thread/subprocess to manage all this work.

The function `all()` decide if run subprocess or thread and then invoke all the necessary function inside a different thread/subprocess.

At this point the required number of thread are spawned for any of the logical piece of the crawler, including the webui.

When we finish and close the webui we will closing each thread in a nice and clean way.



[pyspider]: https://github.com/binux/pyspider
[binux]: https://github.com/binux
