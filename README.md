# RPM Packaging Lab

## Preface

For this lab you will need to have a pretty good familiarity with using SSH and the PASE/Unix command line. You will be editing text files and running PASE commands to build rpm packages. This will simulate porting an existing open source package which is available on Linux and other operating systems already, but may need some modification to work within the PASE environment.

As noted above, we will be editing files in the IFS. For the lab, you can use whatever means you are most comfortable with for this. `nano` is installed if you wish to run from within the terminal for everything. Otherwise, you can edit locally with your favorite text editor and use SFTP or similar to send the files. Again, this is something that you should already be comfortable with.

## Step 0: Getting connected

This lab will be conducted using SSH to log in to the lab system. No access will be provided via 5250. You have been provided the hostname, username, and password to connect. The lab machines will have PuTTY installed on the desktop or you can use your own device, so long as it has an SSH terminal.

*Make sure that your PATH is set up correctly:*

```shell
PATH=/QOpenSys/pkgs/bin:$PATH
```

You can also do the lab on your own IBM i system, so long as you have the Yum bootstrap and the following packages installed on it:

```shell
yum install curl rpm-devel rpm-build gcc-aix gzip make-gnu tar-gnu patch-gnu coreutils-gnu
```

## Step 1: First steps

The first thing we need to do is create a spec file. The convention for the file name is `<package>.spec`. For the first part, we will just be making a dummy package, so create a file called `dummy.spec`. We will add the following contents to it:

```specfile
Name: dummy
Version: 1.0
Release: 0
License: MIT
Summary: Dummy package to test building rpms
Url: https://example.com/project-home-page

%description
This a longer description of the dummy project.

It's got multiple lines and goes in to more detail, well
if it wasn't just a dummy package it would. :)

%files
```

Now we have a basic spec file which we can build with rpmbuild, but first we need to set up a set of directories where rpmbuild will find things. Run the following command to create the directory structure:

```shell
mkdir -p ~/rpmbuild/{BUILD,BUILDROOT,RPMS,SRPMS,SOURCES}
```

You can now copy your spec file in to `~/rpmbuild/SOURCES` and run rpmbuild:

```shell
cp dummy.spec ~/rpmbuild/SOURCES

mkdir ~/rpmbuild/BUILDROOT/br

rpmbuild -ba ~/rpmbuild/SOURCES/dummy.spec --buildroot ~/rpmbuild/BUILDROOT/br
```

If all goes well, you should see output like the following:

```text
Wrote: /home/yum/rpmbuild/SRPMS/dummy-1.0-0.src.rpm
Wrote: /home/yum/rpmbuild/RPMS/ppc64/dummy-1.0-0.ibmi7.3.ppc64.rpm
```

You have just built your first RPM package! You can query it it with `rpm -qp ~/rpmbuild/RPMS/ppc64/dummy-1.0-0.ibmi7.3.ppc64.rpm`. You can add additional options to find more info: `--info`, `--list`, `--requires`, and `--provides`.

## Step 2: Spec file walkthrough

