## Spiking [Rancher Compose](https://github.com/rancher/rancher-compose)

Doing this on a Vagrant-based setup for testing purposes, but this is easily extrapolated to an AWS EC2-based environment. There's even an [Ansible playbook](https://github.com/joshuacox/ansibleplaybook-rancher) if we want finer-grained control over agent provisioning (over just using the UI).

**TL;DR Rancher lets you seamlessly start up an entire Docker-based stack via a clean UI, and Rancher Compose provides a familiar CLI interface to the platform**

### Concepts

**Rancher Server:** UI that talks to all the Rancher Agents. Log into the UI to do initial setup (create Environment(s), add Host(s) to each env, set up ACLs.)

**Rancher Agent:** Docker container installed on each Docker host, responsible for managing the entire lifecycle of service containers & load balancer containers.

**Environment:** separate hosts, ACLs for completely isolated "Dev", "Staging", and "Prod" envs.

**Stack:** multiple services, all linked together (for example, a flask app that talks to a DB). e.g. the entire `docker-compose.yml` Can link to an external service NOT managed by Rancher (e.g. some other AWS service), or even services in other stacks (using "external links").

**Service:** N instances of a particular app. e.g. `web:` in your `docker-compose.yml`, can scale transparently, and Rancher automatically locates these containers based on explicit scheduling rules.

There seems to be multiple ways to slice up services per stack, but the simplest thing to try at first might be to define all of the microservices in a single `docker-compose.yml`. Very loosely-coupled or completely isolated groups of services can have their own compose yml, though.

### Setup

#### One-time

