# inotify-pubsub

Clients for an [inotify][inotify.7]-based event publish-subscribe
system.  Timestamped events are published to a directory, where
watching subscribers can notice them and return them to their callers.
This allows events to be distributed without requiring a long-running
broker process.  Only the filesystem must persist between calls, and
in the absence of long-running subscribers there may be no running
processes involved at all.

## Old-event recovery

To recover messages published before subscription, the reader lists
the event directory contents after the inotify watch is established
and lists past events in chronological order.  Once that historic
listing is complete, the reader begins processing new events as the
watcher reports their publication.

## Example

    termA$ echo red | ./inotify-pub /tmp/a color
    termA$ echo blue | ./inotify-pub /tmp/a color

    termB$ ./inotify-sub /tmp/a
    red
    blue
    …blocks…

    termA$ echo one | ./inotify-pub /tmp/a number

    …termB unblocks…
    one
    …termB blocks…

    termC$ ./inotify-sub /tmp/a color
    red
    blue
    …blocks…

    termA$ echo two | ./inotify-pub /tmp/a number

    …termB unblocks…
    two
    …termB blocks…

    termA$ echo green | ./inotify-pub /tmp/a color

    …termB unblocks…
    green
    …termB blocks…

    …termC unblocks…
    green
    …termC blocks…

## Races on writing

### Writer crash

The full write process is not atomic, and the writing process may
crash with the write partially completed.  Because the final
[`mv`][mv.1] operation is atomic, crashed processes will leave the
event file in the temporary directory (if they got far enough in to
create that file).

If the parent of the crashed writer survives, it can notice the
writer's non-zero exit code and republish.

### Filename collision

Parallel calls with the same event could grab the same timestamp,
leading to filename collisions.  With nanosecond timestamps, this has
a very low chance of happening, although with clock changes
(e.g. [NTP][] updates) it is possible.

If the original has not yet been moved from the temporary directory to
the event directory, the original content may be clobbered by the new
content.  This could be avoided by replacing the `cat >` approach with
one that used [`O_CREAT & O_EXCL`][open.3p] to fail if the target file
already existed.

If the original had already been moved from the temporary directory to
the event directory, the new file will silently fail to move, because
GNU Coreutils [`mv --no-clobber …`][mv.1] does not error out when the
target exists:

    $ touch a b
    $ mv --no-clobber a b
    $ echo $?
    0
    $ mv --version
    mv (GNU coreutils) 8.23
    …

### Time inversion

A slow writer may claim an early timestamp, but take longer than a
subsequent write to complete publication:

1. Writer A claims timestamp 1.
2. Writer B claims timestamp 2.
3. Writer B creates the event file and moves it to the event
   directory.
4. Writer A creates the event file and moves it to the event
   directory.

In this case, the reader may return the event published by writer B
before returning the event published by writer A, even though A has a
lower timestamp.

Even with atomic publication, there would almost certainly be
non-atomic code leading up to the publication invocation, so
desigining a system free of this problem is probably not possible.

For low-volume publishers, the risk of occurrence is low.  On my
laptop, `echo hi | ./inotify-pub /tmp event` takes around 8 ms, and
it's a shell script launching several sub-processes (so there's lots
of room for performance improvement).

### Races on reading

#### Reader crash

If the parent of the crashed reader survives, it can notice the
reader's non-zero exit code and resubscribe.

