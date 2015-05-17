# growrepo

A framework for using `germinate` and `reprepro` to create partial mirrors of Debian packages.

For a full description on how to use these scripts, go to the tutorial on [setting up the ev3dev
build ecosystem][ev3dev-ecosystem].

This is a set of notes that are helpful to get started using the [`germinate`] and [`reprepro`]
packages in your development environment. We assume that you are comfortable with:

1. Basic Linux command line usage
1. Installing Linux software from packages
1. Setting up a [VirtualBox][VirtualBox] VM
1. Setting up a web server

## Set Up A Local Package Repository

Believe me when I say you'll be happy for the improved virtual machine image build
times if you set up a local package repository now. Fortunately this is nowhere
near as complex as it sounds because of two tools that you can install on your
host machine - `germinate` and `reprepro`.

The [`germinate`][germinate] tool takes a list of the top level packages that you want
to have on a disk image and generates a list of all the required dependencies.
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

- The official Debian Jessie package repository (i386 amd64)
- The official Ubuntu Trusty package repository (i386 amd64)
- The offical Trusty Updates and Security repos (amd64)
- The ev3dev specific package repository (amd64 armel)
- The third party repos for other packages (i386 amd64 armel)

These repositories are accessed through different URLs when
using `apt-get`, so I've added the concept of _farms_ to
the metaphor. Each farm stores a `conf` file that is used to
tell `germinate` where it should try to get the package dependency
files that it needs, and where on our local machine the partial
repository will be created.

The whole thing is tied together by a script called `grow_crop` - it takes 
the name of a farm (which matches the name of the seed directory)
and creates output in a matching folder in the `crops` directory. You
can delete the `crops` folder any time you like, the `grow_crop` script will
regenerate the `crops` file structure as needed.

I've provided an assortment of individual scripts that are used to build the
package lists and populate the partial repositories - but when you have a lot
of farms to manage, that gets tiresome. There is also a script that
can be run to update ALL of the repositories, it takes a while but you don't
need to run it that often.

### Generating the seeds for ev3dev

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

Currently, I only use the `required`, `minimal`, and `standard` seed files.

Each of the files in the specific `seeds` folder contains a list of packages
that looks something like this:

```
 * apt
 * conspy
 * dosfstools
...
```

Note the ` * ` at the beginning of each line - in particular the space in
front of the asterisk! 

By convention, the `standard` package list is all the files required to
pull together a working installation - and for `ev3dev` that's the
`packages` list. Where do we get this list of packages? For `ev3dev` it's from
the [`packages` folder in `ev3dev-jessie` in the `brickstrap` package][brickstrap-ev3dev-jessie].

Assuming that I have the `brickstrap` and `growrepo` folders at the same
location, and that I'm currently in `growrepo` folder, I do something like this
to create the package lists:

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

One more step here - we need to create a package list for all of the
`essential` and `required` packages in the main `Packages` file, there's
a little script to handle that called `find_seeds` that is used like 
this:

```
./find_seeds ev3dev.debian.jessie.armel
```

You only need to do it for main distribution repositories.

## Set Up The reprepro System

Now we can focus on setting up [`reprepro`][reprepro] which is really not
very difficult at all. Recall that when we set up the `germinate`
ecosystem, we used the idea of "farms" to grow the package lists. We will
re-use this idea and give the local repositories their own directories - and
the directory name will be the same as the name of the farm.

```
sudo apt-get --no-install-recommends install reprepro
```

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

### Create Folder Structure For Each Local Respository 

For `ev3dev` development, we're going to create a bunch of repositories.
Some are for the `ev3dev.org` packages that we'll assemble and put on
the SD card for the EV3, and the rest are for setting up the ev3dev 
developmnent machine. The name of the repository is specified in the
corresponding farm's `conf` file.

These repositories will live in the `/srv/reprepro` folder, and to
extend the farm metaphor just a bit further, we're going to call
the repository a "silo".

There's a script to make a silo, and it warns you if the silo already
exists. The script must be run as `reprepro`, like this:

```
sudo -u reprepro make_silo ev3dev.ev3dev.jessie.armel
sudo -u reprepro make_silo ev3dev.debian.jessie.armel
```

This will manually create the silo we need, if you have a lot of silos
to manage the `farm_all` script that comes later is your friend.

The main thing to remember is that the `farms/proj.dist.rel.arch/conf`
file is where you set up the silo-specific values that may need to be
customized. The most likely candidates for customization are:

