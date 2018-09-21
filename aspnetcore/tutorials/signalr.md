---
title: "Tutorial: Get started with SignalR on ASP.NET Core"
author: tdykstra
description: In this tutorial, you create a chat app that uses SignalR for ASP.NET Core.
monikerRange: '>= aspnetcore-2.1'
ms.author: tdykstra
ms.custom: mvc
ms.date: 08/31/2018
uid: tutorials/signalr
---

# Tutorial: Get started with SignalR on ASP.NET Core

This tutorial teaches the basics of building a real-time app using SignalR. You learn how to:

> [!div class="checklist"]
> * Create a web app that uses SignalR on ASP.NET Core.
> * Create a SignalR hub on the server.
> * Connect to the SignalR hub from JavaScript clients.
> * Use the hub to send messages from any client to all connected clients.

At the end, you'll have a working chat app:

![SignalR sample app](signalr/_static/signalr-get-started-finished.png)

[View or download sample code](https://github.com/aspnet/Docs/tree/master/aspnetcore/tutorials/signalr/sample) ([how to download](xref:tutorials/index#how-to-download-a-sample)).

## Prerequisites

# [Visual Studio](#tab/visual-studio)

* [Visual Studio 2017 version 15.8 or later](https://www.visualstudio.com/downloads/) with the **ASP.NET and web development** workload
* [.NET Core SDK 2.1 or later](https://www.microsoft.com/net/download/all)

# [Visual Studio Code](#tab/visual-studio-code)

* [Visual Studio Code](https://code.visualstudio.com/download)
* [.NET Core SDK 2.1 or later](https://www.microsoft.com/net/download/all)
* [C# for Visual Studio Code](https://marketplace.visualstudio.com/items?itemName=ms-vscode.csharp)

# [Visual Studio for Mac](#tab/visual-studio-mac)

* [Visual Studio for Mac version 7.5.4 or later](https://www.visualstudio.com/downloads/)
* [.NET Core SDK 2.1 or later](https://www.microsoft.com/net/download/all) (included in the Visual Studio install)

---

## Create the project

# [Visual Studio](#tab/visual-studio/)

* From the menu, select **File > New Project**.

* In the **New Project** dialog, select **Installed > Visual C# > Web > ASP.NET Core Web Application**. Name the project *SignalRChat*.

  ![New Project dialog in Visual Studio](signalr/_static/signalr-new-project-dialog.png)

* Select **Web Application** to create a project that uses Razor Pages.

* Select a target framework of **.NET Core**, select **ASP.NET Core 2.1**, and click **OK**.

  ![New Project dialog in Visual Studio](signalr/_static/signalr-new-project-choose-type.png)

# [Visual Studio Code](#tab/visual-studio-code/)

* Open a folder that you can use for a new project.

* In the **Integrated Terminal**, run the following command:

   ```console
   dotnet new webapp -o SignalRChat
   ```

# [Visual Studio for Mac](#tab/visual-studio-mac)

* From the menu, select **File > New Solution**.

* Select **.NET Core > App > ASP.NET Core Web App** (Don't select **ASP.NET Core Web App (MVC)**).

* Select **Next**.

* Name the project *SignalRChat*, and then select **Create**.

---

## Add the SignalR client library

The SignalR server library is included in the [Microsoft.AspNetCore.App metapackage](xref:fundamentals/metapackage-app). The JavaScript client library isn't automatically included in the project. For this tutorial, you use [Library Manager (LibMan)](xref:client-side/libman/index) to get the client library from *unpkg*. [unpkg](https://unpkg.com/#/) is a [content delivery network](https://wikipedia.org/wiki/Content_delivery_network) that can deliver anything found in [npm, the Node.js package manager](https://www.npmjs.com/get-npm).

# [Visual Studio](#tab/visual-studio/)

* In **Solution Explorer**, right-click the project, and select **Add** > **Client-Side Library**.

* In the **Add Client-Side Library** dialog, for **Provider** select **unpkg**. 

* For **Library**, enter _@aspnet/signalr@1_, and select the latest version that isn't preview.

  ![Add Client-Side Library dialog - select library](signalr/_static/libman1.png)

* Select **Choose specific files**, expand the *dist/browser* folder, and select *signalr.js* and *signalr.min.js*.

* Set **Target Location** to *wwwroot/lib/signalr/*, and select **Install**.

  ![Add Client-Side Library dialog - select files and destination](signalr/_static/libman2.png)

  [LibMan](xref:client-side/libman/index) creates a *wwwroot/lib/signalr* folder and copies the selected files to it.

# [Visual Studio Code](#tab/visual-studio-code/)

* In the **Integrated Terminal**, run the following command to install LibMan.

  ```console
  dotnet tool install -g Microsoft.Web.LibraryManager.Cli
  ```

* Navigate to the project folder (the one that contains the *SignalRChat.csproj* file).

* Run the following command to get the SignalR client library by using LibMan. You might have to wait a few seconds before seeing output.

  ```console
  libman install @aspnet/signalr -p unpkg -d wwwroot\lib\signalr --files dist/browser/signalr.js --files dist/browser/signalr.min.js
  ```

  The parameters specify the following options:
  * Use the unpkg provider.
  * Copy files to the *wwwroot/lib/signalr* destination.
  * Copy only the specified files.

  The output looks like the following example:

  ```console
  wwwroot/lib/signalr/dist/browser/signalr.js written to disk
  wwwroot/lib/signalr/dist/browser/signalr.min.js written to disk
  Installed library "@aspnet/signalr@1.0.3" to "wwwroot\lib\signalr"
  ```

# [Visual Studio for Mac](#tab/visual-studio-mac)

* In the **Terminal**, run the following command to install LibMan.

  ```console
  dotnet tool install -g Microsoft.Web.LibraryManager.Cli
  ```

* Navigate to the project folder (the one that contains the *SignalRChat.csproj* file).

* Run the following command to get the SignalR client library by using LibMan.

  ```console
  libman install @aspnet/signalr -p unpkg -d wwwroot\lib\signalr --files dist/browser/signalr.js --files dist/browser/signalr.min.js
  ```

  The parameters specify the following options:
  * Use the unpkg provider.
  * Copy files to the *wwwroot/lib/signalr* destination.
  * Copy only the specified files.

  The output looks like the following example:

  ```console
  wwwroot/lib/signalr/dist/browser/signalr.js written to disk
  wwwroot/lib/signalr/dist/browser/signalr.min.js written to disk
  Installed library "@aspnet/signalr@1.0.3" to "wwwroot\lib\signalr"
  ```

---

## Create the SignalR hub

A [hub](xref:signalr/hubs) is a class that serves as a high-level pipeline that handles client-server communication.

* In the SignalRChat project folder, create a *Hubs* folder.

* In the *Hubs* folder, create a *ChatHub.cs* file with the following code:

  [!code-csharp[Startup](signalr/sample/Hubs/ChatHub.cs)]

  The `ChatHub` class inherits from the SignalR [Hub](/dotnet/api/microsoft.aspnetcore.signalr.hub) class. The `Hub` class manages connections, groups, and messaging.

  The `SendMessage` method can be called by any connected client. It sends the received message to all clients. SignalR code is asynchronous to provide maximum scalability.

## Configure the project to use SignalR

The SignalR server must be configured to pass SignalR requests to SignalR.

* Add the following highlighted code to the *Startup.cs* file.

  [!code-csharp[Startup](signalr/sample/Startup.cs?highlight=7,33,52-55)]

  These changes add SignalR to the [dependency injection](xref:fundamentals/dependency-injection) system and the [middleware](xref:fundamentals/middleware/index) pipeline.

## Create the SignalR client code

* Replace the content in *Pages\Index.cshtml* with the following code:

  [!code-cshtml[Index](signalr/sample/Pages/Index.cshtml)]

  The preceding code:

  * Creates text boxes for name and message text, and a submit button.
  * Creates a list with `id="messagesList"` for displaying messages that are received from the SignalR hub.
  * Includes script references to SignalR and the *chat.js* application code that you create in the next step.

* In the *wwwroot/js* folder, create a *chat.js* file with the following code:

  [!code-javascript[Index](signalr/sample/wwwroot/js/chat.js)]

  The preceding code:

  * Creates and starts a connection.
  * Adds to the submit button a handler that sends messages to the hub.
  * Adds to the connection object a handler that receives messages from the hub and adds them to the list.

## Run the app

# [Visual Studio](#tab/visual-studio)

* Press **CTRL+F5** to run the app without debugging.

# [Visual Studio Code](#tab/visual-studio-code)

* Press **CTRL+F5** to run the app without debugging.

# [Visual Studio for Mac](#tab/visual-studio-mac)

* From the menu, select **Run > Start Without Debugging**.

---

* Copy the URL from the address bar, open another browser instance or tab, and paste the URL in the address bar.

* Choose either browser, enter a name and message, and select the **Send** button.

  The name and message are displayed on both pages instantly.

  ![SignalR sample app](signalr/_static/signalr-get-started-finished.png)

> [!TIP]
> If the app doesn't work, open your browser developer tools (F12) and go to the console. You might see errors related to your HTML and JavaScript code. For example, suppose you put *signalr.js* in a different folder than directed. In that case the reference to that file won't work and you'll see a 404 error in the console.
> ![signalr.js not found error](signalr/_static/f12-console.png)

## Next steps

If you want clients to connect to a SignalR app from different domains, you have to enable Cross-Origin Resource Sharing (CORS). For more information, see [Cross-origin resource sharing](xref:signalr/security?view=aspnetcore-2.1#cross-origin-resource-sharing).

To learn more about SignalR, hubs, and JavaScript clients, see these resources:

* [Introduction to SignalR for ASP.NET Core](xref:signalr/introduction)
* [Use hubs in SignalR for ASP.NET Core](xref:signalr/hubs)
* [ASP.NET Core SignalR JavaScript client](xref:signalr/javascript-client)
