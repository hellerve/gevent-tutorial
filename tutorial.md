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

<pre>
<code class="python">
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
</code>
</pre>

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

Leider kommen wir nun zu den dunklen Ecken von Gevent. Ich habe es vermieden,
Monkeypatching bis jetzt zu erwähnen, um die mächtigen Koroutinen zu betonen,
aber die Zeit ist gekommen die dunklen Künste des Monkeypatching zu erklären.
Wir haben oben bereits das Kommando ``monkey.patch_socket()`` ausgeführt.
Dies ist ein reines Seiteneffekt-Kommando um die Socket-Programme der 
Standard-Bibliothek zu modifizieren.

<pre>
<code class="python">import socket
print(socket.socket)

print("Nach dem monkey patch")
from gevent import monkey
monkey.patch_socket()
print(socket.socket)

import select
print(select.select)
monkey.patch_select()
print("Nach dem monkey patch")
print(select.select)
</code>
</pre>

<pre>
<code class="python">class 'socket.socket'
Nach dem monkey patch
class 'gevent.socket.socket'

built-in function select
Nach dem monkey patch
function select at 0x1924de8
</code>
</pre>

Pythons Runtime erlaubt es, dass die meisten Objekte zur Laufzeit
modifiziert werden, auch Module, Klassen und sogar Funktionen.
Das ist normalerweise eine unglaublich schlechte Idee, da es zu einem
"impliziten Seiteneffekt" führt, der meistens extrem schwer zu
debuggen ist, falls Probleme auftreten; nichtsdestotroz können 
Monkey Patches in extremen Situationen benutzt werden, in denen
eine Bibliothek das fundamentale Verhalten von Python selbst
verändern muss. In diesem Fall ist gevent in der Lage, die meisten
blockierenden System Calls in der Standardbibliothek so zu patchen,
dass sie stattdessen kooperativ arbeiten,
einschliesslich der Module in  ``socket``, ``ssl``, ``threading`` und
``select``.

Zum Beispiel nutzt die Redis-Anbindung an Python reguläre TCP-Sockets,
um mit der ``redis-server``-Instanz zu kommunizieren. Nur durch den
Aufruf von ``gevent.monkey.patch_all()`` können wir die Redis-Anbindung
dazu bringen, Anfragen kooperativ zu behandeln und mit dem Rest unseres
gevent-Überbaus zu interagieren.

Dies lässt und Bibliotheken integrieren, die normalerweise nicht mit
gevent arbeiten würden, ohne jemals eine einzige Zeile Code schreiben
zu müssen. Obwohl Monkey Patching immer noch schlecht ist, ist es
in diesem Fall ein "nützliches Übel".

# Daten-Strukturen

## Events

Events sind eine Form der asynchronen Kommunikation zwischen
Greenlets.

<pre>
<code class="python">import gevent
from gevent.event import Event

'''
Illustriert den Nutzen von Events
'''


evt = Event()

def setter():
    '''Nach 3 Sekunden werden alle Threads die auf den Wert von evt warten
       aufgeweckt'''
	print('A: Hey, warte auf mich, ich muss etwas besorgen')
	gevent.sleep(3)
	print("Ok, ich bin fertig")
	evt.set()


def waiter():
	'''Nach 3 Sekundenwird der get-Aufruf entblockt'''
	print("Ich werde auf dich warten")
	evt.wait()  # blockierend
	print("Wird ja auch Zeit")

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

Eine Erweiterun des Event-Objekts ist ein AsyncResult, welches
es erlaubt, zusammen mit dem Weckruf einen Wert zu versenden.
Dies wird manchmal future oder deferred genannt, da es eine
Referenz auf einen zukünftigen Wert hält, der zu einem frei
wählbaren Zeitpunkt gesetzt werden kann.

<pre>
<code class="python">import gevent
from gevent.event import AsyncResult
a = AsyncResult()

def setter():
    """
    Setze das Ergebnis von a nach 3 Sekunden.
    """
    gevent.sleep(3)
    a.set('Hallo!')