In the previous step, we built a dummy package to make sure things were set up and get our feet wet. From now on, we'll build a real package. For this we'll be building a relatively simple open source project called `libwordcount`, which as its name might suggest is a library for counting words in text. You can find it here: [https://github.com/kadler/libwordcount](https://github.com/kadler/libwordcount)

Let's make a new spec file for `libwordcount` called `libwordcount.spec` in `~/rpmbuild/SOURCES`:

```specfile
Name: libwordcount
Version: 1.0
Release: 0
License: MIT
Summary: Word count library
Url: https://github.com/kadler/libwordcount

%description
libwordcount is a library to count the occurrences of a given word in chunks of text, even occurrences split across chunks. This allows you to process large text files in a streaming fashion without having to read the whole thing in to memory.

A sample application is included, which counts the word "butter".

%files
```

Before we continue, let's go through some explanations of spec files. Spec files are split in to multiple sections. The first section is the header. This is the only section which doesn't start with a section macro. This section contains metadata tags about the package:

```specfile
Name: libwordcount
Version: 1.0
Release: 0
License: MIT
Summary: Word count library
Url: https://github.com/kadler/libwordcount
```

Tags are words (generally capitalized), followed by a `:` and a value. At a minimum, a spec file needs a `Name`, `Version`, `Release`, `License`, and `Summary` tags as well as a `%description` section. Though the `Url` tag is not required, it is heavily suggested.

- `Name` - package name. This will determine the output of the rpm name (not the spec filename), and generally matches the project name
- `Version` - package version. This can usually be taken straight from the project itself, though in rare circumstances you may have to do some special processing on this to match how rpm understands versioning.
- `Release` - spec version. This should be updated any time you do an rpm build and is used to denote updates to a package when the project version hasn't been updated, like when adding new build options or patching bugs. (this can be reset to 0 when updating `Version`)
- `License` - the project license. Usually something like `MIT`, `BSD`, `GPLv3`, etc. Check `COPYING`, `LICENSE`, etc to determine what the package license is.
- `Summary` - a short one-sentence description of the project. It should not end in a period.
- `Url` - this should usually point to the project's "home page". If they just have a git repository, point at the main repository page.

After the header, comes the `%description` section. The `%description` section contains a longer, more descriptive description of the project and is is plain-text only. Like every section, the description ends when the next section begins.

Finally we have the `%files` section. This section will list all the files, directories, documentation, etc... that are owned by the package. In our example so far, we have not shipped any files. NOTE: if you do not include this section, `rpmbuild` will not produce an RPM file, but only a source RPM.

## Step 3: Building a real package

Ok, so you should now have a template for a real spec file for `libwordcount`. Before we can build it, we need to add a few more things to the spec file. First, we need to add a source using the `Source` tag. This tag should go after the `Url` tag in the header. Looking on the GitHub page, under the Releases tab you should see the tar.gz listed for version 1.0. We can simply add the URL in to the spec:

```specfile
Name: libwordcount
Version: 1.0
Release: 0
License: MIT
Summary: Word count library
Url: https://github.com/kadler/libwordcount

Source: https://github.com/kadler/libwordcount/releases/download/1.0/wordcount-1.0.tar.gz
```

Let's try to build the package now and see what happens:

```shell
mkdir ~/rpmbuild/BUILDROOT/br

rpmbuild -ba ~/rpmbuild/SOURCES/libwordcount.spec --buildroot ~/rpmbuild/BUILDROOT/br
```

Oops, an error:

```text
    Bad file: /home/yum/rpmbuild/SOURCES/wordcount-1.0.tar.gz: No such file or directory
```

While you can give a URL in the `Source` tag, `rpmbuild` will not automatically download the source for you. There are other tools which do, though, and it's good for documentation to list where you got the source file. If there is no publicly accessible download URL you can just list a file name without a path/URL and document the steps to download the source in the spec file. For now, we're just going to download the tar file ourselves so that `rpmbuild` can find it. Regardless of whether you specify a URL or file name, the base file must be in the `~/rpmbuild/SOURCES` directory for `rpmbuild` to find it. You can download it like so:

```shell
pushd ~/rpmbuild/SOURCES

curl -OLk https://github.com/kadler/libwordcount/releases/download/1.0/wordcount-1.0.tar.gz

popd
```

After downloading the tarball, re-run the `rpmbuild` and it should succeed now, though the rpm is still pretty much a dummy.

## Step 4: Macros

You may have noticed various words prefixed with `%`: `%description`, `%files`, etc. These are RPM macros. Macros can be used like basic variables as well as functions. There are many built-in macros that we will use throughout this lab. In this step we're going to tidy up the spec file and reduce the redundancy.

When we set the `Name` and `Version` tags in the spec header, RPM uses these values to automatically define two macros: `%name` and `%version`. Let's use these to adjust our download URL:

```specfile
Source: https://github.com/kadler/%{name}/releases/download/%{version}/wordcount-%{version}.tar.gz
```

## Step 5: Unpacking

The first step to building any package is almost always to unpack the source. RPM spec files have a special section for this: `%prep`. Let's add a `%prep` section after the `%description`:

```specfile
Name: libwordcount
Version: 1.0
Release: 0
License: MIT
Summary: Word count library
Url: https://github.com/kadler/libwordcount

Source: https://github.com/kadler/libwordcount/releases/download/1.0/wordcount-1.0.tar.gz

%description
libwordcount is a library to count the occurrences of a given word in chunks of text, even occurrences split across chunks. This allows you to process large text files in a streaming fashion without having to read the whole thing in to memory.

A sample application is included, which counts the word "butter".

%prep
%setup -q

%files
```

The `%prep` section is where you do "preporatory" work before the build happens: Unpacking tar files, copying individual source files listed in the spec to the build directory, applying patches, etc - the `%prep` section gets turned in to a shell script, so you can put any shell commands directly in it. To extract the source, though, we can use a special macro: `%setup`. Here we use the `-q` option to do so "quietly". Let's run our `rpmbuild` command again.

```text
Executing(%prep): /bin/sh -e /QOpenSys/var/tmp/rpm-tmp.--a2qa
+ umask 022
+ cd /home/yum/rpmbuild/BUILD
+ cd /home/yum/rpmbuild/BUILD
+ rm -rf libwordcount-1.0
+ /QOpenSys/pkgs/bin/tar -xof -
+ /QOpenSys/pkgs/bin/gzip -dc /home/yum/rpmbuild/SOURCES/wordcount-1.0.tar.gz
+ STATUS=0
+ [ 0 -ne 0 ]
+ cd libwordcount-1.0
/QOpenSys/var/tmp/rpm-tmp.--a2qa[40]: libwordcount-1.0:  not found
error: Bad exit status from /QOpenSys/var/tmp/rpm-tmp.--a2qa (%prep)


RPM build errors:
    Bad exit status from /QOpenSys/var/tmp/rpm-tmp.--a2qa (%prep)
```

Oh no, another error! Here we can see the output from running the shell script that the `%prep` section was turned in to. The shell script is run with the `-e` option, so any command which returns a non-zero return code will cause the shell to exit and the `rpmbuild` processing to stop. It appears that after extracting `wordcount-1.0.tar.gz` it could not find the `libwordcount-1.0` directory. The `%setup` macro expects that the source will extract a directory called `%name-%version`. If this is not the case, you can manually specify the output directory with the `-n` option:

```specfile
%prep
%setup -q -n wordcount-%version
```

Make the above change to your spec file and re-run the rpmbuild command. It should succeed this time.

## Step 6: Running the build

Ok, we have extracted our code and now we need to actuall build it. First, we need to know how to do so. For most open source projects, this info can be found in the `README` or `INSTALL` file. Usually, the documented method is the traditional:

```shell
./configure
make
make install
```

Coincidentally, this is exactly the process used for `libwordcount`!

First we need to add a `%build` section to our spec file:

```specfile
%build

./configure
make
```

The `%build` section should include all the steps needed to actually build the RPM, but not actually run the install (that is done later).

The configure step should run and succeed. In this example, the configure step is pretty quick, but in many open source packages it can take quite a long time even on very fast hardware due to all the checks that it does. You should see a failure during `make`, however:

```text
+ make
gcc -g -O2 -fPIC   -c -o wordcount.o wordcount.c
gcc -shared -o wordcount.so wordcount.o
./mkshrlib.sh os400 wordcount.so libwordcount.so 1
Unknown platform: os400
```

Here the `Makefile` is calling the package's `mkshrlib.sh` script to build the shared library but it doesn't know how to build a shared library for our platform. Most open source packages do not know what IBM i or PASE is, but many do know what AIX is, which usually is the same as what we want. Usually we can "lie" and pretend to be AIX:

```specfile
%build

./configure \
  --build=powerpc-ibm-aix6 \
  --host=powerpc-ibm-aix6

make
```

Now if you run the `rpmbuild`, it should succeed.

## Step 7: Patching

Sometimes a package doesn't work, even on AIX so lying that we're AIX doesn't help.
Or maybe PASE behaves slightly differently than AIX...
Or there's a bug which hasn't been fixed upstream, yet...
Or the bug has been fixed upstream, but only in a new major version and you don't want to break compatibility...

In that case, we need to modify the existing source code to fix the problem and we can do that by generating and applying patches. RPM spec files support patch files very well and has lots of tools for making them as easy as possible to apply.

First we need to get the patch. For now, let's just assume that we have generated the patch and stuck it here: [https://github.com/kadler/rpm-lab/raw/master/libwordcount-mkshrlib-ibmi.patch](https://github.com/kadler/rpm-lab/raw/master/libwordcount-mkshrlib-ibmi.patch). We can download the patch and stick it in `~/rpmbuild/SOURCES`:

```bash
pushd ~/rpmbuild/SOURCES

curl -OLk https://github.com/kadler/rpm-lab/raw/master/libwordcount-mkshrlib-ibmi.patch

popd
```

Now we can adjust our spec file to use the patch:

```specfile
Name: libwordcount
Version: 1.0
Release: 0
License: MIT
Summary: Word count library
Url: https://github.com/kadler/libwordcount

Source: https://github.com/kadler/libwordcount/releases/download/1.0/wordcount-1.0.tar.gz

Patch0: libwordcount-mkshrlib-ibmi.patch

%description
libwordcount is a library to count the occurrences of a given word in chunks of text, even occurrences split across chunks. This allows you to process large text files in a streaming fashion without having to read the whole thing in to memory.

A sample application is included, which counts the word "butter".

%prep
%setup -q

%patch0 -p0

%build

./configure

make

%files
```

Here we've added a new tag to our header:

```specfile
Patch0: mkshrlib-ibmi.patch
```

Notice there is a trailing `0`. This is optional and used to distinguish multiple patches. The `Source` tag can have a trailing number as well, but most packages can be built with only one `Source` tag so it's not as big of a deal there, while packages usually tend to accumulate more and more patches over time so it's always good to include the number on `Patch` tags from the beginning.

Just listing the patch file in the header will only cause `rpmbuild` to ensure the file exists - it won't actually apply the patch, but luckily there is a handy `%patch` macro! Like the `Patch` tag, it has a numbered suffix and you can pass it paramters you would pass to the `patch` command. In this case, our patch is built at patch level 0, so we pass `-p0` to it. Patches should be applied in the `%prep` section.

With this patch, `mkshrlib.sh` now supports building shared libraries on IBM i, so we do not need to lie anymore. Let's run the `rpmbuild` and see what happens.

If you look closely, you can see the patch get applied:

```text
+ echo Patch #0 (libwordcount-mkshrlib-ibmi.patch):
Patch #0 (libwordcount-mkshrlib-ibmi.patch):
+ /QOpenSys/pkgs/bin/patch --no-backup-if-mismatch -p0 --fuzz=0
+ 0< /home/yum/rpmbuild/SOURCES/libwordcount-mkshrlib-ibmi.patch
patching file mkshrlib.sh
+ exit
```

### Step 8: Packaging files

So far we have built the project, but our resulting rpm has not included any files. Let's fix that!

There's two parts to this:
1) First we create an `%install` section, which installs the files in to a dummy directory as if we were really installing it
2) Then we list destination files in the `%files` section, which will be packaged from the dummy directory above.

