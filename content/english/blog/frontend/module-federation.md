---
title: "Micro-frontend using Module federation"
meta_title: ""
description: "Micro-frontend using module federation with vite and docker"
date: 2024-02-27T05:00:00Z
categories: ["frontend", "react", "vite"]
author: "Hugo Sjoberg"
tags: ["frontend", "react", "vite"]
draft: false
---

# Micro-frontend

Micro frontends is an architectural used to break down a big website into smaller frontends. Imagine you work for a big-ass company and you have one team responsible for the navbar, one team responsible for the footer etc.

{{< notice "warning" >}}
This is why you stay away from big companies!
{{< /notice >}}

It can also be useful if you have several static websites which you would like to run next to each other, sharing things such as login but one website is using react-12 and the other website is running the newest a shiniest version of react.

Let's see how we can do this using vite and deploy everything in a docker container(This kind of defeats the purpose of being able to deploy different components independently ¯\\_(ツ)_/¯ )

# Host

Let's start with the host-app, this will be the main app pulling in the other remote app(s).

```bash
npm create vite@latest
```

Replace the `vite.config.ts` with the following:

```ts
import { defineConfig, loadEnv } from 'vite';
import react from '@vitejs/plugin-react-swc';
import federation from '@originjs/vite-plugin-federation';

interface Imodules {
  remote: string;
}

export default ({ mode }) => {
  process.env = { ...process.env, ...loadEnv(mode, process.cwd()) };
  const modules: Imodules = {
    remote: '/remote/assets/remoteEntry.js',
  };
  if (mode === 'development') {
    modules.remote = `http://localhost:3002${modules.remote}`;
  }
  return defineConfig({
    plugins: [
      react(),
      federation({
        name: 'host-app',
        remotes: {
          remote: modules.remote,
        },
        shared: ['react'],
      }),
    ],
    build: {
      target: 'esnext',
    },
  });
};
```

The `modules.remote` will tell the host-app where to find the remote app.

In your `app.tsx` add the following line to import the remote-app:

```ts
const RemoteApp = lazy(() => import('remote/Remote'));
```

# Remote

In your remote app, update your `vite.config.ts` to look something like this:

```ts
import { defineConfig, loadEnv } from 'vite';

import federation from '@originjs/vite-plugin-federation';
import react from '@vitejs/plugin-react-swc';

export default ({ mode }) => {
  process.env = { ...process.env, ...loadEnv(mode, process.cwd()) };
  const base = process.env.DEV ? '' : '/remote';
  return defineConfig({
    plugins: [
      react(),
      federation({
        name: 'remote-app',
        filename: 'remoteEntry.js',
        // Modules to expose
        exposes: {
          './Remote': './src/app.tsx',
        },
        shared: ['react'],
      }),
    ],
    base: base,
    build: {
      target: 'esnext',
    },
  });
};
```

Here we specify what to expose (`app.tsx`)

# Run everything together

To run everything together I like to use docker, here is an example [Dockerfile](https://github.com/hugosjoberg/blog-code/blob/main/vite-module-federation/Dockerfile) I used.

In the Dockerfile I use nginx as a reverse-proxy to serve the static content, my `nginx.conf` file:

```conf
server {
  listen 3000;

  location / {
    root /usr/share/nginx/html/host;
    include /etc/nginx/mime.types;
    try_files $uri $uri/ /index.html;
  }

  location /remote/assets/ {
    alias /usr/share/nginx/html/remote/assets/;
    include /etc/nginx/mime.types;
    try_files $uri $uri/ /index.html;
  }
}
```

The `/` path is pointing to my host app and `/remote/assets/` is pointing to my remote app. When you hit `/` the host app will tell your browser to fetch data over at `/remote/assets/`.

And just like that we have created a trendy micro frontend.

# Code

All the code can be found [here](https://github.com/hugosjoberg/blog-code/tree/main/vite-module-federation)
