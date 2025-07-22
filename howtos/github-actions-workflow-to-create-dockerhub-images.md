# Github actions workflow to create dockerhub images

## Basic requirements

### DockerHub account / API token

For the requirements of this document the account on docker hub should be a service account with details located in 1Password so that it isn’t tied to a single account.

Each action should use a different API token so that if one is found to have been compromised only that one has to be removed

#### Docker hub account

This can easily be created by using the google account sign up link.

Once signed up with docker you will then need to create a user tag / spacename. This is the same as GitHub’s space names so **all container repos will live under that tag and be pulled using it.**

#### Create a Dockerhub repo

* Click on the dockerhub logo to reset the view to your dashboard
* Towards the right side there’s a button “Create repository”, click it
* give the repo a sensible name and a general description as required
* the repo is most likely to be used publicly - no sign in required, so leave it public. If private is preferred then click that however there’s only one allowance for a free account

#### Create an API access token

* In the top right click the avatar circle for your account
* Click on “my account”
* Click on “security”
* In the new page click on “New Access Token”
* Give the access token a sensible description “github actions account for \<github repo name goes here>”
* select the scope of access that hte new token will have (likely read,write,delete or read & write)
  * Read, Write, Delete - allow you to manage your repositories.
  * Read & Write - allow you to push images to any repository managed by your account.
  * Read-only - allow you to view, search, and pull images from any public repositories and any private repositories that you have access to.
  * Public Repo Read-only - allow to view, search, and pull images from any public repositories.
* Click “generate”
* store the API key somewhere sensible, potentially 1pass although the requirement might just be github secrets as you can generate a new token easily enough

### Github repo

For this there needs to be a repo in github which you can access the “Settings” area of

The repo should also have a dockerfile with which to build the container

## Main process

### Add secrets to github repo

Github will use the “secrets” added to login to your dockerhub space and upload container images

* Go to the github repo which you want to have actions automatically push docker images for
* Click on “Settings”
* on the left side under “Security” click “Secrets and variables” then “Actions”
* Click on “New repository secret”
* enter a sensible name for the secret ( this will be the variable which you cal in the action )
* then the secret you wish to store. In the case of this document the two secrets were
  * “DOCKERHUB\_USER” - the user tag in dockerhub to login to
  * “DOCKERHUB\_API\_KEY” - The API token created at “ [Create an API access token](https://www.notion.so/Create-an-API-access-token-f2da0f4402f444e78cb54027d6aa4bc5?pvs=21) “
* click “add secret” to store it in github

### Create a workflow

#### What’s a workflow

Workflows are what github actions uses as an instruction set. Located within a “.github” folder in your repo and are of “.yml” type

They are normally of an instruction basis - if this happens do that

For each desired function there should be a separate file. This keeps the actions easily readable and identifiable (github will automatically gather all the relevant files )

#### Creating an action

The following is under the impression that there aren’t currently any actions in the repo and you’re creating one to create a docker container and put it into dockerhub on a push to the repo

* at the top level of the repo create a directory called “.github” and a sub directory “workflows” ( “-p” creates the full path
  * mkdir -p .github/workflows
*   For this example take the below and paste it into a new file

    * vim /.github/workflows/**deploy\_latest\_container\_on\_master\_push.yml**

    ```jsx
    name: Build and Push Docker image to Docker Hub

    on: 
      push:
        branches: [ "cudos-master" ]

    jobs:
      push_to_registry:
        name: Push Docker image to Docker Hub
        runs-on: ubuntu-latest
        steps:
          - name: Check out the repo
            uses: actions/checkout@v3
          
          - name: Login to Docker Hub
            uses: docker/login-action@v2
            with:
              username: ${{ secrets.DOCKERHUB_USER }}
              password: ${{ secrets.DOCKERHUB_API_KEY }}
        
          - name: Build and push Docker image
            uses: docker/build-push-action@v4
            with:
              context: .
              push: true
              tags: cudostest/dashboard:latest
    ```
* This script involves the following areas
  * name - the name github will list the job as running under
  * “on” - the event on which the job will trigger. In the above usecase the job will trigger on a push to the “cudos-master” branch
  * job - the task to run with groupings
    * name - the top level st4ep name
    * runs-on - what type of system the action has been written for. NOTE: there are 3x Ubuntu types, 3x windows types and 4x macOS types - [https://docs.github.com/en/actions/using-workflows/workflow-syntax-for-github-actions#choosing-github-hosted-runners](https://docs.github.com/en/actions/using-workflows/workflow-syntax-for-github-actions#choosing-github-hosted-runners)
    * steps
      * checkout - checks out the code for the action to consume
      * docker/login - confirms that it can login to dockerhub with the user and API token configured previously as a secret [Add secrets to github repo](https://www.notion.so/Add-secrets-to-github-repo-8e2a9ba49f5e464eb7ad8153bd33aabb?pvs=21) using the variable names assigned
      * docker/build-push -
        * As the dockerfile for dashboard is at the top level the “context” is set to that ( “.” ).
        * Push is set to true so once the container is built it it will be pushed up to dockerhub ( this can be set to false and used as a confirmation step in say a merge request to confirm that everything will build correctly so that you don’t have broken code pushed to master branches).
        * The tag is what you want to have the container tag as. Since this isn’t a specific tagged release / is done on any push to master then the tag is set to “latest”. This will overwrite “latest” in dockerhub everytime there’s a push to cudos-master
*   you can then save the file, make git aware of it and push up

    ```jsx
    :wq 
    git add *
    git commit -a -m "<sensible commit message>"
    git push origin <branch name>
    ```
* if pushed to the master branch this will trigger the workflow straight away. If you click on the “actions” tab in git you should be able to see if it’s working and at what stage it might be at.

#### Other possible actions

There are various different types of actions that can be achieved using different combinations of “if. then” blocks (each to a separate file).

```jsx
on:
  workflow_dispatch:
    inputs:
      tag:
        description: 'New release tag name'
        required: true

```

workflow\_dispatch allows you to do a manual run based on something. In the case of the above it’s creating an entry against a variable called “tag” which then has a description of what’s expected and is forced to be entered before the workflow can run.

This allows you to augment the example actions job with something like:

```jsx
- name: Build and push Docker image
        uses: docker/build-push-action@v4
        with:
          context: .
          push: true
          tags: cudostest/dashboard:${{ github.event.inputs.tag }}

- name: Add tag to repo
        uses: rickstaa/action-create-tag@v1
        id: "tag_create"
        with:
          tag: "${{ github.event.inputs.tag }}"
          tag_exists_error: false
          message: "${{ github.event.inputs.tag }} release"
```

As you’re running the workflow based on a tag that you’ve entered you can then utilise that tag. Rather than pushing a container to “latest” you can instead push it to a specific version of the container so that you can fulfill devops standards of desired state and being fairly confident that running the pipeline for the container will always result in the same container being deployed.

While we’re at it we may as well also create a tag in the repo that can be mapped to the container so that you can see what code version was released in the container again taking the tag number but this time uing rickstaa’s action to create a github tag in the repo

NOTE: each item that the workflow’s job uses has a specified version number. Again this is so that you can be sure that it will run the same each time and that no functionalities have been added / removed since writing the workflow