def waiter():
    """
    Nach 3 Sekunden wird der get call entblockt, nachdem der Setter
    einen Wert in das AsyncResult schreibt.
    """
    print(a.get())

gevent.joinall([
    gevent.spawn(setter),
    gevent.spawn(waiter),
])

</code>
</pre>

## Queues

Queues sind geoordnete Datensets, die die üblichen ``put``/``get``
Operationen unterstützt, aber auf eine solche Weise implementiert
sind, dass sie sicher zwischen Greenlets manipuliert werden können.

Zum Beispiel wird bei simultanem Zugriff zweier Greenlets auf
ein Item der Queue nicht zweimal das selbe Item herausgenommen.

[[[cog
import gevent
from gevent.queue import Queue

tasks = Queue()

def worker(n):
    while not tasks.empty():
        task = tasks.get()
        print('Arbeiter %s hat Task %s bekommen' % (n, task))
        gevent.sleep(0)

    print('Ende!')

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

Queues können auch ``put`` und ``get``-Operationen blockieren,
falls es nötig wird.

Jede der ``put`` und ``get``-Operationen hat einen nicht-blockierenden
Gegensata, ``put_nowait`` und ``get_nowait``, welche anstatt zu blockieren
entweder ``gevent.queue.Empty`` oder ``gevent.queue.Full`` zurückgeben,
falls die Operation nicht möglich ist.

In diesem beispiel läuft der Boss zur gleichen Zeit wie die Arbeiter
und auf der Queue liegt eine Restriktion, die verhindert, dass darauf
mehr als drei Elemente liegen. Diese Restriktion bedeutet, dass die ``put``
Operation blockiert bis kein Platz mehr in der Queue ist. Umgekehrt blockiert
die ``get``-Operation, falls keine Elemente mehr in der Queue sind und
nimmt ausserdem ein Timeout-Argument, das es erlaubt, dass die Queue 
mit der Exception ``gevent.queue.Empty`` beendet wird, falls innerhalb
der Zeitspanne des Timeouts keine Arbeit mehr gefunden wird.

[[[cog
import gevent
from gevent.queue import Queue, Empty

tasks = Queue(maxsize=3)

def worker(name):
    try:
        while True:
            task = tasks.get(timeout=1) # decrements queue size by 1
            print('Arbeiter %s hat Task %s bekommen' % (name, task))
            gevent.sleep(0)
    except Empty:
        print('Ende!')

def boss():
    """
    Der Boss wartet mit dem Austeilen der Arbeit, bis ein individueller
    Arbeiter frei ist, da die Maximalgrösse der Task-Queue 3 ist.
    """

    for i in xrange(1,10):
        tasks.put(i)
    print('Alle Arbeit in Iteration 1 ausgeteilt')

    for i in xrange(10,20):
        tasks.put(i)
    print('Alle Arbeit in Iteration 2 ausgeteilt')

gevent.joinall([
    gevent.spawn(boss),
    gevent.spawn(worker, 'steve'),
    gevent.spawn(worker, 'john'),
    gevent.spawn(worker, 'bob'),
])
]]]
[[[end]]]

## Groups und Pools

Eine Group ist eine Sammlung laufender Greenlets, die zusammen
als Gruppe verwaltet und geleitet werden. Es dient ausserdem
als paralleler Dispatcher, der die Python ``multiprocessing``-Bibliothek
spiegelt.

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

Das ist sehr nützlich, um Gruppen asynchroner Aufgaben zu verwalten.

Wie oben erwähnt, hat ``Group`` auch eine API, um Jobs an gruppierte
Greenlets auszuliefern und deren Resultate wieder einzusammeln; all
dies auf mehreren verschiedenen Wegen.

[[[cog
import gevent
from gevent import getcurrent
from gevent.pool import Group

group = Group()

def hello_from(n):
    print('Ausmass der Gruppe: %s' % len(group))
    print('Hallo von Greenlet %s' % id(getcurrent()))

group.map(hello_from, xrange(3))


def intensive(n):
    gevent.sleep(3 - n)
    return 'task', n

print('Geordnet')

ogroup = Group()
for i in ogroup.imap(intensive, xrange(3)):
    print(i)

print('Ungeordnet')

igroup = Group()
for i in igroup.imap_unordered(intensive, xrange(3)):
    print(i)

]]]
[[[end]]]

Ein Pool ist eine Struktur, die dafür entwickelt wurde, eine
dynamische Anzahl von Greenlets, die in ihrer Nebenläufigkeit 
limitiert werden müssen, zu handhaben. Dies ist oftmals gewünscht,
wenn viele Netzwerk- oder IO-basierte Aufgaben parallel bearbeitet
werden sollen.

[[[cog
import gevent
from gevent.pool import Pool

pool = Pool(2)

def hello_from(n):
    print('Ausmass des Pools %s' % len(pool))

pool.map(hello_from, xrange(3))
]]]
[[[end]]]

Oftmals wird beim Bau eines gevent-basierten Services der gesamte
Service um eine Pool-Struktur herum gebaut. Ein Beispiel könnte
eine Klasse sein, die auf verschiedenen Sockets arbeitet.

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
            raise Exception("Pool hat Maximum erreicht")
        else:
            self.pool.spawn(self.listen, socket)

    def shutdown(self):
        self.pool.kill()

</code>
</pre>

## Locks und Semaphoren

Eine Semaphore ist eine Low-Level Synchronisations-Primitive, die
es Greenlets ermöglicht, gleichzeitige Zugriffe und Ausführung
zu koordinieren und zu limitieren. Eine Semaphore hat zwei Methoden,
``acquire`` und ``release``. Die Maximalanzahl an gleichzeitigen
acquires nennt sich die Grenze der Semaphore. Falls die Grenze
der Semaphore 0 erreicht(alle freien Plätze besetzt sind), blockiert
sie bis ein Greenlet seinen Platz wieder freigibt.

[[[cog
from gevent import sleep
from gevent.pool import Pool
from gevent.coros import BoundedSemaphore

sem = BoundedSemaphore(2)

def worker1(n):
    sem.acquire()
    print('Arbeiter %i akquiriert Semaphore' % n)
    sleep(0)
    sem.release()
    print('Arbeiter %i git Semaphore frei' % n)

def worker2(n):
    with sem:
        print('Arbeiter %i akquiriert Semaphore' % n)
        sleep(0)
    print('Arbeiter %i gibt Semaphore frei' % n)

pool = Pool()
pool.map(worker1, xrange(0,2))
pool.map(worker2, xrange(3,6))
]]]
[[[end]]]

Eine Semaphore mit Grenze 1 wird Lock genannt. Dieses lässt
die exklusive Ausführung eines einzelnen Greenlets zu. Sie
werden oft benutzt, um sicherzustellen, dass Resourcen nur
von einer Stelle im Kontext des Programms gleichzeitig 
benutzt werden.

## Thread Locals

Gevent erlaubt es auch, den Greenlets Daten zur Verfügung zu stellen,
die lokal für dessen Kontext verfügbar ist. Intern ist dies durch ein
globales Nachlagen eines privaten Namespaces implementiert, wobei
der Wert von ``getcurrent()`` des Greenlets der Schlüssel ist.

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
        print("x ist nicht lokal in f2")

g1 = gevent.spawn(f1)
g2 = gevent.spawn(f2)

gevent.joinall([g1, g2])
]]]
[[[end]]]

