## Dockerfile Best Practices


### General Guidelines

* Containers should be as ephemeral as possible
* Avoid installing unnecessary packages
* Minimise concerns 
   * Avoid multiple processes/apps in one container


### `.dockerignore`
* A text file called `.dockerignore`
* Top level of your project
* Very similar to <code>.gitignore</code>
* `COPY . dest/` will not copy files ignored in `.dockerignore`
* Include things you don't want in your image:
   * <code>.git</code> directory
   * <code>node_modules</code>, <code>virtualenv</code> directories


#### Sample `.dockerignore` File

```
.git*
.dockerignore
Dockerfile
README*
# don't import python virtualenv
.venv
```


### Best Practices for Images

* Use current official repositories in FROM as base image
* Image size may be a factor on cloud hosts where space is limited
   * debian 124 MB
   * ubuntu 117 MB
   * alpine 3.99 MB
   * busybox 1.11 MB
* Choice of image depends on other factors



### Layer Caching
```
cd ~/docker-introduction/sample-code/caching
docker build -t caching-example -f Dockerfile.layering . 
```

* Build image in <!-- .element: class="fragment" data-fragment-index="0" --><code>sample-code/caching</code> directory
* Run build a second time. What happens? <!-- .element: class="fragment" data-fragment-index="1" -->
* Change line with Change me! and run again <!-- .element: class="fragment" data-fragment-index="2" -->
* Each instruction creates a layer in an image <!-- .element: class="fragment" data-fragment-index="3" -->
* Docker caches layers when building <!-- .element: class="fragment" data-fragment-index="4" -->
* When a layer is changed Docker rebuilds from changed layer <!-- .element: class="fragment" data-fragment-index="5" -->



### Layer Caching
<div style="width:100%">
<div style="float:left;width:50%">
<pre style="font-size:14pt;"><code class="dockerfile" data-trim>
\# Example 1
FROM ubuntu:latest
RUN apt-get update
RUN apt-get install -y curl
\#RUN apt-get install -y nginx
</code></pre>
</div>
<div style="float:left;width:50%">
<pre style="font-size:14pt;"><code class="dockerfile" data-trim>
\# Example 2
FROM ubuntu:latest
RUN apt-get update \
  && apt-get install -y curl #nginx
</code></pre>
</div>
</div>
* Example 1: `apt-get update` does not refresh index
   * apt repos might change
* Best to combine apt-get update and install packages to force apt to refresh index (Example 2)



### Optimising Image Size

* Image size is sum of intermediate layers <!-- .element: class="fragment" data-fragment-index="0" -->
* Even if you remove something it exists as a diff on previous layer <!-- .element: class="fragment" data-fragment-index="2" -->
* Run clean up in same layer whenever possible <!-- .element: class="fragment" data-fragment-index="3" -->


#### Example: Optimising Image Size

<div  class="fragment" data-fragment-index="0">
    <pre style="font-size:10pt;"><code
          data-noescape>FROM ubuntu:latest <mark class="fragment" data-fragment-index="1">112MB</mark>

RUN apt-get update \ 
&& apt-get install -y \
automake \
build-essential \
curl \
wget \
libcap-dev \
reprepro         <mark class="fragment" data-fragment-index="2">284MB</mark>
RUN rm -rf /var/lib/apt/lists/\* <mark class="fragment" data-fragment-index="3">0MB</mark>

ADD https://dl.google.com/android/android-sdk_r24.4.1-linux.tgz .  <mark class="fragment" data-fragment-index="4">326MB</mark>    
RUN tar xf android-sdk_r24.4.1-linux.tgz  <mark class="fragment" data-fragment-index="5">678MB</mark>
RUN rm -f android-sdk_r24.4.1-linux.tgz <mark  class="fragment" data-fragment-index="6">0 MB</mark></code></pre>

<p class="fragment" data-fragment-index="7" style="font-size:19pt;">Image size: 1.4 GB</p>
</div>
<div  class="fragment" data-fragment-index="8">
    <pre style="font-size:10pt;"><code
          data-noescape>FROM ubuntu:latest <mark class="fragment" data-fragment-index="9">112MB</mark>

RUN apt-get update \ 
&& apt-get install -y \
automake \
build-essential \
curl \
wget \
libcap-dev \
reprepro \
&& rm -rf /var/lib/apt/lists/\* <mark class="fragment" data-fragment-index="10">244MB</mark>

RUN  wget https://dl.google.com/android/android-sdk_r24.4.1-linux.tgz && \     
   tar xf android-sdk_r24.4.1-linux.tgz && \     
   rm -f android-sdk_r24.4.1-linux.tgz <mark class="fragment"
    data-fragment-index="11">678 MB</mark></code></pre>

    <p class="fragment" data-fragment-index="12" style="font-size:19pt;">Image size: 1 GB</p>
