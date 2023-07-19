---
title: "Sveltekit API Route benchmark"
date: 2023-07-18T10:00:00+00:00
tags:
  - svelte
  - sveltekit
  - api
  - benchmark
---

## Sveltekit

[Sveltekit](https://kit.svelte.dev/) is a metaframework for the [svelte](https://svelte.dev/) javascript frontend framework.  

It is for svelte, what [next.js](https://nextjs.org/) is to [react](https://react.dev/).   

It is a great fit to run in a backend/frontend setup, but it can also serve as a fullstack solution.
In the cases where you need to expose an API, you can use [api routes](https://kit.svelte.dev/docs/routing#server).  

The question is, how does this API route perform, when compared to a dedicated API framework, like [fastify](https://fastify.dev/)?  

This benchmark is just meant to give a rough idea, comparing performance and DX, and is not scientific or trustable in any way.  

## Result  
Fastify has about 3x performance, compared to Sveltekit, but at 2k RPS SvelteKit might be more than performant enough.  

|Framework|RPS|
|:-:|:-:|
|Sveltekit|2000|
|Fastify|6000|

## Source  
To run this test yourself, you can clone the repo here:  
[https://github.com/askblaker/sveltekit-api-benchmark](https://github.com/askblaker/sveltekit-api-benchmark)