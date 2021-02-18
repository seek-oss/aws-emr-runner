Run Spark application in aws emr

## Download
You can download the executables in releases page or by using the following script:

Macos:
```bash
curl -sSL https://github.com/weixu365/aws-emr-runner/releases/latest/download/aws-emr-runner-macos.bz2 | \
  bunzip2 > aws-emr-runner | \
  chmod +x aws-emr-runner
```

Linux:
```bash
curl -sSL https://github.com/weixu365/aws-emr-runner/releases/latest/download/aws-emr-runner-linux.bz2 | \
  bunzip2 > aws-emr-runner | \
  chmod +x aws-emr-runner
```
## Usage
#### Validate config files (optional)
```bash
./aws-emr-runner validate -f samples/enrichment-pipeline.yml -s samples/enrichment-pipeline.settings.yml
```

#### Setup resources (optional)
```bash
./aws-emr-runner resources -f samples/enrichment-pipeline.yml -s samples/enrichment-pipeline.settings.yml
```

#### Run EMR Cluster and spark application
```bash
./aws-emr-runner run -f samples/enrichment-pipeline.yml -s samples/enrichment-pipeline.settings.yml
```

#### Start an EMR Cluster and keep it alive until manually terminated
```bash
./aws-emr-runner start-cluster -f samples/enrichment-pipeline.yml -s samples/enrichment-pipeline.settings.yml
```

#### Terminate an EMR Cluster (optional)
```bash
./aws-emr-runner terminate-cluster -f samples/enrichment-pipeline.yml -s samples/enrichment-pipeline.settings.yml
```

## Prerequisite
- EMR Service Role. You can use either the default EMR role `EMR_DefaultRole` (created by `aws emr create-default-roles`) or create a custom role in the resources stack

## Resources stack
- S3 Bucket. Upload package files to this s3 bucket then run EMR steps using this package
- IAM Role and instance profile for emr instance
- (Optional) Lambda function to clean up idled clusters which has the same name
- Any other resource you want to put in the resource stack

## Underlying steps when running a spark application
- Load setting files
- Load config file and evaluate variables except resources variables
- Create or update resources stack
- Get resources from resources stack
- Evaluate all variables in config file
- Generate EMR steps from config file
- Create EMR cluster with defined steps 
- Wait until all steps completed

## Configuration
#### Sample config files
- samples/enrichment-pipeline.yml
- samples/enrichment-pipeline.settings.yml

#### Supported variables
- Environment variable, e.g. `{{env.BUILD_NUMBER}}`
- Values in a settings file, reference through `Values` prefix, e.g. `{{Values.environment}}`
- Resources in resource stack, e.g. `{{Resources.EMRInstanceProfile.PhysicalResourceId}}`
- Predefined variables
  - `{{EmrHadoopDebuggingStep}}` enable debugging in EMR
  - `{{AWSAccountID}}` The current aws account id

#### Supported configurations of EMR cluster
Support all the configs for aws nodejs sdk `new EMR().runJobFlow()` method
https://docs.aws.amazon.com/AWSJavaScriptSDK/latest/AWS/EMR.html#runJobFlow-property

#### Life cycle hook scripts
You could run any command or javascript file at any of the life cycle events, e.g. package your spark application at `package` event
```
scripts:
  package:
    - make docker-package

or if only a single command

scripts:
  package: make docker-package

```

or run `aws emr create-default-roles` before deploying resources stack

```
scripts:
  beforeDeployResources:
    - aws emr create-default-roles
```

Here are all supported life cycle events in order:

For command `run` and `run-step`:
- beforeDeployResources
- afterDeployResources
- beforeLoadResources
- afterLoadResources
- beforePackage
- package
- afterPackage
- beforeUploadPackage
- afterUploadPackage

- beforeRun (only available in `run` command)
- afterRun (only available in `run` command)
- afterComplete (only available in `run` command)

- beforeSubmit (only available in `run-step` command)
- afterSubmit (only available in `run-step` command)
- afterStepComplete (only available in `run-step` command)

For command `start-cluster`:
- beforeDeployResources
- afterDeployResources
- beforeLoadResources
- afterLoadResources
- beforeStartCluster
- beforeWaitForClusterStarted
- afterClusterStarted

For command `terminate-cluster`:
- beforeTerminateCluster
- beforeWaitForClusterTerminated
- afterClusterTerminated

## FAQ

#### How to automatically terminate an idled cluster
By using `maxIdleMinutes` in the config file, aws-emr-runner will setup a scheduled task to check the idled clusters, and terminate cluster if it has idled longer than `maxIdleMinutes`, e.g.

```yaml
deploy:
  ...
  maxIdleMinutes: 30 # Will automatically terminate the cluster if exceed max idle minutes
```

#### How to assume a different role to access s3 bucket
You can assume different roles by pre-defined rules using [`AWS::EMR::SecurityConfiguration`](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-emr-securityconfiguration.html), e.g.
```yaml
cluster:
  ...
  SecurityConfiguration: '{{Resources.EMRSecurityConfiguration.PhysicalResourceId}}'

resources:
  EMRSecurityConfiguration:
    Type: AWS::EMR::SecurityConfiguration
    Properties:
      Name: sample-spark-pipeline-emr-securityconfiguration
      SecurityConfiguration:
        AuthorizationConfiguration:
          EmrFsConfiguration:
            RoleMappings:
              -
                Role: "arn:aws:iam::<account-id>:role/<role>"
                IdentifierType: Prefix
                Identifiers:
                  - "s3://your-bucket/"
```

## Contribute

### Commit log format
Use `npx cz` to commit changes, so that we automatically create release notes
