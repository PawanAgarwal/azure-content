<properties 
	pageTitle="Copying an Existing Blob into a Media Services Asset" 
	description="This topic shows how to copy an existing blob into a Media Services Asset." 
	services="media-services" 
	documentationCenter="" 
	authors="Juliako" 
	manager="dwrede" 
	editor=""/>

<tags 
	ms.service="media-services" 
	ms.workload="media" 
	ms.tgt_pltfrm="na" 
	ms.devlang="ne" 
	ms.topic="article" 
	ms.date="09/07/2015" 
	ms.author="juliako"/>

#Copying an Existing Blob into a Media Services Asset

This topic shows how to copy blobs from a storage account into a new Microsoft Azure Media Services asset.

Your blobs could exist in a storage account that is associated with Media Services account, or storage account that is not associated with Media Services account. This topic demonstrates how to copy blobs from a storage account into a Media Services Asset. Note that you can also copy across Data Centers. However, there may be charges incurred by doing so. For more information about pricing, see [Data Transfers](http://azure.microsoft.com/pricing/#header-11).

>[AZURE.NOTE] You should not attempt to change the contents of blob containers that were generated by Media Services without using Media Service APIs.

##Prerequisites

- Two Media Services accounts in a new or existing Azure subscription. See the topic [How to Create a Media Services Account](media-services-create-account.md).
- Operating Systems: Windows 7, Windows 2008 R2, or Windows 8.
- .NET Framework 4.5.
- Visual Studio 2013, Visual Studio 2012 or Visual Studio 2010 SP1 (Professional, Premium, Ultimate, or Express).

##Set up your project

In this section you will create and set up a C# Console Application project.

1. Use Visual Studio 2012 or Visual Studio 2010 SP1 to create a new solution that contains the C# Console Application project. 
2. Enter CopyExistingBlobsIntoAsset for the Name and then click OK.
1. Use Nuget to add references to Media Services related DLLs. In Visual Studio Main Menu, select TOOLS -> Library Package Manager -> Package Manager Console. In the console window type Install-Package windowsazure.mediaservices and press enter.
1. Add other references that are required for this project: System.Configuration.
1. Replace using statements that were added to the Programs.cs file by default with the following ones:
		
		using System;
		using System.Linq;
		using System.Configuration;
		using System.IO;
		using System.Text;
		using System.Threading;
		using System.Threading.Tasks;
		using System.Collections.Generic;
		using Microsoft.WindowsAzure.Storage;
		using Microsoft.WindowsAzure.MediaServices.Client;
		using Microsoft.WindowsAzure;
		using System.Web;
		using Microsoft.WindowsAzure.Storage.Blob;
		using Microsoft.WindowsAzure.Storage.Auth;

1. Add the appSettings section to the .config file and update the values based on your Media Services and Storage key and name values. 

		<appSettings>
		  <add key="MediaServicesAccountName" value="Media-Services-Account-Name"/>
		  <add key="MediaServicesAccountKey" value="Media-Services-Account-Key"/>
		  <add key="MediaServicesStorageAccountName" value="Media-Services-Storage-Account-Name"/>
		  <add key="MediaServicesStorageAccountKey" value="Media-Services-Storage-Account-Key"/>
		  <add key="ExternalStorageAccountName" value="External-Storage-Account-Name"/>
		  <add key="ExternalStorageAccountKey" value="External-Storage-Account-Key"/>
		</appSettings>


##Copy Blobs from a Storage Account into a Media Services Asset

The code example below performs the following tasks:

1. Creates the CloudMediaContext instance. 
1. Creates CloudStorageAccount instances: _sourceStorageAccount and _destinationStorageAccount.
1. Uploads Smooth Streaming files from a local directory into a blob container that is located in _sourceStorageAccount. 
1. Creates a new asset. The blob container that that is created for this asset is located in _destinationStorageAccount. 
1. Uses Azure Storage SDK to copy the specified blobs into the container associated with the asset.

>[AZURE.NOTE]The copy operation does not throw an exception if the locator has expired.

1. Sets the .ism file to be the primary file.
1. Creates the Smooth Streaming URL for the OnDemandOrigin locator associated with the asset. 
		
		class Program
		{
		    // Read values from the App.config file. 
		    static string _accountName = ConfigurationManager.AppSettings["MediaServicesAccountName"];
		    static string _accountKey = ConfigurationManager.AppSettings["MediaServicesAccountKey"];
		    static string _storageAccountName = ConfigurationManager.AppSettings["MediaServicesStorageAccountName"];
		    static string _storageAccountKey = ConfigurationManager.AppSettings["MediaServicesStorageAccountKey"];
		    static string _externalStorageAccountName = ConfigurationManager.AppSettings["ExternalStorageAccountName"];
		    static string _externalStorageAccountKey = ConfigurationManager.AppSettings["ExternalStorageAccountKey"];
		
		    private static MediaServicesCredentials _cachedCredentials = null;
		    private static CloudMediaContext _context = null;
		
		    private static CloudStorageAccount _sourceStorageAccount = null;
		    private static CloudStorageAccount _destinationStorageAccount = null;
		
		    static void Main(string[] args)
		    {
		        _cachedCredentials = new MediaServicesCredentials(
		                        _accountName,
		                        _accountKey);
		        // Use the cached credentials to create CloudMediaContext.
		        _context = new CloudMediaContext(_cachedCredentials);
		
		        // In this example the storage account from which we copy blobs is not 
		        // associated with the Media Services account into which we copy blobs.
		        // But the same code will work for coping blobs from a storage account that is 
		        // associated with the Media Services account.
		        //
		        // Get a reference to a storage account that is not associated with a Media Services account
		        // (an external account).  
		        StorageCredentials externalStorageCredentials =
		            new StorageCredentials(_externalStorageAccountName, _externalStorageAccountKey);
		        _sourceStorageAccount = new CloudStorageAccount(externalStorageCredentials, true);
		
		        //Get a reference to the storage account that is associated with a Media Services account. 
		        StorageCredentials mediaServicesStorageCredentials =
		            new StorageCredentials(_storageAccountName, _storageAccountKey);
		        _destinationStorageAccount = new CloudStorageAccount(mediaServicesStorageCredentials, false);
		
		        // Upload Smooth Streaming files into a storage account.
		        string localMediaDir = @"C:\supportFiles\streamingfiles";
		        CloudBlobContainer blobContainer =
		            UploadContentToStorageAccount(localMediaDir);
		
		        // Create a new asset and copy the smooth streaming files into 
		        // the container that is associated with the asset.
		        IAsset asset = CreateAssetFromExistingBlobs(blobContainer);
		
		        // Get the streaming URL for the smooth streaming files 
		        // that were copied into the asset.   
		        string urlForClientStreaming = CreateStreamingLocator(asset);
		        Console.WriteLine("Smooth Streaming URL: " + urlForClientStreaming);
		
		        Console.ReadLine();
		    }
		
		    /// <summary>
		    /// Uploads content from a local directory into the specified storage account.
		    /// In this example the storage account is not associated with the Media Services account.
		    /// </summary>
		    /// <param name="localPath">The path from which to upload the files.</param>
		    /// <returns>The container that contains the uploaded files.</returns>
		    static public CloudBlobContainer UploadContentToStorageAccount(string localPath)
		    {
		        CloudBlobClient externalCloudBlobClient = _sourceStorageAccount.CreateCloudBlobClient();
		        CloudBlobContainer externalMediaBlobContainer = externalCloudBlobClient.GetContainerReference("streamingfiles");
		
		        externalMediaBlobContainer.CreateIfNotExists();
		
		        // Upload files to the blob container.  
		        DirectoryInfo uploadDirectory = new DirectoryInfo(localPath);
		        foreach (var file in uploadDirectory.EnumerateFiles())
		        {
		            CloudBlockBlob blob = externalMediaBlobContainer.GetBlockBlobReference(file.Name);
		
		            blob.UploadFromFile(file.FullName, FileMode.Open);
		        }
		
		        return externalMediaBlobContainer;
		    }
		
		    /// <summary>
		    /// Creates a new asset and copies blobs from the specifed storage account.
		    /// </summary>
		    /// <param name="mediaBlobContainer">The specified blob container.</param>
		    /// <returns>The new asset.</returns>
		    static public IAsset CreateAssetFromExistingBlobs(CloudBlobContainer mediaBlobContainer)
		    {
		        // Create a new asset. 
		        IAsset asset = _context.Assets.Create("Burrito_" + Guid.NewGuid(), AssetCreationOptions.None);
		
		        IAccessPolicy writePolicy = _context.AccessPolicies.Create("writePolicy", TimeSpan.FromHours(24), AccessPermissions.Write);
		        ILocator destinationLocator = _context.Locators.CreateLocator(LocatorType.Sas, asset, writePolicy);
		
		        CloudBlobClient destBlobStorage = _destinationStorageAccount.CreateCloudBlobClient();
		
		        // Get the asset container URI and Blob copy from mediaContainer to assetContainer. 
		        string destinationContainerName = (new Uri(destinationLocator.Path)).Segments[1];
		
		        CloudBlobContainer assetContainer = destBlobStorage.GetContainerReference(destinationContainerName);
		
		        if (assetContainer.CreateIfNotExists())
		        {
		            assetContainer.SetPermissions(new BlobContainerPermissions
		            {
		                PublicAccess = BlobContainerPublicAccessType.Blob
		            });
		        }
		
		        var blobList = mediaBlobContainer.ListBlobs();
		        foreach (var sourceBlob in blobList)
		        {
		            var assetFile = asset.AssetFiles.Create((sourceBlob as ICloudBlob).Name);
		            CopyBlob(sourceBlob as ICloudBlob, assetContainer);
		            assetFile.ContentFileSize = (sourceBlob as ICloudBlob).Properties.Length;
		            assetFile.Update();
		        }
		
		        destinationLocator.Delete();
		        writePolicy.Delete();
		
		        // Since we copied a set of Smooth Streaming files, 
		        // set the .ism file to be the primary file. 
		        SetISMFileAsPrimary(asset);
		
		        return asset;
		    }
		
		    /// <summary>
		    /// Creates the OnDemandOrigin locator in order to get the streaming URL.
		    /// </summary>
		    /// <param name="asset">The asset that contains the smooth streaming files.</param>
		    /// <returns>The streaming URL.</returns>
		    static public string CreateStreamingLocator(IAsset asset)
		    {
		        var ismAssetFile = asset.AssetFiles.ToList().
		            Where(f => f.Name.EndsWith(".ism", StringComparison.OrdinalIgnoreCase)).First();
		
		        // Create a 30-day readonly access policy. 
		        IAccessPolicy policy = _context.AccessPolicies.Create("Streaming policy",
		            TimeSpan.FromDays(30),
		            AccessPermissions.Read);
		
		        // Create a locator to the streaming content on an origin. 
		        ILocator originLocator = _context.Locators.CreateLocator(LocatorType.OnDemandOrigin, asset,
		            policy,
		            DateTime.UtcNow.AddMinutes(-5));
		
		        return originLocator.Path + ismAssetFile.Name + "/manifest";
		    }
		
		    /// <summary>
		    /// Copies the specified blob into the specified container.
		    /// </summary>
		    /// <param name="sourceBlob">The source container.</param>
		    /// <param name="destinationContainer">The destination container.</param>
		    static private void CopyBlob(ICloudBlob sourceBlob, CloudBlobContainer destinationContainer)
		    {
		        var signature = sourceBlob.GetSharedAccessSignature(new SharedAccessBlobPolicy
		        {
		            Permissions = SharedAccessBlobPermissions.Read,
		            SharedAccessExpiryTime = DateTime.UtcNow.AddHours(24)
		        });
		
		        ICloudBlob destinationBlob = destinationContainer.GetBlockBlobReference(sourceBlob.Name);
		
		        if (destinationBlob.Exists())
		        {
		            Console.WriteLine(string.Format("Destination blob '{0}' already exists. Skipping.", destinationBlob.Uri));
		        }
		        else
		        {
		            try
		            {
		                Console.WriteLine(string.Format("Copy blob '{0}' to '{1}'", sourceBlob.Uri, destinationBlob.Uri));
		                destinationBlob.StartCopy(sourceBlob.Uri);

	                    while (destinationBlob.CopyState.Status == CopyStatus.Pending)
	                    {
	                        Thread.Sleep(10000);
	                    }
	
	                    Console.WriteLine("Final Copy status = " + destinationBlob.CopyState.Status);

		            }
		            catch (Exception ex)
		            {
		                Console.WriteLine(string.Format("Error copying blob '{0}': {1}", sourceBlob.Name, ex.Message));
		            }
		        }
		    }
		
		    /// <summary>
		    /// Sets a file with the .ism extension as a primary file.
		    /// </summary>
		    /// <param name="asset">The asset that contains the smooth streaming files.</param>
		    static private void SetISMFileAsPrimary(IAsset asset)
		    {
		        var ismAssetFiles = asset.AssetFiles.ToList().
		            Where(f => f.Name.EndsWith(".ism", StringComparison.OrdinalIgnoreCase)).ToArray();
		
		        if (ismAssetFiles.Count() != 1)
		            throw new ArgumentException("The asset should have only one, .ism file");
		
		        ismAssetFiles.First().IsPrimary = true;
		        ismAssetFiles.First().Update();
		    }
		}
 

##Media Services learning paths

You can view AMS learning paths here:

- [AMS Live Streaming Workflow](http://azure.microsoft.com/documentation/learning-paths/media-services-streaming-live/)
- [AMS on Demand Streaming Workflow](http://azure.microsoft.com/documentation/learning-paths/media-services-streaming-on-demand/)

