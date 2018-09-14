# Lab 05: DevOps Pipelines with Containers and AKS

---
## Objective
---

In this lab, you'll use the Azure DevOps platform to automate Continuous
Integration (CI) and Continuous Deployment (CD) pipelines from at least one
containerized project to your Azure Kubernetes Service (AKS) cluster.

Prerequisite:
* For this lab, you will need an Azure DevOps account (VSTS account) with
  permissions to create new Build and Release Pipelines.
* This lab also assumes you have a local command-line git client configured. If not,
  use whatever git tool you have to be able to commit your code to a source control
  repository.

---
## Setup an Azure DevOps Team Project
---

### Create an Azure DevOps Project for your build pipelines 

1. In a browser, navigate to your Azure DevOps account (VSTS account) home,
e.g., https://microsoft.visualstudio.com/. 

2. Click the Create Project button to create a new Azure DevOps project.

If you have any issues, visit the [how-to guide](https://docs.microsoft.com/en-us/azure/devops/organizations/projects/create-project?view=vsts&tabs=new-nav) for creating an Azure DevOps Services project.

### Commit your code to a source code repository

Upon completion of previous labs, you should have a local root directory with
three project directories and a deployment directory:

```
 <root>/
 +-- deployments
 |  +-- songrequest.yaml
 +-- SongRequest.MessageConsumer 
 +-- SongRequest.MessageProducer
 +-- SongRequest.WebFrontEndService
```

To enable Continuous Integration, we need to commit this project to a
source control system that can trigger an Azure DevOps build or release
pipeline such as Azure DevOps Repos or GitHub. It may be easiest for you
to use the repository created as a part of the project you just created,
but you may host it in a different system if you prefer.

> Note: It is recommended you include the VisualStudio .gitignore at the root of
> your repository to limit the artifacts committed to source control.

1. Copy the URL to clone the repository to your clipboard.
2. In a console window, navigate to the root directory and execute:
   ```console
   git clone <URL_FROM_REPO> .
   ```
3. To add all of your new files, execute:
   ```console
   git add *
   git commit -m 'Initial commit'
   git push
   ```

Once the repository contains the same file structure as outlined above, you can
create a continuous integration build triggered by changes to the repository.

---
## Create your CI Build Pipeline
---

1. Navigate to the Builds page found under the project pipelines in the navigation tree
   on your Azure DevOps Project home.
2. Select **New** and then **New build pipeline** from the button at the top of the list
   of build pipelines.
3. Configure your repository. Make sure you select the correct repository to which you
   just committed your application code. Click **Continue**
4. Choose a template from which to start. Search for, or scroll to the **Docker container**
   template and click the **Apply** button visible to the right when you hover over the
   template.
5. The Build Pipeline view should now be displayed with the Pipeline level selected.
   Verify the *Agent pool* selected is either the **Hosted Linux Preview** or **Hosted Ubuntu 1604** option.
6. Select the first *Build an image* task to edit the details.
   * Update the *Version* to 1.*
   * Update the *Display name* to be **Build SongRequest.MessageConsumer image**
   * Select your *Azure subscription* (it may require authorization if you have not already
     configured a Service Connection to your Azure subscription in this Azure DevOps project).
   * Select your *Azure Container Registry* to which you will push your images.
   * Explicitly select the SongRequest.MessageConsumer/Dockerfile instead of the default        **/Dockerfile by clicking the ellipses icon to the right of the *Docker File* field.
   * Update the *Image Name* field to be **songrequest.messageconsumer:$(Build.BuildId)**. 
   * Check the **Include Latest Tag** box.
7. Select the *Push an Image* to edit the details.
   * Update the *Version* to 1.*
   * Update the *Display name* to be **Push SongRequest.MessageConsumer image**
   * Select your *Azure subscription* (it may require authorization if you have not already
     configured a Service Connection to your Azure subscription in this Azure DevOps project).
   * Select your *Azure Container Registry* to which you will push your images.
   * Update the *Command* to be **push** (it changed to build when you changed the version)
   * Update the *Image Name* field to be **songrequest.mesageconsumer:$(Build.BuildId)**.
   * Check the **Include Latest Tag** box.
8. Repeat steps 6 and 7 for the SongRequest.WebFrontEnd project so that you are building
   and pushing the **songrequest.webfrontend** image correctly. You will need to add two
   new taks to the Agent job (by clicking the plus sign to the right of the Agent job 1 header).
9. Add a Copy Files task to the end of the Agent job tasks list. Configure it to copy
   the deployment directory files to a staging directory.
   * Select or type **deployments** as your *Source Folder*
   * Set the *Target Folder* to **$(Build.ArtifactStagingDirectory)**
10. Add a Publish Build Artifacts task to the end of the Agent job tasks list. The
    default values for this task should be correct.
11. Click on the **Triggers** tab at the top of the build pipeline editor. Click the
    checkbox to *Enable continuous integration*.
12. Click the **Save & Queue** button at the top of the build pipeline editor. This will
    save your build and manually queue up a build for execution.

Monitor your build by clicking the link displayed in the successful queueing notification.
After the build successfully completes, click the **Release** button to begin creating
your release pipeline.

---
## Create your CD Release Pipeline
---

When starting a new Release pipeline, you should immediately be prompted to *Select a
template*. Find the *Deploy to a Kubernetes cluster* template and click **Apply**.

1. Click the **Add** link in the *Artifacts* box to add another artifact to your release
   pipeline.
   * Select the **Azure Container Registry** as your *Source type*. It may only have
     *Azure Contai...* visible, and if it's not check if you see a *4 more artifact types*
     link to expand the options.
   * Select all of the correct settings until you can select your **songrequest.messageconsumer** *Repository*.
   * Click Add.
2. Repeat step one for the **songrequest.webfrontend** ACR repository.
3. Click on the *1 job, 1 task* link in the *Stage 1* box to edit the release task.
   * Select the *Agent job* level and update the *Agent pool* to be either **Hosted Linux       Preview** or **Hosted Ubuntu 1604**
4. Select the *kubectl apply* task from the task list to edit the details
   * Update the *Version* to 1.*
   * Select your *Azure subscription*, the *Resource group* to which your AKS cluster is
     deployed, and your *Kubernetes cluster*.
   * Check the *Use configuration files* checkbox
   * Use the file selector pop-up to navigate to your "drop" directory (in the
     Build linked artifact). Once the directory is display, append **/songrequest.yaml**
     to the end manually.
5. Save the Release Pipeline
6. Queue a Relese by selecting the **Release** button at the top of the release pipeline
   editor followed by **Create a release**. You will need to select the version to use
   (**latest**) for each of your ACR image artifacts.

   Monitor your release by clicking on the link displayed in the successful queueing
   notification. 

---
## Summary
---

With these steps, you now have a full CI/CD pipeline. Although we didn't trigger
the release to work as a continuous deployment at this time and must be manually
triggered. You would need to decide at what point a release makes sense for your
environement -- when the code repository is changed, or when images have been updated
in the container registry.