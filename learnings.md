- The @types/node package provides TypeScript type definitions for Node.js's built-in APIs, global variables, and native modules.
    - When you write TypeScript code that runs in Node.js, you frequently use built-in modules like fs (file system), path, http, or global variables like process.env and __dirname. Because Node.js is fundamentally written in JavaScript and C++, TypeScript has no native way of knowing what these modules are, what functions they contain, or what arguments those functions accept.


- Why should you pin dependency versions?
    - Pinning dependencies—specifying an exact, immutable version of a package (e.g., 1.4.2 instead of ^1.4.0)ensures that your software builds and behaves exactly the same way across every environment, from a local development machine to your production servers ie guarantee deterministic builds.