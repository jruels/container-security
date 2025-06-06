# Docker Scout quickstart

Docker Scout analyzes image contents and generates a detailed report of packages and vulnerabilities that it detects. It can provide you with suggestions for how to remediate issues discovered by image analysis.

This guide takes a vulnerable container image and shows you how to use Docker Scout to identify and fix the vulnerabilities, compare image versions over time, and share the results with your team.

In order to do this lab, you will need a free Docker Hub account. If you don't have one, then the instructor will show you how to get one.


## Prerequisites 

The Docker CLI must have the Docker Scout plugin installed for the following to work. Please run the preloaded script to install it. 

Download scout

```shell
curl -fsSL https://raw.githubusercontent.com/docker/scout-cli/main/install.sh -o install-scout.sh
```
The installation command will be looking for the plugins' directory. To create it, use the following bash command

```shell
mkdir -p ~/.docker/cli-plugins
```

Now install scout 

```shell
bash ~/install-scout.sh
```

Verify the script ran successfully 

```bash
docker scout version 
```

You should see some fun ASCII output and the version. 

## Setup

[This example project](https://github.com/docker/scout-demo-service) contains a vulnerable Node.js application that you can use to follow along.

1. Clone its repository:

   ```console
   git clone https://github.com/docker/scout-demo-service.git
   ```

2. Move into the directory:

   ```console
   cd scout-demo-service
   ```

3. Ensure sure you're signed in to your Docker account by running the `docker login -u <DOCKER_HUB_USERNAME>` command.

4. Build the image and push it to `<DOCKER_HUB_USERNAME>/scout-demo:v1`, where `<DOCKER_HUB_USERNAME>` is the Docker Hub account you created.

   ```console
   docker build -t <DOCKER_HUB_USERNAME>/scout-demo:v1 .
   docker push <DOCKER_HUB_USERNAME>/scout-demo:v1
   ```

## Enable Docker Scout

Docker Scout analyzes all local images by default. However, to analyze images in remote repositories, you must first enable this feature. You can do this from Docker Hub, the Docker Scout Dashboard, or the CLI. 

1. Enroll your organization with Docker Scout, using the `docker scout enroll` command.

   ```console
   docker scout enroll <DOCKER_HUB_USERNAME>
       ✓ Successfully enrolled organization ORG_NAME with Docker Scout Free
   ```

2. Enable Docker Scout for your image repository with the `docker scout repo enable` command.

   ```console
   docker scout repo enable --org <DOCKER_HUB_USERNAME> <DOCKER_HUB_USERNAME>/scout-demo
   ```

## Analyze image vulnerabilities

After building, use the `docker scout` CLI command to see vulnerabilities detected by Docker Scout.

The example application for this guide uses a vulnerable version of Express. The following command shows all CVEs affecting Express in the image you just built:

```console
docker scout cves --only-package express
```

By default, Docker Scout analyzes the image you recently built, so you don't need to specify the image's name.

Learn more about the `docker scout cves` command in the [`CLI reference documentation`](https://docs.docker.com/reference/cli/docker/scout/cves).

The output should look something like this

```text
    ✓ Image stored for indexing
    ✓ Indexed 79 packages
    ✗ Detected 1 vulnerable package with 3 vulnerabilities


## Overview

                    │       Analyzed Image         
────────────────────┼──────────────────────────────
  Target            │                              
    digest          │  ef581530d802                
    platform        │ linux/amd64                  
    vulnerabilities │    0C     1H     1M     1L   
    size            │ 22 MB                        
    packages        │ 1                            


## Packages and Vulnerabilities

   0C     1H     1M     1L  express 4.17.1
pkg:npm/express@4.17.1

    ✗ HIGH CVE-2022-24999 [OWASP Top Ten 2017 Category A9 - Using Components with Known Vulnerabilities]
      https://scout.docker.com/v/CVE-2022-24999
      Affected range : <4.17.3                                       
      Fixed version  : 4.17.3                                        
      CVSS Score     : 7.5                                           
      CVSS Vector    : CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:N/I:N/A:H  
    
    ✗ MEDIUM CVE-2024-29041 [Improper Validation of Syntactic Correctness of Input]
      https://scout.docker.com/v/CVE-2024-29041
      Affected range : <4.19.2                                       
      Fixed version  : 4.19.2                                        
      CVSS Score     : 6.1                                           
      CVSS Vector    : CVSS:3.1/AV:N/AC:L/PR:N/UI:R/S:C/C:L/I:L/A:N  
    
    ✗ LOW CVE-2024-43796 [Improper Neutralization of Input During Web Page Generation ('Cross-site Scripting')]
      https://scout.docker.com/v/CVE-2024-43796
      Affected range : <4.20.0                                                          
      Fixed version  : 4.20.0                                                           
      CVSS Score     : 2.3                                                              
      CVSS Vector    : CVSS:4.0/AV:N/AC:L/AT:P/PR:N/UI:P/VC:N/VI:N/VA:N/SC:L/SI:L/SA:L  
    


3 vulnerabilities found in 1 package
  CRITICAL  0  
  HIGH      1  
  MEDIUM    1  
  LOW       1  


```
## Fix application vulnerabilities

The fix suggested by Docker Scout is to update the underlying vulnerable, express version to a later version. Since the last error requires 4.20.0, we will use that one

1. Update the `package.json` file with the new package version.

   ```diff
      "dependencies": {
   -    "express": "4.17.1"
   +    "express": "4.20.0"
      }
   ```

2. Rebuild the image with a new tag and push it to your Docker Hub repository:

   ```console
   docker build -t <DOCKER_HUB_USERNAME>/scout-demo:v2 .
   docker push <DOCKER_HUB_USERNAME>/scout-demo:v2
   ```

Now, if you view the latest tag of the image, you can see that you have fixed the vulnerability.



```console
docker scout cves --only-package express
    ✓ Image stored for indexing
    ✓ Indexed 100 packages
    ✓ No vulnerable package detected


## Overview

                    │       Analyzed Image         
────────────────────┼──────────────────────────────
  Target            │                              
    digest          │  34c1167f6cca                
    platform        │ linux/amd64                  
    vulnerabilities │    0C     0H     0M     0L   
    size            │ 22 MB                        
    packages        │ 1                            


## Packages and Vulnerabilities

  No vulnerable packages detected

```

## Evaluate policy compliance

While inspecting vulnerabilities based on specific packages can be useful, it isn't the most effective way to improve your supply chain conduct.

Docker Scout also supports policy evaluation, a higher-level concept for detecting and fixing issues in your images. Policies are a set of customizable rules that let organizations track whether images are compliant with their supply chain requirements.

Because policy rules are specific to each organization, you must specify which organization's policy you're evaluating against. Use the `docker scout config` command to configure your Docker organization.



```console
docker scout config organization ORG_NAME
    ✓ Successfully set organization to ORG_NAME
```

Now you can run the `quickview` command to get an overview of the compliance status for the image you just built. The image is evaluated against the default policy configurations.



```console
docker scout quickview
```
```console
docker scout quickview
✓ SBOM of image already cached, 100 packages indexed
✓ Policy evaluation completed

    i Base image was auto-detected. To get more accurate results, build images with max-mode provenance attestations.
      Review docs.docker.com ↗ for more information.

Target               │  local://us2507/scout-demo:v2  │    2C    17H     7M     2L     1?   
digest               │  34c1167f6cca                  │                                     
Base image           │  alpine:3                      │    2C    15H     7M     0L     1?   
Refreshed base image │  alpine:3                      │    0C     0H     0M     0L          
                     │                                │    -2    -15     -7            -1   
Updated base image   │  alpine:3.20                   │    0C     0H     0M     0L          
│                                                     │    -2    -15     -7            -1

Policy status  FAILED  (2/7 policies met, 2 missing data)

Status │                     Policy                     │           Results            
─────────┼────────────────────────────────────────────────┼──────────────────────────────
!      │ No default non-root user found                 │                              
✓      │ No AGPL v3 licenses                            │    0 packages                
!      │ Fixable critical or high vulnerabilities found │    2C    17H     0M     0L   
✓      │ No high-profile vulnerabilities                │    0C     0H     0M     0L   
?      │ No outdated base images                        │    No data                   
       │                                                │    Learn more ↗                    
?      │ No unapproved base images                      │    No data                   
!      │ Missing supply chain attestation(s)            │    2 deviations

What's next:
View policy violations → docker scout policy local://us2507/scout-demo:v2 --org us2507
View vulnerabilities → docker scout cves local://us2507/scout-demo:v2
View base image update recommendations → docker scout recommendations local://us2507/scout-demo:v2
Compare with the latest in the registry → docker scout compare --to-latest local://us2507/scout-demo:v2 --org us2507
```

Exclamation marks in the status column indicate a violated policy. Question marks indicate that there isn't enough metadata to complete the evaluation. A check mark indicates compliance.

## Improve compliance

The output of the `quickview` command shows that there's room for improvement. Some of the policies couldn't evaluate successfully (`No data`) because the image lacks provenance and SBOM attestations. The image also failed the check on a few of the evaluations.

Policy evaluation does more than just check for vulnerabilities. Take the `Default non-root user` policy for example. This policy helps improve runtime security by ensuring that images aren't set to run as the `root` superuser by default.

To address this policy violation, edit the Dockerfile by adding a `USER` instruction, specifying a non-root user:



```diff
  CMD ["node","/app/app.js"]
  EXPOSE 3000
+ USER appuser
```



Rebuild the image with a new `v3` tag. 

```console
docker build  -t <DOCKER_HUB_USERNAME>/scout-demo:v3 .
docker push <DOCKER_HUB_USERNAME>/scout-demo:v3
```



After rebuilding the image, rerun `docker scout quickview,` and you'll notice the `Default non-root user` policy status has changed. This issue has been remediated. 



## View in Dashboard

After pushing the updated image with attestations, it's time to view the results through a different lens: the Docker Scout Dashboard.

1. Open the [Docker Scout Dashboard](https://scout.docker.com/).
2. Sign in with your Docker account.
3. Select **Images** in the left-hand navigation.

The images page lists your Scout-enabled repositories. Select the image in the list to open the **Image details** sidebar. The sidebar shows a compliance overview for the last pushed tag of a repository.

> **Note**
>
> If policy results have yet to appear, try refreshing the page. If this is your first time using the Docker Scout Dashboard, it might take a few minutes before the results appear.