Viele Web-Frameworks, die gevent benutzen, speichern
HTTP-Session-Objekte in gevent Thread Locals. Zum Beispiel
können wir unter Benutzung der Werkzeug-Bibliothek und
dessen Proxy-Objekt Request-Objekte im Stile des Flask-Frameworks
nachbauen.

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
    return "Hallo " + request.remote_addr

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

Flasks System ist ein bisschen ausgeklügelter als dieses Beispiel, aber
die Idee, Thread Locals als lokalen Session-Speicher zu verwenden ist
trotzdem der gleiche.

## Subprocess

Seit gevent 1.0 ist ``gevent.subprocess`` - eine gepatchte Version von
Pythons ``subprocess``-Modul - in gevent verfügbar. Es unterstützt kooperatives
Warten auf Subprozesse.

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

Einige Benutzer wollen ``gevent`` und ``multiprocessing`` zusammen verwenden.
Eine der offensichtlichen Herausforderungen hierbei ist, dass 
Interprozesskommunikation in ``multiprocessing`` nicht standardmässig 
kooperativ ist. Da  ``multiprozessing.Connection``-basierte Objekte (wie ``Pipe``)
ihre ihnen zugrundeliegenen Dateideskriptoren offenlegen, können
``gevent.socket.wait_read`` und ``wait_write`` benutzt werden, um kooperativ
auf ready-to-read/ready-to-write-Events zu warten, bevor tatsächlich 
gelesen/geschrieben wird:

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

