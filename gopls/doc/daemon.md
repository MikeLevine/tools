# Running gopls as a daemon

**EXPERIMENTAL - This feature is experimental, and under active development.
If you encounter bugs, please [file an issue](troubleshooting.md#file-an-issue).**

If you just want to try this out, skip ahead to the [quickstart](#quickstart).

## Background: gopls execution modes

Gopls was originally implemented as an LSP sidecar: a process started by
editors or editor plugins, and communicated with using jsonrpc 2.0 over
stdin/stdout. By executing as a stateful process, gopls can maintain a
significant amount of cache and can eagerly perform analysis on the source code
being edited.

This execution mode does not work as well when there are many separate editor
processes or when editor processes are short-lived, as is often the case for
users of non-IDE editors such as Vim or Emacs. Having many processes means
having many caches, consuming a significant amount of system resources. Using
short-lived sessions means paying a start-up cost each time a session is
created.

To support these types of workflows, a new mode of gopls execution is supported
wherein a single, persistent, shared gopls "daemon" process is responsible for
managing all gopls sessions. In this mode, editors still start a gopls sidecar,
but this sidecar merely acts as a thin "forwarder", responsible for forwarding
the LSP to the shared gopls instance and recording metrics, logs, and rpc
traces.

## Quickstart

To use a shared gopls instance you must either manage the daemon process
yourself, or let the gopls forwarder processes start the shared daemon as
needed.

### Running with `-remote=auto`

Automatic management of the daemon is easiest, and can be done by passing the
flag `-remote=auto` to the gopls process started by your editor. This will
cause this process to auto-start the gopls daemon if needed, connect to it, and
forward the LSP. For example, here is a reasonable gopls invocation, that sets
some additional flags for easier [debugging](#debugging):
```
$ gopls -remote=auto -logfile=auto -debug=:0 -rpc.trace
```

Note that the shared gopls process will automatically shut down after one
minute with no connected clients.

### Managing the daemon manually

To manage the gopls daemon process via external means rather than having the
forwarders manage it, you must start a gopls daemon process with the
`-listen=<addr>` flag, and then pass `-remote=<addr>` to the gopls processes
started by your editor.

For example, to host the daemon on the TCP port `37374`, do:
```
$ gopls -listen=:37374 -logfile=auto -debug=:0
```

And then from the editor, run
```
$ gopls -remote=:37374 -logfile=auto -debug=:0 -rpc.trace
```

If you are on a POSIX system, you can also use unix domain sockets by prefixing
the flag values with `unix;`. For example:
```
$ gopls -listen="unix;/tmp/gopls-daemon-socket" -logfile=auto -debug=:0
```
And connect via:
```
$ gopls -remote="unix;/tmp/gopls-daemon-socket" -logfile=auto -debug=:0 -rpc.trace
```

(Note that these flag values MUST be enclosed in quotes, because ';' is a
special shell character. For this reason, this syntax is subject to change in
the future.)

## Debugging

Debugging a shared gopls session is more complicated than a singleton session,
because there are now two gopls processes involved with handling the LSP. Here
are some tips:

### Traversing debug pages

When `-debug=:0` is passed to gopls, it runs a webserver that serves stateful
debug pages (see [troubleshooting.md](troubleshooting.md)).  You can find the
actual port hosting these pages by checking the start of the logfile, for
example`head /tmp/gopls-<pid>.log` if using `-logfile=auto`.

The debug pages of the forwarder process will have a link to the debug pages of
the daemon server process. Correspondingly, the debug pages of the daemon
process will have a link to each of its clients.

This can help you find metrics, traces, and log files for all of the various
servers and clients.

### Using logfiles

By default, the gopls daemon is not started with the `-rpc.trace` flag, so its
logfile will only contain actual debug logs from the gopls process.

It is recommended to start the forwarder gopls process with `-rpc.trace`, so
that its logfile will contain rpc trace logs specific to the LSP session.

## Using multiple shared gopls instances

There may be environments where it is desirable to have more than one shared
gopls instance. If managing the daemon manually, this can be done by simply
choosing different `-listen` addresses for each distinct daemon process.

On POSIX systems, there is also support for automatic management of distinct
shared gopls processes: distinct daemons can be selected by passing
`-remote="auto;<id>"`. Any gopls forwarder passing the same value for `<id>`
will use the same shared daemon.

## FAQ

(these questions are speculative -- as of writing nobody has asked any questions yet!)

**Q: Why am I not saving as much memory as I expected when using a shared gopls?**

A: As described in [implementation.md](implementation.md), gopls has a concept
of view/session/cache. Each session and view map onto exactly one editor
session (because they contain things like edited but unsaved buffers). The
cache contains things that are independent of any editor session, and can
therefore be shared.

When, for example, three editor session are sharing a single gopls process,
they will share the cache but will each have their own session and view. The
memory savings in this mode, when compared to three separate gopls processes,
corresponds to the amount of cache overlap across sessions.

Because this hasn't mattered much in the past, it is likely that there is state
that can be moved out of the session/view, and into the cache, thereby
increasing the amount of memory savings in the shared mode.

**Q: When using `-remote=auto`, why does my gopls daemon gets killed when I
close my first editor session?**

A: This is most likely a result of your editor sending a SIGTERM to the gopls
daemon. For example, this is how Vim's native job control behaves by default.
We are actively investigating workarounds for this. If it is causing you
problems, please consider managing the gopls daemon manually.
