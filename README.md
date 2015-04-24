# growrepo

A framework for using `germinate` and `reprepro` to create partial mirrors of Debian packages.

For a full description on how to use these scripts, go to the tutorial on [setting up the ev3dev
build ecosystem][ev3dev-ecosystem].

[ev3dev-ecosystem]: <http://www.ev3dev.org/docs/devtools/setting-up-the-ev3dev-build-ecosystem/>

This is a set of notes that are helpful to get started using the [`germinate`] and [`reprepro`]
packages in your development environment. We assume that you are comfortable with:

1. Basic Linux command line usage
1. Installing Linux software from packages
1. Setting up a [VirtualBox][VirtualBox] VM
1. Setting up a web server
1. Setting up `nfs` to share folders

## Set Up A Local Package Repository

Believe me when I say you'll be happy for the improved virtual machine image build
times if you set up a local package repository now. Fortunately this is nowhere
near as complex as it sounds because of two tools that you can install on your
host machine - `germinate` and `reprepro`.

The [`germinate`][germinate] tool takes a list of the top level packages that you want
to have on the SD card image and generates a list of all the required dependencies.
We'll massage the output of `germinate` to create a package list that can
be used to set up the local repositories using `reprepro`. Let's start with
the `germinate` installation.

```
sudo apt-get --no-install-recommends install germinate
```

### Seeds and Crops and Packages

If there's one thing I get a kick out of, it's a good metaphor. The idea of
`germinate` is to provide a list of top level _seed_ packages from which
we can create a list of all the other packages that are required for an
install of the seed packages. The goal is to have a local mirror of everything
you need to set up your virtual machines, which is going to be a small
subset of all the available Debian packages.

I've extended the metaphor a bit further because I actually need to create
a variety of package lists that satisfy different target requirements, such as:

- The official Debian package repository (i386 amd64)
- The ev3dev specific package repository (amd64 armel)
- The third party repos for other packages (i386 amd64 armel)

These repositories are accessed through different URLs when
using `apt-get`, so I've added the concept of _farms_ to
the metaphor. Each farm stores a `conf` file that is used to
tell `germinate` where it should try to get the package dependency
files that it needs.

The whole thing is tied together by a script called `grow_crop` - it takes 
the name of a farm (which matches the name of the seed directory)
and creates output in a matching folder in the `crops` directory. You
can delete the `crops` folder any time you like, the `grow_crop` script will
regenerate the `crops` file structure as needed.

The easiest way to get the folders all set up for `ev3dev` is to grab the
latest release from the [`growrepo`][growrepo] repository on GitHub - it
contains some of the `seeds` and `farms` that I created. You can use them
as examples for your own partial repositories.

Each of the matching folders within `seeds` has a file called `STRUCTURE` that
specifies how all the other files are related. I'm sticking with the default
`STRUCTURE` file contents:

```
required:
minimal: required
standard: required minimal
custom:
blacklist:
supported:
```

Each of the files in the specific `seeds` folder contains a list of packages
that looks something like this:

```
 * apt
 * conspy
 * dosfstools
...
```

Note the ` * ` at the beginning of each line - in particular the space in
front of the asterix! 

By convention, the `standard` package list is all the files required to
pull together a minimal working installation - and for `ev3dev` that's the
`packages` list. Where do we get this list of packages? For `ev3dev` it's from
the [`packages` folder in `ev3dev-jessie` in the `brickstrap` package][brickstrap-ev3dev-jessie].

Assuming that I have the `brickstrap` and `growrepo` folders at the same
location, and that I'm currently in `growrepo` folder, I do something like this
to create the package lists:

```
cat ../brickstrap/ev3dev-jessie/packages/base        \
    | sed 's/^/ * /' > ./seeds/ev3dev.ev3dev.jessie.armel/required

cat ../brickstrap/ev3dev-jessie/packages/audio       \
    ../brickstrap/ev3dev-jessie/packages/filesystems \
    ../brickstrap/ev3dev-jessie/packages/firmware    \
    ../brickstrap/ev3dev-jessie/packages/graphics    \
    ../brickstrap/ev3dev-jessie/packages/network     \
    ../brickstrap/ev3dev-jessie/packages/utils       \
    | sed 's/^/ * /' > ./seeds/ev3dev.ev3dev.jessie.armel/minimal

cat ../brickstrap/ev3dev-jessie/packages/programming-languages \
    | sed 's/^/ * /' > ./seeds/ev3dev.ev3dev.jessie.armel/standard
```

We could have been a little less precise and just put everything into standard, like this:

```
cat ../brickstrap/ev3dev-jessie/packages/* \
    | sed 's/^/ * /' > ./seeds/ev3dev.ev3dev.jessie.armel/standard
```

To grow the crops that we'll use to populate our local partial
package mirror, we do the following:

```
./grow_crop ev3dev.ev3dev.jessie.armel
```

This will create a bunch of output in the `crops/ev3dev.ev3dev.jessie.armel`
folder. These files will contain formatted output that contains all the packages you'll
need for each crop in `required`, and incrementally more packages in `minimal`
and `standard`.

The one we're most interested in is `crops/ev3dev.ev3dev.jessie.armel/all` - it's
got all of the packages that are available from the mirror specified in
`farms/ev3dev.ev3dev.jessie.armel/conf`.

We'll need the same package list files for the `ev3dev.debian.jessie.armel` farm. When
`grow_crop` is executed on that farm, we'll get the list of packages that are required
to build the `ev3dev` root file system from the official Debian repositories.

Rather than duplicate the process above (and possibly forget to do it later) it's
much easier to simply create a soft link and run the same script as before, but
using the seeds in `ev3dev.debian.jessie.armel`:

```
cd seeds
ln -s ev3dev.ev3dev.jessie.armel ev3dev.debian.jessie.armel
cd ..

./grow_crop ev3dev.debian.jessie.armel
```

## Set Up The reprepro System

Now we can focus on setting up [`reprepro`][reprepro] which is really not
very difficult at all. Recall that when we set up the `germinate`
ecosystem, we used the idea of "farms" to grow the package lists. We will
re-use this idea and give the local repositories their own directories - and
the directory name will be the same as the name of the farm.

Using the Debian conventions, the top level repository mirror folders will
be stored in the `/srv/reprepro/` directory. We don't strictly need it
now, but for future automation of `reprepro` you're going to want to create
a `reprepro` system user and add yourself to the group like this:

```
sudo adduser --system --disabled-password --disabled-login \
             --home /srv/reprepro \
             --group reprepro
sudo adduser username reprepro
```

You'll need to log out and log back in for the group changes to take effect - sorry.