This dummy directory is known as our "buildroot". So lets start off adding an `%install` section after the `%build` section. We already know from the `INSTALL` file  that to install we need to run `make install`, so lets put add that:

```specfile
%install

make install
```

However, if we run this it will install into the final destionation and not our buildroot. Most makefiles support a way to specify a buildroot, usually with the `DESTDIR` variable. In addition, `rpmbuild` defines a `%buildroot` macro for us. Putting those two bits of info together, we can get the correct result:

```specfile
%install

make DESTDIR=%{buildroot} install
```

Now the `%install` section will install the built software in to our buildroot, but we still have to package it by listing files in our `%files` section. The question is: What files are actually installed? For this, there are a couple methods.

- You could pre-build the software outside of rpmbuild and generate a list (`find` is really handy for this).

- Another option is to not package anything and let rpm's unpackaged file checking kick in. The drawback to this is that if the package installs a lot of files, the output will get lost in your terminal.

- What is often easiest is to just package the everything and list the contents of the built rpm.

The last option is what we're going to do. Update your `%files` section to this:

```specfile
%files
/*
```

Now when you run your `rpmbuild` command, your resulting rpm should have files in it! You can check with the `-l` or `--list` option to `rpm -q`:

```shell
rpm -qpl ~/rpmbuild/RPMS/ppc64/libwordcount-1.0-0.ibmi7.3.ppc64.rpm
/usr
/usr/local
/usr/local/bin
/usr/local/bin/butter
/usr/local/include
/usr/local/include/wordcount.h
/usr/local/lib
/usr/local/lib/libwordcount.so
/usr/local/lib/libwordcount.so.1
```

