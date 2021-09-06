# How to use the softcover docker image to publish your book

This article will show you how to use the softcover docker image to run the softcover commands needed to create and publish your book or article to the softcover service.

When I tried to use the softcover CLI command line tooling on my Mac OS machine, I ran into a few installation issues.

Fortunately I found this Docker image with the softcover tools pre installed that was build by one of the softcover team members.

I was able to use the Docker image to build and publish book (in progress) to the platform.

Since I had to figure out a few things along the way, I decided to document the commands I used so that others may benefit from what I discovered.

This article assumes that you have Docker installed on your machine.

## Pulling the image

The first thing you want to do is pull down the image from Docker hub. It is fairly large image so be warned that it will take a bit of time.

The image is hosted at `[https://hub.docker.com/r/softcover/softcover](https://hub.docker.com/r/softcover/softcover)` and the its repo is at [https://github.com/softcover/softcover-docker](https://github.com/softcover/softcover-docker)

Let's pull the image:

```console
docker pull softcover/softcover
```

## Creating a new book

Now that we have the image downloaded we can run the docker command to create the book.

Let's first setup a directory to hold all our books/articles and move into it:

```consloe
mkdir mybooks && cd mybooks
```

Next run the the command to create a new book, named `mybook` in this case:

```console
docker run --rm -v `pwd`:/book softcover/softcover:latest sc new mybook
```

Here we have told the container to run the `sc new mybook` command.

The command will create a new directory named `mybook` in the current directory.

In case we wanted to create an article instead of a book, we would just need to add the `-a` option to the `sc new` command.

```console
docker run --rm -v `pwd`:/book softcover/softcover:latest sc new -a myarticle
```

Ok, let move into the `mybook` directory and explore its content in the next section.

```console
cd mybook
```

> Note: Although we have not specified a working directory option on the command line, the Docker image dockerfile specifies a `/book` working directory. This is why we have a specified a volume map option that maps the current directory `pwd` to the `/book` directory in the container. With this mapping, when we run the `sc new mybook` command, the `mybook` directory is created in the container's `/book` working directory and the working directory content is synched into our current directory.

## Exploring the book directory

The book directory that we just created contains contains a `book.txt` file that lists the table of contents, the preface and the individual book chapters.

Lets see the contents of this file:

```console
$ cat book.txt

cover
frontmatter:
maketitle
tableofcontents
preface.md
mainmatter:
a_chapter.md
yet_another_chapter.md
```

The order of the chapter filenames listed under the `mainmatter:` section determines their corresponding chapter order in the published book. These filenames correspond to files in the `./chapters` directory.

The `preface.md` file also resides in the `./chapters` directory.

The `./chapters` directory contains the set of markdown files that are the source files used to generate the html and ebook file formats of our book.

Lets see the contents of the chapters directory:

```console
$ ls -l chapters

a_chapter.md
yet_another_chapter.md
```

Note, if we had created an article instead of a book, there would only be a single chapter and no preface or table of content and there would only be a single chapter file with a matching name in the `./chapters` directory.

Here is the content of the Book.txt file for an article:

```console
$ cat book.txt
cover
maketitle
a_chapter.md
```

## Building and Serving the generated html files

As we author our book or article, by adding and editing markdown files in the chapters directory, we might like to build and preview the content in a web browser.

One way to do this is to build the html files using the following command from within the `mybook` directory.

```console
docker run --rm -v `pwd`:/book softcover/softcover:latest sc build:html
```

This command uses the markdown files in the `./chapters` directory as source files that are used to build corresponding html output files in the `./html` directory.

In addition to building the html files we can build and also serve them to our local web browser by running the following command instead:

```console
docker run --rm -v `pwd`:/book -d -p 4000:4000 softcover/softcover:latest sc server
```

The above docker command runs the `sc server` command in the container which in turn runs the `sc build:html` to build and publish the markdown files to the `/book` container directory. The `sc server` command then starts up a web server inside the Docker container that serves the html files on port 4000 of the container.(The port number is exposed in the dockerfile)

Our docker command also maps the container port 4000 to our local port 4000 which allows us to navigate to localhost:4000 with our local web browser to view our book html content.

When we change the content of the `chapters` directory the change is detected and the `sc build:html` command is re-run and web server is reloaded, allowing us to instantly preview the output of our changes.

Of course we can always run the `sc build:html` command using the container to force a rebuild.

In fact, I ran into an issue sometimes where the server running in the container would hang or the container itself would exit. This happened when I renamed the chapter markdown files or there was an error in the markdown file that caused a file parsing error.

In this scenario running the `sc build:html` command will display the errors so you can debug the issue.

With the file renaming issue, stoping and restarting the container and running the `sc server` command again seemed to fix the issue.

> Tip: Don't forget to rename the filenames in Book.txt when you rename the markdown files in the chapters directroy.

To check if the container exited you can run this Docker command:

```console
docker ps -a
```

This command will show all running and stopped containers on your system.

Assuming you have resolved any issue you are having, you can stop the container (which will automatically remove it since we are using the --rm flag) and then run the command again:

```console
docker stop <contained-hash-id>
docker run --rm -v `pwd`:/book -d -p 4000:4000 softcover/softcover:latest sc server
```

## Publishing the book to softcover.io

In order to publish your book you need to login to softcover.io service and then issue the softcover CLI publishing commands.

Because of the need to login, the only way you can accomplish this is to interactively log into the running Docker container and then from the container bash shell log into softcover.io and run the publishing commands.

This section will show you this process and the commands you need to run.

To run the container in interactive mode we need to run the container with the -it flag and execute the bash command.

```console
docker run --rm -it -v `pwd`:/book softcover/softcover:latest bash
```

This will drop us into the container in the `/book` working directory:

```console
# pwd
\book
```

Now run the following commands in the bash shell to login, build and publish to the softcover service then logout:

```console
sc login
sc clean
sc deploy
sc logout
```

Instead of running `sc deploy` above we can explicitly run the build and publish commands:

```console
sc login
sc clean
# the next three commands replace the sc deploy command
sc build:all
sc build:preview
sc publish
sc logout
```

We can re-run these commands anytime we wish to update our published book on softcover.io.

Ofcource you can ommit running `sc publish` to inspect the files before you decide to publish.

> Please refer to softcover documentation for full description of all commands

Finally once we are done publishing, we can exit the bash shell by typing `exit` which will terminate the bash session and the container process.

## Inspecting the published content

Normally if we were logging into softcover by executing the `sc login` command from our host machine we could navigate to the softcover service and be automatically logged in.

However since we logged in from a container instead, we will need to manually log into the softcover website to be able to see our published book or article.

## Conclusion

In this article I showed you how you can use the softcover docker container to build, locally serve and publish your book or article to the softcover service.

By using the softcover docker image, you dont have to deal with any installationa and upgrade issues with the softcover CLI tooling. Furthermore you don't have to pollute your work environment with additional tooling that you dont need to have permanently installed

## Resources

The following links may be helpful:
