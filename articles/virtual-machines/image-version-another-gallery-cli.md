---
title: Copy an image version from another gallery using the CLI
description: Copy an image version from another gallery with the Azure CLI.
author: cynthn
ms.service: virtual-machines
ms.subservice: shared-image-gallery
ms.topic: how-to
ms.workload: infrastructure
ms.date: 05/04/2020
ms.author: cynthn
ms.reviewer: akjosh

---

# Copy an image from another gallery using the Azure CLI

**Applies to:** :heavy_check_mark: Linux VMs :heavy_check_mark: Windows VMs :heavy_check_mark: Flexible scale sets :heavy_check_mark: Uniform scale sets

If you have multiple galleries in your organization, you can also create image versions from existing image versions stored in other galleries. For example, you might have a development and test gallery for creating and testing new images. When they are ready to be used in production, you can copy them into a production gallery using this example. You can also create an image from an image in another gallery using [Azure PowerShell](image-version-another-gallery-powershell.md).



## Before you begin

To complete this article, you must have an existing source gallery, image definition, and image version. You should also have a destination gallery. 

The source image version must be replicated to the region where your destination gallery is located. 

When working through this article, replace the resource names where needed.



## Get information from the source gallery

You will need information from the source image definition so you can create a copy of it in your new gallery.

List information about the available image galleries using [az sig list](/cli/azure/sig#az_sig_list) to find information about the source gallery.

```azurecli-interactive 
az sig list -o table
```

List the image definitions in a gallery, using [az sig image-definition list](/cli/azure/sig/image-definition#az_sig_image_definition_list). In this example, we are searching for image definitions in the gallery named *myGallery* in the *myGalleryRG* resource group.

```azurecli-interactive 
az sig image-definition list \
   --resource-group myGalleryRG \
   --gallery-name myGallery \
   -o table
```

List the versions of an image in a gallery, using [az sig image-version list](/cli/azure/sig/image-version#az_sig_image_version_list) to find the image version that you want to copy into your new gallery. In this example, we are looking for all of the image versions that are part of the *myImageDefinition* image definition.

```azurecli-interactive
az sig image-version list \
   --resource-group myGalleryRG \
   --gallery-name myGallery \
   --gallery-image-definition myImageDefinition \
   -o table
```

Once you have all of the information you need, you can get the ID of the source image version using [az sig image-version show](/cli/azure/sig/image-version#az_sig_image_version_show).

```azurecli-interactive
az sig image-version show \
   --resource-group myGalleryRG \
   --gallery-name myGallery \
   --gallery-image-definition myImageDefinition \
   --gallery-image-version 1.0.0 \
   --query "id" -o tsv
```


## Create the image definition 

You need to create an image definition that matches the operating system, operating system state, and Hyper-V generation of the image definition containing your source image version. You can see all of the information you need to recreate the image definition in your new gallery using [az sig image-definition show](/cli/azure/sig/image-definition#az_sig_image_definition_show).

```azurecli-interactive
az sig image-definition show \
   --resource-group myGalleryRG \
   --gallery-name myGallery \
   --gallery-image-definition myImageDefinition
```

The output will look something like this:

```output
{
  "description": null,
  "disallowed": null,
  "endOfLifeDate": null,
  "eula": null,
  "hyperVgeneration": "V1",
  "id": "/subscriptions/1111abcd-1a23-4b45-67g7-1234567de123/resourceGroups/myGalleryRG/providers/Microsoft.Compute/galleries/myGallery/images/myImageDefinition",
  "identifier": {
    "offer": "myOffer",
    "publisher": "myPublisher",
    "sku": "mySKU"
  },
  "location": "eastus",
  "name": "myImageDefinition",
  "osState": "Specialized",
  "osType": "Linux",
  "privacyStatementUri": null,
  "provisioningState": "Succeeded",
  "purchasePlan": null,
  "recommended": null,
  "releaseNoteUri": null,
  "resourceGroup": "myGalleryRG",
  "tags": null,
  "type": "Microsoft.Compute/galleries/images"
}
```

Create a new image definition, in your new gallery, using the information from the output above.


```azurecli-interactive 
az sig image-definition create \
   --resource-group myNewGalleryRG \
   --gallery-name myNewGallery \
   --gallery-image-definition myImageDefinition \
   --publisher myPublisher \
   --offer myOffer \
   --sku mySKU \
   --os-type Linux \
   --hyper-v-generation V1 \
   --os-state specialized 
```

> [!NOTE]
> For image definitions that will contain images descended from third-party images, the plan information must match exactly the plan information from the third-party image. Include the plan information in the image definition by adding `--plan-name`, `--plan-product`, and `--plan-publisher` when you create the image definition.
>


## Create the image version

Create versions using [az image gallery create-image-version](/cli/azure/sig/image-version#az_sig_image_version_create). You will need to pass in the ID of the managed image to use as a baseline for creating the image version. You can use [az image list](/cli/azure/image?view#az_image_list) to get information about images that are in a resource group. 

Allowed characters for image version are numbers and periods. Numbers must be within the range of a 32-bit integer. Format: *MajorVersion*.*MinorVersion*.*Patch*.

In this example, the version of our image is *1.0.0* and we are going to create 1 replica in the *South Central US* region and 1 replica in the *East US* region using zone-redundant storage.


```azurecli-interactive 
az sig image-version create \
   --resource-group myNewGalleryRG \
   --gallery-name myNewGallery \
   --gallery-image-definition myImageDefinition \
   --gallery-image-version 1.0.0 \
   --target-regions "southcentralus=1" "eastus=1=standard_zrs" \
   --replica-count 2 \
   --managed-image "/subscriptions/<Subscription ID>/resourceGroups/myGalleryRG/providers/Microsoft.Compute/galleries/myGallery/images/myImageDefinition/versions/1.0.0"
```

> [!NOTE]
> You need to wait for the image version to completely finish being built and replicated before you can use the same managed image to create another image version.
>
> You can also store your image in Premium storage by adding `--storage-account-type  premium_lrs`, or [Zone Redundant Storage](../storage/common/storage-redundancy.md) by adding `--storage-account-type  standard_zrs` when you create the image version.
>

## Next steps

Create a VM from a [generalized](vm-generalized-image-version-cli.md) or a [specialized](vm-specialized-image-version-cli.md) image version.

Also, try out [Azure Image Builder](./image-builder-overview.md) can help automate image version creation, you can even use it to update and [create a new image version from an existing image version](./linux/image-builder-gallery-update-image-version.md). 

For information about how to supply purchase plan information, see [Supply Azure Marketplace purchase plan information when creating images](marketplace-images.md).
