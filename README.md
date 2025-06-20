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

![Default output](https://github.com/kolyaiks/argo-deployment-demo-app/blob/main/argo-deployment-demo-app.drawio.png)