If you install the rpm, you should see those files installed. Congrats, you've just built and packaged software! However, there's more we can do to improve this package.

## Step 9: Getting the right prefix

By default, `configure` will install to `/usr/local`, but it would be nicer if we integrated in to the rest of the environment and installed it to `/QOpenSys/pkgs`. Luckily, this is usually supported easily by passing the `--prefix` argument to `configure`:

```specfile
%build

./configure --prefix=/QOpenSys/pkgs

make
```

Simple enough, but even simpler `rpmbuild` already defines a macro to do all this for us: `%configure`. In addition to defining `--prefix`, it also does more specialized paths like `--bindir`, `--libdir`, `--sysconfidir`, etc. This macro can be used for pretty much all GNU autoconf packages without needing any other options (though any `configure` arguments can be passed to it, if needed):

```specfile
%build

%configure

make
```

In addition to that, `rpmbuild` defines two make macros: `%make_build` and `%make_install`. These can replace our regular make commands we used before:

```specfile
%build

%configure
%make_build

%install

%make_install

```

If you rebuild the rpm, when you run the `rpm -qpl` command from before, you should now see the files are listed under `/QOpenSys/pkgs` instead of `/usr/local`:

```text
/QOpenSys
/QOpenSys/pkgs
/QOpenSys/pkgs/bin
/QOpenSys/pkgs/bin/butter
/QOpenSys/pkgs/include
/QOpenSys/pkgs/include/wordcount.h
/QOpenSys/pkgs/lib
/QOpenSys/pkgs/lib/libwordcount.so
/QOpenSys/pkgs/lib/libwordcount.so.1
```

