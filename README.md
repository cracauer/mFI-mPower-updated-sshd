# mFI-mPower-updated-sshd
The Ubinquity Network's powerswitches have old firmware with an old sshd. This is how to update

This is a new ssh daemon that you can run on MFi switches.

The problem with the original one is that it has been compiled with
algorithms for host key exchange and the cipher that OpenSSH 7.0
rejects as unsafe.  Debian Buster now defaults to ssh 7.x, so the heat
is on.

I provide both, sshd binaries and also instructions on how to build
those binaries yourself, making the appropriate cross-compilation
chain yourself, with my config files.  Once this is up you can build
more binaries to use on the MFi.  Note, however, that at this point I
compile static binaries, because the default uLibc used by the chain
below is too new to make dynamic executables work with the older uLibc
on the MFi.  More later.

# Information about the system

Host information:
- Atheros AR9330 CPU
- mips 32 bit, run as big endinan
- linux kernel 2.6.32.29
- most of the Unix commandline env is busybox v1.11.2
- the ssh daemon is standalone (not integrated into busybox, not a
  symlink to a shared binary)
- ssh daemon is dropbear 0.51
- linked against uClibc-0.9.29

Most of the filesystem is readonly, except for /var, which is a
tmpfs, containing:
- /var/run
- /var/tmp
- /var/lock
- /var/etc, /etc is symlinked to this
- /var/etc/persistent
- /.ssh/ -> /var/tmp
- $HOME is set to /etc/persistent, which has a .ssh:
  /etc/persistent/.ssh
  $ (cd /.ssh ; /bin/pwd) ; (cd $HOME/.ssh ; /bin/pwd)
  /var/tmp
  /var/etc/persistent/.ssh

Obviously it means the sshd/dropbear binary provided is readonly, as
is the directory it is in (can't make it a symlink).

Note that you put your authorized_keys file in /etc/persistent/.ssh
which is not the same thing as /.ssh

Although this means you have write permissions to /etc, you cannot use
this plainly, as any use of the web interface to change configuration
will wipe out your changes.

/var/etc/persistent holds a set of files the user can write, but
nothing directly concerned with startup.  Google instructions on mfi
ssh configuration to see what's there.

I will talk about how to make the new sshd the default later.  For now
I am providing a procedure to replace the running sshd after boot.
The new one runs on a different port, so while I'm slacking off you
can also firewall off port 22 really hard and use 2399 instead.

I would be very thankful is people gave me feedback up these
procedures on their particular system.  If I get a script to replace
(as opposed to add) the sshd wrong, then users would be shut out of
their system until a factory reset.  That would make me unpopular with
those who continue to use these things in the first place.


# building and installing

Building:
- I am using buildroot-2017.11.2.
  https://buildroot.org/
- My result of `make menuconfig` is in git: 
  buildroot_dot.config
- dropbear-2017.75

Building dropbear:
  buildroot={whereever_your_buildroot_untar_is}
  export CC=$buildroot/output/host/bin/mips-buildroot-linux-uclibc-cc
  export LDFLAGS=-static
  ./configure --host=mips --disable-zlib --disable-syslog
  make
  cp -p dropbear dropbear_static
  scp -oKexAlgorithms=+diffie-hellman-group1-sha1 \
      -c +aes256-cbc -p ./dropbear_static mfi1:/var/run

You can, but I did not, modify things in options.h.  I use the
commandline later.  You get a bunch of warnings about xauth missing
etc that you can turn off by disabling the feature in options.h

# Configuration on the target system

# luckily we can do this:
mkdir /etc/dropbear

# As for host keys:
# - you must make and install a new ecdsa key
# - dss and rsa you can either reuse what is there or made new
#   - reusing what is there has the advantage that you won't have
#     existing clients with stored ~/.ssh/knows_hosts complain
#   - reusing has the disadvantage that you don't know how
#     carefully the old ones have been created

# here is how you make all of them on a real computer:
for type in rsa dss ecdsa ; do
    ./dropbearkey -t $type -f dropbear_${type}_host_key > dropbear_${type}_host_key.pub
done

# you will need to make and use a new ecdsa key.  The other two you
# can decide whether you want to re-use what's there (might already be
# in a gazillion client ~/.ssh/known_hosts but on the other hand who
# knows how these have been generated?).
for type in rsa dss ecdsa ; do
    ./dropbearkey -t $type -f dropbear_${type}_host_key > dropbear_${type}_host_key.pub
done

# now either just copy 
scp /etc/dropbear/dropbear_ecdsa_host_key mfi1:/etc/dropbear/.
# from the host and
  # do this if you want to reuse the old keys
    ln -s /var/run/dropbear_[dr]*_host_key /etc/dropbear/.
  # or do this to copy all new keys
    scp /etc/dropbear/dropbear_ecdsa_host_key mfi1:/etc/dropbear/.

chmod 600 /etc/dropbear/dropbear_*_host_key

# start the daemon, interactive first, errors to stderr

# note that we do not need any -r options because all our host keys
# are in the default location /etc/dropbear
/var/run/dropbear_static -F -E -m -j -k -p 2399 -P /var/run/dropbear2399.pid &

# "production version":
# please note that you do not have working syslog on these things.  If
# you want a log then I would keep the -E and -F options and do this:
/var/run/dropbear_static -F -E -m -j -k -p 2399 \
  -P /var/run/dropbear2399.pid 2>&1 | tee /run/run/dropbear.log &

# Obviously the small filesystem might fill up quickly.  I did a quick
# test, this thing does not crash and burn when the filesystem is
# full, however I don't know what happens on reboot and I get the web
# interface cannot write a new config when the filesystem is full.