</div>



### ADD

* Large intermediate layers
  ```
   ADD http://domain.com/big.tar.gz /usr/path/ # large intermediate layer
   RUN cd /usr/path && tar -xvf big.tar.gz \
      && rm big.tar.gz
  ```
* Increased overall image size


### Better to use COPY
* Better solution:
   ```
   RUN curl -SL http://domain.com/big.tar.gz  \ 
   | tar -xJC /usr/path

   ```
    * Smaller image size
* COPY only copies files <pre><code>COPY . /usr/path/</code></pre>
* Recommend to only use COPY and never ADD


### Multistage Builds

* Best practices intended to optimise image size by keeping them small
* Come at the expense of readability 
   * Layers with long complicated commands
* Multistage builds
   * Introduced with Docker 17.05
   * Enable optimised image size
   * maintain readability


### Multistage Builds

<div style="width:50%;float:left;">
    <ul>
        <li class="fragment" data-fragment-index="0">Multiple FROM directives in a Dockerfile</li>
        <li class="fragment" data-fragment-index="1">Each FROM represents a new build</li>
        <li class="fragment" data-fragment-index="2">Selectively copy artifacts from one of the previous builds</li>
        <li class="fragment" data-fragment-index="3">Leave behind what is not needed</li>
    </ul>
</div>

<div style="width:50%;float:left;">
<pre style="font-size:12pt;" class="fragment" data-fragment-index="4"><code data-trim>
FROM ubuntu:16.04 as builder
WORKDIR /bin
COPY . /bin/
RUN make install

FROM alpine
COPY --from=builder /bin/myprogram /root
ENTRYPOINT ['/root/myprogram'] </code></pre>
</div>


#### Example: Multistage Build
<pre style="font-size:14pt;"  class="fragment" data-fragment-index="0"><code data-trim>
cd ~/href-counter $ docker build -t href-counter -f Dockerfile.build .
$ docker image ls | grep href
REPOSITORY    TAG     IMAGE ID          SIZE
href-counter  latest  b0eb64a75c55      687MB
</code></pre>
<pre style="font-size:14pt;"  class="fragment" data-fragment-index="1"><code data-trim>$ docker build -t href-counter-multi -f Dockerfile.multi .
$ docker image ls | grep href
REPOSITORY          TAG           SIZE
href-counter-multi  latest        10.3MB
</code></pre>


### CMD & ENTRYPOINT revisited

* Avoid using <!-- .element: class="fragment" data-fragment-index="0" -->_shell_ form
   * <code>ENTRYPOINT "executable param1 param2 ..."</code>
* Docker directs POSIX commands at process with PID 1 <!-- .element: class="fragment" data-fragment-index="1" -->
* Using <!-- .element: class="fragment" data-fragment-index="2" -->_shell_ form, process is run internally using <code>/bin/sh -c</code> and do not have PID 1
* <!-- .element: class="fragment" data-fragment-index="3" -->It can be difficult to stop container since process does not receive SIGTERM from <code>docker stop container</code>


### Example: Shell vs Exec
```
cd ~/docker-introduction/sample-code/entrypoint_cmd_examples
$ docker build -t runtop-shell -f Dockerfile.top_shell .
$ docker run --rm --name topshell runtop-shell
```
What happens when you want to stop container
<em>topshell</em>?
                        


### Example: Shell vs Exec

* Best practice to use <!-- .element: class="fragment" data-fragment-index="0" --><strong>exec</strong> form: 
   * `CMD ["executable", "param1", "param2", ..]`
* Or in form that creates interactive shell like <!-- .element: class="fragment" data-fragment-index="1" -->
   * `ENTRYPOINT ["python"]`
   * `CMD ["/bin/bash"]`
   ```
   $ docker build -t runtop-exec -f Dockerfile.top_exec .
   $ docker run runtop-exec
   ```
* Sometimes app constraints don't allow single process on PID1 <!-- .element: class="fragment" data-fragment-index="2" -->
* For this purpose recommended to use <!-- .element: class="fragment" data-fragment-index="3" --> [dumb-init](https://github.com/Yelp/dumb-init)


### Summary

* Dockerfile best practices aim to
   * Keep image footprint small 
      * Sometimes at expense readability
      * Multistage builds are a good compromise
   * Maintain clean control over containers
      * Easy to top and start
      * Flexible in the way they are executed
* Official Dockerfile [best practices](https://docs.docker.com/develop/develop-images/dockerfile_best-practices/)
