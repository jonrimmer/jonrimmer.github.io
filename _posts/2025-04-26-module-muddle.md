---
title: Module Muddle
date: 2025-04-26 00:00:00
categories: [Code, JavaScript]
tags: [node, js, arghhh]
---

So lets say you're creating a `Sidebar` component and you put it in `/src/components/sidebar/index.tsx`. 
Then you want to use it somewhere, but you don't want some ugly relative path like `import Sidebar from '../../components/sidebar`.
You consider using old-school `@` path aliases, but they always felt kinda hacky, so you think instead you can use
[subpath imports](https://nodejs.org/api/packages.html#subpath-imports) in your `package.json`. Something like...

```json
{
  "imports": {
    "#*": "./src/*"
  }
}
```
{: file="package.json"}

So now you should be able to `import Sidebar from '#components/sidebar'`, right? Except now you get a TypeScript error:

```
Cannot find module '#components/sidebar' or its corresponding type declarations
```

What? You search a bit and you find some suggestions to add fallbacks to your `package.json` a la:

```json
{
  "imports": {
    "#*": [
      "./src/*.ts",
      "./src/*.tsx",
      "./src/*/index.ts",
      "./src/*/index.tsx"
    ]
  }
}
```

Except now TypeScript is happily resolving it, but Vite is erroring saying:

```
[plugin:vite:import-analysis] Failed to resolve import "#components/sidebar" from "src/App.tsx". Does the file exist?
```

What the hell? Finally you realise you need to keep the original subpath mapping, so you end up with:

```json
{
  "imports": {
    "#*": [
      "./src/*",
      "./src/*.ts",
      "./src/*.tsx",
      "./src/*/index.ts",
      "./src/*/index.tsx"
    ]
  }
}
```

And now Vite works and TypeScript is still happy. Although if you put that `"./src/*"` entry anywhere _else_ in the list of "fallbacks",
Vite stops working again.

Then a few weeks/month/years from now, TypeScript releases a new version and it breaks again (probably).

So what's happening? Well it seems like it goes back to Node's CommonJS relative module resolution algorithm, where you could specify `require('./foo');` and
it would resolve to `./foo.js` or to `./foo/index.js` if the first file didn't exist. Kinda neat, but later considered to be a mistake, because
ambiguity isn't fun and resolving it in a browser environment would require making multiple network requests, unless your server understands this
algorithm as well, but how do you know if it does?

Anyway the doyens of the JS ecosystem didn't like this kind of ambigous resolution and have been trying to nudge us all towards using fully 
specified paths like `import Sidebar from '#components/sidebar/index.tsx'` even if some people really don't like suddenly having to use
file extensions as part of "upgrading" from CJS. And you don't always have to use them, because some bundlers and tools, in some modes,
still allow extensionless imports. But it seems nothing fully agrees on exactly how things should work.

In this case Vite will resolve subpath imports without an extension, but TypeScript says "those arent relative paths, so they need to be fully specified"
and won't resolve `#components/sidebar` to `#components/sidebar/index.tsx`.

So you add fallbacks to map to `*/index.tsx`, but Vite (and Node.js) says "those are validation fallbacks, and the algorithm says a file not existing
isn't a validation failure, so I only need to look at the first one, and then ignore the rest, even if the first one isn't a file on disk, so the first
one better work". But TypeScript _does_ treat the file not existings as a reason to fallback. So you need the first entry to work for Vite, that's
the `"./src/*"`, and then you include your TS fallbacks.

Unless and until TypeScript realises their mistake and adjusts their resolution algorithm to treat validation fallbacks like Vite and Node.js does.

The upshot of all of this is, we probably need to accept our fate and just start using fully-specified paths everywhere. So, `import Sidebar from #components/sidebar/index.tsx` it is.