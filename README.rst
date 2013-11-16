+++++++++++++++++++++++++++++++++++++++++++++++++++
Upload files directly to Rackspace from the browser
+++++++++++++++++++++++++++++++++++++++++++++++++++

Background
==========

We need to let users take a file (possibly a really big file) from their
hard drive and push it to the `Rackspace Cloud Files storage service
<http://www.rackspace.com/cloud/files/>`.

Our approach
============

**warning: this depends on HTML5 stuff!**

1.  Create a container on rackspace cloudfiles.

2.  Set metadata on that container to allow CORS from the domain hosting
    our HTML file.

3.  Write python code to build a temp URL for the PUT method.

4.  Write HTML to build a simple input type=file widget.

5.  Write some javascript so that when a user picks a file with the file
    input tag, we use a `FileReader <https://developer.mozilla.org/en-US/docs/Web/API/FileReader>`
    to read the contents of the file into a buffer.

6.  Use some more javascript to do an ajax PUT to the temp URL created
    in step 3.

Run the example
===============

The apr.py script depends on the the fantastic `Pyrax
<https://github.com/rackspace/pyrax>` package, so install that first,
and then do this stuff::

$ git clone git@github.com:216software/ajax-put-rackspace.git
$ cd ajax-put-rackspace
$ python apr.py YOUR_RACKSPACE_USER_NAME YOUR_API_KEY

Then in your browser, open up http://localhost:8765 and you should see
something like the screenshot in before-upload.png

.. image:: before-upload.png

Now upload a file.  And hopefully, you'll be able to watch a pretty blue
scrollbar track the upload's progress.

.. image:: after-upload.png




The python part
===============


Here's how to create a
container in the rackspace cloudfiles service::

    $ pip install pyrax # this takes a while!
    $ python
    >>> import pyrax
    >>> pyrax.set_setting('identity_type',  'rackspace')
    >>> pyrax.set_credentials(YOUR_USER_NAME, YOUR_API_KEY)

    >>> uploads_container = pyrax.cloudfiles.create_container('uploads')

Don't worry!  Nothing stupid will happen if your "uploads" container
already exists.  You'll get a reference to the existing container.

Then the last thing we need to do in python is set some metadata, so
that this container will accept AJAX PUT uploads::

    >>> uploads_container.set_metadata({
        'Access-Control-Allow-Origin': 'http://example.com'})

Replace example.com with your domain and maybe replace http with https
if that's how you serve your site.

Next we need to make a URL that we can give to the user's browser and
say "send your data to this".

    >>> import uuid
    >>> filename = str(uuid.uuid4())
    >>> upload_url = pyrax.cloudfiles.get_temp_url(
        uploads_container, filename, 60*60, method='PUT')

In my situation want every upload to be stored separately, so I'm using
uuid.uuid4 to make a unique name.

Don't worry about how I'm using ugly-looking uuid filenames.  When I
draw download links later, I can set the filename that the browser sees
to something else entirely.  So even though the file is stored
internally with a name like::

    e710cbd0-b067-47d5-b18f-422d484c3d7d

when somebody downloads that file, they'll see::

    your-mom-jokes.pdf

The other two arguments to get_temp_url are the time that the URL lives
for (60*60 means one hour) and the method='PUT' means this is a URL that
the browser will push to, not pull from.

In other words, after an hour, requests won't succeed, and only PUT
requests are allowed.

Like I said, the python part is really pretty easy!

Security considerations
=======================

The rackspace cloudfiles servers don't check if the request comes from
from a user that authenticated with your web application.

So, if a third person eavesdrops on the temporary URL we make, then they
can use curl or whatever tool they want to push a different file.

When you make a temporary URL, you need to make sure that the right
person and only the right person gets it.

The javascript part
===================

The javascript part is not so fun.  We have to juggle callbacks for a
bunch of asynchronous calls.

**I would love it if somebody forked this repository and sent me a pull
request with a more elegant way to handle this stuff.**

Here's what the code does:

*   Sets an event listener on the on <input type="file"> tag.

*   That event listener makes a FileReader instance named fr.

*   Then it sets a callback on the fr instance to object to handle when
    the fr instance finishes loading a file.

*   Then it tells the fr instance to load in the file chosen by the user
    in the <input type="file"> tag.

*   When the fr instance finishes reading all the data from inside the
    file, the onload callback fires.

*   Inside the onload callback, we use the good ol' jQuery $.ajax method
    to send the data from the file to rackspace.

*   The success callback for $.ajax request in this scenario doesn't do
    anything interesting.  But in the "real code", I do another AJAX
    request back to my server to tell the database to record that a file
    was successfully uploaded to the upload URL.  And I store the
    original file name and the mime type into the database.

*   It isn't strictly necessary, but I want to show a progress bar in
    the browser as the file uploads.  So I define my own xhr object for
    the $.ajax code to use, and that xhr object will spit out a
    "progress" event.

How to set the filename for downloads
=====================================

If you want to alter the name of the Build a temp URL, but pass in "GET"
for the method.  And then add a query-string parameter filename pointing
to whatever you want.






Alternate solutions
===================

Handle the upload and then push to rackspace
--------------------------------------------

The typical solution involves writing some web application code to
accept the file upload from the browser, and then upload it up to
rackspace.

Maybe to be a little fancier, just the first half happens during during
the web request, and some unrelated background process uploads the file
to rackspace later.

Risks with this approach
------------------------

We're using the fantastic `gunicorn <http://gunicorn.org>` WSGI server
with regular plain-jane vanilla sync workers and we're building a web
application where users upload lots of really big files.

Remember that with a synchronous worker, when a user makes a request,
that request completely ties up the back-end worker process until it
replies.  That's why you run a bunch of sync workers.  A request that
comes in will get handled by one of the idle workers -- as long as
somebody is idle.

When too many users try to upload too many really big files at the same
time, then all of the workers could be tied up, and the application
would become unresponsive.

We could always just keep a ton of web application processes around, so
that no matter how busy the application gets, we always have some idle
workers, but that's a worst-case solution.  That's like dealing with a
weight problem by buying a bigger pair of pants.





What about using async workers?
===============================

Well, first of all, I want to get the files up to rackspace, and this
way gets that done better.

Here's the typical use case for async workers: a request comes in and
and you need to talk to some remote API before you can reply, and that
API sometimes takes a second to respond.

After sending the message to the API, your worker is just sitting there
idly, waiting for a reply.

An async worker can go back to answer other requests while waiting for
that API to finish.

Under the hood, these async libraries all monkey-patch stuff like the
socket library, so that when you read or write from a socket, you
automatically yield.

Here's the problem that we ran into (which is likely totally fixable, or
even never was broken).

We're using the excellent werkzeug library to parse file uploads.  It
internally pulls data from the socket named "wsgi.input" passed in with
the WSGI environ.

Reading from that wsgi.input socket doesn't seem to yield out control,
so while our async worker was reading the gigantic file being uploaded,
even though the async worker was idle, it was not switching to go back
and answer other requests.

We couldn't figure out a nice way to force the werkzeug request object
to intermittently yield while reading from the wsgi.input socket.  We
can't always force it to do this -- lots of people use werkzeug without
also using gevent.

The werkzeug code doesn't know it is being run inside a gunicorn
async gevent worker.





.. vim: set syntax=rst: