---
title: "Fireship FKIT Course Notes"
description: "notes from fireship.io sveltekit course"
slug: "FKIT"
date: 2023-08-06
tags: ["programming", "svelte", "course-notes", "firebase", "tailwind"]
---
I recently completed the [Sveltekit Full Course](https://fireship.io/courses/sveltekit/) from [Fireship.io](https://fireship.io) where you build a [Linktree](https://linktr.ee) clone using [Sveltekit](https://kit.svelte.dev/), [Firebase](https://firebase.google.com/),[Tailwind](https://tailwindui.com/), and [Daisy UI](https://daisyui.com/). 

As someone new to web development in general, I ran into some problems along the way. It is an intermediate course aimed at those with some experience, but hopefully these notes can help other beginners who want to jump right in, but hit a wall and get discouraged.  

Big thanks to everyone in the {{< icon "discord" >}}
[Discord](https://discord.gg/SpDdJ3qaKK) who helped me out!

My demo is live [🌴 here](https://fkit-peach.vercel.app/) & you can find the source code on {{< icon "github" >}} [Github](https://github.com/kmalloy24/linktree-clone).

## 🔥 Let's Build

### 🔥 Firebase Client Setup
This thread in Discord helped me get through this step when I ran into package issues (an ongoing problem throughout the project with Firebase versions and dependencies). 

[Package vulnerabilities issues with SvelteKit Full Course - Firebase, Tailwind, & DaisyUI Setup](https://discord.com/channels/1015095797689360444/1129162283931684957)

### 🔥 Firebase Admin Setup
More help from Discord. 

[Module env/static/private has no exported member](https://discord.com/channels/1015095797689360444/1124021431047565402)

## 👤User Management

### 🎞️ Animated Routes
- Svelte component file names must start with capitalized letter
- Components must be imported to be called
```svelte 
  import AnimatedRoute from "$lib/components/AnimatedRoute.svelte";
```

### ⚡ Auth Store
- Need to add imports to top of `login/+page.svelte` and `firebase.ts` - see [course source code](https://github.com/fireship-io/fkit-course/blob/main/src/routes/login/%2Bpage.svelte)
- Could do userStore() as a component for larger apps rather than dumping it into `firebase.ts`

### ⚡ AuthCheck Component
- Import in `login/+layout.svelte`

## 🧑‍🎤 Advanced Features

### 🖼️ File Uploads
Add to storage rules on Firebase
```js
rules_version = '2';

// Craft rules based on data in your Firestore database
// allow write: if firestore.get(
//    /databases/(default)/documents/users/$(request.auth.uid)).data.isAdmin;
service firebase.storage {
  match /b/{bucket}/o {
    match /users/{userId}/{allPaths=**} {
        allow read;
      allow write: if userId == request.auth.uid;
    }
  }
}
```

## 🍪 Server Auth

### 💥 More Actions
Insert this into `src/routes/[username]/bio/+page.server.ts`

```typescript
export const load = (async ({ locals, params }) => {

    const uid = locals.userID;

    if (!uid) {
      throw redirect(301, "/login");
    }
  
    const userDoc = await adminDB.collection("users").doc(uid!).get();
    const { username, bio } = userDoc.data()!;
  
    if (params.username !== username) {
      throw error(401, "That username does not belong to you");
    }
  
  
    return {
      bio,
    };
}) satisfies PageServerLoad;
```
{{< alert >}}
If getting redirected to /login try clearing cookie data from browser history or open in different web browser.
{{< /alert >}}

## 🚀 Deployment
{{< alert >}}
On Windows you will need to have scripts enabled in your settings - privacy & security > for developers > powershell > toggle change execution policy on
{{< /alert >}}

I ran into a slew of issues with firebase hosting, finally got it to deploy by doing two things,

 1. Rolling back Firebase to 9.0.0, Node to 18.0.0, because of dependency issues with firebase-frameworks after running  `npm i -g firebase-tools` to then do `firebase experiments:enable webframeworks`.
	- Install [nvm](https://www.educative.io/answers/how-to-downgrade-node-version) if using Windows to roll back Node.
	- `npm i firebase@9.0.0` to rollback Firebase.
 2.  Upgrading the Firebase project to the Blaze plan to enable cloud functions (you would need significant traffic to actually be charged anything).

But then once the project successfully deployed to Firebase hosting the link would just load into an error...

I also tried `firebase experiments:disable webframeworks` and then use [svelte-adapter-firebase](https://github.com/jthegedus/svelte-adapter-firebase) but ran into more issues.

Finally I deployed through Vercel via the CLI and it worked perfectly. 
Follow [these instructions](https://vercel.com/docs/frameworks/sveltekit) to install the CLI, adapter, and then `svelte.config.js` should look like this:

```js
import adapter from '@sveltejs/adapter-vercel';
import { vitePreprocess } from '@sveltejs/kit/vite';

/** @type {import('@sveltejs/kit').Config} */

const config = {
    preprocess: vitePreprocess(),
    kit: {
        adapter: adapter(),
    }
};

export default config;
```

To deploy updates going forward simply run `vercel` from the CLI in the root of the project. 

A lot of people seemed to have issues deploying through Firebase Hosting so I suggest digging through the Discord if you run into issues. 

## 🎉 Go Further 
Keep the lesson going!
- Update the homepage 
	- Copy code from [course source code](https://github.com/fireship-io/fkit-course/blob/main/src/routes/login/%2Bpage.svelte)
	- Add gradient headings - [see my source code](https://github.com/kmalloy24/linktree-clone)
	- Change Daisy UI theme 
- Update static folder to change favicon, add link icons
- Finish the edit page, add a Daisy UI header to the layout
- Finish UX flow of login pages

To change the Daisy UI theme update `tailwind.config.js` then change `data-theme` in `app.html`.

```javascript
/** @type {import('tailwindcss').Config} */

module.exports = {
  content: ['./src/**/*.{html,js,svelte,ts}'],
  theme: {
    extend: {},
  },
  plugins: [require("daisyui")],  daisyui: {
    themes: ["light", "dark", "forest"],
  },
}
```