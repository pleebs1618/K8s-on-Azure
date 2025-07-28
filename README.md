# K8s-on-Azure
Notes created while following Kelsey Hightower's 'K8s The Hard Way'

## Kubernetes the Hard Way on Azure

This guide documents how to manually set up a Kubernetes cluster on Microsoft Azure using Windows as the management machine.
It follows the concepts of Kelsey Hightower’s “Kubernetes the Hard Way”, adapted for Azure and Windows PowerShell.

## Prerequisites

Install Azure CLI and Create a Resource Group

```
az login
az group create -n kubernetes -l eastus2
```