For the following sections on setting up `reprepro`, you'll need to become
the `reprepro` user, but that user was configured with no login privileges.
We can do some `sudo` magic to work around that - note that your shell
prompt (if you're using the default that comes with most Linux distributions)
will have changed:

```
userid@machine:~$ sudo -u reprepro bash
reprepro@machine:/home/userid$ cd $HOME
reprepro@machine:~$ pwd
/srv/reprepro
reprepro@machine:~$ exit
userid@machine:~$
```

### Create Folder Structure For Each Local Respository 

For `ev3dev` development, we're going to create two repositories, one for 
the `ev3dev.org` packages and another for the standard Debian packages.
You can guess that to make things easy, the repository names will be
exactly the same as the `farms` that we grow the `crops` in.

Remember to become the `reprepro` user and then create the following
directories:

```
for d in "conf" "gpg" "logs" "www"; do
    mkdir -p "/srv/reprepro/ev3dev.debian.jessie.armel/$d"
    mkdir -p "/srv/reprepro/ev3dev.ev3dev.jessie.armel/$d"
done

chmod 775 /srv/reprepro/ev3dev.debian.jessie.armel
chmod 775 /srv/reprepro/ev3dev.ev3dev.jessie.armel

chmod 775 /srv/reprepro/ev3dev.debian.jessie.armel/logs
chmod 775 /srv/reprepro/ev3dev.ev3dev.jessie.armel/logs

chmod 775 /srv/reprepro/ev3dev.debian.jessie.armel/www
chmod 775 /srv/reprepro/ev3dev.ev3dev.jessie.armel/www

chmod 700 /srv/reprepro/ev3dev.debian.jessie.armel/gpg
chmod 700 /srv/reprepro/ev3dev.ev3dev.jessie.armel/gpg

```

Note that the `conf` directories are not group writeable, but `logs` and `www`
are - that's to avoid accidentally clobbering the configuration as a plain 
user - but you can always regenerate the `www` contents.

Now create the file `conf/options` like this"

```
cat <<EOF > "/srv/reprepro/ev3dev.debian.jessie.armel/conf/options"
outdir +b/www
logdir +b/logs
gnupghome +b/gpg
EOF
    
cat <<EOF > "/srv/reprepro/ev3dev.ev3dev.jessie.armel/conf/options"
outdir +b/www
logdir +b/logs
gnupghome +b/gpg
EOF
```

and set up the ev3dev repository `conf` files:

```
cat <<EOF > "/srv/reprepro/ev3dev.ev3dev.jessie.armel/conf/distributions"
Codename: jessie
Architectures: armel
Description: ev3dev (required packages only)
Components: main
Update: ev3dev-jessie-update
EOF
    
cat <<EOF > "/srv/reprepro/ev3dev.ev3dev.jessie.armel/conf/options"
outdir +b/www
logdir +b/logs
gnupghome +b/gpg
EOF

cat <<EOF > "/srv/reprepro/ev3dev.ev3dev.jessie.armel/conf/updates"
Name:  ev3dev-jessie-update
Method: http://ev3dev.org/debian
VerifyRelease: 93178A7C
Suite: jessie
Components: main
Architectures: armel
FilterList: purge ../packages
EOF
```

I'm using the University of Waterloo Debian mirror locations here, you
should use the closest mirror:

```
cat <<EOF > "/srv/reprepro/ev3dev.debian.jessie.armel/conf/distributions"
Codename: jessie
Architectures: armel
Description: debian (required packages only)
Components: main contrib non-free
Update: debian-jessie-update
EOF

cat <<EOF > "/srv/reprepro/ev3dev.debian.jessie.armel/conf/options"
outdir +b/www
logdir +b/logs
gnupghome +b/gpg
EOF

cat <<EOF > "/srv/reprepro/ev3dev.debian.jessie.armel/conf/updates"
Name:  debian-jessie-update
Method: http://mirror.csclub.uwaterloo.ca/debian
VerifyRelease: 8B48AD6246925553
Suite: jessie
Components: main contrib non-free
Architectures: armel
FilterList: purge ../packages
EOF
```

### Get the GPG Keys For Each Repository

We are (hopefully) still the `reprepro` user - remember that the `$HOME`
for this user is `/srv/reprepro` and that's where the `.gnupg` directory
will get created. Don't worry, only the `reprepro` user has access to
it.

Pulling in and saving the keys for each repository is simple:

```
pushd .
cd /srv/reprepro

# Get the ev3dev.org public signing key
    
gpg2 --keyserver pgp.mit.edu --recv-keys 2B210565
cd /srv/reprepro/ev3dev.ev3dev.jessie.armel
gpg2 --export 2B210565 | GNUPGHOME=gpg gpg --import --no-permission-warning
    
# Get the Debian Jessie public signing key
    
gpg2 --keyserver pgp.mit.edu --recv-keys 8B48AD6246925553
cd /srv/reprepro/ev3dev.debian.jessie.armel
gpg2 --export 8B48AD6246925553 | GNUPGHOME=gpg gpg --import --no-permission-warning

popd
```

That's it, now we can go back to being ourselves, just exit the current shell:

```
exit
```

### Harvesting Crops and Populating a Local Mirror

We're almost ready - all we need to do now is create the package
lists and populate the repositories. We've got the `harvest_crop` script
to do all the work for us.

I was hoping this would work as a normal user, but it turns out that
if you ever want to automate things, it's easier to always run the
`harvest_crops` script as the `reprepro` user.

Assuming we're in the `growrepo` script directory

```
sudo -u reprepro bash

./harvest_crop ev3dev.ev3dev.jessie.arm required minimal standard
./harvest_crop ev3dev.debian.jessie.arm required minimal standard

exit
```

## Configuring Your Webserver To Serve Packages

As discussed near the beginning of this tutorial, I happen to like
using Hiawatha for serving up content. We need to configure Hiawatha
to create virtual hosts - this allows the server to associate a URL
with a particular source directory.

Whatever web server you use, the steps will be similar to these.

I've got Hiawatha set up to listen to the following adresses:

- `127.0.0.1:8080` localhost
- `192.168.56.1:8080` VirtualBox HostOnly

For Hiawatha, that looks like the following configuration stanzas:

```
Binding {
    	Port = 8080
}
```
    
Next, we need to configure the virtual host URLs, and associate
the root directory for the content. These stanzas handle it:

```
VirtualHost {
	Hostname = debian.jessie.armel.domain.com
    	WebsiteRoot = /srv/reprepro/ev3dev.debian.jessie.armel/www
    	ShowIndex = yes
     	AccessLogfile = /var/log/hiawatha/debian-access.log
     	ErrorLogfile = /var/log/hiawatha/debian-error.log
    	FollowSymlinks = yes
}
    
VirtualHost {
    	Hostname = ev3dev.jessie.armel.domain.com
    	WebsiteRoot = /srv/reprepro/ev3dev.ev3dev.jessie.armel/www
    	ShowIndex = yes
     	AccessLogfile = /var/log/hiawatha/ev3dev-access.log
     	ErrorLogfile = /var/log/hiawatha/ev3dev-error.log
    	FollowSymlinks = yes
}
```

No magic here - you'll notice of course that the `farm` name
shows up again in the `Hostname` definition, and in the `WebsiteRoot`. 
Remember to chage `domain.com` to your local domain name.

The last step is to set up your local DNS server to direct the
`Hostname` URLs to one of the specific IP addresses that Hiawatha is bound
to.

If you don't have a full blown DNS server, have no fear. The old-school
local DNS file under Linux is `/etc/hosts`, just add these two stanzas
to the `/etc/hosts` file:

```
127.0.0.1	debian.jessie.armel.domain.com
127.0.0.1	ev3dev.jessie.armel.domain.com
```

Of course, substitute your actual domain name for `domain.com`.

Point your webserver at `debian.jessie.armel.domain.com:8080` and you
should see a directory listing with `dists/` and `pool/` in it.

## Conclusion

If you've followed along this far and had success, then give yourself
a pat on the back and go for a walk and enjoy the outdoors. You have
set a partial repository of the main Debian repository, built the `ev3dev`
compatible Linux kernel using a cross compiler, and possibly build the
SD card image for `ev3dev`.

[ev3dev]: <http://www.ev3dev.org>
[ev3dev-buildscripts]: <https://github.com/ev3dev/ev3dev-buildscripts>
[brickstrap]: <https://github.com/ev3dev/brickstrap>
[germinate]: <http://manpages.ubuntu.com/manpages/utopic/en/man1/germinate.1.html>
[growrepo]: <https://github.com/rhempel/growrepo>
[reprepro]: <http://mirrorer.alioth.debian.org>

[Hiawatha]: <https://www.hiawatha-webserver.org/>
[VirtualBox]: <https://www.virtualbox.org/>
[brickstrap-ev3dev-jessie]: <https://github.com/ev3dev/brickstrap/tree/ev3dev/ev3dev-jessie>
