# chillscope
##### A script to intelligently suspend backgrounded X11 applications

This script optionally runs a child program in its own `systemd-run --user`
scope, then suspends (freezes) and resumes (thaws) the scope subject to
certain conditions; it can also move unrelated grandchildren out of the
scope cgroup.

Hence the name: it chills a scope. Ahahaha! How original! I know, right?

See near the bottom for a usage example.

### About the certain conditions:

* The scope is thawed if one of its member processes receives the input focus on the current X display.
* For the first `--gracetime seconds`, the scope won't be suspended but is still subject to renicing (see below).
* The scope is frozen if it hasn't had the input focus for more than `--suspendafter` seconds (if set).
  * The scope is not frozen while it has at least `--trafficthreshold` bytes/second of incoming or outgoing traffic in total, if set; thus, `chillscope` won't suspend a browser that's downloading something in the background.
    * This requires the ability to run `sudo --non-interactive pktstat-bpf`, and relies on parsing that tool's output, so it could be fragile.
    * Traffic sampling time is currently hard-coded to be 2s.
* The scope is thawed for `--resumefor` seconds every `--resumeevery` seconds, if both are set.
  * If `--trafficthreshold` is set, it's checked before re-freezing.
* Before freezing the scope, the script enumerates the processes that are in it, and moves any whose name (comm and/or cmdline) doesn't match the regex passed via the `--procname` switch from the child scope to the script's own cgroup.
  * TODO: perhaps support moving to a different cgroup.
* The script will exit if the controlled scope becomes empty or disappears.
* In addition to freezing the cgroup, the script can also nice and un-nice the cgroup members when the cgroup loses or gains focus (see `--nice_bg` and `--nice_fg`).
  * Increasing niceness is only possible up to `20-$(ulimit -e nice)`, the "max nice" rlimit (see `getrlimit(2)`).
  * Note: without configuration in `/etc/security/limits*`, `rlimit_nice` will generally be 0, which means the lowest attainable nice value is 20. Setting either `--nice_bg` or `--nice_fg` in this situation without `--sudo_nice` will effectively set the nice value to 19, always.
  * If `--sudo_nice` is passed on the command line, the script will use `sudo --non-interactive renice` instead of `renice`.

If `--cgroup /sys/fs/cgroup/path/to/cgroup` is specified, and the cgroup
exists, chillscope manages that cgroup (regardless of its type, and whether
it was created by systemd) and doesn't spawn a child process.

### The parameters can be set and changed at runtime as follows:

* via inherited environment variables
* via command line switches like `--suspendafter` (these take precedence over inherited envvars)
* via files in a configuration directory:
  * the name of the directory is passed in via the `--confdir` switch;
  * in this directory, you can have files named for the command line switches; e.g. `suspendafter`, `nice_fg` etc.
  * the script will periodically read the values in these files, allowing you to reconfigure it at runtime without restarting it.
  * (The `loopdelay` tunable is only exposed via the configuration directory; it sets the maximum time between runs of the main loop. There should normally be no reason to change the default; see comments in the script.)

### Further notes:

* -p/--property switches are passed to systemd-run unchanged and can be mixed in with regular arguments.
* The script will only work correctly with unified (v2) cgroups.
* Changing the `--procname` regex at runtime will only apply to future decisions; processes already moved out of the monitored cgroup won't be moved back even if their name matches the new regex.

## Motivation

I was frustrated by the fact that many "modern" applications are actually
browsers, which, due to their complexity, will often run CPU intensive
subtasks with no obvious benefit to the user.

One such application is Ferdium, which, while very useful, will, after an
hour or so of runtime, start chugging away merrily, constantly executing
some JavaScript, consuming 250% CPU on my laptop; this would then cause the
CPU scaling governor to increase the CPU frequency, which would increase
power usage and temperature, and cause the fans to spin up.

(Incidentally, the `ondemand` governor has an `ignore_nice_load` tunable; so
if you run Ferdium at a nice level of 1, it'll still hog the CPU but at
least the governor won't turn up the frequency for Ferdium alone.)

I have found VSCodium (strictly speaking, some of its plugins) to be another
frequent offender. There is no reason why I should tolerate it using 80% CPU
on a core when it's not in the foreground. (It's aggravating enough when
it _is_ in the foreground.)

While Firefox isn't quite as bad, some pages can also contain obnoxious
JavaScript, or some CPU intensive animation, which you don't want to expend
CPU cycles on when you're not using the browser.

In general, fat, so-called "native" chat clients (which are actually mostly
browsers) are good candidates for management by `chillscope`: they'll get a
chance to run and receive notifications every few minutes, but won't be able
to hog your CPU if they don't have focus.

## Dependencies

* Linux with cgroupv2 mounted under `/sys/fs/cgroup`
* X11 (functionality under Wayland would be limited to XWayland applications, IIUC)
* zsh
* xprop, xwininfo
* access to /proc
* For spawning a child cgroup via `systemd-run`, a running `systemd --user` instance, and `systemd-run` itself.
* For the traffic-based decision on whether to suspend, `sudo` and `pktstat-bpf` from https://github.com/dkorunic/pktstat-bpf.
* For effective renicing, `sudo` or an appropriate `rlimit_nice` value.

## Limitations

* Remote X displays untested; may not work.
* If a GUI terminal process is removed from the monitored scope, but the processes running in it are not, those will be suspended, but never resumed due to the terminal receiving focus.
* If there is a high rate of focus changes, CPU usage can be higher than ideal.
* Pasting stuff copied from a suspended process doesn't work.
  * Install a clipboard manager like copyq as a workaround.
* The renicing is inherently, unfixably racy, because the cgroup membership can change while the renice is in progress.
* Applications that minimize to the system tray can't receive focus while they have no window, and they can't create a window while they're suspended. `chillscope` will suspend them, but you won't be able to resume them by switching to them.
  * Workaround: configure them to minimize to the taskbar; or find some way of writing a 0 into the `cgroup.freeze` of their cgroup when you want to resume them.
* The script makes no effort to work out if a niceness adjustment would be an increase or decrease; however, it does try to make sure `renice` will succeed, so it only sets values possible under the current `rlimit_nice`.
  * So while it would be possible to renice a process with a nice value of e.g. 0 to e.g. 5 even with the default `rlimit_nice`, the script won't do this because it would only succeed if the starting nice value were less than 5, and it doesn't want to deal with that.
    * Instead, it will treat the `--nice_bg` value of 5 as 20, because that's what the default `rlimit_nice` of 0 allows.
  * Since there are two valid ways (the rlimit and `sudo_nice`) to make sure you get the nice values you actually want, I don't see this as a problem.

## Further ideas, currently unimplemented

* Limit resource usage (CPUWeight, CPUQuota, AllowedCPUs, IOWeight) when no member process of the scope has input focus, perhaps as an alternative to suspending.
  * ionice could also be increased, but lowering it would need root. Same for switching to a different CPU scheduler (e.g. SCHED_IDLE).
    * We do already use sudo for `pktstat-bpf` and optionally for renicing, though.
  * Many resouce management settings are only available for services, not scopes. TODO: evaluate switching from --scope to a service unit.
* Add a mode where we don't start a child, but manage several existing cgroups, not just one, each with its own configuration. Yay for feeping creaturism! Would this be worth it? You can just run multiple instances of the script, one per cgroup.

## Examples

### Basic: run Firefox, but suspend it when it's in the background

```
chillscope \
	--gracetime 300 \
	--nice_bg 19 \
	--nice_fg 0 \
	--sudo_nice \
	--suspendafter 10 \
	--trafficthreshold 50000 \
	--resumeevery 180 \
	--resumefor 5 \
	--unit firefox \
	-- firefox
```

### Complex: run Ferdium, creating overrideable runtime configuration

We'll also manage an existing Ferdium cgroup if it already exists.

```
#!/bin/zsh
mkdir -p ~/.config/chillscope/ferdium
pushd ~/.config/chillscope/ferdium
# Create a configuration for the user to override:
for key value in \
	debug 0 \
	loglevel info \
	nice_bg 19 \
	nice_fg 1 \
	procname ferdium \
	resumefor 5 \
	resumeevery 60 \
	suspendafter 10 \
	trafficthreshold 50000; do
	[[ -f $key ]] || echo $value >$key
done
popd
cd /
exec chillscope \
	--cgroup /sys/fs/cgroup/user.slice/user-$UID.slice/user@$UID.service/app.slice/ferdium.scope \
	--confdir ~/.config/chillscope/ferdium \
	--gracetime 180 \
	--unit ferdium \
	--logfile ~/.xsession-errors \
	--sudo_nice \
	-- \
	stdbuf -oL -eL /usr/bin/ferdium 2>&1 \
	| stdbuf -oL \
		sed 's/^/ferdium: /' \
	>>~/.xsession-errors &!
```

## Author, licence

Copyright (c) 2025 András Korn; licensed under the GPLv3 (see LICENSE).
