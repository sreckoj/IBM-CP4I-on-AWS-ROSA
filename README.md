# Installation of IBM CP4I 2022.2.1 on AWS ROSA with EBS storage

## Introduction notes

We assume here that the following steps are already completed:
1. Catalog sources for IBM operators are added to the OpenShift cluster - please see [https://www.ibm.com/docs/en/cloud-paks/cp-integration/2022.2?topic=installing-adding-catalog-sources-your-openshift-cluster](https://www.ibm.com/docs/en/cloud-paks/cp-integration/2022.2?topic=installing-adding-catalog-sources-your-openshift-cluster)  
2. Operators for the IBM Cloud Pak for Integration and for the required capabilities installed- please see [https://www.ibm.com/docs/en/cloud-paks/cp-integration/2022.2?topic=installing-operators](https://www.ibm.com/docs/en/cloud-paks/cp-integration/2022.2?topic=installing-operators) 

We also asume that there is an OpenShift project (namespace) with the name **cp4i** and that the IBM Entitlement Key secret is already created in that namespace. For the instructions on how to create that secret see the follwing chapter from the documentation: [https://www.ibm.com/docs/en/cloud-paks/cp-integration/2022.2?topic=installing-applying-your-entitlement-key-online-installation](https://www.ibm.com/docs/en/cloud-paks/cp-integration/2022.2?topic=installing-applying-your-entitlement-key-online-installation)