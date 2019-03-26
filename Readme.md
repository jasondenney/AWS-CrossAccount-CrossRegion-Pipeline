Cross Account Pipeline
============================

In order to create the cross account pipeline you must follow the steps below *in order*.

Create a stack using `CrossAccountPrerequisites.template`. This should be run in the build account. This is the account where builds will run and the pipeline will live. This should be deployed in the region in which you want to perform builds and manage the pipeline. It does not need to be the same region as any of your deployments. There is only one parameter that need to be set the first time you run the stack:

- _RootAccountArns_ - set this value to a comma separated list of ARNs representing the root accounts for deployment (arn:aws:iam::${AccountId}:root). This should include the build account.
The remaining values should be left with default or blank settings.

Once the prerequisites stack is completed you will need to create a stack using `CloudFormationDeployer.template` in each of the accounts to which you wish to deploy. This stack can be created in any region because it only creates IAM resources, which are global. Most often this should be created in the same region as you are deploying to. This template has two parameters:
- _BuildAccount_ - set this value to the AWS Account ID of the build account (the account you created the prerequisites stack in).
- _CMKARNs_ - set this to the CMK output value from the prerequisite stack from step one. Later you may need to update this value to include multiple CMK ARNs. If you need to do so you will separate them using commas.

Once the deployer stacks have completed in development, staging, and production you'll need to update the prerequisite stack in the build account. You need to change two values:

- _DeployReady_ - set this value to true.
- _ArtifactBucketAccessRoleArns_ - set this value a comma separated list of the output ARNs from the deployer stacks you created in the previous step (arn:aws:iam::${AccountId}:role/CrossAccountCodePipelineCloudFormation,arn:aws:iam::${AccountId}:role/CrossAccountCloudFormation).
This will update the stack to create an S3 bucket policy with permissions for the roles created in previous steps to access the S3 bucket.

Cross Region
----------------------------

If any of your deployments are in regions other than the build region you will need perform some additional steps. These steps will need to be run once per region you are deploying to.

Create a stack using `CrossAccountRegional.template` in the build account, in the region you are deploying to. There are only two parameters for this stack:
- _RootAccountArns_ - set this value to a comma separated list of ARNs representing the root accounts for deployment (arn:aws:iam::${AccountId}:root). This should include the build account.
- _ArtifactBucketAccessRoleArns_ - set this value a comma separated list of the output ARNs from the deployer stacks you created in the previous step (arn:aws:iam::${AccountId}:role/CrossAccountCodePipelineCloudFormation,arn:aws:iam::${AccountId}:role/CrossAccountCloudFormation).

Once the stack(s) has/have completed you'll need to update the deployer stacks in each account. You will need to change one value:

- _CMKARNs_ - this value should be a comma separated list of all the CMK ARNs created in the build account (one from the prerequisites stack and one for each regional stack).

Once the deployer stacks are complete you will need to update the prerequisites stack. There are a number of settings you'll need to change:
- _BuildS3Bucket_ - set this to the ArtifactBucket output of **this** stack. Whilst the resource is created in this stack it cannot rely on that value because of dependency issues.
- _Replication_ - set this value to true.
- _ReplicationBucketList_ - set this value to a comma separated list of the ArtifactBucket output of each of the regional stacks.
- _ReplicationBucketStarArns_ - set this value to a comma separated list of the S3 ARN with star access for each of the ArtifactBucket outputs. The format is arn:aws:s3:::{ArtifactBucket name}/*
- _ReplicationCMKs_ - set this value to a comma separated list of the CMK output of each of the regional stacks.

This will create a lambda function that replicates the data from the main bucket to each of the regional buckets.

That is all that is needed to create a pipeline that is cross account/cross region. For an example pipeline that uses the values above please look at the `ExamplePipeline.template`.

Developer Account
-----------------
If you are running in a developer account you'll want to take some steps to be sure things are always up to date in your account.

First, go through the steps of creating a cross account pipeline, treating your developer account as both the build account and the deployment account (you'll only deploy to one account), but you'll want to make a few small changes.

In the prerequisites stack:
- you'll need to add the "real" build account's root account ARN to the _RootAccountArns_ parameter.
- you'll need to add the "real" build account's `CrossAccountPipelineS3Replication` role (arn:aws:iam::${AccountId}:role/CrossAccountPipelineS3Replication) to the _ArtifactBucketAccessRoleArns_.

In the "real" build account you'll need to modify the replication settings:
- add your artifact bucket to the _ReplicationBucketList_ parameter.
- add your artifact bucket to the _ReplicationBucketStarArns_ parameter.
- add your CMK to the _ReplicationCMKs_ parameter.

Once this is done you will have a bucket that gets data both from your account's builds and the "real" build account's builds. This will keep your account in sync at all times, whilst allowing you to test on a private fork.


Note On GitHub
----------------
The ExampleProject project uses a GitHub hook for CodeBuild. This hook uses an OAuth connection to AWS, so no GitHub credentials are stored in AWS. In order to configure this you'll need to go to the CodeBuild page and start the process of creating a build project. Follow the directions in [this article](https://www.itonaut.com/2018/06/18/use-github-source-in-aws-codebuild-project-using-aws-cloudformation/) for direction of what you need to do (just the last part of the article).

**_NOTE_**: Some of the permission grants in this code are beyond what you should grant. In order to simplify the build/deploy the grants in the `CloudFormationDeployer.template` are very open (`CrossAccountCloudFormationRole` is granted `arn:aws:iam::aws:policy/AdministratorAccess`). You should tighten these permissions to match the permissions you wish to have on your deployment process.