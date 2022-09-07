# Installation of IBM CP4I 2022.2.1 on AWS ROSA with EBS storage

## Introduction notes

We assume here that the following steps are already completed:
1. Catalog sources for IBM operators are added to the OpenShift cluster - please see [https://www.ibm.com/docs/en/cloud-paks/cp-integration/2022.2?topic=installing-adding-catalog-sources-your-openshift-cluster](https://www.ibm.com/docs/en/cloud-paks/cp-integration/2022.2?topic=installing-adding-catalog-sources-your-openshift-cluster)  
2. Operators for the IBM Cloud Pak for Integration and for the required capabilities installed- please see [https://www.ibm.com/docs/en/cloud-paks/cp-integration/2022.2?topic=installing-operators](https://www.ibm.com/docs/en/cloud-paks/cp-integration/2022.2?topic=installing-operators)

We also assume that there is an OpenShift project (namespace) with the name **cp4i** and that the IBM Entitlement Key secret is already created in that namespace. For the instructions on how to create that secret see the following chapter from the documentation: [https://www.ibm.com/docs/en/cloud-paks/cp-integration/2022.2?topic=installing-applying-your-entitlement-key-online-installation](https://www.ibm.com/docs/en/cloud-paks/cp-integration/2022.2?topic=installing-applying-your-entitlement-key-online-installation)

The following storage classes exist in ROSA out-of-the-box:
```
NAME            PROVISIONER             RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION
gp2             kubernetes.io/aws-ebs   Delete          WaitForFirstConsumer   true                
gp2-csi         ebs.csi.aws.com         Delete          WaitForFirstConsumer   true                
gp3 (default)   ebs.csi.aws.com         Delete          WaitForFirstConsumer   true                
gp3-csi         ebs.csi.aws.com         Delete          WaitForFirstConsumer   true                
```

All of those classes are of the block, RWO type. For the instance of CP4I Platform UI (Platform Navigator), we need an RWX class. So to proceed with the existing classes as they are, we followed the alternative installation method described here: [https://www.ibm.com/docs/en/cloud-paks/cp-integration/2022.2?topic=ui-deploying-platform-rwo-storage](https://www.ibm.com/docs/en/cloud-paks/cp-integration/2022.2?topic=ui-deploying-platform-rwo-storage)
This approach means that we create a kind of RWX "facade" or "wrapper" around the existing RWO class. In our case, we decided to use the existing **gp3** class.
