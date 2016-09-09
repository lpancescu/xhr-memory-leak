===============
XHR Memory Leak
===============

This is a self-contained test case for `Jenkins bug #10912`_ (the Jenkins_ web
interface causes a significant memory leak in all major browsers). In Chrome,
the Jenkins Dashboard increases its memory footprint from 120 MB at the
beginning, to 1.7 GB less than three hours later; Safari and Firefox show the
same behavior, although the increase is somewhat less steep (in my case,
Firefox reached 2.95 GB after 48 hours, and its ``about:memory`` tool reported
1.2 GB in orphan DOM nodes).

What happens
============

The Timeline tool from Google Chrome shows the number of listeners and nodes
increasing every 5 seconds (for the nodes, it's a significant increase: it
already reached hundreds of thousands of nodes within minutes), and they appear
to never be garbage-collected. The stack trace corresponding to each of these
increases in the number of nodes points to the function ``refreshPart`` from
``hudson-behavior.js``.

The problem is that ``refreshPart`` keeps producing a new closure every 5
seconds, which contains references to both the new and the old DOM trees of the
``div`` they are refreshing, as well as a new ``XMLHttpRequest`` object. The
latter are somewhat special with regards to garbage collection: according to an
article_ by Chris Wellons, ``XMLHttpRequest`` objects are seldom, if ever,
garbage-collected, and should only be created in limited numbers.

A test case and possible solutions
==================================

I wrote a self-contained ``refreshPart`` function to illustrate the memory
leak, as well as two possible solutions for Jenkins:

1. ``noleak-interval.html`` uses ``window.setInterval`` instead of
   ``window.setTimeout``
2. ``noleak-timeout.html`` reuses the same ``XMLHttpRequest`` object, created
   outside of the ``f`` function

The code uses AJAX to replace the contents of a ``pre`` with the contents of a
file from the same directory. If you have Python installed, you can use its
included web server:

* Python 2.x: ``python2 -m SimpleHTTPServer``
* Python 3.x: ``python3 -m http.server --bind localhost``

You can now open `leak.html`_, `noleak-timeout.html`_ and
`noleak-interval.html`_ in different tabs and use Chrome's Task Manager to
monitor their memory usage for a few minutes.

The memory leak only appears when ``XMLHttpRequest`` is used (see
``noleak-noxhr.html`` for an example that is identical to ``leak.html``, except
for using a hard-coded string as a replacement for the original content).

The simplest solution would be to use the ``Ajax.PeriodicalUpdater`` class from
Prototype.js, as illustrated by ``noleak-prototype.html``. Unfortunately, this
isn't practical for a Jenkins fix, since the server returns the full ``div``
with the id (not just its contents, as ``Ajax.PeriodicalUpdater`` requires).

.. _Jenkins bug #10912: https://issues.jenkins-ci.org/browse/JENKINS-10912
.. _Jenkins: https://jenkins.io
.. _article: http://nullprogram.com/blog/2013/02/08/
.. _leak.html: http://localhost:8000/leak.html
.. _noleak-timeout.html: http://localhost:8000/noleak-timeout.html
.. _noleak-interval.html: http://localhost:8000/noleak-interval.html
