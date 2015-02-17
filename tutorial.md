[TOC]

# Einführung

Die Struktur dieses Tutorials nimmt an, dass der Leser ein
gewisses Level an Wissen über Python-Programmierung hat,
sonst jedoch nicht viel. Kein Wissen über Nebenläufigkeit
wird erwartet. Das Ziel ist es, dem Leser die Werkzeuge in
die Hand zu geben, um mit gevent zu arbeiten, ihm zu helfen,
seine vorhandenen Probleme mit Nebenläufigkeit zu bezwingen
und es ihm ermöglichen, noch heute asynchrone Applikationen 
zu schreiben.

### Mitwirkende

In chronologischer Reihenfolge der Mitarbeit:
[Stephen Diehl](http://www.stephendiehl.com)
[J&eacute;r&eacute;my Bethmont](https://github.com/jerem)
[sww](https://github.com/sww)
[Bruno Bigras](https://github.com/brunoqc)
[David Ripton](https://github.com/dripton)
[Travis Cline](https://github.com/traviscline)
[Boris Feld](https://github.com/Lothiraldan)
[youngsterxyf](https://github.com/youngsterxyf)
[Eddie Hebert](https://github.com/ehebert)
[Alexis Metaireau](http://notmyidea.org)
[Daniel Velkov](https://github.com/djv)
[Veit Heller](https://veitheller.de)

Ausserdem ein Dankeschön an Denis Bilenko dafür, dass er gevent
geschrieben und uns bei der Erstellung dieses Tutorials beraten hat.

Dies ist ein kollaboratives Dokument, veröffentlicht unter der MIT-Lizenz.
Du hast etwas hinzuzufügen? Siehst einen Tippfehler? Forke es und erstelle
einen Pull Request auf [Github](https://github.com/sdiehl/gevent-tutorial).
Jede Mitarbeit ist willkommen.

(Anmerkung des Übersetzers: Falls Tippfehler in der Übersetzung auftauchen
solleten, kannst du sie [hier](https://github.com/hellerve/gevent-tutorial)
melden.)

Diese Seite ist auch [in Japanisch](http://methane.github.com/gevent-tutorial-ja) 
und [Italienisch](http://pbertera.github.io/gevent-tutorial-it/) verfügbar.

# Core

## Greenlets

Die primäre Struktur, die in gevent verwendet wird, ist das <strong>Greenlet</strong>, 
eine leichtgewichtige Koroutine die Python als C-Erweiterungs-Modul zur
Verfügung gestellt wird. Greenlets laufen allesamt innerhalb des OS-Prozesses
des Hauptprogrammes, werden aber kooperativ verwaltet.

> Only one greenlet is ever running at any given time. (Nur ein Greenlet läuft zu jeder gegebenen Zeit.)

Dies unterscheidet sich von allen echten Parallelitäts-Konstrukten,
die von den ``multiprocessing``- oder ```threading``-Bibliotheken
implementiert werden; diese verwenden Spin-Prozesse und POSIX-Threads,
welche vom Betriebssystem verwaltet werden und echt parallel ablaufen.

## Synchrone & Asynchrone Ausführung

Die Kernidee von Nebenläufigkeit ist, dass eine grössere Aufgabe in
eine Ansammlung von Sub-Aufgaben unterteilt werden kann, welche
simultan ausgeführt werden sollen oder *asynchron* anstatt nacheinander
oder *synchron*. Ein Übergang zwischen zwei Sub-Aufgaben nennt man
*Kontext-Wechsel*(Context Switch).

Ein Kontext-Wechsel in gevent wird durch *Yielding* umgesetzt. In diesem
Beispiel haben wir zwei Kontexte, die sich gegenseitig "yielden", indem
sie ``gevent.sleep(0)```aufrufen.

[[[cog
import gevent

def foo():
    print('Expliziter Kontext bei foo')
    gevent.sleep(0)
    print('Expliziter Kontextwechsel zu foo')

def bar():
    print('Expliziter Kontext bei bar')
    gevent.sleep(0)
    print('Expliziter Kontextwechsel zu bar')

gevent.joinall([
    gevent.spawn(foo),
    gevent.spawn(bar),
])
]]]
[[[end]]]

Es ist erhellend, den Kontrollfluss des Programms zu visualisieren
oder mit einem Debugger hindurchzugeben, um die Kontext-Wechsel in Echtzeit
zu betrachten.

(Anmerkung des Übersetzers: An dieser Stelle will ich mich entschuldigen,
die Gif nicht übersetzt zu haben, aber ich glaube, dass es dem Verständnis nicht
abträglich ist.)

![Greenlet Control Flow](flow.gif)

Die wirkliche Stärke von gevent ist zu sehen, wenn wir es für
Netzwerk- und IO-lastige Funktionen benutzen, welche kooperativ
verwaltet werden können. Gevent kümmert sich um all die Details,
die nötig sind, damit deine Netzwerkbibliotheken immer implizit ihre
Greenlet-Kontexte liefern, wenn dies möglich ist. Ich kann nicht
genug betonen, was für ein mächtiges Idiom das ist. Aber vielleicht
illustriert ein Beispiel das.

In diesem Fall ist die ``select()``-Funktion normalerweise ein
blockierender Aufruf, der verscheidene Dateideskriptoren abfragt.

[[[cog
import time
import gevent
from gevent import select

start = time.time()
tic = lambda: 'bei %1.1f Sekunden' % (time.time() - start)

def gr1():
    # Wartet eine Sekunde lang, aber wir wollen nicht herumlungern
    print('Beginn der Wartezeit: %s' % tic())
    select.select([], [], [], 2)
    print('Ende der Wartezeit: %s' % tic())

def gr2():
    # Wartet eine Sekunde lang, aber wir wollen nicht herumlungern
    print('Beginn der Wartezeit: %s' % tic())
    select.select([], [], [], 2)
    print('Ende der Wartezeit: %s' % tic())

def gr3():
    print("Hey, lass uns irgendwas tun, solange die Greenlets warten, %s" % tic())
    gevent.sleep(1)

gevent.joinall([
    gevent.spawn(gr1),
    gevent.spawn(gr2),
    gevent.spawn(gr3),
])
]]]
[[[end]]]

Ein weiteres etwas synthetisches Beispiel definiert eine 
``task``-Funktion, die *nichtdeterministisch* ist(d.h. es wird nicht
garantiert, dass ihre Rückgabe immer das selbe Ergebnis bei gleicher
Eingabe ist). In diesem Fall ist der Nebeneffekt dieser Funktion,
dass die Ausführung der Aufgabe für eine zufällige Anzahl an Sekunden
pausiert wird.

[[[cog
import gevent
import random

def task(pid):
    """
    Eine nichtdeterministische Aufgabe
    """
    gevent.sleep(random.randint(0,2)*0.001)
    print('Aufgabe %s beendet' % pid)

def synchronous():
    for i in range(1,10):
        task(i)

def asynchronous():
    threads = [gevent.spawn(task, i) for i in xrange(10)]
    gevent.joinall(threads)

print('Synchron:')
synchronous()

print('Asynchron:')
asynchronous()
]]]
[[[end]]]

In der synchronen Funktion laufen alle Aufrufe von ``task()``
sequentiell ab, was ein *blockierendes* (d.h. die Ausführung
des Hauptprogrammes pausierendes) Programm erzeugt. Es "wartet"
auf die Ausführung jedes Aufrufes.

Die wichitgen Teile des Programms sind die Aufrufe von ``gevent.spawn``,
welche die angegebene Funktion in einem Greenlet-Thread verpackt.
Die Liste initialisierter Greenlets werden im Array ``threads`` gespeichert,
welcher an die Funktion ```gevent.joinall```weitergereicht wird, welche
das laufende Programm blockiert, um alle Greenlets auszuführen.
Die Ausführung wird nur weitergeführt, wenn alle Greenlets terminieren.

Der wichtige Umstand hier ist, dass die Reihenfolge, in der der Code
ausgeführt wird, im asynchronen Fall mehr oder weniger zufällig ist und
dass die ganze Ausführungszeit im asynchronen Fall sehr viel geringer ist
als im synchronen Fall. Tatsächlich ist die maximale Zeit, die der
synchrone Code braucht, um einen ```task`` auszuführen 0.002 Sekunden,
was bedeutet, dass die Gesamtzeit sich auf etwa 0.02 Sekunden beläuft.
Im asynchronen Fall beläuft sich die maximale Gesamtzeit auf ungefähr
0.002 Sekunden, da kein ```task```die Ausführung der anderen blockiert.

Bei einem etwas üblicheren Fall, dem asynchronen Abrufen von Daten von
einem Server, wird sich die Laufzeit von ``fetch()`` in verschiedenen
Abfragen unterscheiden, in Abhängigkeit von der Last des Servers zur Zeit
der Abfrage.

[[[cog
import gevent.monkey
gevent.monkey.patch_socket()

import gevent
import urllib2
import simplejson as json

def fetch(pid):
    response = urllib2.urlopen('http://json-time.appspot.com/time.json')
    result = response.read()
    json_result = json.loads(result)
    datetime = json_result['datetime']

    print('Prozess %s: %s' % (pid, datetime))
    return json_result['datetime']

def synchronous():
    for i in range(1,10):
        fetch(i)

def asynchronous():
    threads = []
    for i in range(1,10):
        threads.append(gevent.spawn(fetch, i))
    gevent.joinall(threads)

print('Synchron:')
synchronous()

print('Asynchron:')
asynchronous()
]]]
[[[end]]]

## Determinismus

Wie bereits erwähnt sind Greenlets deterministisch. Gibt man ihnen
den selben Input und konfiguriert man sie gleich, produzieren sie
immer die selbe Ausgabe. Im Folgenenden werden wir zum Beispiel eine
Aufgabe auf einen Multiprozessor-Pool aufteilen und die Resultate mit
denen eines gevent-Pools vergleichen.

<pre>
<code class="python">
import time

def echo(i):
    time.sleep(0.001)
    return i

# Nichtdeterministischer Prozess Pool

from multiprocessing.pool import Pool

p = Pool(10)
run1 = [a for a in p.imap_unordered(echo, xrange(10))]
run2 = [a for a in p.imap_unordered(echo, xrange(10))]
run3 = [a for a in p.imap_unordered(echo, xrange(10))]
run4 = [a for a in p.imap_unordered(echo, xrange(10))]

print(run1 == run2 == run3 == run4)

# Deterministischer Gevent Pool

from gevent.pool import Pool

p = Pool(10)
run1 = [a for a in p.imap_unordered(echo, xrange(10))]
run2 = [a for a in p.imap_unordered(echo, xrange(10))]
run3 = [a for a in p.imap_unordered(echo, xrange(10))]
run4 = [a for a in p.imap_unordered(echo, xrange(10))]

print(run1 == run2 == run3 == run4)
</code>
</pre>

<pre>
<code class="python">False
True</code>
</pre>

Obwohl gevent normalerweise deterministisch ist, können Quellen
von Nichtdeterminismus sich in ein Programm einschleichen,
wenn man beginnt, mit der Aussenwelt, zum Beispiel Sockets
und Dateien, zu kommunizieren. Daher können Green Threads,
obwohl sie eine Form der "deterministischen Nebenläufigkeit"
sind, trotzdem einige der gleichen Probleme haben, die
POSIX Threads und Prozesse durchmachen.

Das immerwährende Problem mit Nebenläufigkeit ist bekannt als
die *Wettlaufsituation*(Race Condition). Einfach gesagt passiert
eine Wettlaufsituation, wenn zwei nebenläufige Threads/Prozesse
von einer geteilten Resource abhängen, sie jedoch auch zu modifizieren
versuchen. Dies resultiert in Resourcen, deren Werte abhängig
von Zeit und Ablauf der Ausführung werden. Dies ist ein Problem
und im Allgemeinen sollte man sehr stark versuchen, Wettlaufsituationen
zu vermeiden, da sie global nicht-deterministisches Programmverhalten
zur Folge haben.

Der beste Ansatz hierfür ist es, jeglichen globalen Zustand zu jedem
Zeitpunkt zu vermeiden. Globaler Zustand und Seiten-Effekte zur Import-Zeit
werden dich immer wieder belästigen!

## Greenlets kreieren

gevent bietet einige Wrapper um die Initialisierung von Greenlets
an. Einige der meist verwendeten Muster sind:

[[[cog
import gevent
from gevent import Greenlet

def foo(message, n):
    """
    Jeder Thread bekommt die Argumente message und n
    bei seiner Initialisierung
    """
    gevent.sleep(n)
    print(message)

# Initialisiert eine neue Greenlet-Instanz, die die Funktion
# foo ausführt
thread1 = Greenlet.spawn(foo, "Hallo", 1)

# Wrapper, um ein neues Greenlet mit der Funktion foo
# zu erstellen und auszuführen, mit den übergebenen Argumenten
thread2 = gevent.spawn(foo, "Ich lebe!", 2)

# Lambda-Ausdruck
thread3 = gevent.spawn(lambda x: (x+1), 2)

threads = [thread1, thread2, thread3]

# Blockiert, bis alle Threads fertig sind.
gevent.joinall(threads)
]]]
[[[end]]]

Zusätzlich zur Verwendung der Greenlet-Klasse kann man auch eine
Subklasse dieser erstellen und die ``_run`` Methode überschreiben.

[[[cog
import gevent
from gevent import Greenlet

class MyGreenlet(Greenlet):

    def __init__(self, message, n):
        Greenlet.__init__(self)
        self.message = message
        self.n = n

    def _run(self):
        print(self.message)
        gevent.sleep(self.n)

g = MyGreenlet("Hallo!", 3)
g.start()
g.join()
]]]
[[[end]]]


## Greenlet-Zustände

Wie jeder andere Code-Teil können Greenlets auf verschiedene
Wege fehlschlagen. Ein Greenlet mag fehlschlagen, weil es eine
Exception wirft, um das Program zu beenden oder weil es zu
viele System-Resourcen benötigt.

Der interne Zustand eines Greenlets ist normalerweise ein
von der Zeit abhängiger Parameter. Es gibt einige Flags in
Greenlets, die es ermöglichen, den Zustand des Threads zu
beobachten:

- ``started`` -- Boolean, zeigt an, ob das Greenlet gestartet wurde
- ``ready()`` -- Boolean, zeigt an, ob das Greenlet angehalten ist
- ``successful()`` -- Boolean, zeigt an, ob das Greenlet angehalten ist und keine Exception geworfen hat
- ``value`` -- jeglicher Wert, der Rückgabewert des Greenlets
- ``exception`` -- Exception, nicht aufgefangene Exception, die innerhalb des Greenlets geworfen wurde

[[[cog
import gevent

def win():
    return 'Gewonnen!'

def fail():
    raise Exception('Du bist ein Verlierer im Verlieren.')

winner = gevent.spawn(win)
loser = gevent.spawn(fail)

print(winner.started) # True
print(loser.started)  # True

# Exceptions die im Greenlet geworfen wurden bleiben im Greenlet.
try:
    gevent.joinall([winner, loser])
except Exception as e:
    print('Hierhin kommen wir nie')

print(winner.value) # 'Gewonnen!'
print(loser.value)  # None

print(winner.ready()) # True
print(loser.ready())  # True

print(winner.successful()) # True
print(loser.successful())  # False

# Die Exception die in fail geworfen wurde wird nicht ausserhalb
# des Greenlets propagiert. Ein stacktrace wird nach stdout geschrieben,
# aber der Stack des Parents wird nicht aufgerollt.

print(loser.exception)

# Es ist jedoch möglich die Exception auch ausserhalb wieder
# zu werden
# raise loser.exception
# oder mit
# loser.get()
]]]
[[[end]]]

## Programmende

Greenlets, die nicht beenden, wenn das Hauptprogramm ein SIGQUIT
erhält, können die Programmausführung länger als erwartet weiterführen.
Diese werden zu sogenannten "Zombie-Prozessen", die ausserhalb des
Python-Interpreters beendet werden müssen.

Ein häufig verwendetes Muster ist es, auf SIGQUIT-Signale in Richtung
des Hauptprogramms zu hören und ``gevent.shutdown`` vor dem Beenden
des Programms aufzurufen.

<pre>
<code class="python">import gevent
import signal

def run_forever():
    gevent.sleep(1000)

if __name__ == '__main__':
    gevent.signal(signal.SIGQUIT, gevent.kill)
    thread = gevent.spawn(run_forever)
    thread.join()
</code>
</pre>

## Timeouts

Timeouts sind eine Einschränkung der Laufzeit eines Code-Blocks
oder Greenlets.

<pre>
<code class="python">
import gevent
from gevent import Timeout

seconds = 10

timeout = Timeout(seconds)
timeout.start()

def wait():
    gevent.sleep(10)

try:
    gevent.spawn(wait).join()
except Timeout:
    print('Konnte nicht beendet werden')

</code>
</pre>

Sie können auch mit einem Context Manager in einem ``with``-Ausdruck
verwendet werden.

<pre>
<code class="python">import gevent
from gevent import Timeout

time_to_wait = 5 # Sekunden

class TooLong(Exception):
    pass

with Timeout(time_to_wait, TooLong):
    gevent.sleep(10)
</code>
</pre>

Zusätzlich liefert gevent auch Timeout-Argumente für eine
Vielzahl von Greenlet- und Datenstrukturen-basierten Aufrufen.
Zum Beispiel:

[[[cog
import gevent
from gevent import Timeout

def wait():
    gevent.sleep(2)

timer = Timeout(1).start()
thread1 = gevent.spawn(wait)

try:
    thread1.join(timeout=timer)
except Timeout:
    print('Timeout in Thread 1')

# --

timer = Timeout.start_new(1)
thread2 = gevent.spawn(wait)

try:
    thread2.get(timeout=timer)
except Timeout:
    print('Timeout in Thread 2')

# --

try:
    gevent.with_timeout(1, wait)
except Timeout:
    print('Timeout in Thread 3')

]]]
[[[end]]]

## Monkeypatching

Alas we come to dark corners of Gevent. I've avoided mentioning
monkey patching up until now to try and motivate the powerful
coroutine patterns, but the time has come to discuss the dark arts
of monkey-patching. If you noticed above we invoked the command
``monkey.patch_socket()``. This is a purely side-effectful command to
modify the standard library's socket library.

<pre>
<code class="python">import socket
print(socket.socket)

print("After monkey patch")
from gevent import monkey
monkey.patch_socket()
print(socket.socket)

import select
print(select.select)
monkey.patch_select()
print("After monkey patch")
print(select.select)
</code>
</pre>

<pre>
<code class="python">class 'socket.socket'
After monkey patch
class 'gevent.socket.socket'

built-in function select
After monkey patch
function select at 0x1924de8
</code>
</pre>

Python's runtime allows for most objects to be modified at runtime
including modules, classes, and even functions. This is generally an
astoudingly bad idea since it creates an "implicit side-effect" that is
most often extremely difficult to debug if problems occur, nevertheless
in extreme situations where a library needs to alter the fundamental
behavior of Python itself monkey patches can be used. In this case gevent
is capable of patching most of the blocking system calls in the standard
library including those in ``socket``, ``ssl``, ``threading`` and
``select`` modules to instead behave cooperatively.

For example, the Redis python bindings normally uses regular tcp
sockets to communicate with the ``redis-server`` instance. Simply
by invoking ``gevent.monkey.patch_all()`` we can make the redis
bindings schedule requests cooperatively and work with the rest
of our gevent stack.

This lets us integrate libraries that would not normally work with
gevent without ever writing a single line of code. While monkey-patching
is still evil, in this case it is a "useful evil".

# Data Structures

## Events

Events are a form of asynchronous communication between
Greenlets.

<pre>
<code class="python">import gevent
from gevent.event import Event

'''
Illustrates the use of events
'''


evt = Event()

def setter():
    '''After 3 seconds, wake all threads waiting on the value of evt'''
	print('A: Hey wait for me, I have to do something')
	gevent.sleep(3)
	print("Ok, I'm done")
	evt.set()


def waiter():
	'''After 3 seconds the get call will unblock'''
	print("I'll wait for you")
	evt.wait()  # blocking
	print("It's about time")

def main():
	gevent.joinall([
		gevent.spawn(setter),
		gevent.spawn(waiter),
		gevent.spawn(waiter),
		gevent.spawn(waiter),
		gevent.spawn(waiter),
		gevent.spawn(waiter)
	])

if __name__ == '__main__': main()

</code>
</pre>

An extension of the Event object is the AsyncResult which
allows you to send a value along with the wakeup call. This is
sometimes called a future or a deferred, since it holds a
reference to a future value that can be set on an arbitrary time
schedule.

<pre>
<code class="python">import gevent
from gevent.event import AsyncResult
a = AsyncResult()

def setter():
    """
    After 3 seconds set the result of a.
    """
    gevent.sleep(3)
    a.set('Hello!')

def waiter():
    """
    After 3 seconds the get call will unblock after the setter
    puts a value into the AsyncResult.
    """
    print(a.get())

gevent.joinall([
    gevent.spawn(setter),
    gevent.spawn(waiter),
])

</code>
</pre>

## Queues

Queues are ordered sets of data that have the usual ``put`` / ``get``
operations but are written in a way such that they can be safely
manipulated across Greenlets.

For example if one Greenlet grabs an item off of the queue, the
same item will not be grabbed by another Greenlet executing
simultaneously.

[[[cog
import gevent
from gevent.queue import Queue

tasks = Queue()

def worker(n):
    while not tasks.empty():
        task = tasks.get()
        print('Worker %s got task %s' % (n, task))
        gevent.sleep(0)

    print('Quitting time!')

def boss():
    for i in xrange(1,25):
        tasks.put_nowait(i)

gevent.spawn(boss).join()

gevent.joinall([
    gevent.spawn(worker, 'steve'),
    gevent.spawn(worker, 'john'),
    gevent.spawn(worker, 'nancy'),
])
]]]
[[[end]]]

Queues can also block on either ``put`` or ``get`` as the need arises.

Each of the ``put`` and ``get`` operations has a non-blocking
counterpart, ``put_nowait`` and
``get_nowait`` which will not block, but instead raise
either ``gevent.queue.Empty`` or
``gevent.queue.Full`` if the operation is not possible.

In this example we have the boss running simultaneously to the
workers and have a restriction on the Queue preventing it from containing
more than three elements. This restriction means that the ``put``
operation will block until there is space on the queue.
Conversely the ``get`` operation will block if there are
no elements on the queue to fetch, it also takes a timeout
argument to allow for the queue to exit with the exception
``gevent.queue.Empty`` if no work can be found within the
time frame of the Timeout.

[[[cog
import gevent
from gevent.queue import Queue, Empty

tasks = Queue(maxsize=3)

def worker(name):
    try:
        while True:
            task = tasks.get(timeout=1) # decrements queue size by 1
            print('Worker %s got task %s' % (name, task))
            gevent.sleep(0)
    except Empty:
        print('Quitting time!')

def boss():
    """
    Boss will wait to hand out work until a individual worker is
    free since the maxsize of the task queue is 3.
    """

    for i in xrange(1,10):
        tasks.put(i)
    print('Assigned all work in iteration 1')

    for i in xrange(10,20):
        tasks.put(i)
    print('Assigned all work in iteration 2')

gevent.joinall([
    gevent.spawn(boss),
    gevent.spawn(worker, 'steve'),
    gevent.spawn(worker, 'john'),
    gevent.spawn(worker, 'bob'),
])
]]]
[[[end]]]

## Groups and Pools

A group is a collection of running greenlets which are managed
and scheduled together as group. It also doubles as parallel
dispatcher that mirrors the Python ``multiprocessing`` library.

[[[cog
import gevent
from gevent.pool import Group

def talk(msg):
    for i in xrange(3):
        print(msg)

g1 = gevent.spawn(talk, 'bar')
g2 = gevent.spawn(talk, 'foo')
g3 = gevent.spawn(talk, 'fizz')

group = Group()
group.add(g1)
group.add(g2)
group.join()

group.add(g3)
group.join()
]]]
[[[end]]]

This is very useful for managing groups of asynchronous tasks.

As mentioned above, ``Group`` also provides an API for dispatching
jobs to grouped greenlets and collecting their results in various
ways.

[[[cog
import gevent
from gevent import getcurrent
from gevent.pool import Group

group = Group()

def hello_from(n):
    print('Size of group %s' % len(group))
    print('Hello from Greenlet %s' % id(getcurrent()))

group.map(hello_from, xrange(3))


def intensive(n):
    gevent.sleep(3 - n)
    return 'task', n

print('Ordered')

ogroup = Group()
for i in ogroup.imap(intensive, xrange(3)):
    print(i)

print('Unordered')

igroup = Group()
for i in igroup.imap_unordered(intensive, xrange(3)):
    print(i)

]]]
[[[end]]]

A pool is a structure designed for handling dynamic numbers of
greenlets which need to be concurrency-limited.  This is often
desirable in cases where one wants to do many network or IO bound
tasks in parallel.

[[[cog
import gevent
from gevent.pool import Pool

pool = Pool(2)

def hello_from(n):
    print('Size of pool %s' % len(pool))

pool.map(hello_from, xrange(3))
]]]
[[[end]]]

Often when building gevent driven services one will center the
entire service around a pool structure. An example might be a
class which polls on various sockets.

<pre>
<code class="python">from gevent.pool import Pool

class SocketPool(object):

    def __init__(self):
        self.pool = Pool(1000)
        self.pool.start()

    def listen(self, socket):
        while True:
            socket.recv()

    def add_handler(self, socket):
        if self.pool.full():
            raise Exception("At maximum pool size")
        else:
            self.pool.spawn(self.listen, socket)

    def shutdown(self):
        self.pool.kill()

</code>
</pre>

## Locks and Semaphores

A semaphore is a low level synchronization primitive that allows
greenlets to coordinate and limit concurrent access or execution. A
semaphore exposes two methods, ``acquire`` and ``release`` The
difference between the number of times a semaphore has been
acquired and released is called the bound of the semaphore. If a
semaphore bound reaches 0 it will block until another greenlet
releases its acquisition.

[[[cog
from gevent import sleep
from gevent.pool import Pool
from gevent.coros import BoundedSemaphore

sem = BoundedSemaphore(2)

def worker1(n):
    sem.acquire()
    print('Worker %i acquired semaphore' % n)
    sleep(0)
    sem.release()
    print('Worker %i released semaphore' % n)

def worker2(n):
    with sem:
        print('Worker %i acquired semaphore' % n)
        sleep(0)
    print('Worker %i released semaphore' % n)

pool = Pool()
pool.map(worker1, xrange(0,2))
pool.map(worker2, xrange(3,6))
]]]
[[[end]]]

A semaphore with bound of 1 is known as a Lock. it provides
exclusive execution to one greenlet. They are often used to
ensure that resources are only in use at one time in the context
of a program.

## Thread Locals

Gevent also allows you to specify data which is local to the
greenlet context. Internally, this is implemented as a global
lookup which addresses a private namespace keyed by the
greenlet's ``getcurrent()`` value.

[[[cog
import gevent
from gevent.local import local

stash = local()

def f1():
    stash.x = 1
    print(stash.x)

def f2():
    stash.y = 2
    print(stash.y)

    try:
        stash.x
    except AttributeError:
        print("x is not local to f2")

g1 = gevent.spawn(f1)
g2 = gevent.spawn(f2)

gevent.joinall([g1, g2])
]]]
[[[end]]]

Many web frameworks that use gevent store HTTP session
objects inside gevent thread locals. For example, using the
Werkzeug utility library and its proxy object we can create
Flask-style request objects.

<pre>
<code class="python">from gevent.local import local
from werkzeug.local import LocalProxy
from werkzeug.wrappers import Request
from contextlib import contextmanager

from gevent.wsgi import WSGIServer

_requests = local()
request = LocalProxy(lambda: _requests.request)

@contextmanager
def sessionmanager(environ):
    _requests.request = Request(environ)
    yield
    _requests.request = None

def logic():
    return "Hello " + request.remote_addr

def application(environ, start_response):
    status = '200 OK'

    with sessionmanager(environ):
        body = logic()

    headers = [
        ('Content-Type', 'text/html')
    ]

    start_response(status, headers)
    return [body]

WSGIServer(('', 8000), application).serve_forever()


<code>
</pre>

Flask's system is a bit more sophisticated than this example, but the
idea of using thread locals as local session storage is nonetheless the
same.

## Subprocess

As of gevent 1.0, ``gevent.subprocess`` -- a patched version of Python's
``subprocess`` module -- has been added. It supports cooperative waiting on
subprocesses.

<pre>
<code class="python">
import gevent
from gevent.subprocess import Popen, PIPE

def cron():
    while True:
        print("cron")
        gevent.sleep(0.2)

g = gevent.spawn(cron)
sub = Popen(['sleep 1; uname'], stdout=PIPE, shell=True)
out, err = sub.communicate()
g.kill()
print(out.rstrip())
</pre>

<pre>
<code class="python">
cron
cron
cron
cron
cron
Linux
<code>
</pre>

Many people also want to use ``gevent`` and ``multiprocessing`` together. One of
the most obvious challenges is that inter-process communication provided by
``multiprocessing`` is not cooperative by default. Since
``multiprocessing.Connection``-based objects (such as ``Pipe``) expose their
underlying file descriptors, ``gevent.socket.wait_read`` and ``wait_write`` can
be used to cooperatively wait for ready-to-read/ready-to-write events before
actually reading/writing:

<pre>
<code class="python">
import gevent
from multiprocessing import Process, Pipe
from gevent.socket import wait_read, wait_write

# To Process
a, b = Pipe()

# From Process
c, d = Pipe()

def relay():
    for i in xrange(10):
        msg = b.recv()
        c.send(msg + " in " + str(i))

def put_msg():
    for i in xrange(10):
        wait_write(a.fileno())
        a.send('hi')

def get_msg():
    for i in xrange(10):
        wait_read(d.fileno())
        print(d.recv())

if __name__ == '__main__':
    proc = Process(target=relay)
    proc.start()

    g1 = gevent.spawn(get_msg)
    g2 = gevent.spawn(put_msg)
    gevent.joinall([g1, g2], timeout=1)
</code>
</pre>

Note, however, that the combination of ``multiprocessing`` and gevent brings
along certain OS-dependent pitfalls, among others:

* After [forking](http://linux.die.net/man/2/fork) on POSIX-compliant systems
gevent's state in the child is ill-posed. One side effect is that greenlets
spawned before ``multiprocessing.Process`` creation run in both, parent and
child process.
* ``a.send()`` in ``put_msg()`` above might still block the calling thread
non-cooperatively: a ready-to-write event only ensures that one byte can be
written. The underlying buffer might be full before the attempted write is
complete.
* The ``wait_write()`` / ``wait_read()``-based approach as indicated above does
not work on Windows (``IOError: 3 is not a socket (files are not supported)``),
because Windows cannot watch pipes for events.

The Python package [gipc](http://pypi.python.org/pypi/gipc) overcomes these
challenges for you in a largely transparent fashion on both, POSIX-compliant and
Windows systems. It provides gevent-aware ``multiprocessing.Process``-based
child processes and gevent-cooperative inter-process communication based on
pipes.

## Actors

The actor model is a higher level concurrency model popularized
by the language Erlang. In short the main idea is that you have a
collection of independent Actors which have an inbox from which
they receive messages from other Actors. The main loop inside the
Actor iterates through its messages and takes action according to
its desired behavior.

Gevent does not have a primitive Actor type, but we can define
one very simply using a Queue inside of a subclassed Greenlet.

<pre>
<code class="python">import gevent
from gevent.queue import Queue


class Actor(gevent.Greenlet):

    def __init__(self):
        self.inbox = Queue()
        Greenlet.__init__(self)

    def receive(self, message):
        """
        Define in your subclass.
        """
        raise NotImplemented()

    def _run(self):
        self.running = True

        while self.running:
            message = self.inbox.get()
            self.receive(message)

</code>
</pre>

In a use case:

<pre>
<code class="python">import gevent
from gevent.queue import Queue
from gevent import Greenlet

class Pinger(Actor):
    def receive(self, message):
        print(message)
        pong.inbox.put('ping')
        gevent.sleep(0)

class Ponger(Actor):
    def receive(self, message):
        print(message)
        ping.inbox.put('pong')
        gevent.sleep(0)

ping = Pinger()
pong = Ponger()

ping.start()
pong.start()

ping.inbox.put('start')
gevent.joinall([ping, pong])
</code>
</pre>

# Real World Applications

## Gevent ZeroMQ

[ZeroMQ](http://www.zeromq.org/) is described by its authors as
"a socket library that acts as a concurrency framework". It is a
very powerful messaging layer for building concurrent and
distributed applications.

ZeroMQ provides a variety of socket primitives, the simplest of
which being a Request-Response socket pair. A socket has two
methods of interest ``send`` and ``recv``, both of which are
normally blocking operations. But this is remedied by a briliant
library by [Travis Cline](https://github.com/traviscline) which
uses gevent.socket to poll ZeroMQ sockets in a non-blocking
manner.  You can install gevent-zeromq from PyPi via:  ``pip install
gevent-zeromq``

[[[cog
# Note: Remember to ``pip install pyzmq``
import gevent
import zmq.green as zmq

# Global Context
context = zmq.Context()

def server():
    server_socket = context.socket(zmq.REQ)
    server_socket.bind("tcp://127.0.0.1:5000")

    for request in range(1,10):
        server_socket.send("Hello")
        print('Switched to Server for %s' % request)
        # Implicit context switch occurs here
        server_socket.recv()

def client():
    client_socket = context.socket(zmq.REP)
    client_socket.connect("tcp://127.0.0.1:5000")

    for request in range(1,10):

        client_socket.recv()
        print('Switched to Client for %s' % request)
        # Implicit context switch occurs here
        client_socket.send("World")

publisher = gevent.spawn(server)
client    = gevent.spawn(client)

gevent.joinall([publisher, client])

]]]
[[[end]]]

## Simple Servers

<pre>
<code class="python">
# On Unix: Access with ``$ nc 127.0.0.1 5000``
# On Window: Access with ``$ telnet 127.0.0.1 5000``

from gevent.server import StreamServer

def handle(socket, address):
    socket.send("Hello from a telnet!\n")
    for i in range(5):
        socket.send(str(i) + '\n')
    socket.close()

server = StreamServer(('127.0.0.1', 5000), handle)
server.serve_forever()
</code>
</pre>

## WSGI Servers

Gevent provides two WSGI servers for serving content over HTTP.
Henceforth called ``wsgi`` and ``pywsgi``:

* gevent.wsgi.WSGIServer
* gevent.pywsgi.WSGIServer

In earlier versions of gevent before 1.0.x, gevent used libevent
instead of libev. Libevent included a fast HTTP server which was
used by gevent's ``wsgi`` server.

In gevent 1.0.x there is no http server included. Instead
``gevent.wsgi`` is now an alias for the pure Python server in
``gevent.pywsgi``.


## Streaming Servers

**If you are using gevent 1.0.x, this section does not apply**

For those familiar with streaming HTTP services, the core idea is
that in the headers we do not specify a length of the content. We
instead hold the connection open and flush chunks down the pipe,
prefixing each with a hex digit indicating the length of the
chunk. The stream is closed when a size zero chunk is sent.

    HTTP/1.1 200 OK
    Content-Type: text/plain
    Transfer-Encoding: chunked

    8
    <p>Hello

    9
    World</p>

    0

The above HTTP connection could not be created in wsgi
because streaming is not supported. It would instead have to
buffered.

<pre>
<code class="python">from gevent.wsgi import WSGIServer

def application(environ, start_response):
    status = '200 OK'
    body = '&lt;p&gt;Hello World&lt;/p&gt;'

    headers = [
        ('Content-Type', 'text/html')
    ]

    start_response(status, headers)
    return [body]

WSGIServer(('', 8000), application).serve_forever()

</code>
</pre>

Using pywsgi we can however write our handler as a generator and
yield the result chunk by chunk.

<pre>
<code class="python">from gevent.pywsgi import WSGIServer

def application(environ, start_response):
    status = '200 OK'

    headers = [
        ('Content-Type', 'text/html')
    ]

    start_response(status, headers)
    yield "&lt;p&gt;Hello"
    yield "World&lt;/p&gt;"

WSGIServer(('', 8000), application).serve_forever()

</code>
</pre>

But regardless, performance on Gevent servers is phenomenal
compared to other Python servers. libev is a very vetted technology
and its derivative servers are known to perform well at scale.

To benchmark, try Apache Benchmark ``ab`` or see this
[Benchmark of Python WSGI Servers](http://nichol.as/benchmark-of-python-web-servers)
for comparison with other servers.

<pre>
<code class="shell">$ ab -n 10000 -c 100 http://127.0.0.1:8000/
</code>
</pre>

## Long Polling

<pre>
<code class="python">import gevent
from gevent.queue import Queue, Empty
from gevent.pywsgi import WSGIServer
import simplejson as json

data_source = Queue()

def producer():
    while True:
        data_source.put_nowait('Hello World')
        gevent.sleep(1)

def ajax_endpoint(environ, start_response):
    status = '200 OK'
    headers = [
        ('Content-Type', 'application/json')
    ]

    start_response(status, headers)

    while True:
        try:
            datum = data_source.get(timeout=5)
            yield json.dumps(datum) + '\n'
        except Empty:
            pass


gevent.spawn(producer)

WSGIServer(('', 8000), ajax_endpoint).serve_forever()

</code>
</pre>

## Websockets

Websocket example which requires <a href="https://bitbucket.org/Jeffrey/gevent-websocket/src">gevent-websocket</a>.


<pre>
<code class="python"># Simple gevent-websocket server
import json
import random

from gevent import pywsgi, sleep
from geventwebsocket.handler import WebSocketHandler

class WebSocketApp(object):
    '''Send random data to the websocket'''

    def __call__(self, environ, start_response):
        ws = environ['wsgi.websocket']
        x = 0
        while True:
            data = json.dumps({'x': x, 'y': random.randint(1, 5)})
            ws.send(data)
            x += 1
            sleep(0.5)

server = pywsgi.WSGIServer(("", 10000), WebSocketApp(),
    handler_class=WebSocketHandler)
server.serve_forever()
</code>
</pre>

HTML Page:

    <html>
        <head>
            <title>Minimal websocket application</title>
            <script type="text/javascript" src="jquery.min.js"></script>
            <script type="text/javascript">
            $(function() {
                // Open up a connection to our server
                var ws = new WebSocket("ws://localhost:10000/");

                // What do we do when we get a message?
                ws.onmessage = function(evt) {
                    $("#placeholder").append('<p>' + evt.data + '</p>')
                }
                // Just update our conn_status field with the connection status
                ws.onopen = function(evt) {
                    $('#conn_status').html('<b>Connected</b>');
                }
                ws.onerror = function(evt) {
                    $('#conn_status').html('<b>Error</b>');
                }
                ws.onclose = function(evt) {
                    $('#conn_status').html('<b>Closed</b>');
                }
            });
        </script>
        </head>
        <body>
            <h1>WebSocket Example</h1>
            <div id="conn_status">Not Connected</div>
            <div id="placeholder" style="width:600px;height:300px;"></div>
        </body>
    </html>


## Chat Server

The final motivating example, a realtime chat room. This example
requires <a href="http://flask.pocoo.org/">Flask</a> ( but not necessarily so, you could use Django,
Pyramid, etc ). The corresponding Javascript and HTML files can
be found <a href="https://github.com/sdiehl/minichat">here</a>.


<pre>
<code class="python"># Micro gevent chatroom.
# ----------------------

from flask import Flask, render_template, request

from gevent import queue
from gevent.pywsgi import WSGIServer

import simplejson as json

app = Flask(__name__)
app.debug = True

rooms = {
    'topic1': Room(),
    'topic2': Room(),
}

users = {}

class Room(object):

    def __init__(self):
        self.users = set()
        self.messages = []

    def backlog(self, size=25):
        return self.messages[-size:]

    def subscribe(self, user):
        self.users.add(user)

    def add(self, message):
        for user in self.users:
            print(user)
            user.queue.put_nowait(message)
        self.messages.append(message)

class User(object):

    def __init__(self):
        self.queue = queue.Queue()

@app.route('/')
def choose_name():
    return render_template('choose.html')

@app.route('/&lt;uid&gt;')
def main(uid):
    return render_template('main.html',
        uid=uid,
        rooms=rooms.keys()
    )

@app.route('/&lt;room&gt;/&lt;uid&gt;')
def join(room, uid):
    user = users.get(uid, None)

    if not user:
        users[uid] = user = User()

    active_room = rooms[room]
    active_room.subscribe(user)
    print('subscribe %s %s' % (active_room, user))

    messages = active_room.backlog()

    return render_template('room.html',
        room=room, uid=uid, messages=messages)

@app.route("/put/&lt;room&gt;/&lt;uid&gt;", methods=["POST"])
def put(room, uid):
    user = users[uid]
    room = rooms[room]

    message = request.form['message']
    room.add(':'.join([uid, message]))

    return ''

@app.route("/poll/&lt;uid&gt;", methods=["POST"])
def poll(uid):
    try:
        msg = users[uid].queue.get(timeout=10)
    except queue.Empty:
        msg = []
    return json.dumps(msg)

if __name__ == "__main__":
    http = WSGIServer(('', 5000), app)
    http.serve_forever()
</code>
</pre>
