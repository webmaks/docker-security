# docker-security
Advices to Docker security.


Youtube https://www.youtube.com/watch?v=rqsrmeLn65w 

## Security recommendations for Docker on Linux servers, in order of priority.

### First, research and learn

1. Watch my [DockerCon session on production Docker concerns](https://www.youtube.com/watch?v=V4f_sHTzvCI). This gives you a good baseline of production things before you dive into specific security concerns.
1. Read the [Docker security guide](https://docs.docker.com/engine/security/security/).
1. Read some blog posts on security tools and topics. These skims a lot of things I list here: https://blog.sqreen.io/docker-security/ and https://sysdig.com/blog/20-docker-security-tools/ 


### Then consider each of these a project to implement. Easier ones at top:
Note: I've marked each as something you do to the host config *or* something done in the Dockerfile for the container app itself. This is useful if you're someone who maybe can't control one or the other and only need the below items that are under your purview. (maybe you're a sysadmin and don't control Dockerfiles, or vice-versa).

1. `both` **Just use Docker.** Running an app in a default-settings Docker Linux container [greatly reduces your risk profile](https://docs.docker.com/engine/security/security/) vs. just running that app on a full Linux VM OS. It applies default apparmor and seccomp profiles, and also uses a small whitelist of Linux Kernel capabilities.
1. `host` **Scan your hosts for proper Docker config.** Run https://github.com/docker/docker-bench-security and see if you can improve things with your setup. It checks the host config and how Docker is installed and gives you red/green metrics. Note the goal isn't to have all green, but to understand the reds you have and decide if you're OK with that.
1. `host` **Don't Expose the Docker TCP Socket to the Internet** - This is hard to get right, and mostly ends in tears (see below comments). As of 2018 you can now use docker's built-in SSH connection method to remotely control the Docker Engine without exposing anything other than SSH. This is perfect for remote docker admin and CI/CD solutions. Here's a [demo of using it with the `DOCKER_HOST=ssh://user@host` environment variable](https://youtu.be/5EuDY6ayNs8?t=382), and [here's a demo of using it with the new `docker context`](https://youtu.be/fJTaUvcCJts?t=1621) in 2019. Note that most tools that need the Docker socket (Portainer, Traefik) have a way to do it without exposing the TCP port to remote connections. Once you have higher-level admin tools like web admins, you now need to protect them, as they become the target. To protect them, use VPN-only or private network access, high random port, or add a stronger web proxy auth in front of them.
1. `Dockerfile` **Don't run apps in containers as root.** Use `USER <appname>` to prevent your app from running as the root user in the container. Often there is already a user created in official images, but just not enabled until you add the `USER` line in your own downstream Dockerfile.
1. `both` **App and OS dependency scanning.** You want to scan for [CVE's](https://cve.mitre.org/) in both your app dependencies (npm, pip, composer, etc.) and OS dependency's (apt, apk, yum). There are three main stages where you can do this: git commits, image build, and after image build:
    - `git` For app dependency-only CVE scanning in your git repos, use something like [Snyk](https://snyk.io/) (or just GitHub built-in) to scan your git repos for known vulnerabilities. This is great for [shift-left security](https://www.twistlock.com/resources/shift-left-security/).
    - `CI` Scan during image build. Aqua Security [Microscanner](https://github.com/aquasecurity/microscanner) does OS CVE scanning inside a container build, which I think is very handy, but doesn't look for app dependencies and doesn't have the full Alpine support if you need it. Compliments the git-repo scanning above.
    - `CI` Scan images after build or on servers. Docker Hub used to do this for a fee, now you'll need Docker EE or 3rd party. I think the best open-source scanner right now might be Aqua Security's [Trivy](https://github.com/aquasecurity/trivy#os-packages) which claims to have the most complete scanner, especially for Alpine-based images. It scans images from outside Docker.  This could replace the above two methods.
1. `Dockerfile` **Use Slimmed-Down Base Images**, which will set off less CVE's when scanning images, and prevent attack vectors that could potentially use the vulnerable software that wasn't included in the larger, more "full-featured" base images. There are several downsides to doing this and there are several choices in choosing your base.
    - Smaller base images mean less utilities are included, and that usually means that the smaller the base is, the more skilled you need to be at managing/using containers. What do you do when your container needs troubleshooting, but it doesn't have curl or ping? Or even a shell? There are ways to deal with this limitation, but it's more than just `docker exec`.
    - A `scratch` (empty) or `busybox` image is the smallest. I see very few able to get away with those.
    - Alpine is a popular choice, and most official images have a version using alpine, but I've seen specific issues in rare cases by choosing Alpine over the default Debian, and I only know of one CVE scanner that correctly handles it: [trivy](https://github.com/aquasecurity/trivy), and if you're forced to use a different scanner, then I recommend comparing results to trivy to ensure it does what the scanner maker claims.
    - Slim is an option that some official images have, and I prefer it. It's just a bit larger than Alpine, but is Debian-based so that doesn't have the limitations of Alpine.
1. `Dockerfile` **Use Multi-Stage Images To Prevent Dev and CI Dependencies In Production**, which can add complexity to your Dockerfile, but is a big win once you get it right. The [basics of multi-stage are in the Docs](https://docs.docker.com/develop/develop-images/multistage-build/), but the key is to keep the same code and prod dependencies in each Stage, while layering on dev or CI testing dependencies only when they are needed, all-the-while never touching the underlying production Stage (layer). I show a step-by-step example of this for Node.js, which can be adapted to most languages, in my [DockerCon19 talk repo](https://github.com/BretFisher/dockercon19).
1. `host` **Enable "[user namespaces](https://integratedcode.us/2015/10/13/user-namespaces-have-arrived-in-docker/)"** so even the root user in a container isn't really root on the host. (so even if a hacker broke out of the container they are just limited user on host OS.) This won't work in all use cases, so consider having different sets of servers where the high-security set has user namespaces enabled and runs all your apps that still work in that setup. Then run the rest on the default Docker config on a different set of servers.
1. `host` **Runtime Bad Behavior Monitoring** [Sysdig Falco](https://sysdig.com/opensource/falco/) has the great idea of watching your containers for unusual behavior, like an `exec` command, or bind-mounting sensitive host files in `/etc`, or something trying to mess with certain files *in* a container... it will watch the system and log these actions (based on a default set of rules that you can add to). You can run this small binary on every host, and have it log to a central location.
1. `both` **Content Trust**, which has to do with signing your code, and then the images all the way through the CI/CD pipeline and finally making certain your servers won't deploy new containers on images you didn't cryptographically trust. [Docker does this](https://docs.docker.com/engine/security/trust/content_trust/), Docker Enterprise makes it easier, also other platforms may do this.
1. `both` **Later, check out AppArmor, SELinux, Seccomp, and Linux "capabilities"** for locking down what a container can access/do on host. There are already defaults running in Docker, but they can be customized per container if you want that next-level hardening. It's something you'll need to customize for each image you want to run, so I don't recommend them as a first step, but good for systems you really want to lockdown.
1. `host` **Docker root-less.** This is an option for running the dockerd daemon as a normal user on the host. It's new in 2019 and if you already do all the above, it may be more work then it's worth to use this, but it can also be handy when you don't have local admin, or you're interested in CI automation without local admin. See [Docker's blog post](https://engineering.docker.com/2019/02/experimenting-with-rootless-docker/). Note the biggest limitation is lack of virtual networking (which still requires root). [Watch this DockerCon session on it](https://www.youtube.com/watch?v=Qq78zfXUq18), and install it with a script as a normal Linux user https://get.docker.com/rootless
1. `host` Besides Docker EE, there are 3rd party platforms that cover many of these features like https://www.blackducksoftware.com ,  https://www.aquasec.com/, https://sysdig.com/, and https://www.twistlock.com/

_Originally posted by @BretFisher in https://github.com/BretFisher/ama/issues/17#issuecomment-400163774_
