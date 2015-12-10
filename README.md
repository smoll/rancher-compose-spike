## Spiking [Rancher Compose](https://github.com/rancher/rancher-compose)

Doing this on a Vagrant-based setup for testing purposes, but this is easily extrapolated to an AWS EC2-based environment. There's even an [Ansible playbook](https://github.com/joshuacox/ansibleplaybook-rancher) if we want finer-grained control over agent provisioning (over just using the UI).

### Setup

#### One-time

0. I started a Vagrant-based Rancher server by cloning [rancher/rancher](https://github.com/rancher/rancher) and doing a `vagrant up` and was able to access the UI at http://172.19.8.100:8080/
  0. Had to [hack the vagrant plugin](https://github.com/rancher/rancher/issues/2129#issuecomment-162847782) to get it to work. A subsequent comment to the issue indicates the problem is fixed on the `Vagrantfile` now. BTW, we shouldn't run into this issue at all if using a non-Vagrant-based Rancher server.

0. I set up a new environment titled "Staging" and renamed the default env to "Dev" to test out the separation of envs.

0. I set up ACL using GitHub authentication, and registered it as a Developer app (very easy step-by-step directions from the Rancher UI).

0. I added my first host manually (Vagrant-based) using the copy+paste commands from the UI, but they provide a way to bootstrap a [EC2-based host](http://172.19.8.100:8080/infra/hosts/add/amazonec2) by supplying AWS creds in the UI. Note that I did *not* use a RancherOS-based box, it seems to work fine with an Ubuntu/Debian/CentOS box.

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
    # Set the url that Rancher is on
    $ export RANCHER_URL=http://172.19.8.100:8080/
    # Set the access key, i.e. username
    $ export RANCHER_ACCESS_KEY=2C28400B70260B8A3542
    # Set the secret key, i.e. password
    $ export RANCHER_SECRET_KEY=GiYWRoNTz2jv79xU3WHknkcuHYeDgJDtEE8k31ih
    ```

0. Make a code change to the version-controlled compose files, e.g. change `smoll/flask-nanoservice:latest` to `smoll/flask-nanoservice:1.0`

0. Run the `rancher-compose` command to up the service(s):

    ```
    rancher-compose -p stacksonstacks rm --force # hard kill any running containers
    rancher-compose -p stacksonstacks up # bring up containers with new config
    ```

### TODO: test failover

0. Add 2 more hosts

0. Scale up to 2 instances of the app (should now live on `host1` and `host2`)

0. Suspend 1 of the 2 hosts running an app container

0. Check

    0. that no requests were dropped
    0. the time it takes for a new instance of the app container to be launched on host 3
