# SpringBoot dummy application

This is an example of the basic SpringBoot application that uses Thymeleaf to render the html page that has environment name that comes from properties file.

## Building the app
```aidl
docker build -t argo-deployment-demo-app .
```


## Running the app locally:

```aidl
docker run -p 8080:8080 argo-deployment-demo-app
```

As a result of the command above you should see the output below using `http://localhost:8080` in your browser:

![Default output](https://github.com/kolyaiks/argo-deployment-demo-app/blob/main/argo-deployment-demo-app.drawio.png)