## Step 10: Building a real `%files` section

Currently we're just telling rpm to ship everything under `/`, which includes shared directories like `/QOpenSys/pkgs/bin` and `/QOpenSys/pkgs/include`. What happens when we attempt to install two rpms which both ship these directories? In that case we get a conflict and yum will prevent you from installing one of them. Instead, what we want to do is only ship those files and directories which the package itself will own. In this case: `/QOpenSys/pkgs/lib/libwordcount.so.1`, `/QOpenSys/pkgs/lib/libwordcount.so`, `/QOpenSys/pkgs/include/wordcount.h`, and `/QOpenSys/pkgs/bin/butter`. Let's adjust our `%files` section:

```specfile
%files
/QOpenSys/pkgs/bin/butter
/QOpenSys/pkgs/include/wordcount.h
/QOpenSys/pkgs/lib/libwordcount.so
/QOpenSys/pkgs/lib/libwordcount.so.1
```

Now we've only listed the files that our package installs, but we can do even better using more macros. Instead of hard-coding the paths, we can use macros so that the paths listed in the `%files` section always matches the paths given by the `%configure` macro:

```specfile
%files
%{_bindir}/butter
%{_includedir}/wordcount.h
%{_libdir}/libwordcount.so
%{_libdir}/libwordcount.so.1
```

