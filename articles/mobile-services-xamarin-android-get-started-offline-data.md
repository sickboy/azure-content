<properties linkid="develop-mobile-tutorials-get-started-offline-data-android" urlDisplayName="Using Offline Data" pageTitle="Using offline data in Mobile Services (Xamarin Android) | Mobile Dev Center" metaKeywords="" description="Learn how to use Azure Mobile Services to cache and sync offline data in your Xamarin Android application" metaCanonical="" disqusComments="1" umbracoNaviHide="1" documentationCenter="Mobile" title="Using offline data in Mobile Services" authors="donnam" editor="wesmc" manager="dwrede"/>

<tags ms.service="mobile-services" ms.workload="mobile" ms.tgt_pltfrm="mobile-xamarin-android" ms.devlang="dotnet" ms.topic="article" ms.date="09/25/2014" ms.author="donnam" />

# Using offline data sync in Mobile Services

[WACOM.INCLUDE [mobile-services-selector-offline](../includes/mobile-services-selector-offline.md)]

This topic shows you how to use use the offline capabilities of Azure Mobile Services. These features allow you to interact with a local database when you are in an offline scenario with your Mobile Service. The offline features allow you to sync your local changes with the mobile service when you are online again. 

In this tutorial, you will update the app from the [Get started with Mobile Services] or [Get Started with Data] tutorial to support the offline features of Azure Mobile Services. Then you will add data in a disconnected offline scenario, sync those items to the online database, and then log in to the Azure Management Portal to view changes to data made when running the app.

>[WACOM.NOTE] This tutorial is intended to help you better understand how Mobile Services enables you to use Azure to store and retrieve data in a Windows Store app. As such, this topic walks you through many of the steps that are completed for you in the Mobile Services quickstart. If this is your first experience with Mobile Services, consider first completing the tutorial [Get started with Mobile Services].
>
> To complete this tutorial, you need a Azure account. If you don't have an account, you can create a free trial account in just a couple of minutes. For details, see <a href="http://www.windowsazure.com/en-us/pricing/free-trial/?WT.mc_id=AE564AB28" target="_blank">Azure Free Trial</a>. 

A project with the completed state of this tutorial is available [here](https://github.com/Azure/mobile-services-samples/tree/master/TodoOffline/Xamarin.Android).

This tutorial walks you through these basic steps:

1. [Update the app to support offline features]
2. [Test the app connected to the Mobile Service]

This tutorial requires the following:

* Visual Studio with the [Xamarin extension] **or** [Xamarin Studio] 
* Completion of the [Get started with Mobile Services] or [Get Started with Data] tutorial
* [Azure Mobile Services SDK version 1.3.0-beta2][Mobile Services SDK Nuget]
* [Azure Mobile Services SQLite Store version 1.0.0-beta2][SQLite store nuget]

>[WACOM.NOTE] The instructions below assume you are using Visual Studio 2012 or higher with the Xamarin extension. If you are using Xamarin Studio, use the built-in NuGet package manager support.

## <a name="enable-offline-app"></a>Update the app to support offline features

Azure Mobile Services offline sync allows end users to interact with a local database when the network is not accessible. To use these features in your app, you initialize `MobileServiceClient.SyncContext` to a local store. Then reference your table through the `IMobileServiceSyncTable` interface.

1. In Visual Studio open the project that you completed in the [Get started with Mobile Services] or [Get Started with Data] tutorial. In Solution Explorer, remove the reference to **Azure Mobile Services SDK** in **Components**.

2. Install the prerelease package of the Mobile Services SQLiteStore using the following command in Package Manager Console: 
    
        install-package WindowsAzure.MobileServices.SQLiteStore -Pre

    This will also install all of the required dependencies.
    
3. In the references node, remove the references to `System.IO`, `System.Runtime` and `System.Threading.Tasks`.

### Edit ToDoActivity.cs

1. Add the declarations

		using Microsoft.WindowsAzure.MobileServices.Sync;
		using Microsoft.WindowsAzure.MobileServices.SQLiteStore;
		using System.IO;

2. Change the type of the member `ToDoActivity.toDoTable` from  `IMobileServiceTable<>` to `IMobileServiceSyncTable<>`

3. In the method `OnCreate(Bundle)`, after the line that initializes the member `client`, add the following code:

	    // existing initializer
	    client = new MobileServiceClient (applicationURL, applicationKey, progressHandler);
	
		// new code to initialize the SQLite store
	    string path = Path.Combine(System.Environment.GetFolderPath(System.Environment.SpecialFolder.Personal), "test1.db");
	
	    if (!File.Exists(path))
	    {
	        File.Create(path).Dispose();
	    }

	    var store = new MobileServiceSQLiteStore(path);
	    store.DefineTable<ToDoItem>();
	
	    await client.SyncContext.InitializeAsync(store, new TodoSyncHandler(this));

4. In the same method, change the line that initializes `toDoTable` to use the method `GetSyncTable<>` instead of `GetTable<>`:

		toDoTable = client.GetSyncTable <ToDoItem> ();

5. Modify the method `OnRefreshItemsSelected` to add calls to `PushAsync` and `PullAsync`:

		async void OnRefreshItemsSelected ()
		{
		    await client.SyncContext.PushAsync();
			await this.todoTable.PullAsync("todoItems", todoTable.CreateQuery());
		    await RefreshItemsFromTableAsync();
		}

### Edit ToDoItem.cs 

1. Add the using statement: 

        using Microsoft.WindowsAzure.MobileServices; 


2. Add the following members to the class `ToDoItem`:
 
		[Version]
		public string Version { get; set; }
		
		
		public override string ToString()
		{
		    return "Text: " + Text + "\nComplete: " + Complete + "\n";
		}

## <a name="test-online-app"></a>Test the app 

In this section you will test the  `SyncAsync` method that synchronizes the local store with the mobile service database.

1. In Visual Studio, press the **Run** button to build the project and start the app in the iPhone emulator, which is the default for this project.

2. Notice that the list of items in the app is empty. As a result of the code changes in the previous section, the app no longer reads items from the mobile service, but rather from the local store. 

3. Add items to the To Do list.

    ![][1]


4. Log into the Microsoft Azure Management portal and look at the database for your mobile service. If your service uses the JavaScript backend for mobile services, you can browse the data from the **Data** tab of the mobile service. If you are using the .NET backend for your mobile service, you can click on the **Manage** button for your database in the SQL Azure Extension to execute a query against your table.

    Notice the data has not been synchronized between the database and the local store.

5. In the app, push the **Refresh** button. This causes the app to call `MobileServiceClient.SyncContext.PushAsync` and `IMobileServiceSyncTable.PullAsync()`, then `RefreshTodoItems` to refresh the app with the items from the local store. 

    The push operation results in the mobile service database receiving the data from the store. It is executed off the `MobileServiceClient.SyncContext` instead of the `IMobileServicesSyncTable` and pushes changes on all tables associated with that sync context. This is to cover scenarios where there are relationships between tables.
    
    In contrast, the pull operation retrieves records from only the table that was specified. If there are pending operations for this table in the sync context, a `PushAsync` operation will be implictly called by the Mobile Services SDK.
        
    ![][3] 



    ![][2]


  

##Summary

In order to support the offline features of mobile services, we used the `IMobileServiceSyncTable` interface and initialized `MobileServiceClient.SyncContext` with a local store. In this case the local store was a SQLite database.

The normal CRUD operations for mobile services work as if the app is still connected but, all the operations occur against the local store.

When we wanted to synchronize the local store with the server, we used the `IMobileServiceSyncTable.PullAsync` and `MobileServiceClient.SyncContext.PushAsync` methods.

*  To push changes to the server, we called `IMobileServiceSyncContext.PushAsync()`. This method is a member of `IMobileServicesSyncContext` instead of the sync table because it will push changes across all tables:

    Only records that have been modified in some way locally (through CUD operations) will be sent to the server.
   
* To pull data from a table on the server to the app, we called `IMobileServiceSyncTable.PullAsync`.

    A pull always issues a push first.  

    This sample uses an overload of **PullAsync()** that allows specifying a query key and a query. The query key is used for incremental sync. The Mobile Services SDK tracks the last updated timestamp after each successful pull operation. On the next pull, only newer records will be retrieved. If no query key is specified, a full sync will be performed for the sync table.

## Next steps

You can download the completed version of this tutorial in our [GitHub samples repository](https://github.com/Azure/mobile-services-samples/tree/master/TodoOffline/Xamarin.Android).

<!--* [Handling conflicts with offline support for Mobile Services]
-->
* [How to use the Xamarin Component client for Azure Mobile Services]

<!-- Anchors. -->
[Update the app to support offline features]: #enable-offline-app
[Test the app connected to the Mobile Service]: #test-online-app
[Next Steps]:#next-steps

<!-- Images -->
[1]: ./media/mobile-services-xamarin-android-get-started-offline-data/mobile-quickstart-startup-android.png
[2]: ./media/mobile-services-xamarin-android-get-started-offline-data/mobile-data-browse.png
[3]: ./media/mobile-services-xamarin-android-get-started-offline-data/mobile-quickstart-completed-android.png



<!-- URLs. -->
[Handling conflicts with offline support for Mobile Services]: /en-us/documentation/articles/mobile-services-xamarin-android-handling-conflicts-offline-data/ 
[Get started with data]: /en-us/documentation/articles/partner-xamarin-mobile-services-android-get-started-data/
[Get started with Mobile Services]: /en-us/documentation/articles/partner-xamarin-mobile-services-android-get-started/
[How to use the Xamarin Component client for Azure Mobile Services]: /en-us/documentation/articles/partner-xamarin-mobile-services-how-to-use-client-library/

[Mobile Services SDK Nuget]: http://www.nuget.org/packages/WindowsAzure.MobileServices/1.3.0-beta2
[SQLite store nuget]: http://www.nuget.org/packages/WindowsAzure.MobileServices.SQLiteStore/1.0.0-beta2
[Xamarin Studio]: http://xamarin.com/download
[Xamarin extension]: http://xamarin.com/visual-studio
[NuGet Addin for Xamarin]: https://github.com/mrward/monodevelop-nuget-addin
