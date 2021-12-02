How to Request Tokens
=====================

As part of the [GridFTP and GSI migration](../policy/gridftp-gsi-migration.md), the OSG will be transitioning authentication
away from X.509 certificates to the use of bearer tokens such as [SciTokens](http://scitokens.org/) or
[WLCG JWT](https://twiki.cern.ch/twiki/bin/view/LCG/WLCGAuthorizationWG).
This document is intended as a guide for OSG developers for requesting tokens necessary for software development.

## Before Starting

Before you can request the appropriate tokens, you must have the following:

-   A [WLCG INDIGO IAM](https://wlcg.cloud.cnaf.infn.it/) account belonging to the `wlcg`, `wlcg/pilots`, and `wlcg/xfers`
    groups.
-   One of the following:
    -   The ability to run containers through tools like `docker` or `podman`
    -   An installation of [oidc-agent](https://indigo-dc.gitbook.io/oidc-agent/) available as an RPM from the OSG
        repositories

## Requesting Tokens Using a Container

[oidc-agent](https://indigo-dc.gitbook.io/oidc-agent/) is a process that runs in the background that can request access
and refresh tokens from OpenID Connect token providers.

### Registering an OIDC profile

1. Start an agent container in the background and name it `my-agent` to easily run subsequent commands against it:

        :::console
        docker run -d --name my-agent opensciencegrid/oidc-agent:release

1. Generate a local client profile and follow the prompts:

        :::console
        docker exec -it my-agent oidc-gen -w device <CLIENT PROFILE>

    1. Specify the WLCG INDIGO IAM instance as the client issuer:

            Issuer [https://iam-test.indigo-datacloud.eu/]: https://wlcg.cloud.cnaf.infn.it/

    1. Request `wlcg`, `offline_access`, and other scopes for the capabilities that you need:

        | **Capability**   | **Scope**                     |
        |------------------|-------------------------------|
        | HTCondor `READ`  | `compute.read`                |
        | HTCondor `WRITE` | `compute.modify compute.cancel compute.create` |
        | XRootD read      | `read:/`                      |
        | XRootD write     | `write:/`                     |

        For example, to request HTCondor `READ` and `WRITE` access, specify the following scopes:

            This issuer supports the following scopes: openid profile email address phone offline_access wlcg iam wlcg.groups
            Space delimited list of scopes or 'max' [openid profile offline_access]: wlcg offline_access compute.read compute.modify compute.cancel compute.create
    
        Note that, prior to HTCondor 8.9.7, the server also needed `condor:/ALLOW` in all cases.

    1. When prompted, open <https://wlcg.cloud.cnaf.infn.it/device> in a browser, enter the code provided by `oidc-gen`,
       and click "Submit".

    1. On the next page, verify the scopes and client profile name, and click "Authorize".

    1. Enter a password to encrypt your local client profile.
       You'll need to remember this if you want to re-use this profile in subsequent sessions.

### Requesting access tokens

!!! note
    You must first [register a new profile](#registering-an-oidc-profile).

1. Request a token using the client profile that you used with `oidc-gen`:
	
        :::console
        docker exec -it my-agent oidc-token --aud="<SERVER AUDIENCE>" <CLIENT PROFILE>


    For tokens used against an HTCondor-CE, set `<SERVER AUDIENCE>` to  
    `<CE FQDN>:<CE PORT>`.

1. Copy the output of `oidc-token` into a file on the host where you need SciToken authentication, e.g. an HTCondor or
   XRootD client.

### Reloading an OIDC profile

!!! note
    Required after restarting the running container. You must have an existing [registered profile](#registering-an-oidc-profile).

1. If your existing container is not already running, start it:

        :::console
        docker start my-agent

1. Reload profile:

        :::console
        docker exec -it my-agent oidc-add <CLIENT PROFILE>


1. Enter password used to encrypt your `<CLIENT PROFILE>` created during profile registration.


## Requesting Tokens with an RPM installation

### Registering an OIDC profile

1. Start the agent and add the appropriate variables to your environment:

        :::console
        eval `oidc-agent`

1. Generate a local client profile and follow the prompts:

        :::console
        oidc-gen -w device <CLIENT PROFILE>

    1. Specify the WLCG INDIGO IAM instance as the client issuer:

            Issuer [https://iam-test.indigo-datacloud.eu/]: https://wlcg.cloud.cnaf.infn.it/

    1. Request `wlcg`, `offline_access`, and other scopes for the capabilities that you need:

        | **Capability**   | **Scope**                     |
        |------------------|-------------------------------|
        | HTCondor `READ`  | `compute.read`                |
        | HTCondor `WRITE` | `compute.modify compute.cancel compute.create` |
        | XRootD read      | `read:/`                      |
        | XRootD write     | `write:/`                     |
         For example, to request HTCondor `READ` and `WRITE` access, specify the following scopes:

            This issuer supports the following scopes: openid profile email address phone offline_access wlcg iam wlcg.groups
            Space delimited list of scopes or 'max' [openid profile offline_access]: wlcg offline_access compute.read compute.modify compute.cancel compute.create
    
        Note that, prior to HTCondor 8.9.7, the server also needed `condor:/ALLOW` in all cases.

    1. When prompted, open <https://wlcg.cloud.cnaf.infn.it/device> in a browser, enter the code provided by `oidc-gen`,
       and click "Submit".

    1. On the next page, verify the scopes and client profile name, and click "Authorize".

    1. Enter a password to encrypt your local client profile.
       You'll need to remember this if you want to re-use this profile in subsequent sessions.

### Requesting access tokens

!!! note
    You must first [register a new profile](#registering-an-oidc-profile_1).

1. Request a token using the client profile that you used with `oidc-gen`:

        :::console
        oidc-token --aud="<SERVER AUDIENCE>" <CLIENT PROFILE>

    For tokens used against an HTCondor-CE, set `<SERVER AUDIENCE>` to  
    `<CE FQDN>:<CE PORT>`.

1. Copy the output of `oidc-token` into a file on the host where you need SciToken authentication, e.g. an HTCondor or
   XRootD client.

### Reloading an OIDC profile

!!! note
    Required if you log out of the running machine. You must have an existing [registered profile](#registering-an-oidc-profile_1).

1. If you do not already have a running 'oidc-agent', start one:

        :::console
        eval 'oidc-agent'

1. Reload profile:

        :::console
        oidc-add <CLIENT PROFILE>


1. Enter password used to encrypt your `<CLIENT PROFILE>` created during profile registration.

##Troubleshooting Tokens

You can inspect the payload  by copy-pasting the token into the "Encoded" text box here: <http://jwt.io/>.
Mouse over the fields and values for details.