These macros match the arguments to `configure`, but prefixed by an underscore. Thus `--bindir` becomes `%{_bindir}`, `--libdir` becomes `%{_libdir}`, and so on.

One final thing in the `%files` section for now: we should set the ownership to match what IBM i expects. The default ownership for rpm is that files are owned by the user `root` and group `root`, but instead we should have the files owned by `qsys`. There's no equivalent to the `root` group on IBM i, so we can instead default to no group owner by using the special value `*none`. We can set the ownership (along with Unix file permissions) with the `%attr` macro on each file, eg.

```specfile
%files
%attr(-, qsys, *none) %{_bindir}/butter
%attr(-, qsys, *none) %{_includedir}/wordcount.h
%attr(-, qsys, *none) %{_libdir}/libwordcount.so
%attr(-, qsys, *none) %{_libdir}/libwordcount.so.1
```

However, it's far easier to just set the default attributes with `%defattr` instead:

```specfile
%files
%defattr(-, qsys, *none)
%{_bindir}/butter
%{_includedir}/wordcount.h
%{_libdir}/libwordcount.so
%{_libdir}/libwordcount.so.1
```

By using the `-` above, we're telling `rpmbuild` to ignore that parameter and leave it unchanged.

Now that our all our sections are referencing the macro paths defined by `rpmbuild`, our rpm spec file will not need any adjusting should we want to change the install prefix. We can test that by changing the definition of `%{_prefix}`:

```shell
$ rpmbuild -ba ~/rpmbuild/SOURCES/libwordcount.spec --buildroot ~/rpmbuild/BUILDROOT/br --define '_prefix /opt/freeware'
#...
$ rpm -qp -l ~/rpmbuild/RPMS/ppc64/libwordcount-1.0-0.ibmi7.3.ppc64.rpm
/opt/freeware/bin/butter
/opt/freeware/include/wordcount.h
/opt/freeware/lib/libwordcount.so
/opt/freeware/lib/libwordcount.so.1
```

## Step 11: Adding a build dependencies

You may have noticed that `libwordcount` can build against `zlib`, either by reading the readme, seeing the references to `zlib` in the `configure` output or looking at the output of `configure -h`. Let's add zlib support and make our package more useful. To do so, we need to pass `--with-zlib=yes` to configure, like so:

```specfile
%build

%configure --with-zlib=yes

%make_build

%install

%make_install
```

Now run your `rpmbuild` and see what happens. You should encounter an error:

```text
checking for library containing deflate... no
configure: error: cannot find zlib library
error: Bad exit status from /QOpenSys/var/tmp/rpm-tmp.-6g5qb (%build)
```

`configure` is telling us that it can't find the `zlib` library, though what it really means is that it can't find the `zlib` development library. Usually, libraries are split in to two or more packages: the main package, eg.`foo` and a development sub-package, eg. `foo-devel`. We need to install `zlib-devel` for our package to build against it:

```shell
yum install zlib-devel
```

Now that the `zlib` development package is installed, the `rpmbuild` command should succeed again. If we look at the rpm requirements, we should see that the rpm now depends on `libz.so.1(shr_64.o)`:

