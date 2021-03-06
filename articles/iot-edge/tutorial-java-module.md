---
# Mandatory fields. See more on aka.ms/skyeye/meta.
title: Tutorial create custom Java module - Azure IoT Edge | Microsoft Docs 
description: This tutorial shows you how to create an IoT Edge module with Java code and deploy it to an edge device.
services: iot-edge
author: kgremban
manager: philmea

ms.author: kgremban
ms.date: 11/25/2018
ms.topic: tutorial
ms.service: iot-edge
ms.custom: "mvc, seodec18"
---

# Tutorial: Develop a Java IoT Edge module and deploy to your simulated device

You can use Azure IoT Edge modules to deploy code that implements your business logic directly to your IoT Edge devices. This tutorial walks you through creating and deploying an IoT Edge module that filters sensor data. You'll use the simulated IoT Edge device that you created in the Deploy Azure IoT Edge on a simulated device in [Windows](quickstart.md) or [Linux](quickstart-linux.md) quickstarts. In this tutorial, you learn how to:    

> [!div class="checklist"]
> * Use Visual Studio Code to create an IoT Edge Java module based on the Azure IoT Edge maven template package and Azure IoT Java device SDK.
> * Use Visual Studio Code and Docker to create a Docker image and publish it to your registry.
> * Deploy the module to your IoT Edge device.
> * View generated data.


The IoT Edge module that you create in this tutorial filters the temperature data that's generated by your device. It only sends messages upstream if the temperature is above a specified threshold. This type of analysis at the edge is useful for reducing the amount of data that's communicated to and stored in the cloud. 

[!INCLUDE [quickstarts-free-trial-note](../../includes/quickstarts-free-trial-note.md)]


## Prerequisites

An Azure IoT Edge device:

* You can use your development machine or a virtual machine as an Edge device by following the steps in the quickstart for [Linux](quickstart-linux.md).
* Java modules for IoT Edge don't support Windows devices.

Cloud resources:

* A free or standard-tier [IoT Hub](../iot-hub/iot-hub-create-through-portal.md) in Azure. 

Development resources:

* [Visual Studio Code](https://code.visualstudio.com/). 
* [Java Extension Pack](https://marketplace.visualstudio.com/items?itemName=vscjava.vscode-java-pack) for Visual Studio Code.
* [Azure IoT Edge extension](https://marketplace.visualstudio.com/items?itemName=vsciot-vscode.azure-iot-edge) for Visual Studio Code. 
* [Java SE Development Kit 10](https://aka.ms/azure-jdks), and [set the `JAVA_HOME` environment variable](https://docs.oracle.com/cd/E19182-01/820-7851/inst_cli_jdk_javahome_t/) to point to your JDK installation.
* [Maven](https://maven.apache.org/)
* [Docker CE](https://docs.docker.com/install/)
   * If you're developing on a Windows device, make sure Docker is [configured to use Linux containers](https://docs.docker.com/docker-for-windows/#switch-between-windows-and-linux-containers). 


## Create a container registry

In this tutorial, you use the Azure IoT Edge extension for Visual Studio Code to build a module and create a **container image** from the files. Then you push this image to a **registry** that stores and manages your images. Finally, you deploy your image from your registry to run on your IoT Edge device.  

You can use any Docker-compatible registry to hold your container images. Two popular Docker registry services are [Azure Container Registry](https://docs.microsoft.com/azure/container-registry/) and [Docker Hub](https://docs.docker.com/docker-hub/repos/#viewing-repository-tags). This tutorial uses Azure Container Registry. 

If you don't already have a container registry, follow these steps to create a new one in Azure:

1. In the [Azure portal](https://portal.azure.com), select **Create a resource** > **Containers** > **Container Registry**.

2. Provide the following values to create your container registry:

   | Field | Value | 
   | ----- | ----- |
   | Registry name | Provide a unique name. |
   | Subscription | Select a subscription from the drop-down list. |
   | Resource group | We recommend that you use the same resource group for all of the test resources that you create during the IoT Edge quickstarts and tutorials. For example, **IoTEdgeResources**. |
   | Location | Choose a location close to you. |
   | Admin user | Set to **Enable**. |
   | SKU | Select **Basic**. | 

5. Select **Create**.

6. After your container registry is created, browse to it, and then select **Access keys**. 

7. Copy the values for **Login server**, **Username**, and **Password**. You use these values later in the tutorial to provide access to the container registry. 

## Create an IoT Edge module project
The following steps create an IoT Edge module project that's based on the Azure IoT Edge maven template package and Azure IoT Java device SDK by using Visual Studio Code and the Azure IoT Edge extension.

### Create a new solution

Create a Java solution template that you can customize with your own code. 

1. In Visual Studio Code, select **View** > **Command Palette** to open the VS Code command palette. 

2. In the command palette, enter and run the command **Azure IoT Edge: New IoT Edge solution**. Follow the prompts in the command palette to create your solution.

   | Field | Value |
   | ----- | ----- |
   | Select folder | Choose the location on your development machine for VS Code to create the solution files. |
   | Provide a solution name | Enter a descriptive name for your solution or accept the default **EdgeSolution**. |
   | Select module template | Choose **Java Module**. |
   | Provide value for groupId | Enter a group ID value or accept the default **com.edgemodule**. |
   | Provide a module name | Name your module **JavaModule**. |
   | Provide Docker image repository for the module | An image repository includes the name of your container registry and the name of your container image. Your container image is prepopulated from the last step. Replace **localhost:5000** with the login server value from your Azure container registry. You can retrieve the login server from the Overview page of your container registry in the Azure portal. The final string looks like \<registry name\>.azurecr.io/javamodule. |
 
   ![Provide Docker image repository](./media/tutorial-java-module/repository.png)
   
If it's the first time to create Java module, it might take several minutes to download maven packages. Then the VS Code window loads your IoT Edge solution workspace. The solution workspace contains five top-level components. The **modules** folder contains the Java code for your module as well as Dockerfiles for building your module as a container image. The **\.env** file stores your container registry credentials. The **deployment.template.json** file contains the information that the IoT Edge runtime uses to deploy modules on a device. And **deployment.debug.template.json** file containers the debug version of modules. You won't edit the **\.vscode** folder or **\.gitignore** file in this tutorial. 

If you didn't specify a container registry when creating your solution, but accepted the default localhost:5000 value, you won't have a \.env file. 

### Add your registry credentials

The environment file stores the credentials for your container registry and shares them with the IoT Edge runtime. The runtime needs these credentials to pull your private images onto the IoT Edge device. 

1. In the VS Code explorer, open the .env file. 
2. Update the fields with the **username** and **password** values that you copied from your Azure container registry. 
3. Save this file. 

### Update the module with custom code

1. In the VS Code explorer, open **modules** > **JavaModule** > **src** > **main** > **java** > **com** > **edgemodule** > **App.java**.

5. Add the following code at the top of the file to import new referenced classes.

    ```java
    import java.io.StringReader;
    import java.util.concurrent.atomic.AtomicLong;
    import java.util.HashMap;
    import java.util.Map;
    
    import javax.json.Json;
    import javax.json.JsonObject;
    import javax.json.JsonReader;
    
    import com.microsoft.azure.sdk.iot.device.DeviceTwin.Pair;
    import com.microsoft.azure.sdk.iot.device.DeviceTwin.Property;
    import com.microsoft.azure.sdk.iot.device.DeviceTwin.TwinPropertyCallBack;
    ```

5. Add the following definition into class **App**. This variable sets the value that the measured temperature must exceed for the data to be sent to the IoT hub. 

    ```java
    private static final String TEMP_THRESHOLD = "TemperatureThreshold";
    private static AtomicLong tempThreshold = new AtomicLong(25);
    ```

7. Replace the execute method of **MessageCallbackMqtt** with the following code. This method is called whenever the module receives an MQTT message from the IoT Edge hub. It filters out messages that report temperatures below the temperature threshold set via the module twin.

    ```java
        private int counter = 0;
       @Override
        public IotHubMessageResult execute(Message msg, Object context) {
            this.counter += 1;
 
            String msgString = new String(msg.getBytes(), Message.DEFAULT_IOTHUB_MESSAGE_CHARSET);
            System.out.println(
                   String.format("Received message %d: %s",
                            this.counter, msgString));
            if (context instanceof ModuleClient) {
                try (JsonReader jsonReader = Json.createReader(new StringReader(msgString))) {
                    final JsonObject msgObject = jsonReader.readObject();
                    double temperature = msgObject.getJsonObject("machine").getJsonNumber("temperature").doubleValue();
                    long threshold = App.tempThreshold.get();
                    if (temperature >= threshold) {
                        ModuleClient client = (ModuleClient) context;
                        System.out.println(
                            String.format("Temperature above threshold %d. Sending message: %s",
                            threshold, msgString));
                        client.sendEventAsync(msg, eventCallback, msg, App.OUTPUT_NAME);
                    }
                } catch (Exception e) {
                    e.printStackTrace();
                }
            }
            return IotHubMessageResult.COMPLETE;
        }
    ```

8. Add the following two static inner classes into class **App**. These classes receive updates on the desired properties from the module twin, and updates the **tempThreshold** variable to match. All modules have their own module twin, which lets you configure the code that's running inside a module directly from the cloud.

    ```java
    protected static class DeviceTwinStatusCallBack implements IotHubEventCallback {
        @Override
        public void execute(IotHubStatusCode status, Object context) {
            System.out.println("IoT Hub responded to device twin operation with status " + status.name());
        }
    }
 
    protected static class OnProperty implements TwinPropertyCallBack {
        @Override
        public void TwinPropertyCallBack(Property property, Object context) {
            if (!property.getIsReported()) {
                if (property.getKey().equals(App.TEMP_THRESHOLD)) {
                    try {
                        long threshold = Math.round((double) property.getValue());
                        App.tempThreshold.set(threshold);
                    } catch (Exception e) {
                        System.out.println("Faile to set TemperatureThread with exception");
                        e.printStackTrace();
                    }
                }
            }
        }
    }
    ```

9. Add the following lines in to **main** method after **client.open()** to subscribe the module twin updates.

    ```java
    client.startTwin(new DeviceTwinStatusCallBack(), null, new OnProperty(), null);
    Map<Property, Pair<TwinPropertyCallBack, Object>> onDesiredPropertyChange = new HashMap<Property, Pair<TwinPropertyCallBack, Object>>() {
        {
            put(new Property(App.TEMP_THRESHOLD, null), new Pair<TwinPropertyCallBack, Object>(new OnProperty(), null));
        }
    };
    client.subscribeToTwinDesiredProperties(onDesiredPropertyChange);
    client.getTwin();
    ```

11. Save this file.

12. In the VS Code explorer, open the deployment.template.json file in your IoT Edge solution workspace. This file tells the **$edgeAgent** to deploy two modules: **tempSensor** and **JavaModule**. The default platform of your IoT Edge is set to **amd64** in your VS Code status bar, which means your **JavaModule** is set to Linux amd64 version of the image. Change the default platform in status bar from **amd64** to **arm32v7** or **windows-amd64** if that is your IoT Edge device's architecture. 

   To learn more about deployment manifests, see [Understand how IoT Edge modules can be used, configured, and reused](module-composition.md).

   In the deployment.template.json file, the **registryCredentials** section stores your Docker registry credentials. The actual username and password pairs are stored in the .env file, which is ignored by git.  

13. Add the **JavaModule** module twin to the deployment manifest. Insert the following JSON content at the bottom of the **moduleContent** section, after the **$edgeHub** module twin: 

   ```json
       "JavaModule": {
           "properties.desired":{
               "TemperatureThreshold":25
           }
       }
   ```

   ![Add module twin to deployment template](./media/tutorial-java-module/module-twin.png)

14. Save this file.

## Build your IoT Edge solution

In the previous section, you created an IoT Edge solution and added code to the **JavaModule** to filter out messages where the reported machine temperature is below the acceptable threshold. Now you need to build the solution as a container image and push it to your container registry. 

1. Sign in to Docker by entering the following command in the Visual Studio Code integrated terminal. Then you can push your module image to your Azure container registry.
     
   ```csh/sh
   docker login -u <ACR username> -p <ACR password> <ACR login server>
   ```
   Use the username, password, and login server that you copied from your Azure container registry in the first section. You can also retrieve these values from the **Access keys** section of your registry in the Azure portal.

2. In the VS Code explorer, right-click the deployment.template.json file and select **Build and Push IoT Edge solution**. 

When you tell Visual Studio Code to build your solution, it first takes the information in the deployment template and generates a deployment.json file in a new folder named **config**. Then, it runs two commands in the integrated terminal: `docker build` and `docker push`. These two commands build your code, containerize the Java app, and then push the code to the container registry that you specified when you initialized the solution. 

You can see the full container image address with tag in the VS Code integrated terminal. The image address is built from information that's in the module.json file with the format \<repository\>:\<version\>-\<platform\>. For this tutorial, it should look like registryname.azurecr.io/javamodule:0.0.1-amd64.

## Deploy and run the solution

In the quickstart article that you used to set up your IoT Edge device, you deployed a module by using the Azure portal. You can also deploy modules using the Azure IoT Hub Toolkit extension (formerly Azure IoT Toolkit extension) for Visual Studio Code. You already have a deployment manifest prepared for your scenario, the **deployment.json** file. All you need to do now is select a device to receive the deployment.

1. In the VS Code command palette, run the command **Azure: Sign in** and follow the instructions to sign in your Azure account. If you're already signed in, you can skip this step.

2. In the VS Code command palette, run **Azure IoT Hub: Select IoT Hub**. 

3. Choose the subscription and IoT hub that contain the IoT Edge device that you want to configure. 

4. In the VS Code explorer, expand the **Azure IoT Hub Devices** section. 

5. Right-click the name of your IoT Edge device, then select **Create Deployment for Single Device**. 

   ![Create deployment for single device](./media/tutorial-java-module/create-deployment.png)

6. Select the **deployment.json** file in the **config** folder and then click **Select Edge Deployment Manifest**. Do not use the deployment.template.json file. 

7. Click the refresh button. You should see the new **JavaModule** running along with the **TempSensor** module and the **$edgeAgent** and **$edgeHub**.  

## View generated data

Once you apply the deployment manifest to your IoT Edge device, the IoT Edge runtime on the device collects the new deployment information and starts executing on it. Any modules running on the device that aren't included in the deployment manifest are stopped. Any modules missing from the device are started. 

You can view the status of your IoT Edge device using the **Azure IoT Hub Devices** section of the Visual Studio Code explorer. Expand the details of your device to see a list of deployed and running modules. 

On the IoT Edge device itself you can see the status of your deployment modules using the command `iotedge list`. You should see four modules: the two IoT Edge runtime modules, tempSensor, and the custom module that you created in this tutorial. It may take a few minutes for all the modules to start, so rerun the command if you don't see them all initially. 

To view the messages being generated by any module, use the command `iotedge logs <module name>`. 

You can view the messages as they arrive at your IoT hub using Visual Studio Code. 

1. To monitor data that arrives at the IoT hub, select the ellipsis (**...**), and then select **Start Monitoring D2C Messages**.
2. To monitor the D2C message for a specific device, right-click the device in the list, and select **Start Monitoring D2C Messages**.
3. To stop monitoring data, run the command **Azure IoT Hub: Stop monitoring D2C message** in the command palette. 
4. To view or update the module twin, right-click the module in the list, and select **Edit module twin**. To update the module twin, save the twin JSON file, right-click the editor area, and select **Update Module Twin**.
5. To view Docker logs, install [Docker](https://marketplace.visualstudio.com/items?itemName=PeterJausovec.vscode-docker) for VS Code. You can find your running modules locally in Docker explorer. In the context menu, select **Show Logs** to view in the integrated terminal.
 
## Clean up resources 

If you plan to continue to the next recommended article, you can keep the resources and configurations that you created and reuse them. You can also keep using the same IoT Edge device as a test device. 

Otherwise, you can delete the local configurations and the Azure resources that you created in this article to avoid charges. 

[!INCLUDE [iot-edge-clean-up-cloud-resources](../../includes/iot-edge-clean-up-cloud-resources.md)]

[!INCLUDE [iot-edge-clean-up-local-resources](../../includes/iot-edge-clean-up-local-resources.md)]


## Next steps

In this tutorial, you created an IoT Edge module with code to filter raw data that's generated by your IoT Edge device. You can continue on to the next tutorials to learn other ways that Azure IoT Edge can help you turn data into business insights at the edge.

> [!div class="nextstepaction"]
> [Store data at the edge with SQL Server databases](tutorial-store-data-sql-server.md)

