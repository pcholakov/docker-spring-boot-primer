Short primer on packaging applications as Docker containers
===========================================================

For an excellent explanation of why Docker is generating so much buzz in the developer community, watch this excellent presentation on [AWS Elastic Beanstalk and Docker](https://www.youtube.com/watch?v=OzLXj2W2Rss). And while it's super easy to [get started](https://docker.com/tryit/), I wanted to explore in a bit more detail how I might integrate Docker into a [deployment pipeline](http://martinfowler.com/bliki/DeploymentPipeline.html).

I am writing this from the perspective of a Mac OS X user; Windows users can likewise make use of boot2docker, and Linux users with native Docker can simply ignore the boot2docker setup parts.

We are going to convert a sample Java micro-service application packaged as a standalone JAR into a ready-to-run Docker container. Even if you don't plan on developing JVM-based applications,the same process applies to any Linux-based application stack.

You will need the following tools to run everything locally:

 * git
 * gradle
 * docker
 * boot2docker
 * virtualbox

Mac OS X users can install all of the above via [Homebrew](http://brew.sh) (all except VirtualBox) / [Cask](http://caskroom.io) (for VirtualBox).

Since Mac OS X can't host Docker containers natively, we'll use the boot2docker tooling and Linux distribution to run our containers in a virtual machine. The first time it runs, boot2docker will download the latest VM image so this could take a bit:

    $ boot2docker init
    $ boot2docker start
    $ $(boot2docker shellinit)

The last command exposes the hosted Docker runtime's endpoint to the docker command line tool running in the Mac environment. (Try run just ```boot2docker shellinit``` to see the exported variable. If you want to open multiple terminals/shell tabs, you'll need to execute this last command in each of them to let the ```docker``` utility know where to find the container host.) At this point you should be able to run ```docker info``` to confirm that everything is running. For more info see the [boot2docker usage page](https://docs.docker.com/installation/mac/).

Next, clone the [sample code repository](https://github.com/pcholakov/docker-spring-boot-primer) (which is largely derived from the [Spring Boot Getting Started](http://spring.io/guides/gs/spring-boot/) sample):

    $ git clone https://github.com/pcholakov/docker-spring-boot-primer.git

A Docker container's file system is built up using a [layered approach](https://docs.docker.com/terms/layer/). A base image is grown by successive layers which add something new. Every layer can be individually snapshotted and shared. Thus we could have a base operating environment, extended with a customised application runtime environment, extended with a particular release of our application. The fact that individual layers are versioned allows us to achieve better repeatability while still making it easy to update the lower layers when the need arises.

![Illustration &copy; Docker, https://docs.docker.com/terms/layer/](docker-filesystems-multilayer.png)

While you could start from absolute scratch, we will begin with the [phusion/baseimage-docker](http://phusion.github.io/baseimage-docker/) stripped down Ubuntu-derived image. Think of this as a VM base image where the kernel is already provided by the Docker host -- this just gives us a minimalist userland environment. To build our first Docker image:

    $ cd docker-java-base && docker build -t openjdk7-base .

This executes the first [Dockerfile](https://docs.docker.com/reference/builder/), found in ```docker-java-base```:

	FROM phusion/baseimage
	RUN apt-get update && apt-get install -y openjdk-7-jdk

All this does is tells Docker to begin with the phusion base image, then install OpenJDK 7 using APT, and commit the result to a new image tagged "openjdk7-base". During this step you could install an alternative application stack, e.g. Rails, MEAN, Node, and generally customise the userland environment just how your application expects it. This container only need to be rebuilt if we perform some major updates to our platform, say to apply a security patch, upgrade the JVM version, or switch to a newer Ubuntu LTS release.

Next, we will build the Spring Boot sample application:

    $ cd app && gradle build

This step will produce a standalone JAR inside of ```app/build/libs/```. In a typical deployment pipeline this JAR might be deployed onto some pre-existing infrastructure using techniques varying from simple shell scripting to DevOps automation tools like Ansible. However we want to provide a fully self-sufficient container with our app inside it, ready to run in any Docker-compatible environment. So while still in the ```app``` directory, run:

    $ docker build -t app-0.1 .

This builds a third layer (phusion/baseimage -> openjdk7-base -> app-0.1), using the Dockerfile found in ```app```:

	FROM openjdk7-base
	ENV HOME /root
	RUN /etc/my_init.d/00_regen_ssh_host_keys.sh
	CMD ["/sbin/my_init"]
	EXPOSE 8080
	COPY ../app/build/libs/gs-spring-boot-0.1.0.jar /opt/
	ENTRYPOINT ["/usr/bin/java", "-jar", "/opt/gs-spring-boot-0.1.0.jar"]

This Dockerfile is a little more complex than the first but all it says is that we wish to bundle the JAR output of the Gradle build step, launch it using the ```java``` command whenever the container runs, and declares that it exposes a service on TCP port 8080. All that is left to do is run the containerised application:

	$ docker run -i -p 8080 -t app-0.1

Since this runs the container in interactive mode, you should see the Spring boot startup messages. Note that even though the Dockerfile already declares port 8080 as exposed, we have to tell Docker explicitly to publish this port from within the Docker host with the ```-p``` switch. The application is running within the Docker host VM, in its own isolated container. We can check the details from another local shell (remember to repeat ```$(boot2docker shellinit)```) using:

	$ docker ps
	CONTAINER ID        IMAGE               COMMAND                CREATED             STATUS              PORTS
	8cab6ecbbd6e        app-0.1:latest      "/usr/bin/java -jar    52 seconds ago      Up 51 seconds       0.0.0.0:49154->8080/tcp

How can we locate the application's endpoint? Remember that sicne this runs in a Linux VM under VirtualBox, we have to connect to its unique IP address which we can find from:

	$ boot2docker ip
	The VM's Host only interface IP address is: 192.168.59.103

We can now combine the boot2docker IP with the Docker-assigned dynamic port, and send an HTTP request to the application:

	$ curl http://192.168.59.103:49154
	Greetings from Spring Boot!

And thus we have built a containerised version of our app, which is ready to ship to any compliant hosting environment. One downside is that the app container image is rather chunky compared to the base from which we started:

	$ docker images
	REPOSITORY          TAG                 IMAGE ID            CREATED             VIRTUAL SIZE
	app-0.1             latest              eff211e85165        3 hours ago         666.1 MB
    openjdk7-base       latest              96aee1b66b01        4 hours ago         651.8 MB
    phusion/baseimage   latest              cf39b476aeec        9 days ago          289.4 MB

However, a Docker host running many applications based on e.g. the same openjdk7-base image only needs to store the deltas, thus allowing us to pack apps a lot more densely on the same infrastructure while retaining some degree of isolation between them.

The major advantage that Docker gives us is that we have complete control over the execution environment inside the container and complete consistency from development laptops all the way to production environments. This is not without its caveats -- see this [Hacker News thread about Docker](https://news.ycombinator.com/item?id=8167928) for some real-world experiences. Whether this approach will gain much traction over virtualisation remains to be seen.

I hope that this post has given you a good overview of the steps involved in integrating Docker into your own deployment pipeline.
