# bashreduce : mapreduce in a bash script

bashreduce lets you apply your favorite unix tools in a mapreduce fashion across multiple machines/cores.  There's no installation, administration, or distributed filesystem.  You'll need:

* "br":http://github.com/erikfrey/bashreduce/blob/master/br somewhere handy in your path
* vanilla unix tools: sort, awk, ssh, netcat, pv
* password-less ssh to each machine you plan to use

# Configuration

Edit @~/.br.hosts@ and/or @/etc/br.hosts@ and enter the machines you wish to use as workers.  Or specify your machines at runtime:

```
br -m "host1 host2 host3"
```
To take advantage of multiple cores, repeat the host name.

# Examples

## sorting

```
br < input > output
```

## word count

```
br -r "uniq -c" < input > output
```

## great big join

```
LC_ALL='C' br -r "join - /tmp/join_data" < input > output
```

# Performance

## big honkin' local machine

Let's start with a simpler scenario: I have a machine with multiple cores and with normal unix tools I'm relegated to using just one core.  How does br help us here?  Here's br on an 8-core machine, essentially operating as a poor man's multi-core sort:

```

| command                                    | using     | time       | rate      |
| sort -k1,1 -S2G 4gb_file > 4gb_file_sorted | coreutils | 30m32.078s | 2.24 MBps |
| br -i 4gb_file -o 4gb_file_sorted          | coreutils | 11m3.111s  | 6.18 MBps |
| br -i 4gb_file -o 4gb_file_sorted          | brp/brm   | 7m13.695s  | 9.44 MBps |

```
The job completely i/o saturates, but still a reasonable gain!

## many cheap machines

Here lies the promise of mapreduce: rather than use my big honkin' machine, I have a bunch of cheaper machines lying around that I can distribute my work to.  How does br behave when I add four cheaper 4-core machines into the mix?

```

|   command                                  |   using   |   time      |   rate     |
| sort -k1,1 -S2G 4gb_file > 4gb_file_sorted | coreutils | 30m32.078s  |  2.24 MBps |
| br -i 4gb_file -o 4gb_file_sorted          | coreutils |  8m30.652s  |  8.02 MBps |
| br -i 4gb_file -o 4gb_file_sorted          | brp/brm   |  4m 7.596s  | 16.54 MBps |

```

We have a new bottleneck: we're limited by how quickly we can partition/pump our dataset out to the nodes.  awk and sort begin to show their limitations (our clever awk script is a bit cpu bound, and @sort -m@ can only merge so many files at once).  So we use two little helper programs written in C (yes, I know!  it's cheating!  if you can think of a better partition/merge using core unix tools, contact me) to partition the data and merge it back.

## Future work

I've tested this on ubuntu/debian, but not on other distros.  According to Daniel Einspanjer, netcat has different parameters on Redhat.

br has a poor man's dfs like so:

```
br -r "cat > /tmp/myfile" < input
```

But this breaks if you specify the same host multiple times.  Maybe some kind of very basic virtualization is in order.  Maybe.

Other niceties would be to more closely mimic the options presented in sort (numeric, reverse, etc).