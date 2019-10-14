# CloudBees Core Managed Master for Field Led Workshops
## Custom Managed Master Container (Docker) Image
This repository provides an example for creating a custom container (Docker) image to use as a [Managed Master](https://go.cloudbees.com/docs/cloudbees-core/cloud-admin-guide/operating/#managing-masters) *Docker image template* to be provisioned by CloudBees Core Operations Center running on Kubernetes. 

The image is configured to skip the Jenkins 2 Setup Wizard, install all of the CloudBees recommended plugins (minus a few) and some additional plugins typically used by CloudBees SAs for demos and workshops, and auto-configure the Jenkins instances. This *config-as-code* results in a streamlined CloudBees Core Managed Master provisioning process.

## Config-as-Code and GitOps for CloudBees Core Managed Masters
This project uses the original `install-plugins.sh`, that has been part of the OSS Jenkins Docker image since 2014, for plugin-management-as-code. This is combined with a mix of JCasC and Jenkins Groovy init scripts to manage all Jenkins configuration as code.

### JCasC (Jenkins Configuration as Code)
The Jenkins JCasC plugin provides CasC stored as a yaml file. Not all configurations are currently supported by the JCasC plugin - Groovy init scripts are used when there is not support with JCasC - see below for some examples.

The JCasC yaml for CloudBees Field Workshop Managed Masters is managed as part of [the JCasC configuration for the Field Workshop CloudBees Operations Center](https://github.com/kypseli/cb-core-oc-workshop/blob/master/k8s/casc.yml#L48). The reason for this is to make it easier for individual Managed Masters to use different JCasC configuration. An example of overriding 

### Dockerfile
- The `Dockerfile` starts with a `FROM` value of the CloudBees Core Managed Master Docker image: `cloudbees/cloudbees-core-mm`. 
- The `RUN /usr/local/bin/install-plugins.sh $(cat plugins.txt)` command installs all the plugins.
- The `config-as-code.yml` file provides Configuration-as-Code via [the Jenkins CasC plugin](https://github.com/jenkinsci/configuration-as-code-plugin).
- The `quickstart` scripts further modifies the Managed Master configuration using `init.groovy.d` scripts for configuration not currently supported by the CasC plugin.

#### Proxy Support for Installing Plugins via `install-plugins.sh`
Customize the Docker file to add the proxy environment values. 

Add the following `ARGs` and `ENVs` to your Dockerfile:
```Dockerfile
ARG http_proxy
ARG https_proxy
ARG no_proxy
ENV http_proxy=$http_proxy
ENV https_proxy=$https_proxy
ENV no_proxy=$no_proxy
```
 
Pass them into the build like this:

```shell
docker build --build-arg http_proxy --build-arg https_proxy --build-arg no_proxy -t cb-core-mm-master .
```

This will allow automatically picking up the proxy values set in your environment.  

#### Plugins installed:
See the [`plugins.txt`](plugins.txt) file to see all the plugins that get installed - some *non-CJE standard plugins* highlights include:

- [Blue Ocean with the Blue Ocean Pipeline Editor](https://jenkins.io/doc/book/blueocean/)
- [Pipeline Utilities plugin](https://jenkins.io/doc/pipeline/steps/pipeline-utility-steps/)
- [Jenkins Configuration as Code Plugin](https://github.com/jenkinsci/configuration-as-code-plugin)

Note, the `install-plugins.sh` script will download the specified plugins and their dependencies at build time and include them in the image; it also inspects the Jenkins WAR and skips any plugins already included by CloudBees (embedded in the WAR).

#### initilization scripts (Groovy init scripts that run on Jenkins post initialization)
##### Regular init scripts (on startup)
- `init_99_save.groovy`: Ensures any previous configuration changes are saved, sets flags not to re-run certain scripts; currently not much done here as we prefer to wait for the CJE license to be activated

##### License Activated scripts - utilizes the custom CloudBees `license-activated-or-renewed-after-expiration` hook that will be fired after your CJE license is activated
- `init_01_quickstart_hook.groovy`: This script is only used to trigger another custom groovy hook but only after the CloudBees license has been activated.

##### Quickstart scripts - a custom init groovy hook that fires after required plugins are installed and after a necessary restart
- `init_04_pipeline_model-def_config.groovy`: Configures the agent label to be used for Pipeline Declarative Docker syntax. This is not documented very well, but there is a global and per folder setting to tell Declarative Pipeline what Jenkins agent `label` to use when using the `` syntax - to ensure that the underlying Pipeline Model Definition will be able to spin up the Docker image to use *inside* the agent.
- [`init_10_global_flow_durability.groovy`](quickstart/init_10_global_flow_durability.groovy): Sets the global **Pipeline Speed/Durability Setting** to `PERFORMANCE_OPTIMIZED`, this may be overridden per Pipeline job or per-branch for a Pipeline Multibranch project. See [Scaling Pipelines](https://jenkins.io/doc/book/pipeline/scaling-pipeline/) for more details. We might as well use maximum performance since we are using the One Shot Executor Strategy for agents.



