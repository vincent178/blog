title: react-native-performance
date: 2015-12-09 12:05:09
tags:
---


For most React Native applications, your business logic will run on the JavaScript thread. 
Updates to native-backed views are batched and sent over to the native side at the end of each iteration of the event loop, before the frame deadline
