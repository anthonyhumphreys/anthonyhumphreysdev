---
title: Deploying a Gatsby Site to Google Cloud Run
category: "100 Days of Code"
cover: cloud.jpg
author: Anthony Humphreys
---

# Building the Gatsby site

You don't need to do anything _particularly_ special to build a Gatsby site for deployment to Cloud Run, but there are some steps between building the project and seeing it live.

For this tutorial (and for my blogs) I'll use [gatsby-starter-hero-blog](https://github.com/greglobinski/gatsby-starter-hero-blog) starter.

Getting up and running is simple (make sure you have the [gatsby cli](https://www.gatsbyjs.org/docs/gatsby-cli/) installed correctly first)

`gatsby new anthonyhumphreysdev https://github.com/greglobinski/gatsby-starter-blog`,

then you can run your site locally with

`gatsby develop`.

After customising the template to your liking (have a poke around, check out the gatsby & starter docs for more guidance!), setting up some content and an initial post, it's time to deploy a test build.

I decided to use [GitHub Actions](https://github.com/features/actions) and [Cloud Run](https://cloud.google.com/run) to do this. GitHub actions is the new kid on the block for CI/CD, but it's such a nice experience, especially being so closely coupled to your actual source repo. I use Cloud Run for Lexio and love its ease of use and general developer experience.

You'll need to set up some environment variables for the gatsby starter and for the GitHub action workflow. These should be clear from the starter's docs, and from the source below. You can [set secrets in the GitHub repo](https://help.github.com/en/actions/configuring-and-managing-workflows/creating-and-storing-encrypted-secrets)

You can checkout the full action YAML [here](https://github.com/anthonyhumphreys/anthonyhumphreysdev/blob/master/.github/workflows/main.yml):

I simply use the Node action to install dependencies and build the site.

```YAML
- name: Setup NodeJS
  uses: actions/setup-node@v1
  with:
    node-version: "10.x"
- name: Install dependencies
  run: |-
    yarn global add gatsby-cli
    yarn
- name: Gatsby Build
  run: yarn build
```

That's all there really is to it as far as building the site goes - no different to building on your own machine...but we still need a few bits and pieces yet.

# Cloud Run

Before continuing, you'll need to provision a new service in Cloud Run (assuming you have a Google Cloud account and Project set up!). Make a note of the Service Account Email Address, Project ID, Service Name, as you will need these later.

# Building and Deploying the Docker Image

I had a few issues with the Gatsby Docker image so rolled my own...probably should've stuck with it and resolved my issues, but it worked so that's just a `// TODO: Use gatsby image` instead!

## Dockerfile

```YAML
FROM nginx:latest

COPY public /usr/share/nginx/html
COPY nginxconf/nginx.conf /etc/nginx/nginx.conf

EXPOSE 8080
```

If you're not familiar with Docker - all that's happening here is I use the latest version of the [nginx image from dockerhub](https://hub.docker.com/_/nginx). I copy the files built in the previous step, which are in the `public` directory, to the `/usr/share/nginx/html` directory in the container, and then copy the `nginx.conf` file from the project to the container too. The last thing I do is `EXPOSE 8080` which opens up port 8080 for the container.

## Nginx Config

I won't go into Nginx as a reverse proxy, there are plenty of blog posts about that around already. You can however find the config I used below:

```conf
events {}
http {
    server {
        listen 8080;
        server_tokens off;
        location / {
            include /etc/nginx/mime.types;
            autoindex on;
            root   /usr/share/nginx/html;
            index  index.html index.htm;
            try_files $uri $uri/ /index.html;
        }
    }
}
```

Before I can push the image I need to setup GCloud in order to talk to Google's Cloud Registry:

```YAML
- name: Setup GCloud
  uses: GoogleCloudPlatform/github-actions/setup-gcloud@master
  with:
    version: "286.0.0"
    service_account_email: ${{ secrets.RUN_SA_EMAIL }}
    service_account_key: ${{ secrets.GCLOUD_AUTH }}
    project_id: ${{ secrets.RUN_PROJECT }}
```

Then I build the image

```YAML
- name: Build Docker Image
  run: docker build . -t "eu.gcr.io/$PROJECT_ID/$SERVICE_NAME:$GITHUB_SHA"
```

Now, I authenticate and publish the image

```YAML
- name: Authenticate for gcr
  run: gcloud auth print-access-token | docker login -u oauth2accesstoken --password-stdin https://eu.gcr.io/$PROJECT_ID
- name: Push Docker Image to gcr
  run: docker push eu.gcr.io/$PROJECT_ID/$SERVICE_NAME:$GITHUB_SHA
```

The final step is to deploy a new revision of the service to Cloud Run

```YAML
- name: Deploy
  run: |-
    gcloud run deploy $SERVICE_NAME \
      --quiet \
      --region $RUN_REGION \
      --image eu.gcr.io/$PROJECT_ID/$SERVICE_NAME:$GITHUB_SHA \
      --platform managed \
      --allow-unauthenticated
```

## Adding a post

Simply add a new post under `content/posts` following the naming convention, commit your changes and push - when your changes hit the master branch, the Action will run and update your site. Magic, right?

![Magic GIF](https://media.giphy.com/media/12NUbkX6p4xOO4/giphy.gif)

Hopefully you can now browse to the url for the service and see your brand new site! If I have skimmed over any steps or if anything isn't clear, hit me up on [Twitter](https://twitter.com/aphumphreys) and I'll clear things up!
