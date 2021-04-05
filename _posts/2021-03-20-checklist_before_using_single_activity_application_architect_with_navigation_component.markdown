---
layout: single
classes: wide
title: "Checklist before using Single activity application architecture with Navigation component"
categories: Android
date: 2021-03-20 22:51:00 +0900
tags:
  - Android
  - Single Activity Application
  - Navigation
  - Fragment
  - State management
---

Recently, Lots of developers start to try using SAA(Single Activity Application) with Navigation components. Surely there are lots of benefits to Single Activity Application, But you might need to know about these factors before boosting your project. 

## TL;DR

* ViewPager2 doesn't keep its state with the recycler view adapter. 
* Dialog cannot be stacked with a fragment.
* Leak! Leak!

## ViewPager2 doesn't keep its state with the recycler view adapter

As you can see from [the android developers' reference](https://developer.android.com/guide/fragments/saving-state), Android system operations ensure the user's state is saved unless you omit to give an id for the view.
 
![Image1](/assets/images/2021-03-20-checklist_before_using_single_activity_application_architect_with_navigation_component-0.png) 

![Image2](/assets/images/2021-03-20-checklist_before_using_single_activity_application_architect_with_navigation_component-1.png) 

For keeping the state with view state, however, **You should use FragmentStateAdapter.**

RecyclerView Adapter, which is generally use.



