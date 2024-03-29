# Nginx Keycloak

[![Linux build of nginx-keycloak](https://github.com/flavienbwk/nginx-keycloak/actions/workflows/linux-build.yml/badge.svg)](https://github.com/flavienbwk/nginx-keycloak/actions/workflows/linux-build.yml)

Setting NGINX as a reverse proxy with Keycloak SSO in front of your web applications.

## Getting started

This setup build a cluster of Docker components that demonstrate how to support SSO for a set of applications (app_1, app_2 and app_3) using a reverse proxy and Keycloak. 
These applications are: 
1. app_1: simple NGINX http proxy. 
2. app_2: simple Apache proxy with a page that retrieve and render user information from http headers based on Keycloak token. 
3. app_3: A web application that is developed using Vue framework and interacts with two microservices (user roles management). 

Note that the app_3 should be accessible to demonstrate the functionalities.  

### Configuring Keycloak

1. Set-up `.env` and edit variable values

    ```bash
    cp .env.example .env
    ```

2. Start Keycloak

    ```bash
    docker-compose up -d keycloak
    ```

3. Go to `http://localhost:3333` and login with your credentials

4. In the [master realm](http://localhost:3333/auth/admin/master/console/#/realms/master), we are going to create a client

    1. In sidebar, click ["Clients"](http://localhost:3333/auth/admin/master/console/#/realms/master/clients) and click on the "Create" button. Let's call it `NginxApps`.
    2. In `NginxApps` client parameters :
       1. Add a "Valid Redirect URI" to your app : `http://localhost:3002/*` (don't forget clicking "+" button to add the URL, then "Save" button)
       2. Set the "Access type" to `confidential`
    3. In the "Credentials" tab, retrieve the "Secret" and **set `KEYCLOAK_SECRET` in your `.env`** file

5. Go to ["Users"](http://localhost:3333/auth/admin/master/console/#/realms/master/users) in the sidebar and create one. Edit its password in the "Credentials" tab.

### Simple user authentication

With this method, being a registered user is sufficient to access your apps.

If you choose this method, you're already set. Just run :

```bash
docker-compose up -d nginx app_1 
```

You can now visit `http://localhost:3002` to validate the configuration.

### Add user authentication for a web frontend application 
This section describes how to add user authentication function for the app_3 to support SSO for two microservices that implement the user roles management. 
The app_3 source code should be accessible for the [docker-compose](./docker-compose.yml). The related mount path should be configured in the app_3 section.  

In our [docker-compose](./docker-compose.yml) configuration, edit the NGINX configuration mount point to be `./nginx-app-3.conf.template` instead of `./nginx.conf.template`.

```bash
docker-compose up -d nginx app_1 app_3
```

### Role-based / per-app user authentication

Let's say you want only specific users to be able to access specific apps. We have to create a role for that.

1. In sidebar, click "Clients"
2. Select the `NginxApps` client and go to the "Roles" tab
3. Top right, click the "Add Role" button and create one with name `NginxApps-App1`

    :information_source: 1 role = 1 app

Now we want to attribute this role to our user.

1. In sidebar, click "Users"
2. Click "Edit" on the user you want to add the role to
3. Go to the "Role Mappings" tab
4. Select the "Client Roles" `NginxApps` and assign the `NginxApps-App1` role by selecting it and clicking "Add selected"

In our [docker-compose](./docker-compose.yml) configuration, edit the NGINX configuration mount point to be `./nginx-roles.conf.template` instead of `./nginx.conf.template`.

:information_source: If you want to name your role differently, you can edit the expected name in `./nginx-roles.conf.template` in the `contains(client_roles, "NginxApps-App1")` line.

Start NGINX and the app :

```bash
    docker-compose up -d nginx app_1
```

You can now visit `http://localhost:3002` to validate the configuration.

### Present user information from the identity token 

Let's display few user information retrieved from Keycloak identiy token. The NGINX will be configured to do so and forwoard this information as HTTP headers to the application App_2 runs behind an apache server (port=4090). 

1. The app_2 is added to docker-compose.yml file as new image, where the Apache2 server image is used and three files are mounted: aap.conf, index.html and user.shtml
2. In the nginx-roles.con.template file, three attributes are abstucted fromt the ID token: user ID, user name and user email.  

In LUA block: 

access_by_lua {
    .....
        ngx.req.set_header("X-USER", res.id_token.sub)
        ngx.req.set_header("X-NAME", res.id_token.name)
        ngx.req.set_header("X-EMAIL", res.id_token.email)  
}

3. Start both nginx proxy and app_2

```bash
docker-compose up -d nginx app_2
```
    :information_source: 1 role = 1 app

Now you can visit `http://localhost:4090/user.shtml` to validate the configuration and to see the user information.

## Credits

- [Configure NGINX and Keycloak to enable SSO for proxied applications](https://kevalnagda.github.io/configure-nginx-and-keycloak-to-enable-sso-for-proxied-applications)