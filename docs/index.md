# Overview

This guide will walk you through the process of deploying a sample Java application in CFT.

An application will be generated from a template and deployed to a Kubernetes cluster.

At the end of this tutorial, you will be able to access your application via the HMCTS VPN and will have made changes to it.

## Prerequisites

Before starting this tutorial, make sure you have:

- An account on HMCTS Azure AD. See the [onboarding guide](https://hmcts.github.io/onboarding/person/#azure-ad-groups) to get set up.
- Access to the HMCTS GitHub organisation. See the [onboarding guide](https://hmcts.github.io/onboarding/team/github.html#github) to get setup.
- (Recommended) Join the below Slack channels
  - [#golden-path](https://hmcts-reform.slack.com/app_redirect?channel=golden-path) is for community discussion about the tutorials.
  - [#labs-build-notices](https://hmcts-reform.slack.com/app_redirect?channel=labs-build-notices) Jenkins build notices channel.
  - [#platops-help](https://hmcts-reform.slack.com/app_redirect?channel=platops-help) is for raising support requests to the `Platform Operations` team.

## Steps

### Create a repository

First, we are going to create a GitHub repository with our template, you'll need to fill in a few fields and select a name for the GitHub repository.

Click `Create` in the Backstage sidebar and select [`Spring Boot Service`](https://backstage.platform.hmcts.net/create) template.

   Fill out the fields:

- Page 1 - Provide some simple information
  - Product:                       `labs`

  - Component:                     `YourGithubUsername` - make sure you update this

  - Slack contact channel:         `cloud-native`

  - Description:                   `Deploying a Java application`

  - HTTP port:                     `8080` - do not use any port number lower than 1024

  - GitHub admin team:             `hmcts/reform` - you can also use your own team

  - Owner:                         `dts_cft_developers` - normally you would use your teams AzureAD group here

- Page 2 - Choose a location
  - Host:                          `github.com`

  - Owner:                         `hmcts`

  - Repository:                    `labs-YourGithubUsername`

### Build application

1. Go to your GitHub repository - there should be a link to it in backstage. Your repository needs an appropriate topic added in order for Jenkins to find it when it scans GitHub. Since your repository is called labs-yourGitHubUsername, you need to add the topic `jenkins-cft-j-z`

1. Log in to Sandbox Jenkins and select [HMCTS - Labs](https://sandbox-build.platform.hmcts.net/job/HMCTS_Sandbox_LABS/) folder. Check if your repository is there, if it's not then scan the organization by clicking on `Scan Organization Now`.
The new repository should be listed under repositories after the scan finishes.
Logs can be monitored under `Scan Organization Log`.
Any GitHub repository that starts with `labs-*` and has the `jenkins-cft-j-z` topic will be listed as part of this scan.

1. Click on your repository name.

1. Wait for your build to complete against the `master` branch, (click the Play button if it hasn't started by itself).

### Configure load balancing for high availability

We load balance across Kubernetes clusters using [Azure Application Gateway](https://docs.microsoft.com/en-us/azure/application-gateway/overview).

You will need to add a couple of lines of config for the application to the load balancer config file.

1. Open [sbox/backend_lb_config.yaml](https://github.com/hmcts/azure-platform-terraform/blob/master/environments/sbox/backend_lb_config.yaml) in your browser.
1. Click the pencil button to edit the file in GitHub
1. Search for `#labs` and add your application to the section, see example below:

   ```yaml
   #labs
      - product: "labs"
        component:     # GitHub repository name without "labs" prefix, e.g. `YourGithubUsername`
   ```

1. Scroll down to the bottom to the 'Commit changes' section. Select `create a new branch for this commit` and give your branch a name. Commit the file and create a pull request.

1. To complete this section you will need your pull request to be approved, someone on your team should be able to do this.
If you get stuck try asking in [#platops-code-review](https://hmcts-reform.slack.com/app_redirect?channel=golden-path) on Slack.
Once approved and the build has passed then merge your pull request.
If you have a permissions issue then ask in [#golden-path](https://hmcts-reform.slack.com/app_redirect?channel=golden-path) on Slack.

### Deploy application

We use [GitOps](https://www.weave.works/technologies/gitops/) for application deployment to Kubernetes.

Your application will be deployed in `labs` Kubernetes namespace which has already been created.

For the benefit of of this tutorial we have created a separate [guide](https://github.com/hmcts/cnp-flux-config/blob/master/labs/README.md#creating-the-flux-config-for-your-lab-application) to help you create the flux config needed to deploy your lab application with flux and to enable flux to automate updating image tags to the latest version of your image.

It's also worth taking a look at the app deployment [guide](https://github.com/hmcts/cnp-flux-config/blob/master/docs/app-deployment-v2.md#application) in cnp-flux-config to understand how you would do this normally.

A couple of minutes after the PR you created has been merged, you should see your HelmRelease resource deployed and the tag of your application's image updated to the latest tag with the `prod-xxxxxxx-xxxxxxxxxxxxxx` naming convention.

To check this you can connect to the cluster by running the command below:

```command
 az login
 az aks get-credentials --resource-group cft-sbox-00-rg --name cft-sbox-00-aks --subscription DCD-CFTAPPS-SBOX --overwrite-existing
```

To make sure your pod is running as expected and to check the status of your HelmRelease run the following commands (make sure to swap YourGithubUsername with your GitHub username):

```command
 kubectl get hr labs-YourGithubUsername -n labs
 kubectl get pods -l app.kubernetes.io/name=labs-YourGithubUsername-java -n labs
```

### Access application

If all went well your application should be visible now.

The URL will be (update the GitHub username variable):

   ```text
   http://labs-$yourGitHubUsername-sandbox.service.core-compute-sandbox.internal 
   ```  

Open this in your browser, you should see:

```text
Welcome to labs-$yourGitHubUsername application
```

### Customise application

We are going to update the application by changing the home page with an environment variable.

Environment variables are managed in the [Helm](https://helm.sh) chart that has been created for you.
The chart is in the `charts/$app-name` folder.

1. Open the `values.yaml` file inside the Helm chart.  

1. Add an environment value to the chart:

   ```yaml
   java:
     environment:
       FAVOURITE_FRUIT: plum
   ```

1. Update the code to reference the new environment variable in `RootController.java` the file is in `src/main/java/uk/gov/hmcts/reform/<Component>/controllers/`.

   ```java
    public ResponseEntity<String> welcome() {
        return ok("Welcome to your app, my favourite fruit is " +  System.getenv("FAVOURITE_FRUIT"));
    }
   ```

1. Ask someone on your team to review your `pull request` and then merge it.

1. Run the Jenkins pipeline against the `master` branch (this will trigger automatically on the production Jenkins instance).

1. Reload your application in your browser and check it now shows your favourite fruit.

## Feedback

[comment]: <> (As of December 2021)
This is a new way of onboarding developers that we are trying out.
If you could provide feedback it would really help us improve it for others.
The [survey](https://forms.office.com/r/P2YbcLVAr4) has 4 questions and will only take you a couple of minutes to complete.

If you've found a problem with the guide please [report an issue](https://github.com/hmcts/golden-path-java/issues) instead, or you can create a pull request to correct it yourself.

If you need help with the lab please reach out in [#golden-path](https://hmcts-reform.slack.com/app_redirect?channel=golden-path).

## Troubleshooting

See our [troubleshooting](https://hmcts.github.io/ways-of-working/troubleshooting/#troubleshooting-issues) guide.
