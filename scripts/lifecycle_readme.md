ELB and ASG lifecycle event scripts

Often when running a web service, you'll have your instances behind a load balancer. But when deploying new code to these instances, you don't want the load balancer to continue sending customer traffic to an instance while the deployment is in progress. Lifecycle event scripts give you the ability to integrate your AWS CodeDeploy deployments with instances that are behind an Elastic Load Balancer or in an Auto Scaling group. Simply set the name (or names) of the Elastic Load Balancer your instances are a part of, set the scripts in the appropriate lifecycle events, and the scripts will take care of deregistering the instance, waiting for connection draining, and re-registering after the deployment finishes.
Requirements

The register and deregister scripts have a couple of dependencies in order to properly interact with Elastic Load Balancing and AutoScaling:

    The AWS CLI. In order to take advantage of AutoScaling's Standby feature, the CLI must be at least version 1.3.25. If you have Python and PIP already installed, the CLI can simply be installed with pip install awscli. Otherwise, follow the installation instructions in the CLI's user guide.

    An instance profile with a policy that allows, at minimum, the following actions:

     elasticloadbalancing:Describe*
     elasticloadbalancing:DeregisterInstancesFromLoadBalancer
     elasticloadbalancing:RegisterInstancesWithLoadBalancer
     autoscaling:Describe*
     autoscaling:EnterStandby
     autoscaling:ExitStandby
     autoscaling:UpdateAutoScalingGroup
     autoscaling:SuspendProcesses
     autoscaling:ResumeProcesses

    Note: the AWS CodeDeploy Agent requires that an instance profile be attached to all instances that are to participate in AWS CodeDeploy deployments. For more information on creating an instance profile for AWS CodeDeploy, see the Create an IAM Instance Profile for Your Amazon EC2 Instances topic in the documentation.

    All instances are assumed to already have the AWS CodeDeploy Agent installed.

Installing the Scripts

To use these scripts in your own application:

    Install the AWS CLI on all your instances.
    Update the policies on the EC2 instance profile to allow the above actions.
    Copy the .sh files in this directory into your application source.
    Edit your application's appspec.yml to run deregister_from_elb.sh on the ApplicationStop event, and register_with_elb.sh on the ApplicationStart event.
    If your instance is not in an Auto Scaling Group, edit common_functions.sh to set ELB_LIST to contain the name(s) of the Elastic Load Balancer(s) your deployment group is a part of. Make sure the entries in ELB_LIST are separated by space. Alternatively, you can set ELB_LIST to _all_ to automatically use all load balancers the instance is registered to, or _any_ to get the same behaviour as _all_ but without failing your deployments if the instance is not part of any ASG or ELB. This is more flexible in heterogeneous tag-based Deployment Groups.
    Optionally set up HANDLE_PROCS=true in common_functions.sh. See note below.
    Deploy!

Important notice about handling AutoScaling processes

When using AutoScaling with CodeDeploy you have to consider some edge cases during the deployment time window:

    If you have a scale up event, the new instance(s) will get the latest successful Revision, and not the one you are currently deploying. You will end up with a fleet of mixed revisions.
    If you have a scale down event, instances are going to be terminated, and your deployment will (probably) fail.
    If your instances are not balanced accross Availability Zones and you are using these scripts, AutoScaling may terminate some instances or create new ones to maintain balance (see this doc), interfering with your deployment.
    If you have the health checks of your AutoScaling Group based off the ELB's (documentation) and you are not using these scripts, then instances will be marked as unhealthy and terminated, interfering with your deployment.

In an effort to solve these cases, the scripts can suspend some AutoScaling processes (AZRebalance, AlarmNotification, ScheduledActions and ReplaceUnhealthy) while deploying, to avoid those events happening in the middle of your deployment. You only have to set up HANDLE_PROCS=true in common_functions.sh.

A record of the previously (to the start of the deployment) suspended process is kept by the scripts (on each instance), so when finishing the deployment the status of the processes on the AutoScaling Group should be returned to the same status as before. I.e. if AZRebalance was suspended manually it will not be resumed. However, if the scripts don't run (failed deployment) you may end up with stale suspended processes.

Disclaimer: There's a small chance that an event is triggered while the deployment is progressing from one instance to another. The only way to avoid that completely whould be to monitor the deployment externally to CodeDeploy/AutoScaling and act accordingly. The effort on doing that compared to this depends on the each use case.

WARNING: If you are using this functionality you should only use CodeDepoyDefault.OneAtATime deployment configuration to ensure a serial execution of the scripts. Concurrent runs are not supported.