- `MIRROR` for settig the location of the repo you want to mirror
- `REPO_GPG_KEY` that repositories signing key, so you can verify it
- `SIGN_GPG_KEY` your private signing key

### Your Private Signing Key

The `apt` system that Debian based systems rely on to install packages
has built-in security that relies on `gpg` encryption. The `Release` 
file is generally signed with a private signing key and is saved
as `Release.gpg`. That lets `apt` verify that the files from this
repository are trusted.

Of course, there's a bit of work to do to create a master key and
subkeys for signing, encryption, and authentication. And then you
need to put the public part of the keyset up on a keyserver.

Rather than document this here, I'll [refer you to the excellent 
guide here][gpgKeyGuide].

Once you have your keyset generated, there is one step to do
that is particular to this process. You need to export your 
private keys and import them into the `reprepro` user's keychain.

```
gpg --armour --export-secret-keys | sudo -u reprepro gpg --import
```

Check that it worked like this example:

```
sudo -u reprepro gpg -K

/srv/reprepro/.gnupg/secring.gpg
--------------------------------
sec#  4096R/FA852339 2014-08-06 [expires: 2019-08-05]
uid                  Ralph Hempel <rhempel@hempeldesigngroup.com>
ssb   4096R/E7249370 2014-08-07
ssb   4096R/0148B07D 2014-08-07
ssb   4096R/79A9DE19 2014-08-07
```

Don't worry, I'm not giving anything away here - in fact you can grab
my public keys anytime like this:

```
gpg --keyserver "pgp.mit.edu" --recv-keys FA852339
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
sudo -u reprepro ./harvest_crop ev3dev.ev3dev.jessie.arm required minimal standard
sudo -u reprepro ./harvest_crop ev3dev.debian.jessie.arm required minimal standard
```

### Working Smart, Not Hard

These steps are not too onerous, but if you do them frequently they
get boring, and if you do them infrequently you'll have to reread this
tutorial.

So in the interest of avoiding both problems, I created `farm_all` - a
scripts that buzzes through the `farms` directory and does the following:

- Make a new silo for the farm, if it does not already exist
- Wipe out the previous crop
- Grow a new crop
- Harvest the new crop and update the packages in the silo
- Sign the silo if it has changed

Run it like this as a regular user, you will be prompted for your password
as some of the scripts will `sudo -u reprepro` as needed.

```
./farm-all
```

If a repo changes at all, you'll be promted to re-sign it, so that's a bit
of manual labour, but the rest is pretty much automated.

If you want to grab an updated package list from each of your source repos, then
just run

```
./farm-all clean
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
    	WebsiteRoot = /srv/reprepro/debian.jessie.armel/www
    	ShowIndex = yes
     	AccessLogfile = /var/log/hiawatha/debian-access.log
     	ErrorLogfile = /var/log/hiawatha/debian-error.log
    	FollowSymlinks = yes
}
    
VirtualHost {
    	Hostname = ev3dev.jessie.armel.domain.com
    	WebsiteRoot = /srv/reprepro/ev3dev.jessie.armel/www
    	ShowIndex = yes
     	AccessLogfile = /var/log/hiawatha/ev3dev-access.log
     	ErrorLogfile = /var/log/hiawatha/ev3dev-error.log
    	FollowSymlinks = yes
}

...
```

No magic here - you'll notice of course that the `farm` name
shows up again in the `Hostname` definition, and without the
project name in the `WebsiteRoot`. That's in case you have 
multiple projects that use the same repository mirror.

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

On your ev3dev development machine, you'll need to set up `/etc/hosts` to
have the local DNS settings for all your repositories. Fortunately, this
is all taken care of automatically if you use my [pxeBoot scripts][pxeBoot].

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
[pxeBoot]: <https://github.com/rhempel/pxeBoot>
[reprepro]: <http://mirrorer.alioth.debian.org>
[ev3dev-ecosystem]: <http://www.ev3dev.org/docs/devtools/setting-up-the-ev3dev-build-ecosystem/>

[gpgKeyGuide]: <http://spin.atomicobject.com/2013/11/24/secure-gpg-keys-guide>

[Hiawatha]: <https://www.hiawatha-webserver.org/>
[VirtualBox]: <https://www.virtualbox.org/>
[brickstrap-ev3dev-jessie]: <https://github.com/ev3dev/brickstrap/tree/ev3dev/ev3dev-jessie>