The resubscribed [old-event recovery](#old-event-recovery) will unwind
any [time inversions](#time-inversion) that may have been present in
the old events, ordering them by claimed timestamp while they were
initially ordered by move time.  If the parent records the timestamp
of the last received message, it can ignore older messages and pick up
where it left off.  This replay recovery is vulnerable to further time
inversions, so conservative parents should record all timestamps seen
in the last *s* seconds to protect against timestamp inversion of *s*
seconds or less.

Crashes during [old-event recovery](#old-event-recovery) may leave an
orphan FIFO.  A more sophisticated implementation (i.e. not using
POSIX shells) could use a pipe instead of the FIFO to avoid persisting
the handle to the filesystem.  The FIFO is also at risk for [filename
collision](#filename-collision)

##  Requirements

* A [POSIX shell][sh.1p].
* A POSIX-compliant [`cat`][cat.1p] (e.g. from [GNU
  Coreutils][coreutils]).
* [`date`][date.1] from [GNU Coreutils][coreutils] (POSIX [does not
  specify a way to get sub-second precision][date.1p]).
* A POSIX-compliant [`echo`][echo.1p] (e.g. from [GNU
  Coreutils][coreutils]).
* [`inotifywait`][inotifywait.1] from [inotify-tools][].
* A POSIX-compliant [`mkdir`][mkdir.1p] (e.g. from [GNU
  Coreutils][coreutils]).
* A POSIX-compliant [`mkfifo`][mkfifo.1p] (e.g. from [GNU
  Coreutils][coreutils]).
* [`mv`][mv.1] from [GNU Coreutils][coreutils] (POSIX [does not
  specify `--no-clobber`][mv.1p].)
* A POSIX-compliant [`printf`][printf.1p] (e.g. from [GNU
  Coreutils][coreutils]).
* A POSIX-compliant [`read`][read.1p] (e.g. from your [shell's
  built-ins][regular-built-ins]).
* A POSIX-compliant [`rm`][rm.1p] (e.g. from [GNU
  Coreutils][coreutils]).
* A POSIX-compliant [`sort`][sort.1p] (e.g. from [GNU
  Coreutils][coreutils]).
* A POSIX-compliant [`test`][test.1p] (e.g. from [GNU
  Coreutils][coreutils]).

[coreutils]: https://www.gnu.org/software/coreutils/coreutils.html
[inotify-tools]: https://github.com/rvoicilas/inotify-tools/wiki
[NTP]: http://www.ntp.org/

[execution-environment]: http://pubs.opengroup.org/onlinepubs/9699919799/utilities/V3_chap02.html#tag_18_12
[regular-built-ins]: http://pubs.opengroup.org/onlinepubs/9699919799/utilities/V3_chap02.html#tag_18_09_01_01

[cat.1p]: http://pubs.opengroup.org/onlinepubs/9699919799/utilities/cat.html
[date.1]: http://man7.org/linux/man-pages/man1/date.1.html
[date.1p]: http://pubs.opengroup.org/onlinepubs/9699919799/utilities/date.html
[echo.1p]: http://pubs.opengroup.org/onlinepubs/9699919799/utilities/echo.html
[inotifywait.1]: http://man7.org/linux/man-pages/man1/inotifywait.1.html
[mv.1]: http://man7.org/linux/man-pages/man1/inotifywait.1.html
[mkdir.1p]: http://pubs.opengroup.org/onlinepubs/9699919799/utilities/mkdir.html
[mkfifo.1p]: http://pubs.opengroup.org/onlinepubs/9699919799/utilities/mkfifo.html
[mv.1]: http://man7.org/linux/man-pages/man1/mv.1.html
[mv.1p]: http://pubs.opengroup.org/onlinepubs/9699919799/utilities/mv.html
[printf.1p]: http://pubs.opengroup.org/onlinepubs/9699919799/utilities/printf.html
[read.1p]: http://pubs.opengroup.org/onlinepubs/9699919799/utilities/read.html
[rm.1p]: http://pubs.opengroup.org/onlinepubs/9699919799/utilities/rm.html
[sh.1p]: http://pubs.opengroup.org/onlinepubs/9699919799/utilities/sh.html
[sort.1p]: http://pubs.opengroup.org/onlinepubs/9699919799/utilities/sort.html
[test.1p]: http://pubs.opengroup.org/onlinepubs/9699919799/utilities/test.html
[open.3p]: http://pubs.opengroup.org/onlinepubs/9699919799/functions/open.html
[inotify.7]: http://man7.org/linux/man-pages/man1/inotify.7.html