Man sollte sich jedoch klarmachen, dass die Kombination von ``multiprocessing``
und gevent einige betriebssystemspezifische Fallstricke mit sich bringt,
unter anderem:


* Nach dem [Forken](http://linux.die.net/man/2/fork) ist gevents Zustand im Child
schlecht gestellt. Ein Nebeneffekt ist, dass Greenlets, die vor dem Erstellen
von ``multiprocessing.Process`` hervorgebracht werden in beiden Prozessen
laufen, Parent und Child.
* ``a.send()`` in ``put_msg`` oben könnte den aufrufenden Thread immer noch
nicht kooperativ blockieren: ein ready-to-write-Event versichert nur, dass
ein Byte geschrieben werden kann. Der zugrundeliegende Puffer könnte voll sein,
bevor der versuchte Write beendet wurde.
* Der Ansatz, der auf ``wait_write()``/``wait_read()`` basiert, so wie es oben
verwendet wurde, funktioniert nicht auf Windows(``IOError: 3 is not a 
socket (files are not supported)``), da Windows Pipes nicht auf Events hin 
beobachten kann.

Das Python-Paket [gipc](http://pypi.python.org/pypi/gipc) überwindet diese
Herausforderungen für den Anwender auf eine grössenteils transparente Weise,
sowohl auf POSIX- als auch auf Windows-Systemen. Es liefert gevent-kompatible
``multiprocessing.Process``-basierte Kindprozesse und gevent-kooperative
Interprozesskommunikation basierend auf Pipes.

## Actors

Das Aktoren-Modell ist ein High-Level-Nebenläufigkeitsmodell, das
von der Sprache Erlang popularisiert wurde. Kurz gesagt ist die Hauptidee,
dass man über eine Ansammlung unabhängiger Actors verfügt, die eine
Inbox besitzen, aus welcher sie Nachrichten von anderen Actors bekommen.
Die Hauptschleife im Actor iteriert durch seine Nachrichten und agiert
auf Basis seines gewünschten Verhaltens.

Gevent hat keinen primitiven Actor-Typen, jedoch können wir sehr
einfach einen definieren, indem wir eine Queue innerhalb einer
Greenlet-Subklasse verwenden.

<pre>
<code class="python">import gevent
from gevent.queue import Queue


class Actor(gevent.Greenlet):

    def __init__(self):
        self.inbox = Queue()
        Greenlet.__init__(self)

    def receive(self, message):
        """
        Dies muss in jeder Subklasse anders definiert werden.
        """
        raise NotImplemented()

    def _run(self):
        self.running = True

        while self.running:
            message = self.inbox.get()
            self.receive(message)

</code>
</pre>

Ein Anwendungsfall:

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

# Anwendungen aus der echten Welt

## Gevent ZeroMQ

[ZeroMQ](http://www.zeromq.org/) wird von dessen Autoren als
"eine Socket-Bibliothek, die als ein Nebenläufigkeitsframework
agiert" beschrieben. Es stellt eine sehr mächtige Messaging-Schicht
bereit, die es erlaubt, nebenläufige und verteilte Anwendungen zu
erstellen.

ZeroMQ bietet eine Vielfalt an Socket-Primitiven an, die simpelste
davon ist ein Request-Response-Socket-Paar. Ein Socket hat
zwei interessante Methoden namens ``send`` and ``recv``, beide
davon normale blockierende Operationen. Dies jedoch wird durch
eine geniale Bibliothek von [Travis Cline](https://github.com/traviscline) 
in Ordnung gebracht, welche gevent.socket benutzt, um ZeroMQ-Sockets
auf nicht-blockierende Weise abzufragen. Um dies auszunutzen,
muss lediglich die Python-Anbindung an ZeroMQ via ``pip install pyzmq``
installiert werden.

(Anmerkung des Übersetzers: Diese Bibliothek wurde inzwischen
in den Kern des PyZeroMQ-Projekts eingebettet. Installationshinweise
für gevent-zeromq sind daher hinfällig)

[[[cog
# Anmerkung: Zuvor muss ``pip install pyzmq`` ausgeführt werden
import gevent
import zmq.green as zmq

# Globaler Kontext
context = zmq.Context()

def server():
    server_socket = context.socket(zmq.REQ)
    server_socket.bind("tcp://127.0.0.1:5000")

    for request in range(1,10):
        server_socket.send("Hallo")
        print('Zum Server gewechselt, um %s zu bearbeiten' % request)
        # IHier passiert ein impliziter Kontextwechsel
        server_socket.recv()

def client():
    client_socket = context.socket(zmq.REP)
    client_socket.connect("tcp://127.0.0.1:5000")

    for request in range(1,10):

        client_socket.recv()
        print('Zum Client gewechselt, um %s zu bearbeiten' % request)
        # Implicit context switch occurs here
        client_socket.send("Welt")

publisher = gevent.spawn(server)
client    = gevent.spawn(client)

gevent.joinall([publisher, client])

]]]
[[[end]]]

## Ein simpler Server

<pre>
<code class="python">
# Unter Unix: Zugang mit ``$ nc 127.0.0.1 5000``
# Unter Window: Zugang mit ``$ telnet 127.0.0.1 5000``

from gevent.server import StreamServer

def handle(socket, address):
    socket.send("Hallo von telnet!\n")
    for i in range(5):
        socket.send(str(i) + '\n')
    socket.close()

server = StreamServer(('127.0.0.1', 5000), handle)
server.serve_forever()
</code>
</pre>

## WSGI-Server

Gevent liefert zwei WSGI-Server mit, um Inhalte über HTTP anzubieten,
fortan nennen wir sie ``wsgi`` und ``pywsgi``:

* gevent.wsgi.WSGIServer
* gevent.pywsgi.WSGIServer

In früheren Versionen von gevent(vor 1.0.x) benutzte gevent libevent
anstelle von libev. Libevent schloss einen HTTP-Server mit ein,
welcher von gevents ``wsgi``-Server benutzt wurde.

In gevent 1.0.x gibt es keinen HTTP-Server mehr. Stattdessen
ist ``gevent.wsgi`` nun ein Alias für den reinen Python-Server
in ``gevent.pywsgi``.


## Streaming-Server

**Diese Sektion ist unter gevent 1.0.x nicht anwendbar.**

Für diejenigen Benutzer, die HTTP-Streaming-Services kennen: die
Kernidee ist, dass wir im Header nicht die Länge des Inhalts
spezifizieren. Anstelle dessen halten wir die Verbindung offen
und schicken Teile durch die Pipe, während wir jedem Teil
eine hexadezimalen Zahl voranstellen, die die Länge des Teils
repräsentiert. Der Stream wird geschlossen, sobald ein Teil
von der Grösse 0 gesendet wird.

    HTTP/1.1 200 OK
    Content-Type: text/plain
    Transfer-Encoding: chunked

    8
    <p>Hallo

    9
    Welt!</p>

    0

Die obige HTTP-Verbindung kann nicht in wsgi kreiert werden,
da Streaming nicht unterstützt wird. Anstelle dessen würde es
gepuffert.

<pre>
<code class="python">from gevent.wsgi import WSGIServer

def application(environ, start_response):
    status = '200 OK'
    body = '&lt;p&gt;Hallo Welt!&lt;/p&gt;'

    headers = [
        ('Content-Type', 'text/html')
    ]

    start_response(status, headers)
    return [body]

WSGIServer(('', 8000), application).serve_forever()

</code>
</pre>

Unter Verwendung von pywsgi können wir jedoch unseren Handler
als Generator schreiben und das Ergebnis Teil für Teil versenden.

<pre>
<code class="python">from gevent.pywsgi import WSGIServer

def application(environ, start_response):
    status = '200 OK'

    headers = [
        ('Content-Type', 'text/html')
    ]

    start_response(status, headers)
    yield "&lt;p&gt;Hallo"
    yield "Welt!&lt;/p&gt;"

WSGIServer(('', 8000), application).serve_forever()

</code>
</pre>

Trotzdem ist die Performance von Gevent-Servern phänomenal
im Vergleich zu anderen Python-Servern. Libev ist eine sehr
gut untersuchte Technologie und Server, die sie benutzen,
sind bekannt dafür, dass sie gut skalieren.

Als Massstab kann der geneigte Leser zum Beispiel das
Apache Benchmark ``ab`` verwenden oder diesen
[Vergleich von Python-WSGI-Servern](http://nichol.as/benchmark-of-python-web-servers)
für einen Vergleich mit anderen Servern konsultieren.

<pre>
<code class="shell">$ ab -n 10000 -c 100 http://127.0.0.1:8000/
</code>
</pre>

## Langes Polling

<pre>
<code class="python">import gevent
from gevent.queue import Queue, Empty
from gevent.pywsgi import WSGIServer
import simplejson as json

data_source = Queue()

def producer():
    while True:
        data_source.put_nowait('Hallo Welt')
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

Dies ist ein Beispiel für Websockets, welches
<a href="https://bitbucket.org/Jeffrey/gevent-websocket/src">gevent-websocket</a>
benötigt.

<pre>
<code class="python"># Simpler gevent-websocket server
import json
import random

from gevent import pywsgi, sleep
from geventwebsocket.handler import WebSocketHandler

class WebSocketApp(object):
    '''Sendet Zufallsdaten an den Websocket'''

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

Die dazugehörige HTML-Seite sieht wie folgt aus::

    <html>
        <head>
            <title>Minimale Websocket-Anwendung</title>
            <script type="text/javascript" src="jquery.min.js"></script>
            <script type="text/javascript">
            $(function() {
                // Öffnet eine Verbindung zu unserem Server
                var ws = new WebSocket("ws://localhost:10000/");

                // Was tun wir, wenn wir eine Nachricht erhalten?
                ws.onmessage = function(evt) {
                    $("#placeholder").append('<p>' + evt.data + '</p>')
                }
                // Wir aktualisieren unser conn_status-Feld mit dem Verbindungs-Status
                ws.onopen = function(evt) {
                    $('#conn_status').html('<b>Verbunden</b>');
                }
                ws.onerror = function(evt) {
                    $('#conn_status').html('<b>Fehler</b>');
                }
                ws.onclose = function(evt) {
                    $('#conn_status').html('<b>Geschlossen</b>');
                }
            });
        </script>
        </head>
        <body>
            <h1>WebSocket-Beispiel</h1>
            <div id="conn_status">Nicht verbunden</div>
            <div id="placeholder" style="width:600px;height:300px;"></div>
        </body>
    </html>


## Chat-Server

Das letzte Beispiel sei ein Echtzeit-Chat-Raum. Dieses Beispiel benötigt
<a href="http://flask.pocoo.org/">Flask</a> ( aber nicht notwendigerweise, verwendbar wären auch Django,
Pyramid, usw.). Die entsprechenden Javascript und HTML-Dateien sind <a href="https://github.com/sdiehl/minichat">hier</a>
zu finden.

<pre>
<code class="python"># Mikro-gevent-Chatraum.
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
