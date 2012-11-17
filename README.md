Android Drupal SDK
================================

Android Drupal SDK is an easy to use Android client for REST servers based on Drupal and Services 3.

It is built at [Demotix.com](http://www.demotix.com/ "Demotix.com") by Alexandru Badiu and used in the [Demotix Android app](https://play.google.com/store/apps/details?id=com.demotix).

While being used in an application it is work in progress and implements currently only a small number of the standard functions offered by the Services module.

It uses [Android Asynchronous Http Client](http://loopj.com/android-async-http/).

Features
========

* Easy to use
* Supports basic authentication
* Persistent cookies
* Tiny overhead
* HTTP requests happen outside the UI thread
* Smart retries, gzip, threadpool

General implementation
======================

There is a base class, ServicesClient, which takes care of storing the session information and make the POST, GET and DELETE calls.

Each service group (System services, User services) is implemented in a separate class which uses the base class to make the relevant calls.

Usage
=====

Create a REST client that connects to example.com/api/mobile:

    ServicesClient client;
    client = new ServicesClient("http://www.example.com", "api/mobile");

Set basic auth credentials:
    
    client.setBasicAuth("username","password/token");

Add a persistent cookie store to save the session and reuse it between application runs:
    
    cookieStore = new PersistentCookieStore(this);
    client.setCookieStore(cookieStore);

Create system and user services:

    UserServices us;
    SystemServices ss;

    us = new UserServices(client);
    ss = new SystemServices(client);

Call system.connect and check if we are logged in:
    
    JsonHttpResponseHandler connectHandler = new JsonHttpResponseHandler() {
        @Override
        public void onSuccess(JSONObject response) {
            boolean loggedin = false;
            try {
                JSONObject user = response.getJSONObject("user");
                int uid = user.getInt("uid");
                if (uid > 0) {
                    loggedin = true;
                }
                else {
                    loggedin = false;
                }
            } catch (JSONException e) {
                loggedin = false;
            }

            if (loggedin) {
                // User is already logged in, do something 
            }
            else {
                // User is not logged in, display login activity or automatically login
            }
        }

        @Override
        public void onFailure(Throwable e, JSONObject response) {
            // System.Connect call failed
        }

        @Override
        public void onFinish() {
        }
    };

    ss.Connect(connectHandler);

Call user.login:

    JsonHttpResponseHandler loginHandler = new JsonHttpResponseHandler() {
        @Override
        public void onSuccess(JSONObject response) {
            boolean error = false;
            try {
                JSONObject user = response.getJSONObject("user");
          
            } catch (JSONException e) {
                error = true;
            }

            if (error) {
                // A JSON error occured
            }
            else {
                // User has logged in
            }
        }

        @Override
        public void onFailure(Throwable e, JSONObject response) {
            // Username or password were incorrect
        }

        @Override
        public void onFinish() {
            activity.hideProgressDialog();
        }
    };

    activity.showProgressDialog("Logging you in");
    us.Login("username", "password");

Implementing new services
=========================

New services should live in a class named after their group (System, Node, User) and Services (e.g. NodeServices). 

It should have a ServicesClient property and it's constructor should set that property. 

Parameters should be packaged in a JSONObject.

Calls to services should be made using the provided methods in ServicesClient:

* get(String method, JSONObject params, AsyncHttpResponseHandler responseHandler)
* post(String method, JSONObject params, AsyncHttpResponseHandler responseHandler)
* delete(String method, JSONObject params, AsyncHttpResponseHandler responseHandler)


Uploading large files
=====================

Android Asynchronous Http Client supports file uploading but it reads the whole contents of the file in memory 
which will not work for large files, causing your application to crash with an out of memory exception.

The solution is to go a bit lower level and use the underlying HttpClient to do the upload attaching the files as 
InputStreams which will perform a streaming upload.

    HttpParams httpParams = new BasicHttpParams();
    HttpConnectionParams.setConnectionTimeout(httpParams, 900 * 1000);
    HttpConnectionParams.setSoTimeout(httpParams, 900 * 1000);

    HttpClient httpclient = ServicesClient.client.getHttpClient();
    HttpContext httpContext = ServicesClient.client.getHttpContext();

    HttpPost httppost = new HttpPost("http://www.example.com/api/mobile/service/upload");
    httppost.setParams(httpParams);

    MultipartEntity entity = new MultipartEntity();
    // Add parameters to the post body
    entity.addPart("param1", new StringBody(param1.toString(),"application/json", Charset.forName("UTF-8")));
    entity.addPart("param2", new StringBody(param2.toString(),"application/json", Charset.forName("UTF-8")));

    InputStream istream1 = new FileInputStream("file1.jpg");
    entity.addPart("file1", new InputStreamBody(istream1, "file1"));
    InputStream istream2 = new FileInputStream("file2.jpg");
    entity.addPart("file2", new InputStreamBody(istream2, "file2"));

    httppost.setEntity(entity);
    HttpResponse response = httpclient.execute(httppost, httpContext);


Progress callback for upload
============================

TBD
