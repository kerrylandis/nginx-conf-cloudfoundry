# nginx proxy config for Cloud Foundry on IBM Cloud
The instructions below help you setup an nginx proxy to a Cloud Foundry running a Node.js app. This repo makes use of the `nginx.conf` file to utilize the ngninx buildpack in your app.

The configuration is a customized example pulled from the main [Cloud Foundry nginx buildpack](https://github.com/cloudfoundry/nginx-buildpack). Official buildpack documentation can be found at [here](https://docs.cloudfoundry.org/buildpacks/nginx/index.html).

### Setting up Cloud Foundry

1. Create the [Cloud Foundy Instance](https://cloud.ibm.com/cloudfoundry/overview) on IBM Cloud by provisioning the service.

   - Choose a region
   - For the runtime, choose SDK for Node.js (doesn’t really matter which is chosen, because we change the buildpack in commandline with cf push <appname> -b <buildpack>)
   - App name: lowercase name, no spaces.  This will be what is used in cf push command. i.e. `test-proxy`
   - Host name: name for your URL in front of the .mybluemix.net domain.  Make it similar to your app name. i.e. `test-proxy.mybluemix.net`
   - Choose an organization
   - Choose a space
   - Tags can be ignored

2. Create nginx app

   Create a new GitHub repository to host your nginx configuration.  After you create it, download this boilerplate nginx app and place it in the repo.

3. IBM Cloud login and target

   Make sure you have the latest IBM cloud command-line tool. If not, [download it](https://cloud.ibm.com/docs/cli?topic=cloud-cli-getting-started).
   
   Using command-line, login to IBM cloud
   ```
   ibmcloud login –sso
   # follow the steps shown in console and choose your environment when asked
   ```
   
   Next, we need to target our organization and space in cloud foundry:
   ```
   ibmcloud cf –target
   # choose Organization
   # choose Space
   ```

4. Pushing the app

   In command-line, ‘cd’ to your NGINX configuration folder that was created in Step 2.
   
   Inside our directory, we can push the app to CF:
   ```
   ibmcloud cf push <appname> -b nginx_buildpack
   # Use App Name from Step 1
   ```
   
   If there were no error messages, your nginx server can be found at the host/domain setup in Step 1.  This forces CF to use nginx_buildpack which is the nginx environment, instead of the default Node.js that we selected in Step 1.
   
   For an example, our simulated test-proxy would be at: https://test-proxy.mybluemix.net/

5. Internal routing

   Next, we need to create and map an internal route for our nginx to communicate to Node.js through the internal network, so it doesn’t have to go out into the internet.
   
   Run this command:
   ```
   ibmcloud cf map-route <nodejsAppName> apps.internal --hostname <nodejsAppName>
   # nodeJSAppName = the name of your CF instance with your Node.js web app
   ```

6. Mapping the route between apps

   Next, we tell IBM Cloud to make a network path between the two apps:

   ```
   ibmcloud cf add-network-policy <proxyName> --destination-app <nodejsAppName> --port 8080 --protocol tcp
   ```
   
   This will allow us to use Container-to-Container networking since we are running different containers on Cloud Foundry for nginx and Node.js. 

7. Setup nginx.conf

   We need to setup nginx to utilize the “<nodejsAppName>.app.internal” in our configuration for nginx. By default, Cloud Foundry sets the $PORT environment variable to 8080.  Our NodeJS application is setup to use “http”, with no SSL, at port 8080, when the ‘env.process.PORT’ variable exists.  In nginx.conf, we need to configure the Node.js host used in proxy_pass to use, for example: test.apps.internal:8080.  This will route network communication internally and provide a speed up for communication between proxy and app.

8. Environment variables

   Go to your [Resource list](https://cloud.ibm.com/resources) in Cloud Foundry and select you Cloud Foundry Proxy Application. Find the “Runtime” link on the left, and then go into Environment Variables tab.
 
   Scroll down to the User defined section:

   ```
   #CF NGINX App Environment Variables
 
   COSURL: <cloudObjectStorageContainer> #i.e. mycontainer.s3.us.cloud-object-storage.appdomain.cloud
   APPURL: <appURL> #i.e. test.apps.internal:8080, APPURL is the internal route setup for CF Node.js app from Step 5
   ```
   
   Additional environment variables can be added as needed for your application as long as they are called in the nginx.conf file.

9. Setup a custom domain

   Inside the Cloud Foundry application dashboard, we can find the “Routes” button on the top right. You want to click “Edit Routes”, so we can add the new URL path.
   
   Click “Add Route”, to create a new route, and select one of the free domains provided by IBM Cloud, i.e. mybluemix.net.
   
   This will be the route we use to view the website through a browser.  Behind the scenes it will forward to our internal route mapped to the CF NodeJS App.
