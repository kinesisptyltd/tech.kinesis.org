---
layout: post
title: A sortable table component in Ember.js
tags: emberjs jsonapi
authors: chris justin
---

At Kinesis we deal with a lot of tabular data, and in the past have leaned on
[jQuery DataTables](https://www.datatables.net) heavily prior to using
Ember.js, so naturally when the need for a table in Ember.js came up we tried
using DataTables wrapped up in a Ember component.

Unfortunately, we were not able to get DataTables working with server side
pagination with Ember.js in a way that we were satisfied with.
