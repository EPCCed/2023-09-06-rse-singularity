---
title: "Creating Your Own Container Images"
teaching: 40
exercises: 40
questions:
- "How can I make my own Singularity container images?"
- "How do I document the 'recipe' for a Singularity container image?"
- "How can I make more complex container images? "
objectives:
- "Explain the purpose of a Singularity recipe file and show some simple examples."
- "Demonstrate how to build a Singularity container image from a recipe file."
- "Compare the steps of creating a container image interactively versus a recipe file."
- "Create an installation strategy for a container image."
- "Explain how you can include files within Singularity container images when you build them."
- "Explain how you can access files on the Singularity host from your Singularity containers."
keypoints:
- "Singularity recipe files specify what is within Singularity container images."
- "The `singularity build` command is used to build a container image from a recipe file."
- "`singularity build` requires admin/root privileges so usually needs to be prefixed with `sudo`."
- Singularity allows containers to read and write files from the Singularity host.
- You can copy files from your host system into your Singularity container images at creation time by using the `%files` section in the recipe file.

---

There are lots of reasons why you might want to create your **own** Singularity container image.
- You can't find a container image with all the tools you need online or in local resources
- You want to have a container image to "archive" all the specific software versions you ran for a project.
- You want to share your workflow with someone else.

> ## Building using Docker rather than Singularity
>
> An alternative to building container images using Singularity itself is to use Docker to build the images
> that you then run using Singularity. This has a number of advantages:
> * Docker/Docker Desktop is often easier to install than SingularityCE/Apptainer (particularly on macOS/Windows systems)
> * Docker can build cross-platform - you can build container images for x86 systems on Arm-based systems (such as Mac M1/M2 systems)
> * Docker is generally more efficient in dealing with uploading/downloading container image data that makes it better
>   for moving your container images to remote HPC facilities
>
> This session primarily uses Singularity to build the container images but we also provide the equivalent 
> Dockerfiles in case you want to build container images using Docker rather than Singularity.
>
> > ## Building and uploading images using Docker
> > 
> > Typically, you will build using Docker with a command such as (assuming you are issuing the command
> > from the same directory as the Dockerfile and that your Docker Hub username is `alice`):
> >
> > ```
> > docker image build --platform linux/amd64 -t alice/image-name .
> > ```
> >
> > You can then push your built image to Docker Hub with:
> >
> > ```
> > docker push alice/image-name
> > ```
> >
> > Finally, you can log into the remote system and build a Singularity image from the image on Docker
> > Hub with:
> >
> > ```
> > singularity build image-name.sif docker://alice/image-name
> > ```
> >
> > You can also build directory from a tar archive exported from Docker using the `docker-archive` image
> > type if you do not want to upload via Docker Hub or another online repository.
> > 
> {: .solution}
{: .callout}

## Starting with a basic Alpine Linux image

Before creating a reproducible installation, let's start with a minimal Linux container image.
Create a Singularity container image from an `alpine` Docker container image:

~~~
singularity pull alpine.sif docker://alpine 
~~~
{: .language-bash}

~~~
INFO:    Converting OCI blobs to SIF format
INFO:    Starting build...
Getting image source signatures
Copying blob 31e352740f53 done
Copying config f4b9357049 done
Writing manifest to image destination
Storing signatures
2023/06/17 09:38:24  info unpack layer: sha256:31e352740f534f9ad170f75378a84fe453d6156e40700b882d737a8f4a6988a3
INFO:    Creating SIF file...
~~~
{: .output}

Now, start a shell in a container based on the new container image:

~~~
singularity shell alpine.sif
~~~
{: .language-bash}

Because this is a basic container, there's a lot of things not installed -- for
example, `python3`.

~~~
Singularity> python3
~~~
{: .language-bash}
~~~
/bin/sh: python3: not found
~~~
{: .output}

Python 3 is not provided by the `alpine` container image. However, the Alpine
version of Linux has an installation tool (a package manager) called `apk` that
we can use to install Python 3, or indeed a wide range of other software,
libraries or tools. So, we could build our own container image that adds Python
3 to the `alpine` container image. Software can be installed using the `apk
add` command, e.g. to install Python 3 (as well as a couple of additional
related packages) we might use:

~~~
apk add --update python3 py3-pip python3-dev
~~~
{: .language-bash}

## Interactive installation

You may wonder why we cannot install Python 3 directly in the running container itself by
using the command above. If you try to do this, you will get an error:

~~~
ERROR: Unable to lock database: Read-only file system
ERROR: Failed to open apk database: Read-only file system
~~~
{: .output}
 
This is because the system directories where `apk` would install
Python are in read-only areas of file system in the running container. Installing
software interactively is not ideal anyway from a reproducibility aspect as it 
makes it difficult to know exactly what process was followed to install the software
in the container image and track changes to this process over time and/or versions
of the container image.

> ## Writable container images
>
> There is a way to create an image in a way that can be written to
> but it is a bit convoluted and not as useful as you might first expect; due, in a large
> part to the reproducibility issues discussed above. To be able to install
> Python 3 in a running Alpine container we need to build and run the container in a different
> way. You need to use the `--sandbox` flag:
>
> ~~~
> singularity build --sandbox alpine-writable.sandbox docker://alpine
> ~~~
> {: .language-bash}
>
> Once the sandbox container image has been built, we need to open a shell in a container based
> on the sandbox container image:
>
> ~~~
> singularity shell --writable alpine-writable.sandbox
> ~~~
> {: .language-bash}
>
> Now, finally, we can use the `apk add --update python3 py3-pip python3-dev` command in
> the running container to install Python 3. Note, the installation will persist in
> the sandbox container image even if you shut down the running container and start
> a new one.
>
> If you then want to convert the sandbox container image to a standard container image
> you can use the build command:
>
> ~~~
> singularity build alpine-python.sif alpine-writable.sandbox
> ~~~
> {: .language-bash}
>
> This approach can be useful for exploring the install commands to use to create
> your container images but it is not generally a good way to create reproducible
> container images.
>
> [Singularity CE docs on sandbox images](https://docs.sylabs.io/guides/3.10/user-guide/build_a_container.html#creating-writable-sandbox-directories)
{: .callout}

## Put installation instructions in a Singularity recipe file

A Singularity recipe file is a plain text file with keywords and commands that
can be used to create a new container image. This is a much more reproducible
approach than installing things interactively as it allows us to have a record
of exactly how we installed software in the container image and, as it is plain
text, it lends itself well to being placed under version control (e.g. using
git and Github/Gitlab) to track and manage versions of the recipe file. We will
start by creating a very simple recipe file that defines an image based on
Alpine Linux with Python 3 installed.

Using your favourite text editor, create a file called `alpine-python.def` and add 
the following lines:

~~~
Bootstrap: docker
From: alpine:latest

%post
    apk add --update python3 py3-pip python3-dev

%runscript
    python3 --version
~~~

Let's break this file down:

- The first line, `Bootstrap: docker`, tells Singularity that we want to start
  the build from an existing Docker container image.
- The next line, `From: alpine:latest`, indicates which container image we are
  starting from.  It is the "base" container image we are going to start from.
  As no Docker container image repository URL is specified, Singularity assumes
  that this is a container image from Docker Hub. 
- The `%post` section specifies commands to run as part of the image build
  process. In this case we use the Alpine Linux `apk` command to install
  Python 3.
- The last section, `%runscript`, indicates the default action we want a
  container based on this container image to run (when used with `singularity
  run`). In this case, we ask the container to return the version of Python
  installed in the container.

> ## Build with Docker
>
> > ## Dockerfile
> >
> > ```
> > FROM alpine:latest
> >
> > RUN apk add --update python3 py3-pip python3-dev
> >
> > CMD ["python3", "--version"]
> > ```
> {: .solution}
{: .callout}


## Extending our recipe file

So far, we only have a text file named `alpine-python.def` -- we do not yet
have a container image. Also, in order to create and use your own container
images, you may need to provide some more detailed instructions than the very
basic recipe we've created so far. For example, you may want to include files
in your container image that are currently on your host system by copying them
into the image when it is built. You may also want to undertake more advanced
software installation or configuration.

Before we go ahead and use our recipe to build a container image, we're going
to create a simple Python script on our host system and update our recipe to
have this copied into our container image when it is created.

Use your text editor to create a Python script called `sum.py` with the
following contents:

~~~
#!/usr/bin/env python3

# Comment added in container

import sys
try:
   total = sum(int(arg) for arg in sys.argv[1:])
   print('sum =', total)
except ValueError:
   print('Please supply integer arguments')
~~~
{: .language-python}

### Including your scripts and data within a container image

We're now going to add some configuration to enable us to include one or more
files from our local system into the container image that we're going to build.

You might want to do this if you have a local script that you'd like to include
within your image, for example, or perhaps some static input data that will
always be required by software within your image. To demonstrate the process,
we're going to modify our recipe file to add our `sum.py` script into the
container image.

We'll do this by modifying our recipe file to include a new `%files` section:

~~~
Bootstrap: docker
From: alpine:latest

%files
    sum.py /home

%post
    apk add --update python3 py3-pip python3-dev

%runscript
    python3 --version
~~~

The `%files` section here specifies that the file `sum.py` (in the current
directory from where we initiate the container image build process) should be
copied to the target location `/home` inside the container image that will be
built based on this recipe file. Multiple lines can be added to the `%files`
section to have additional files copied from the host filesystem into the
container image.

> ## Build with Docker
>
> > ## Dockerfile
> >
> > ```
> > FROM alpine:latest
> >
> > COPY sum.py /home
> > 
> > RUN apk add --update python3 py3-pip python3-dev
> >
> > CMD ["python3", "--version"]
> > ```
> {: .solution}
{: .callout}

Note that it's not necessarily a good idea to put your scripts inside the
container image if you're constantly changing or editing them. In this case, 
referencing the scripts in a shared location that is mounted into a running
container from the host system is often a better approach. You should also
think carefully about container size -- if you run `ls -lh *.sif` you'll see
the size of each container image in the current directory. The bigger your
container image becomes, the more impractical it will be to share and download.

## Build a new Singularity container image from the recipe

We now want Singularity to take this recipe file, run the installation commands
contained within it, and then save the resulting container as a new container
image. To do this we will use the `singularity build` command.

We have to provide the `singularity build` command with two pieces of
information:
- the name of the new container image file that we want to create
- the name of the recipe file to use

As we are building a container image we need admin/root privileges so we need
to use the `sudo` command to run our `singularity build` command.

> ## sudo Password
> As you are using `sudo`, you may be asked by the system for your password
> when you run this command. Your system will typically ask for the password
> when using `sudo` for the first time after an expiry period is reached (this
> can be every 5 mins but is sometimes longer, it depends on the system you are
> using).
{: .callout} 

> ## Running `singularity build` with `sudo`
> The statement above that says that we _need_ to run `singularity build` with
> `sudo` is not entirely correct in all cases -- there are some cases where
> `singularity build` can be run without `sudo`. You may, for example, find
> this is the case if you're running via `lima` on `macOS`. However, as a
> general rule we suggest to use `sudo` since this ensures that the running
> process has the necessary privileges to create files with the correct
> ownership and permissions within the generated container image.
{: .callout} 

All together, the build command that you should run on your computer, will have
a structure like the following:

~~~
sudo singularity build <container image file name> <recipe file name>
~~~
{: .language-bash}

For example, if my recipe is in the file `alpine-python.def` and I wanted to
call my container image file `alpine-python.sif`, I would use this command:

~~~
sudo singularity build alpine-python.sif alpine-python.def
~~~
{: .language-bash}

~~~
INFO:    Starting build...
Getting image source signatures
Copying blob 31e352740f53 done  
Copying config f4b9357049 done  
Writing manifest to image destination
Storing signatures
2023/08/03 18:25:46  info unpack layer: sha256:31e352740f534f9ad170f75378a84fe453d6156e40700b882d737a8f4a6988a3
INFO:    Copying sum.py to /home
INFO:    Running post scriptlet
+ apk add --update python3 py3-pip python3-dev
fetch https://dl-cdn.alpinelinux.org/alpine/v3.18/main/x86_64/APKINDEX.tar.gz
fetch https://dl-cdn.alpinelinux.org/alpine/v3.18/community/x86_64/APKINDEX.tar.gz
(1/27) Installing libbz2 (1.0.8-r5)
(2/27) Installing libexpat (2.5.0-r1)
(3/27) Installing libffi (3.4.4-r2)
(4/27) Installing gdbm (1.23-r1)
(5/27) Installing xz-libs (5.4.3-r0)
(6/27) Installing libgcc (12.2.1_git20220924-r10)
(7/27) Installing libstdc++ (12.2.1_git20220924-r10)
(8/27) Installing mpdecimal (2.5.1-r2)
(9/27) Installing ncurses-terminfo-base (6.4_p20230506-r0)
(10/27) Installing libncursesw (6.4_p20230506-r0)
(11/27) Installing libpanelw (6.4_p20230506-r0)
(12/27) Installing readline (8.2.1-r1)
(13/27) Installing sqlite-libs (3.41.2-r2)
(14/27) Installing python3 (3.11.4-r0)
(15/27) Installing python3-pycache-pyc0 (3.11.4-r0)
(16/27) Installing pyc (0.1-r0)
(17/27) Installing py3-setuptools-pyc (67.7.2-r0)
(18/27) Installing py3-pip-pyc (23.1.2-r0)
(19/27) Installing py3-parsing (3.0.9-r2)
(20/27) Installing py3-parsing-pyc (3.0.9-r2)
(21/27) Installing py3-packaging-pyc (23.1-r1)
(22/27) Installing python3-pyc (3.11.4-r0)
(23/27) Installing py3-packaging (23.1-r1)
(24/27) Installing py3-setuptools (67.7.2-r0)
(25/27) Installing py3-pip (23.1.2-r0)
(26/27) Installing pkgconf (1.9.5-r0)
(27/27) Installing python3-dev (3.11.4-r0)
Executing busybox-1.36.1-r0.trigger
OK: 142 MiB in 42 packages
INFO:    Adding runscript
INFO:    Creating SIF file...
INFO:    Build complete: alpine-python.sif
~~~
{: .output}

> ## Exercise: Container images and running a container
>
> 1. How might you check that your container image file was created successfully?
>
> 2. What command will run a container based on the container image you've
>    created and perform the default action?
>
> 3. What is causing this default action to run and how could you change the
>    default action?
>
> 4. Can you make it do something different, like print "hello world"?
>
> > ## Solution
> >
> > 1. To check that the file for your new image has been created, run `ls`.
> >    You should see the name of your new container image file listed.
> >
> > 2. We would use `singularity run alpine-python.sif` to run a container
> >    based on the `alpine-python.sif` container image and perform the default
> >    action.
> >
> > 3. The default action is being triggered by the run script embedded in the
> >    image. This is defined in the `%runscript` section of the recipe file we
> >    created earlier. To update the default action within the image, one
> >    option is to edit the recipe file and rebuild the image.
> >
> > 4. We could use `singularity exec alpine-python.sif echo "hello world"` to
> >    run a container based on the container image and perform the default
> >    action.
> >
> > {: .language-bash}
> {: .solution}
{: .challenge}

> ## Exercise: Explore the script `sum.py` script
>
> Start a container from your image that runs the `sum.py` script. What happens
> if you use the `singularity exec` command above and put numbers after the
> script name?
>
> > ## Solution
> >
> > This script comes from [the Python Wiki](https://wiki.python.org/moin/SimplePrograms)
> > and is set to add all numbers that are passed to it as arguments.
> {: .solution}
{: .challenge}

> ## Exercise: Interactive use
>
> We can also use the script interactively within a running container. What
> commands would you use to start a container from your `alpine-python.sif`
> container image and then run the `sum.py` script interactively in this
> container?
>
> > ## Solution
> >
> > The Singularity command to run the container interactively is:
> > ~~~
> > singularity shell alpine-python.sif
> > Singularity> python3 sum.py 10 12 10
> > ~~~
> > {: .language-bash}
> >
> > ~~~
> > sum = 32
> > ~~~
> > {: .output}
> {: .solution}
{: .challenge}

### Making the `sum.py` script run automatically

To close out our practical work on building containers, there's one thing we
haven't yet done. At present, when we run a container from our image, the
default run script prints the Python version. let's modify this to run the
`sum.py` script by default:

> ## Make the `sum.py` script run automatically
> 
> Can you modify the `alpine-sum.def` recipe file so that the `sum.py` is run
> automatically when using the `singularity run` command?
>
> > ## Solution
> > ~~~
> > Bootstrap: docker
> > From: alpine:latest
> > 
> > %post
> >     apk add --update python3 py3-pip python3-dev
> > 
> > %files
> >     sum.py /home
> > 
> > %runscript
> >     python3 /home/sum.py
> > ~~~
> >
> {: .solution}
{: .challenge}

Build and test it:
~~~
sudo singularity build alpine-sum.sif alpine-sum.def
singularity run alpine-sum.sif
~~~
{: .language-bash}

You'll notice that you can run the container without arguments just fine,
resulting in `sum = 0`, but this is boring. Supplying arguments however
doesn't work:
~~~
singularity run alpine-sum.sif 10 11 12
~~~
{: .language-bash}

still results in

~~~
sum = 0
~~~
{: .output}

This is because the arguments `10 11 12` are not interpreted by
the *runscript* in the container.

To achieve the goal of having a command that *always* runs when a
container is run from the container image *and* can be passed the arguments given on the
command line, we need to tell the *runscript* to use the arguments:

> ## Handling command line arguments in run scripts
> 
> Can you modify update the `alpine-sum.def` recipe file to handle command line
> arguments passed to `sum.py`?
>
> > ## Solution
> > ~~~
> > Bootstrap: docker
> > From: alpine:latest
> > 
> > %post
> >     apk add --update python3 py3-pip python3-dev
> > 
> > %files
> >     sum.py /home
> > 
> > %runscript
> >     python3 /home/sum.py "$@"
> > ~~~
> >
> {: .solution}
{: .challenge}

Build and test your updated image:
~~~
sudo singularity build alpine-sum.sif alpine-sum.def
singularity run alpine-sum.sif 10 11 12
~~~
{: .language-bash}

~~~
sum = 33
~~~
{: .output}

<br/><br/>
While it may not look like you have achieved much at this stage, you have
already created an image that combines a lightweight Linux operating system
installation with your own configuration and software installed. This can now
be used to run a given command and it can also operate reliably across
different platforms that have Singularity or Apptainer installed.

<br/><br/>

> ## Build with Docker
>
> > ## Dockerfile
> >
> > ```
> > FROM alpine:latest
> >
> > COPY sum.py /home
> >
> > RUN apk add --update python3 py3-pip python3-dev
> >
> > ENTRYPOINT ["python3", "/home/sum.py"]
> > ```
> {: .solution}
{: .callout}

## Some additional notes and warnings

> ## Security Warning
> 
> Login credentials including passwords, tokens, secure access tokens or other secrets
> must never be stored in a container image. If secrets are stored, they are at high risk to
> be found and exploited when made public.
{: .callout}

> ## Alternatives for copying files into a container image
>
> Another approach for getting your own files into a container image is by using the `%post`
> section and adding commands that download the files from the Internet. For example, if your code
> is in a GitHub repository, you could include this statement in your recipe file
> to download the latest version every time you build the container image:
>
> ~~~
> %post
>     ...other installation commands...
>     git clone https://github.com/alice/mycode
> ~~~
>
> Similarly, the `wget` command can be used to download any file publicly available
> on the internet:
>
> ~~~
> %post
>     ...other installation commands...
>     wget ftp://ftp.ncbi.nlm.nih.gov/blast/executables/blast+/2.10.0/ncbi-blast-2.10.0+-x64-linux.tar.gz
> ~~~
>
> Note that the above examples depend on commands (`git` and `wget` respectively) that 
> must be available within your container: Linux distributions such as Alpine may require you to 
> install such commands before using them.
{: .callout}

### Boring but important notes about installation

There are a lot of choices when it comes to installing software -- sometimes too many!
Here are some things to consider when creating your own container image:

- **Start smart**, or, don't install everything from scratch! If you're using Python
as your main tool, start with a [Python container image](https://hub.docker.com/_/python). Same with [R](https://hub.docker.com/r/rocker/r-ver/). We've used Alpine Linux as an example
in this lesson, but it's generally not a good container image to start with for initial development and experimentation because it is
a less common distribution of Linux; using [Ubuntu](https://hub.docker.com/_/ubuntu), [Debian](https://hub.docker.com/_/debian) and [CentOS](https://hub.docker.com/_/centos) are all
good options for scientific software installations. The program you're using might
recommend a particular distribution of Linux, and if so, it may be useful to start with a container image for that distribution.
- **How big?** How much software do you really need to install? When you have a choice,
lean towards using smaller starting container images and installing only what's needed for
your software, as a bigger container image means longer download times to use.
- **Know (or Google) your Linux**. Different distributions of Linux often have
  distinct sets of tools for installing software. The `apk` command we used
  above is the software package installer for Alpine Linux. The "package
  managers"/installers for various common Linux distributions are listed below:
    - Ubuntu / Debian: `apt` or `apt-get`
    - CentOS / RedHat: `yum`
    - SUSE / openSUSE: `zypper`

  Most common Linux software is available as packages, although sometimes only
  for a small number of the more common distributions, and can be installed by
  these package management tools. A web search for "install X on Y Linux" is
  usually a good start for common software installation tasks; if something
  isn't available via the Linux distribution's installation tools, try the
  options below.
- **Use what you know**. You've probably used commands like `pip` (Python) or
  `install.packages()` (R) before on your own computer -- these will also work
  to install things in container images (if the basic scripting language is
  installed).
- **README**. Many scientific software tools come with a README file or
  installation instructions that lay out how to install the software. If the
  install instructions include options like those suggested above, try those
  first.

In general, a good strategy for installing software is:
- Make a list of what you want to install.
- Look for pre-existing container images.
- Read through instructions for software you'll need to install.
- Create a Singularity recipe file and then try to build the container image
  from that.

### Sharing your Singularity container images

You have a few different options available to share your container image files with other people,
including:

- Place them on shared storage on the shared facility so other can access them
- Place on external storage that is publicly available via the web
- Add them to an online repository such as the [Sylabs Cloud Library](https://cloud.sylabs.io/library)

## More advanced definition files

Here we've looked at a very simple example of how to create an image. At this stage, you might want to have a go at creating your own definition file for some code of your own or an application that you work with regularly. There are several definition file sections that were _not_ used in the above example, these are:

 - `%setup`
 - `%environment`
 - `%startscript`
 - `%test`
 - `%labels`
 - `%help`

The [`Sections` part of the definition file documentation](https://docs.sylabs.io/guides/3.11/user-guide/definition_files.html) details all the sections and provides an example definition file that makes use of all the sections.

## Additional Singularity features

Singularity has a wide range of features. You can find full details in the [Singularity User Guide](https://docs.sylabs.io/guides/3.11/user-guide/index.html) and we highlight a couple of key features here that may be of use/interest:

**Remote Builder Capabilities:** If you have access to a platform with Singularity installed but you don't have root access to create containers, you may be able to use the [Remote Builder](https://docs.sylabs.io/guides/3.11/user-guide/cloud_library.html#remote-builder) functionality to offload the process of building an image to remote cloud resources. You'll need to register for a _cloud token_ via the link on the [Remote Builder](https://cloud.sylabs.io/builder) page.

**Signing containers:** If you do want to share container image (`.sif`) files directly with colleagues or collaborators, how can the people you send an image to be sure that they have received the file without it being tampered with or suffering from corruption during transfer/storage? And how can you be sure that the same goes for any container image file you receive from others? Singularity supports signing containers. This allows a digital signature to be linked to an image file. This signature can be used to verify that an image file has been signed by the holder of a specific key and that the file is unchanged from when it was signed. You can find full details of how to use this functionality in the Singularity documentation on [Signing and Verifying Containers](https://docs.sylabs.io/guides/3.11/user-guide/signNverify.html).


> ## Best practices for writing container image definition files
> Take a look at Nüst et al.'s "[_Ten simple rules for writing Dockerfiles for reproducible data science_](https://doi.org/10.1371/journal.pcbi.1008316)" \[1\] 
> for some great examples of best practices to use when writing Dockerfiles. 
> The [GitHub repository](https://github.com/nuest/ten-simple-rules-dockerfiles) associated with the paper also has a set of [example `Singularityfile`s](https://github.com/nuest/ten-simple-rules-dockerfiles/tree/master/examples) 
> demonstrating how the rules highlighted by the paper can be applied.
>
> <small>[1] Nüst D, Sochat V, Marwick B, Eglen SJ, Head T, et al. (2020) Ten simple rules for writing Dockerfiles for reproducible data science. PLOS Computational Biology 16(11): e1008316. https://doi.org/10.1371/journal.pcbi.1008316</small>
{: .callout}

{% include links.md %}