0. I started a Vagrant-based Rancher server by cloning [rancher/rancher](https://github.com/rancher/rancher) and doing a `vagrant up` and was able to access the UI at http://172.19.8.100:8080/
  0. Had to [hack the vagrant plugin](https://github.com/rancher/rancher/issues/2129#issuecomment-162847782) to get it to work. A subsequent comment to the issue indicates the problem is fixed on the `Vagrantfile` now. BTW, we shouldn't run into this issue at all if using a non-Vagrant-based Rancher server.

0. I set up a new environment titled "Staging" and renamed the default env to "Dev" to test out the separation of envs.

0. I set up ACL using GitHub authentication, and registered it as a Developer app (very easy step-by-step directions from the Rancher UI).

0. I added my first host manually (Vagrant-based, hardcoded an IP of `192.168.50.101`) using the copy+paste commands from the UI, but they provide a way to bootstrap a [EC2-based host](http://172.19.8.100:8080/infra/hosts/add/amazonec2) by supplying AWS creds in the UI. Note that I did *not* use a RancherOS-based box, it seems to work fine with an Ubuntu/Debian/CentOS box.

0. Generate an [API key/secret pair](http://172.19.8.100:8080/settings/api) for use by CI server.

0. Add DNS entry for load balancer (in my case, just added Vagrant private IP to hosts file: `192.168.50.101 flask-nanoservice.examplez.com`)

#### Every new app

0. Create a [new stack using the UI](http://docs.rancher.com/rancher/rancher-ui/applications/stacks/), like `stacksonstacks`, using [this walkthrough as a guide](http://rancher.com/virtual-host-routing-using-rancher-load-balancer/).

0. Export the `docker-compose.yml` and `rancher-compose.yml` that is [automatically generated](http://172.19.8.100:8080/apps/1e3/code):

    ```
    # docker-compose.yml

    RancherLB:
      ports:
      - '80'
      labels:
        io.rancher.loadbalancer.target.flask-nanoservice: flask-nanoservice.examplez.com=5000
      tty: true
      image: rancher/load-balancer-service
      links:
      - flask-nanoservice:flask-nanoservice
    flask-nanoservice:
      labels:
        io.rancher.container.pull_image: always
      tty: true
      image: smoll/flask-nanoservice:latest
    ```

    ```
    # rancher-compose.yml

    RancherLB:
      scale: 1
      health_check:
        port: 42
        interval: 2000
        unhealthy_threshold: 3
        healthy_threshold: 2
        response_timeout: 2000
    flask-nanoservice:
      scale: 1
    ```

    And add it to version control, so it can be used by our CI server.

0. Verify app is working:

    ```
    while sleep 1; do curl http://flask-nanoservice.examplez.com; done
    ```

#### Every deploy (to Staging env)

0. On remote deployer machine a.k.a. Jenkins slave a.k.a. CI server build executor, set a bunch of environment variables so `rancher-compose` connects to the correct Rancher server.

    ```
    export RANCHER_URL=http://172.19.8.100:8080/
    export RANCHER_ACCESS_KEY=2C28400B70260B8A3542
    export RANCHER_SECRET_KEY=GiYWRoNTz2jv79xU3WHknkcuHYeDgJDtEE8k31ih
    ```

0. Make a code change to the version-controlled compose files, e.g. change `smoll/flask-nanoservice:latest` to `smoll/flask-nanoservice:1.0`

0. Run the `rancher-compose` command to up the service(s):

    ```
    $ rancher-compose -p stacksonstacks rm --force # hard kill any running containers

    INFO[0000] Project [stacksonstacks]: Deleting project
    INFO[0000] [0/2] [flask-nanoservice]: Deleting
    INFO[0000] [0/2] [RancherLB]: Deleting
    INFO[0000] [0/2] [RancherLB]: Deleted
    INFO[0000] [0/2] [flask-nanoservice]: Deleted

    $ time rancher-compose -p stacksonstacks up -d # bring up containers with new config

    INFO[0000] Creating service flask-nanoservice
    INFO[0000] Creating service RancherLB
    INFO[0001] Project [stacksonstacks]: Starting project
    INFO[0001] [0/2] [flask-nanoservice]: Starting
    INFO[0014] [1/2] [flask-nanoservice]: Started
    INFO[0014] [1/2] [RancherLB]: Starting
    INFO[0025] [2/2] [RancherLB]: Started

    real  0m25.189s
    user  0m0.113s
    sys 0m0.039s
    ```

    Can take a less brute-force approach and do a rolling deploy, but doing this for simplicity's sake. The `rancher-compose` CLI also lets you view aggregated logs with:

    ```
    $ rancher-compose -p stacksonstacks logs

    flask-nanoservice_1 | 2015-12-10T22:44:46.340549514Z  * Restarting with stat
    2015-12-10T22:44:46.494737305Z  * Debugger is active!
    2015-12-10T22:44:46.498820134Z  * Debugger pin code: 265-654-587
    2015-12-10T22:45:32.518299140Z 10.42.59.155 - - [10/Dec/2015 22:45:32] "GET / HTTP/1.1" 200 -
    2015-12-10T22:45:49.043567439Z 10.42.59.155 - - [10/Dec/2015 22:45:49] "GET / HTTP/1.1" 200 -
    2015-12-10T22:45:50.081264159Z 10.42.59.155 - - [10/Dec/2015 22:45:50] "GET / HTTP/1.1" 200 -
    flask-nanoservice_1 | 2015-12-10T22:45:51.120578062Z 10.42.59.155 - - [10/Dec/2015 22:45:51] "GET / HTTP/1.1" 200 -
    flask-nanoservice_1 | 2015-12-10T22:45:52.158090131Z 10.42.59.155 - - [10/Dec/2015 22:45:52] "GET / HTTP/1.1" 200 -
    flask-nanoservice_1 | 2015-12-10T22:45:53.196547216Z 10.42.59.155 - - [10/Dec/2015 22:45:53] "GET / HTTP/1.1" 200 -
    ```

### TODO: test failover

0. Add 2 more hosts

0. Scale up to 2 instances of the app (should now live on `host1` and `host2`)

0. Suspend 1 of the 2 hosts running an app container

0. Check

    0. that no requests were dropped
    0. the time it takes for a new instance of the app container to be launched on host 3
    0. what happens after host 1 eventually recovers

#### Other things to explore

* Multiple Rancher Servers (masters)

* [Monitoring](http://rancher.com/comparing-monitoring-options-for-docker-deployments/). There's some basic monitoring [built in](http://172.19.8.100:8080/infra/containers/1i13/labels) too.

* Logging. Rancher already provides built-in rudimentary (but real-time) log aggregation in the UI for all containers; this may be enough for our needs. Visualization in [Loggly](http://rancher.com/aggregating-application-logs-from-docker-containers-with-loggly/) is another possibility, though. [Sysdig](http://rancher.com/sysdig-announces-integrated-docker-monitoring-in-rancher/) too.
