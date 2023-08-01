# Minimizing a Nuxt 3 Docker Image

You've just finished building an awesome Nuxt 3 application, but there's a catch – your Docker image is over 1 *Gigabyte* in size, your laptop is exploding trying to upload the image and deployment feels like a never-ending wait. Fear not, fellow Nuxt enthusiast! In this blog post, we'll dive into the art of minimizing Nuxt 3 Docker images, starting with a basic unoptimized image that is 1.4 Gigabyte large and ending with a image that's only 164 MB in size. That's a 850% reduction in size! Your boss and laptop will thank you :)

You can either follow along with your own Nuxt 3 project or just check out the final [GitHub repository](https://github.com/sliplane/minimizing-nuxt3-docker-image).

## The Basic Nuxt 3 Docker Image

While I don't know the exact Docker Image you are using, I bet it looks something like this:

```Dockerfile
# Use any Node.js base image that you want!
FROM node:18

# Set the working directory to /app
WORKDIR /app

# Copy the package.json file into the working directory before copying the rest of the files to cache the dependencies
COPY package.json /app

# Install the dependencies, you might want to use yarn or pnpm instead
RUN npm install

# Copy the rest of the files into the working directory
COPY . /app

# Build the application, again, use yarn or pnpm if you want
RUN npm run build

# Start the application. This is the default command for Nuxt 3
CMD ["node", ".output/server/index.mjs"] 
```

This Dockerfile is pretty straightforward. It starts with a Node.js 18 base image, copies the `package.json` file, installs the dependencies, copies the rest of the files, builds the application and finally starts the application. This is a great starting point, but it's not very optimized. This results in a 1.4 Gigabyte large Docker image. Let's see if we can do better!

## Step 0: Using an Alpine base image

The first thing we can do is use an Alpine base image instead of a Debian base image. This will reduce the size of our Docker image by about 1 Gigabyte!. This is a very easy change, since Alpine images are provided by Docker. All we need to do is change the base image from `node:18` to `node:18-alpine`:

```Dockerfile
# Use any Node.js base image that you want (as long as it's Alpine)!
FROM node:18-alpine

# Set the working directory to /app
WORKDIR /app

# Copy the package.json file into the working directory before copying the rest of the files to cache the dependencies
COPY package.json /app

# Install the dependencies, you might want to use yarn or pnpm instead
RUN npm install

# Copy the rest of the files into the working directory
COPY . /app

# Build the application, again, use yarn or pnpm if you want
RUN npm run build

# Start the application. This is the default command for Nuxt 3
CMD ["node", ".output/server/index.mjs"] 
```

This is of course a tradeoff, since Alpine images come with less preinstalled software and have a different package manager among other things. But if we are just building a Nuxt 3 application, this is a great way to reduce the size of our Docker image and the change will probably not affect us at all.


## Step 1: Ignoring unnecessary files

The next and probably easiest thing we can do is create a `.dockerignore` file and ignore all files that are not needed in the Docker image. This works similar to a `.gitignore` file you might already be familiar with. The `.dockerignore` file will simply tell Docker to ignore certain files and directories when building the image. This is highly dependent on your project, but here's an example of a `.dockerignore` file for a Nuxt 3 project:

```
node_modules
.output
.nuxt
.git
docs
```

This will ignore your installed `node_modules` folder, since we will install the dependencies in the Docker image. It will also ignore the `.output` and `.nuxt` folders, since we will build the application in the Docker image. Finally, it will ignore the `.git` folder and the `docs` folder, since we don't need them in the Docker image. Depending on your project, you might need to ignore more files and folders. For example, if you are using a CI/CD tool like GitHub Actions, you might want to ignore the `.github` folder. You get the idea. Go wild and try to ignore as many files and folders as possible. This will reduce the size of your Docker image and speed up the build process!


## Step 2: Using a multi-stage build

The next thing we can do is use a multi-stage build. This is a very powerful feature of Docker that allows us to build our application in one Docker image and then copy the built application into a second Docker image. This allows us to use a very large Docker image to build our application, but then use a very small Docker image to run our application. This is a great way to reduce the size of our Docker image, since we don't need all the build tools in our final Docker image. Let's see how this works in practice:


```Dockerfile
# Use a large Node.js base image to build the application and name it "build"
FROM node:18-alpine as build

# Exact same steps as before
WORKDIR /app

COPY package.json /app

RUN npm install

COPY . /app

RUN npm run build

# Create a new Docker image and name it "prod"
FROM node:18-alpine as prod

WORKDIR /app

# Copy the built application from the "build" image into the "prod" image
# This will only copy whatever is in the .output folder and ignore useless files like node_modules!
COPY --from=build /app/.output /app/.output

# Start is the same as before
CMD ["node", ".output/server/index.mjs"] 
```


## Step 3: Using a distroless image

Using a distroless image is another great way to reduce the size of your Docker image, but implies a few tradeoffs for relatively small gains (13 MB in our case). Distroless images are Docker images that don't contain any operating system packages. This means that you can't use any tools like `bash` or `curl` in your Docker image. For most applications, this is not a problem. The best way to find out if this works for you is to simply try it out. For that, you can just change the prodution image to a distroless image like this:

```Dockerfile
# Use a large Node.js base image to build the application and name it "build"
FROM node:18-alpine as build

WORKDIR /app

# Copy the package.json and package-lock.json files into the working directory before copying the rest of the files
# This will cache the dependencies and speed up subsequent builds if the dependencies don't change
COPY package*.json /app

# You might want to use yarn or pnpm instead
RUN npm install

COPY . /app

RUN npm run build

# Instead of using a node:18-alpine image, we are using a distroless image. These are provided by google: https://github.com/GoogleContainerTools/distroless
FROM gcr.io/distroless/nodejs:18 as prod

WORKDIR /app

# Copy the built application from the "build" image into the "prod" image
COPY --from=build /app/.output /app/.output

# Since this image only contains node.js, we do not need to specify the node command and simply pass the path to the index.mjs file!
CMD ["/app/.output/server/index.mjs"]
```

Note that we are using the `node:18-alpine` image for the build stage, since the `gcr.io/distroless/nodejs:18` image doesn't contain any build tools. 

## Step 4: Using a CDN for static assets

If you have a lot of static assets such as images or videos in your application, you can use a CDN to serve those assets. This will reduce the size of your Docker image, since you don't need to copy the static assets into the Docker image. Instead, you can just reference the static assets from the CDN. This doesn't just decrease the size of your Docker image, but also speeds up your application and reduces the load on your server! To serve static assets from a CDN, you will need to [configure your Nuxt App](https://nuxt.com/docs/api/composables/use-runtime-config#appcdnurl) and upload the static assets to the CDN. How you upload your static assets to the CDN depends on what CDN you are using and goes beyond the scope of this blog post. What you will need to do in any case, is remove the static assets from your Docker image. You can do that by only copying the server files into your final production image like this:


```Dockerfile
FROM node:18-alpine as build

WORKDIR /app

COPY package.json /app

RUN npm install

COPY . /app

RUN npm run build

FROM gcr.io/distroless/nodejs:18 as prod

WORKDIR /app

COPY --from=build /app/.output/server /app/.output/server

EXPOSE 3000/tcp

CMD ["/app/.output/server/index.mjs"]
```

## Final Result

Our final Docker image is only 164 MB in size, which is a 850% reduction! If you want to have a reproducible example, you can check out the [GitHub repository](https://github.com/sliplane/minimizing-nuxt3-docker-image).

![Final Result](/final-result.png)


## What's next?

At this point your Docker image should be less than 200 MB in size, which is pretty good! Of course, you can always go further and optimize your Docker image even more. But at some point you are going to hit diminishing returns. If you want to nerd out on Docker image optimizations, [check out our Blog Post](https://google.de) about the smallest possible Docker image that we use for rapid prototyping at [Sliplane](https://sliplane.io). It's less than 3 MB in size! If you are tired of optimizing Docker images and want to focus on shipping your product, [check out Sliplane](https://sliplane.io) to host your next project without giving up control over your own infrastructure!