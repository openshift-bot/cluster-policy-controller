## SDN solutions for Openshift

# WORK IN PROGRESS
# DO NOT USE THIS YET

#### Build and Install

	$ git clone https://github.com/openshift/openshift-sdn
	$ cd openshift-sdn
	$ make clean        # optional
	$ make              # build
	$ make install      # installs in /usr/bin

#### Try it out

##### Use vagrant, pre-define a cluster, and bring it up

	$ export OPENSHIFT_DEV_CLUSTER=1
	$ export OPENSHIFT_NUM_MINIONS=2
	$ export OPENSHIFT_SDN=ovs-simple
	$ vagrant up

##### Manually add minions to a master

Steps to create an OpenShift cluster with openshift-sdn. This requires that each machine (master, minions) have built openshift already. Check [here](https://github.com/openshift/origin).

On OpenShift master,

	$ openshift start master  # start the master openshift server
	$ openshift-sdn           # assumes etcd is running at localhost:4001

On OpenShift node,

	$ openshift-sdn -etcd-endpoints=http://openshift-master:4001 -minion -public-ip=<10.10....>
	where, 
		-etcd-endpoints	: reach the etcd db here
		-minion 	: run it in minion mode (will watch etcd servers for new minion subnets)
		-public-ip	: use this field for suggesting the publicly reachable IP address of this minion
		-hostname	: the name that will be used to register the minion with openshift-master
	$ openshift start node --master=http://openshift-master:8080

Back on the master,

	Create a json file for the new minion resource
        $ cat <<EOF > mininon-1.json
	{
		"kind":"Minion", 
		"id":"openshift-minion-1",
	 	"apiVersion":"v1beta1"
	}
	EOF
	where, openshift-minion-1 is a hostname that is resolvable from the master (or, create an entry in /etc/hosts and point it to the public-ip of the minion).
	$ openshift cli create -f minion-1.json

Done. Create new pods from the master (or just docker containers on the minions), and see that the pods are indeed reachable from each other.

#### Performance Note

The current design has a long path for packets directed for the overlay network.
There are two veth-pairs, a linux bridge, and then the OpenVSwitch, that cause a drop in performance of about 40%

Hand-crafted solutions that eliminate the long-path to just a single veth-pair bring the performance close to the wire. The performance has been measured using sockperf.

    $ docker exec -it openshift-origin bash
    $ ln -s /var/lib/openshift/openshift.local.certificates/admin/.kubernetes_auth $HOME/.kubernetes_auth
    $ osc --help

#### TODO

 - Add more options, so that users can choose the subnet to give to the cluster. The default is hardcoded today to "10.1.0.0/16"
 - Performance enhancements, as discussed above
 - Usability without depending on openshift

You can develop [locally on your host](CONTRIBUTING.adoc#develop-locally-on-your-host) or with a [virtual machine](CONTRIBUTING.adoc#develop-on-virtual-machine-using-vagrant), or if you want to just try out OpenShift [download the latest Linux server, or Windows and Mac OS X client pre-built binaries](CONTRIBUTING.adoc#download-from-github).

First, **get up and running with the** [**Contributing Guide**](CONTRIBUTING.adoc).

Once setup with a Go development environment and Docker, you can:

1.  Build the source code

        $ make clean build

2.  Start the OpenShift server

        $ make run

3.  In another terminal window, switch to the directory and start an app:

        $ cd $GOPATH/src/github.com/openshift/origin
        $ _output/local/go/bin/openshift cli create -f examples/hello-openshift/hello-pod.json

In your browser, go to [http://localhost:6061](http://localhost:6061) and you should see 'Welcome to OpenShift'.


### What's Just Happened?

The example above starts the ['openshift/hello-openshift' Docker image](https://github.com/openshift/origin/blob/master/examples/hello-openshift/hello-pod.json#L11) inside a Docker container, but managed by OpenShift and Kubernetes.

* At the Docker level, that image [listens on port 8080](https://github.com/openshift/origin/blob/master/examples/hello-openshift/hello_openshift.go#L16) within a container and [prints out a simple 'Hello OpenShift' message on access](https://github.com/openshift/origin/blob/master/examples/hello-openshift/hello_openshift.go#L9).
* At the Kubernetes level, we [map that bound port in the container](https://github.com/openshift/origin/blob/master/examples/hello-openshift/hello-pod.json#L13) [to port 6061 on the host](https://github.com/openshift/origin/blob/master/examples/hello-openshift/hello-pod.json#L14) so that we can access it via the host browser.
* When you created the container, Kubernetes decided which host to place the container on by looking at the available hosts and selecting one with available space.  The agent that runs on each node (part of the OpenShift all-in-one binary, called the Kubelet) saw that it was now supposed to run the container and instructed Docker to start the container.

OpenShift brings all of these pieces (and the client) together in a single, easy to use binary.  The following examples show the other OpenShift specific features that live above the Kubernetes runtime like image building and deployment flows.


### Next Steps

We highly recommend trying out the [OpenShift walkthrough](https://github.com/openshift/origin/blob/master/examples/sample-app/README.md), which shows some of the lower level pieces of of OpenShift that will be the foundation for user applications.  The walkthrough is accompanied by a blog series on [blog.openshift.com](https://blog.openshift.com/openshift-v3-deep-dive-docker-kubernetes/) that goes into more detail.  It's a great place to start, albeit at a lower level than OpenShift 2.

Both OpenShift and Kubernetes have a strong focus on documentation - see the following for more information about them:

* [OpenShift Documentation](http://docs.openshift.org/latest/welcome/index.html)
* [Kubernetes Getting Started](https://github.com/GoogleCloudPlatform/kubernetes/blob/master/README.md)
* [Kubernetes Documentation](https://github.com/GoogleCloudPlatform/kubernetes/blob/master/docs/README.md)

You can see some other examples of using Kubernetes at a lower level - stay tuned for more high level OpenShift examples as well:

* [Kubernetes walkthrough](https://github.com/GoogleCloudPlatform/kubernetes/tree/master/examples/walkthrough)
* [Kubernetes guestbook](https://github.com/GoogleCloudPlatform/kubernetes/tree/master/examples/guestbook)
* [OpenShift guestbook template](https://github.com/openshift/origin/blob/master/examples/guestbook) is an example of how entire applications can be packaged together.


### Troubleshooting

If you run into difficulties running OpenShift, start by reading through the [troubleshooting guide](https://github.com/openshift/origin/blob/master/docs/debugging-openshift.md).


API
---

The OpenShift APIs are exposed at `https://localhost:8443/osapi/v1beta1/*`.

* Builds
 * `https://localhost:8443/osapi/v1beta1/builds`
 * `https://localhost:8443/osapi/v1beta1/buildConfigs`
 * `https://localhost:8443/osapi/v1beta1/buildLogs`
 * `https://localhost:8443/osapi/v1beta1/buildConfigHooks`
* Deployments
 * `https://localhost:8443/osapi/v1beta1/deployments`
 * `https://localhost:8443/osapi/v1beta1/deploymentConfigs`
* Images
 * `https://localhost:8443/osapi/v1beta1/images`
 * `https://localhost:8443/osapi/v1beta1/imageRepositories`
 * `https://localhost:8443/osapi/v1beta1/imageRepositoryMappings`
* Templates
 * `https://localhost:8443/osapi/v1beta1/templateConfigs`
* Routes
 * `https://localhost:8443/osapi/v1beta1/routes`
* Projects
 * `https://localhost:8443/osapi/v1beta1/projects`
* Users
 * `https://localhost:8443/osapi/v1beta1/users`
 * `https://localhost:8443/osapi/v1beta1/userIdentityMappings`
* OAuth
 * `https://localhost:8443/osapi/v1beta1/accessTokens`
 * `https://localhost:8443/osapi/v1beta1/authorizeTokens`
 * `https://localhost:8443/osapi/v1beta1/clients`
 * `https://localhost:8443/osapi/v1beta1/clientAuthorizations`

The Kubernetes APIs are exposed at `https://localhost:8443/api/v1beta1/*`:

* `https://localhost:8443/api/v1beta1/pods`
* `https://localhost:8443/api/v1beta1/services`
* `https://localhost:8443/api/v1beta1/replicationControllers`
* `https://localhost:8443/api/v1beta1/operations`

OpenShift and Kubernetes integrate with the [Swagger 2.0 API framework](http://swagger.io) which aims to make it easier to document and write clients for RESTful APIs.  When you start OpenShift, the Swagger API endpoint is exposed at `https://localhost:8443/swaggerapi`. The Swagger UI makes it easy to view your documentation - to view the docs for your local version of OpenShift start the server with CORS enabled:

    $ openshift start --cors-allowed-origins=.*

and then browse to http://openshift3swagger-claytondev.rhcloud.com (which runs a copy of the Swagger UI that points to localhost:8080 by default).  Expand the operations available on v1beta1 to see the schemas (and to try the API directly).


FAQ
---

1. How does OpenShift relate to Kubernetes?

    OpenShift embeds Kubernetes and adds additional functionality to offer a simple, powerful, and
    easy-to-approach developer and operator experience for building applications in containers.
    Kubernetes today is focused around composing containerized applications - OpenShift adds
    building images, managing them, and integrating them into deployment flows.  Our goal is to do
    most of that work upstream, with integration and final packaging occurring in OpenShift.  As we
    iterate through the next few months, you'll see this repository focus more on integration and
    plugins, with more and more features becoming part of Kubernetes.

    OpenShift tracks the Kubernetes upstream at
    [github.com/openshift/kubernetes](https://github.com/openshift/kubernetes).  See the wiki in
    that project for more on how we manage the process of integrating prototyped features.

2. What about [geard](https://github.com/openshift/geard)?

    Geard started as a prototype vehicle for the next generation of the OpenShift node - as an
    orchestration endpoint, to offer integration with systemd, and to prototype network abstraction,
    routing, SSH access to containers, and Git hosting.  Its intended goal is to provide a simple
    way of reliably managing containers at scale, and to offer administrators tools for easily
    composing those applications (gear deploy).

    With the introduction of Kubernetes, the Kubelet, and the pull model it leverages from etcd, we
    believe we can implement the pull-orchestration model described in
    [orchestrating geard](https://github.com/openshift/geard/blob/master/docs/orchestrating_geard.md),
    especially now that we have a path to properly
    [limit host compromises from affecting the cluster](https://github.com/GoogleCloudPlatform/kubernetes/pull/860).  
    The pull-model has many advantages for end clients, not least of which that they are guaranteed
    to eventually converge to the correct state of the server. We expect that the use cases the geard
    endpoint offered will be merged into the Kubelet for consumption by admins.

    systemd and Docker integration offers efficient and clean process management and secure logging
    aggregation with the system.  We plan on introducing those capabilities into Kubernetes over
    time, especially as we work with the Docker upstream to limit the impact of the Docker daemon's
    parent child process relationship with containers, where death of the Docker daemon terminates
    the containers under it

    Network links and their ability to simplify how software connects to other containers is planned
    for Docker links v2 and is a capability we believe will be important in Kubernetes as well ([see issue 494 for more details](https://github.com/GoogleCloudPlatform/kubernetes/issues/494)).

    The geard deployment descriptor describes containers and their relationships and will be mapped
    to deployment on top of Kubernetes.  The geard commandline itself will likely be merged directly
    into the `openshift` command for all-in-one management of a cluster.


Contributing
------------

All contributions are welcome - OpenShift uses the Apache 2 license and does not require any contributor agreement to submit patches.  Please open issues for any bugs or problems you encounter, ask questions on the OpenShift IRC channel (#openshift-dev on freenode), or get involved in the [Kubernetes project](https://github.com/GoogleCloudPlatform/kubernetes) at the container runtime layer.

See [HACKING.md](https://github.com/openshift/origin/blob/master/HACKING.md) for more details on developing on OpenShift including how different tests are setup.

If you want to run the test suite, make sure you have your environment from above set up, and from the origin directory run:

```
# run the unit tests
$ make check

# run a simple server integration test
$ hack/test-cmd.sh

# run the integration server test suite
$ hack/test-integration.sh

# run the end-to-end test suite
$ hack/test-end-to-end.sh

# run all of the tests above
$ make test
```

You'll need [etcd](https://github.com/coreos/etcd) installed and on your path for the integration and end-to-end tests to run, and Docker must be installed to run the end-to-end tests.  To install etcd you should be able to run:

```
$ hack/install-etcd.sh
```

Some of the components of OpenShift run as Docker images, including the builders and deployment tools in `images/builder/docker/*` and 'images/deploy/*`.  To build them locally run

```
$ hack/build-images.sh
```


License
-------

OpenShift is licensed under the [Apache License, Version 2.0](http://www.apache.org/licenses/).
