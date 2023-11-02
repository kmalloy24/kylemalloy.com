---
title: "Configure Vercel Speed Insights API with Sveltekit & Typescript"
description: "How to Configure the Vercel Speed Insights API with Sveltekit & Typescript"
slug: "vercel-speed-insights"
date: 2023-10-27
tags: ["web-dev", "vercel", "svelte", "typescript", "chatgpt"]
---

## Intro
Vercel Speed Insights is a great tool that can track performance of your website or application overtime, providing checkpoints for production builds, so you can more easily troubleshoot performance drops between releases. 

You can always run Lighthouse or [PageSpeed Insights](https://developers.google.com/speed/pagespeed/insights) to test your site but since it runs on your computer it might not reflect real-world actual user experience across browsers, devices, and in different locations. You also have to run the test manually and document them.

What makes Vercel Speed Insights so handy is that it collects vitals from your actual users and displays the data on a nice dashboard within your project. It also calculates an overall metric called the _Real Experience Score_, which is the combined score experienced by your visitors.

If you are already hosting your project on Vercel it makes it really nice to keep things within the same ecosystem so that you can take advantage of other useful features, like Vercel Analytics, [which can be setup with a few lines of code in Sveltekit.](/blog/vercel-analytics/)

If you want to use Speed Insights with a Sveltekit project you will need to use the [Speed Insights API](https://vercel.com/docs/speed-insights/api) as it only supports Next.js, Nuxt.js, and Gatsby with zero configuration. 

The [Vercel docs](https://vercel.com/docs/speed-insights/api#getting-started) are a little light on the topic and their [Sveltekit example](https://github.com/vercel/vercel/tree/main/examples/sveltekit) is done with Javascript. It took me a bit to put the pieces together, and adapt the example to my Typescipt project, so hopefully this step-by-step guide will provide some additional value.

[Ivo Berger](https://ivoberger.com/) wrote a [blog post](https://ivoberger.com/posts/using-vercel-analytics-with-svelte-kit) in 2021 that dives deeper into the API and is an excellent resource as well. 

## 1. Install Web Vitals
To start, make sure you have installed the [Chrome Web Vitals Package](https://www.npmjs.com/package/web-vitals) in your project. If not, run 👇

`npm i web-vitals`

{{< alert cardColor="#7A889D" iconColor="#AAFFBF" textColor="#f1faee">}}
This is a production dependency.
{{< /alert >}}

## 2. Update `vite.config.ts`

Sveltekit uses Vite as the build tool, so you'll have to define the analytics ID for your Vercel project in `vite.config.ts`. To do this add the highlighted code below.

> Note: The Vercel Analytics ID environment variable is inlined during the build process. It is not part of your project environment variables, and is accessed by [`process.env`](https://nodejs.org/dist/latest-v8.x/docs/api/process.html#process_process_env) inside Node.js.

{{< highlight go "linenos=table,hl_lines=9-11,linenostart=1" >}}
import { sveltekit } from '@sveltejs/kit/vite';
import { defineConfig } from 'vitest/config';

export default defineConfig({
	plugins: [sveltekit()],
	test: {
		include: ['src/**/*.{test,spec}.{js,ts}']
	},
	define: {
		'import.meta.env.VERCEL_ANALYTICS_ID': 
    JSON.stringify(process.env.VERCEL_ANALYTICS_ID)
	}
});
{{< / highlight>}}

In Vercel's JS Example the `vite.config.js` looks like this.

```ts
import { sveltekit } from '@sveltejs/kit/vite';

/** @type {import('vite').UserConfig} */
const config = {
  plugins: [sveltekit()],
  define: {
    'import.meta.env.VERCEL_ANALYTICS_ID': JSON.stringify(process.env.VERCEL_ANALYTICS_ID)
  }
};

export default config;
```

## 3. Create `vitals.ts`

Next, create a new file in `src>lib` called `vitals.ts`.

Looking at the [Javascript example](https://github.com/vercel/vercel/blob/main/examples/sveltekit/src/lib/vitals.js) from Vercel, and the [Typescript example](https://github.com/ivoberger/ivoberger.com/blob/b27ccad2b6d2e5a3108e50385ec52c3a745258c8/src/lib/webvitals.ts) from Ivo Berger, we see that they are fairly similar. Both will work, but both also throw a few Typescript errors. The errors won't stop the app from building and working correctly -- so for now let's copy Ivo's Typescript code and save the file. 

<!-- You can read more in depth on Typescript compiler errors and when/how to bypass them [here](/blog/ts-escape). -->

```ts
// vitals.ts

import type { Metric } from 'web-vitals';
import { getCLS, getFCP, getFID, getLCP, getTTFB } from 'web-vitals';

export type AnalyticsOptions = {
	params: Record<string, string>;
	path: string;
	analyticsId: string;
	debug?: true;
};

const vitalsUrl = '<https://vitals.vercel-analytics.com/v1/vitals>';

function getConnectionSpeed(): string {
	return 'connection' in navigator &&
		navigator['connection'] &&
		'effectiveType' in navigator['connection']
		? navigator['connection']['effectiveType']
		: '';
}

function sendToAnalytics(metric: Metric, options: AnalyticsOptions) {
	const page = Object.entries(options.params).reduce(
		(acc, [key, value]) => acc.replace(value, `[${key}]`),
		options.path
	);

	const body = {
		dsn: options.analyticsId,
		id: metric.id,
		page,
		href: location.href,
		event_name: metric.name,
		value: metric.value.toString(),
		speed: getConnectionSpeed()
	};

	if (options.debug) {
		console.log('[Analytics]', metric.name, JSON.stringify(body, null, 2));
	}

	const blob = new Blob([new URLSearchParams(body).toString()], {
		// This content type is necessary for `sendBeacon`:
		type: 'application/x-www-form-urlencoded'
	});
	if (navigator.sendBeacon) {
		navigator.sendBeacon(vitalsUrl, blob);
	} else
		fetch(vitalsUrl, {
			body: blob,
			method: 'POST',
			credentials: 'omit',
			keepalive: true
		});
}

export function webVitals(options: AnalyticsOptions): void {
	try {
		getFID((metric) => sendToAnalytics(metric, options));
		getTTFB((metric) => sendToAnalytics(metric, options));
		getLCP((metric) => sendToAnalytics(metric, options));
		getCLS((metric) => sendToAnalytics(metric, options));
		getFCP((metric) => sendToAnalytics(metric, options));
	} catch (err) {
		console.error('[Analytics]', err);
	}
}
```
## 4. Update `src/routes/+layout.svelte`
The last thing we'll need to do in the codebase is add the following imports into the `<scipt>` tag of the root layout.

```ts
    // Import SvelteKit module
    import { browser } from '$app/environment';
	// Import function from vitals.ts
	import { webVitals } from '$lib/vitals';
	// Retrieve Vercel analytics ID
	let analyticsId = import.meta.env.VERCEL_ANALYTICS_ID;
	// Sync up browser session & Vercel project
	$: if (browser && analyticsId) {
		webVitals({
			path: $page.url.pathname,
			params: $page.params,
			analyticsId
		});
	}
```

## 5. Deploy & Revist Your URL
After deploying the new code, and visiting your project link on the web (you may need to disable adblocker), you should be good to go! Navigate to the Speed Insights tab on your Vercel project dashboard to start using the tool.

>Note: Personal Accounts can enable Speed Insights on a single project at a time. To enable Speed Insights on multiple projects, transfer to a Team Account.
## 6. BONUS: Refactor `vitals.ts`
Now that we know the code works, lets deal with those pesky Typescript errors to help clean up the code, and enforce type safety. 

We are getting two TS errors within the `getConnectionSpeed` function for various reasons. 
>Line 2: Type 'unknown' is not assignable to type 'string'. ts(2322)
>
>Line 4: Type '{}' may represent a primitive value, which is not permitted as the right operand of the 'in' operator. ts(2638)

{{< highlight go "linenos=table,hl_lines=2 4,linenostart=1" >}}
function getConnectionSpeed(): string {
	return 'connection' in navigator &&
		navigator['connection'] &&
		'effectiveType' in navigator['connection']
		? navigator['connection']['effectiveType']
		: '';
}
{{< / highlight >}}

Let's feed the function and errors into ChatGPT and see if it can help us. You can see the chat [here](https://chat.openai.com/share/e095a5f4-ddb6-4227-abce-39c789b0cea0) which explains what is going on at each step.

We end up with the following Typescript `interface` and `function`.

```ts
interface NavigatorWithConnection extends Navigator {
    connection?: {
        effectiveType?: string;
    };
}

function getConnectionSpeed(): string {
    const nav: NavigatorWithConnection = navigator;

    if (nav.connection && nav.connection.effectiveType) {
        return nav.connection.effectiveType;
    }

    return '';
}
```

That's it! Substituting the code above will result in zero TS errors.

---

## Resources
- [Speed Insights API Reference](https://vercel.com/docs/speed-insights/api)
- [How to use Vercel Analytics with SvelteKit](https://ivoberger.com/posts/using-vercel-analytics-with-svelte-kit)
- [Chrome Web Vitals NPM](https://www.npmjs.com/package/web-vitals)
- [Refactor vitals.ts with ChatGPT](https://chat.openai.com/share/e095a5f4-ddb6-4227-abce-39c789b0cea0)