```shell
$ rpm -qp --requires /home/yum/rpmbuild/RPMS/ppc64/libwordcount-1.0-0.ibmi7.3.ppc64.rpmlib:/QOpenSys/pkgs/lib/libgcc_s.so.1(shr_64.o)(ppc64)
lib:/QOpenSys/pkgs/lib/libz.so.1(shr_64.o)(ppc64)
lib:/QOpenSys/usr/lib/libc.a(shr_64.o)(ppc64)
rpmlib(CompressedFileNames) <= 3.0.4-1
rpmlib(PayloadFilesHavePrefix) <= 4.0-1
```

These `lib:/*` dependencies are automatically generated by `rpmbuild` itself by reading the XCOFF binary header and matching the listed library dependencies to where they can be found in the filesystem. This means that you do not need to remember to list these dependencies explicitly in the spec file (or alterntaively, it prevents these requirements from being missed). Thus, whoever installs this package will be ensured that `/QOpenSys/pkgs/lib/libz.so.1` is available when it is installed. But what happens if someone else attempts to build your spec file and doesn't have `zlib-devel` installed? Unlike runtime dependencies, these build-time dependencies need to be explicitly set using the `BuildRequires` tags in the header:

```specfile
Name: libwordcount
Version: 1.0
Release: 0
License: MIT
Summary: Word count library
Url: https://github.com/kadler/libwordcount

BuildRequires: zlib-devel
```

Now if someone attempts to build the spec file and zlib-devel is not installed, they will immediately get an error telling them to install it:

```text
error: Failed build dependencies:
        zlib-devel is needed by libwordcount-1.0-0.ppc64
```

## Step 12: Sub-packages

As noted above, shared libraries are usually split from the parts that are only needed during development (headers, .so files, man pages, etc) in to multiple sub-packages. This can all be done within the same spec file using the `%package` macro:

```specfile
Name: libwordcount
Version: 1.0
Release: 0
License: MIT
Summary: Word count library
Url: https://github.com/kadler/libwordcount

BuildRequires: zlib-devel

Source: https://github.com/kadler/libwordcount/releases/download/1.0/wordcount-1.0.tar.gz

Patch0: libwordcount-mkshrlib-ibmi.patch

%description
libwordcount is a library to count the occurrences of a given word in chunks of text, even occurrences split across chunks. This allows you to process large text files in a streaming fashion without having to read the whole thing in to memory.

A sample application is included, which counts the word "butter".

%package devel
Summary: Word count library development files
Requires: libwordcount == %{version}

%description devel
Files needed to build against libwordcount

%package -n wordcount-samples
Summary: Samples using libwordcount

%description -n wordcount-samples
Samples using libwordcount:
- butter: count occurrences of "butter" in plain-text or gzip-compressed text files

%prep
%setup -q -n wordcount-%{version}

%patch0 -p0

%build
%configure --with-zlib=yes
%make_build

%install

%make_install

%files
%defattr(-, qsys, *none)
%{_libdir}/libwordcount.so.1

%files devel
%defattr(-, qsys, *none)
%{_libdir}/libwordcount.so
%{_includedir}/wordcount.h

%files -n wordcount-samples
%defattr(-, qsys, *none)
%{_bindir}/butter
```

The `%package` sections should start after the main `%description` section. Each `%package` macro needs at least a `Summary` tag and a matching `%description` tag. By passing a string to the macro, you can define a subpackage name based on the main package name or use the `-n` argument to completely specify a new package name. Here we've defined two subpackages: `libwordcount-devel` and `wordcount-samples`. In addition to the matching `%description` section, each package gets its own matching `%files` section, or else no files will be packaged for that package and now rpm will be generated for it.

You may have also noticed the `Requires` tag. This is because there is no binary shipped by `libwordcount-devel` which would automatically pick up that requirement by `rpmbuild`, so it has to be manually specified.

After running `rpmbuild` you should see 3 rpms were generated now:

```text
Wrote: /home/yum/rpmbuild/SRPMS/libwordcount-1.0-0.src.rpm
Wrote: /home/yum/rpmbuild/RPMS/ppc64/libwordcount-1.0-0.ibmi7.3.ppc64.rpm
Wrote: /home/yum/rpmbuild/RPMS/ppc64/libwordcount-devel-1.0-0.ibmi7.3.ppc64.rpm
Wrote: /home/yum/rpmbuild/RPMS/ppc64/wordcount-samples-1.0-0.ibmi7.3.ppc64.rpm
```

## Step 13: Final pieces

Our package is pretty much done, but there's still one thing we could improve. Our `butter` sample isn't very useful without a text file to test against. Let's generate a file and ship it with our samples sub-package. Luckily the package provides a python script which can be used to generate such a text file.

First, run word.py to generate a test file with lots of "butter" mixed in with random data, then let's compress a copy of it:

```shell
~/rpmbuild/BUILD/wordcount-1.0/word.py butter 300 > ~/rpmbuild/SOURCES/butter.txt
gzip -c ~/rpmbuild/SOURCES/butter.txt > ~/rpmbuild/SOURCES/butter.txt.gz
```

Now, to ship them we need to add them as sources to our package with additional `Source` tags:

```specfile
Name: libwordcount
Version: 1.0
Release: 0
License: MIT
Summary: Word count library
Url: https://github.com/kadler/libwordcount

BuildRequires: zlib-devel

Source: https://github.com/kadler/libwordcount/releases/download/1.0/wordcount-1.0.tar.gz

Source1: butter.txt
Source2: butter.txt.gz
```

Since the package's Makefile doesn't provide any mechanism to install these files, we will have to do this manually in the `%install` section:

```specfile
%install

%make_install

# Make a wordcount package directory to hold our data
mkdir -p %{buildroot}%{_datadir}/wordcount
cp %{SOURCE1} %{SOURCE2} %{buildroot}%{_datadir}/wordcount
```

`rpmbuild` helpfully provides a macro for each of the `Source` tags. NOTE: This macro **must** be upper-cased even though the tag is just capitalized and unlike all the other macros used so far.

Finally we can add these files to our `%files` section:

```specfile
%files -n wordcount-samples
%defattr(-, qsys, *none)
%{_bindir}/butter

%dir %{_datadir}/wordcount
%{_datadir}/wordcount/butter.txt
%{_datadir}/wordcount/butter.txt.gz
```

Here we use the `%dir` macro to ensure the directory is owned by our package. Since this package owns the whole directory, we can even take a shortcut and just list the directory itself, which will package all of its contents and subdirectories as well:

```specfile
%files -n wordcount-samples
%defattr(-, qsys, *none)
%{_bindir}/butter

%{_datadir}/wordcount
```

Now run `rpmbuild` for the last time. Everything should build successfully. You can install the rpms and play around with the butter command:

```shell
yum insall ~/rpmbuild/RPMS/ppc64/wordcount-samples-1.0-0.ibmi7.3.ppc64.rpm ~/rpmbuild/RPMS/ppc64/libwordcount-1.0-0.ibmi7.3.ppc64.rpm

$ butter /QOpenSys/pkgs/share/wordcount/butter.txt
$ butter /QOpenSys/pkgs/share/wordcount/butter.txt.gz
```

Both of the above commands should say they found the 300 instances of "butter" in the text file.

## Final words

There are many resources on the web for learning how to package rpms. Here are a couple really good ones:

- [http://rpm-guide.readthedocs.io/en/latest/](http://rpm-guide.readthedocs.io/en/latest/)
- [https://fedoraproject.org/wiki/Packaging:Guidelines](https://fedoraproject.org/wiki/Packaging:Guidelines)

In addition, IBM will be providing IBM i-specific rpm packaging documentation and more in the future. Check this link for more once it is available: [http://ibm.biz/ibmi-rpm-guide](http://ibm.biz/ibmi-rpm-guide)
