---
title: Configure pipeline triggers
description: Configure pipeline triggers
ms.topic: conceptual
ms.author: ashkir
author: ashokirla
ms.date: 10/20/2020
monikerRange: ">=azure-devops-2019"
---

# Trigger one pipeline after another

Large products have several components that are dependent on each other.
These components are often independently built. When an upstream component (a library, for example) changes, the downstream dependencies have to be rebuilt and revalidated.

In situations like these, add a pipeline trigger to run your pipeline upon the successful completion of the **triggering pipeline**.

# [YAML](#tab/yaml)

:::moniker range=">= azure-devops-2020"

To trigger a pipeline upon the completion of another, specify the triggering pipeline as a [pipeline resource](resources.md#resources-pipelines).

> [!NOTE]
> Previously, you may have navigated to the classic editor for your YAML pipeline and configured **build completion triggers** in the UI. While that model still works, it is no longer recommended. The recommended approach is to specify **pipeline triggers** directly within the YAML file. Build completion triggers as defined in the classic editor have various drawbacks, which have now been addressed in pipeline triggers. For instance, there is no way to trigger a pipeline on the same branch as that of the triggering pipeline using build completion triggers.

In the following example, we have two pipelines - `app-ci` (the pipeline defined by the YAML snippet) and `security-lib-ci` (the pipeline referenced by the pipeline resource). We want the `app-ci` pipeline to run automatically every time a new version of the security library is built in the main branch or any releases branch.


```yaml
# this is being defined in app-ci pipeline
resources:
  pipelines:
  - pipeline: securitylib   # Name of the pipeline resource
    source: security-lib-ci # Name of the pipeline referenced by the pipeline resource
    trigger: 
      branches:
      - releases/*
      - main
```

* `pipeline: securitylib` specifies the name of the pipeline resource, and is used when referring to the pipeline resource from other parts of the pipeline, such as pipeline resource variables.
* `source: security-lib-ci` specifies the name of the pipeline referenced by this pipeline resource. You can retrieve a pipeline's name from the Azure DevOps portal in several places, such as the [Pipelines landing page](../get-started/multi-stage-pipelines-experience.md#pipelines-landing-page). To configure the pipeline name setting, edit the YAML pipeline, choose **Triggers** from the settings menu, and navigate to the **YAML** pane. This setting only works if the value is in quotes, if the name of the pipeline being referenced contains whitespace.

    ![Pipeline settings](../repos/media/pipelines-options-for-git/yaml-pipeline-git-options-menu.png)

> [!NOTE] 
> If the triggering pipeline is in another Azure DevOps project, you must specify the
> project name using `project: OtherProjectName`. For more information, see [pipeline resource](resources.md#resources-pipelines).

Similar to CI triggers, you can specify the branches to include or exclude:

```yaml
# this is being defined in app-ci pipeline
resources:
  pipelines:
  - pipeline: securitylib
    source: security-lib-ci
    trigger: 
      branches:
        include: 
        - releases/*
        exclude:
        - releases/old*
```

> [!NOTE]
> If your filters aren't working, try using the prefix `refs/heads/`. For example, use `refs/heads/releases/old*`instead of `releases/old*`.

If the triggering pipeline and the triggered pipeline use the same repository, then both the pipelines will run using the same commit when one triggers the other. This is helpful if your first pipeline builds the code, and the second pipeline tests it. However, if the two pipelines use different repositories, then the triggered pipeline will use the version of the code in the branch specified by the **Default branch for manual and scheduled builds** setting, as described in the following section.

### Branch considerations for pipeline completion triggers

Pipeline completion triggers use the **Default branch for manual and scheduled builds** setting to determine which branch's version of a YAML pipeline's branch filters to evaluate when determining whether to run a pipeline as the result of another pipeline completing. By default this setting points to the default branch of the repository.

When a pipeline completes, the Azure DevOps runtime evaluates the pipeline resource trigger branch filters of any pipelines with pipeline completion triggers that reference the completed pipeline. A pipeline can have multiple versions in different branches, so the runtime evaluates the branch filters in the pipeline version in the branch specified by the **Default branch for manual and scheduled builds** setting. If there is a match, the pipeline runs, but the version of the pipeline that runs may be in a different branch depending on whether the triggered pipeline is in the same repository as the completed pipeline.

- If the two pipelines are in different repositories, the triggered pipeline version in the branch specified by **Default branch for manual and scheduled builds** is run.
- If the two pipelines are in the same repository, the triggered pipeline version in the same branch as the triggering pipeline is run, even if that branch is different than the **Default branch for manual and scheduled builds**, and even if that version does not have branch filters that match the completed pipeline's branch. This is because the branch filters from the **Default branch for manual and scheduled builds** branch are used to determine if the pipeline should run, and not the branch filters in the version that is in the completed pipeline branch. 

If your pipeline completion triggers don't seem to be firing, check the value of the **Default branch for manual and scheduled builds** setting for the triggered pipeline. The branch filters in that branch's version of the pipeline are used to determine whether the pipeline completion trigger initiates a run of the pipeline. By default, **Default branch for manual and scheduled builds** is set to the default branch of the repository, but you can change it after the pipeline is created.

A typical scenario in which the pipeline completion trigger doesn't fire is when a new branch is created, the pipeline completion trigger branch filters are modified to include this new branch, but when the first pipeline completes on a branch that matches the new branch filters, the second pipeline doesn't trigger. This happens if the branch filters in the pipeline version in the **Default branch for manual and scheduled builds** branch don't match the new branch. To resolve this trigger issue you have the following two options.

- Update the branch filters in the pipeline in the **Default branch for manual and scheduled builds** branch so that they match the new branch.
- Update the **Default branch for manual and scheduled builds** setting to a branch that has a version of the pipeline with the branch filters that match the new branch.

To view and update the **Default branch for manual and scheduled builds** setting:

1. [Navigate](../get-started/multi-stage-pipelines-experience.md#navigating-pipelines) to the [pipeline details](../get-started/multi-stage-pipelines-experience.md#view-pipeline-details) for your pipeline, and choose **Edit**.

    :::image type="content" source="media/pipeline-triggers/pipeline-edit.png" alt-text="Edit pipeline."::: 

2. Choose **...** and select **Triggers**.

    :::image type="content" source="media/pipeline-triggers/edit-triggers.png" alt-text="Edit triggers."::: 

3. Select **YAML**, **Get sources**, and view the **Default branch for manual and scheduled builds** setting. If you change it, choose **Save** or **Save & queue** to save the change.

    :::image type="content" source="media/pipeline-triggers/default-branch-setting.png" alt-text="Default branch for manual and scheduled builds setting."::: 

### Behavior when pipeline completion triggers and CI triggers are present

When you specify both CI triggers and pipeline triggers, you can expect new runs to be started every time (a) an update is made to the repository and (b) a run of the upstream pipeline is completed. Consider an example of a pipeline `B` that depends on `A`. Let us also assume that both of these pipelines use the same repository for the source code, and that both of them also have CI triggers configured. When you push an update to the repository, then:

- A new run of `A` is started.
- At the same time, a new run of `B` is started. This run will consume the previously produced artifacts from `A`.
- As `A` completes, it will trigger another run of `B`.

To prevent triggering two runs of `B` in this example, you must remove its CI trigger or pipeline trigger.

:::moniker-end

:::moniker range="< azure-devops-2020"
Triggers in pipeline resources are not in Azure DevOps Server 2019. Choose the **Classic** tab in the documentation for information on build completion triggers.
:::moniker-end

# [Classic](#tab/classic)

In the classic editor, pipeline triggers are called **build completion triggers**. You can select any other build in the same project to be the triggering pipeline.

After you add a **build completion** trigger, select the **triggering build**. If the triggering build is sourced from a Git repo, you can also specify **branch filters**. If you want to use wildcard characters, then type the branch specification (for example, `features/modules/*`) and then press Enter.

> [!NOTE]
> Keep in mind that in some cases, a single [multi-job build](phases.md) could meet your needs.
> However, a build completion trigger is useful if your requirements include different configuration settings, options, or a different team to own the dependent pipeline.

### Download artifacts from the triggering build

In many cases, you'll want to download artifacts from the triggering build. To do this:

1. Edit your build pipeline.

1. Add the **Download Build Artifacts** task to one of your jobs under **Tasks**.

1. For **Download artifacts produced by**, select **Specific build**.

1. Select the team **Project** that contains the triggering build pipeline.

1. Select the triggering **Build pipeline**.

1. Select **When appropriate, download artifacts from the triggering build**.

1. Even though you specified that you want to download artifacts from the triggering build, you must still select a value for **Build**. The option you choose here determines which build will be the source of the artifacts whenever your triggered build is run because of any other reason than `BuildCompletion` (e.g. `Manual`, `IndividualCI`, `Schedule`, and so on).

1. Specify the **Artifact name** and make sure it matches the name of the artifact published by the triggering build.

1. Specify the **Destination directory** to which you want to download the artifacts. For example: `$(Build.BinariesDirectory)`

---
