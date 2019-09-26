---
layout: page
---


<p align="center">
  <img src="https://user-images.githubusercontent.com/55264/45711831-c87dbd80-bb58-11e8-9c67-84ba9427d60a.gif">
</p>

Tutorial
==========

This tutorial serves as a deeper-dive into Graphiti development,
building an Employee Directory application. We purposefully built this
to illustrate common - but non-trivial - scenarios present in many
applications.

A core concept of Graphiti is **Test-First** - the most pleasant way to
develop Graphiti is by starting with an [integration test]({{site.github.url}}/guides/concepts/testing). But that can add a lot of noise to a tutorial like this. Though we'll occasionally touch on testing - and the git diffs at the top of each section contain the necessary tests - we won't test first for the purposes of this tutorial.

<div class="clearfix">
<div markdown="1" class="tutorial col-md-6">

### Server Side: Rails

[Rails Sample Application](https://github.com/graphiti-api/employee_directory)

* [Step 0: Bootstrapping]({{site.github.url}}/tutorial/step_0)
* [Step 1: Initial Resource]({{site.github.url}}/tutorial/step_1)
* [Step 2: Has Many]({{site.github.url}}/tutorial/step_2)
* [Step 3: Belongs To]({{site.github.url}}/tutorial/step_3)
* [Step 4: Customizing Queries]({{site.github.url}}/tutorial/step_4)
* [Step 5: Has One]({{site.github.url}}/tutorial/step_5)
* [Step 6: Customizing Writes]({{site.github.url}}/tutorial/step_6)
* [Step 7: Many-to-Many]({{site.github.url}}/tutorial/step_7)
* [Step 8: Polymorphic
Relationships]({{site.github.url}}/tutorial/step_8)
* [Step 9: Polymorphic Resources]({{site.github.url}}/tutorial/step_9)

</div>

<div markdown="1" class="tutorial col-md-6">

### Client Side: VueJS (diff-only)

[VueJS Sample Application](https://github.com/graphiti-api/employee-directory-vue)


* [Step 0: Setup](https://github.com/graphiti-api/employee-directory-vue/commit/be690c3038380e17e326935d595a0b83fc8004f9)
  * Run after `vue create employee-directory-vue` using [Vue CLI](https://cli.vuejs.org).
* [Step 1: Define Models](https://github.com/graphiti-api/employee-directory-vue/compare/step_0_setup...step_1_models)
* [Step 2: Data Grid](https://github.com/graphiti-api/employee-directory-vue/compare/step_1_models...step_2_data_grid)
* [Step 3: Relationships](https://github.com/graphiti-api/employee-directory-vue/compare/step_2_data_grid...step_3_includes)
* [Step 4: Filtering](https://github.com/graphiti-api/employee-directory-vue/compare/step_3_includes...step_4_filtering)
* [Step 5: Sorting](https://github.com/graphiti-api/employee-directory-vue/compare/step_4_filtering...step_5_sorting)
* [Step 6: Total Count](https://github.com/graphiti-api/employee-directory-vue/compare/step_5_sorting...step_6_stats)
* [Step 7: Pagination](https://github.com/graphiti-api/employee-directory-vue/compare/step_6_stats...step_7_pagination)
* [Step 8: Basic Form Setup](https://github.com/graphiti-api/employee-directory-vue/compare/step_7_pagination...step_8_basic_form_setup)
* [Step 9: Dropdown](https://github.com/graphiti-api/employee-directory-vue/compare/step_8_basic_form_setup...step_9_dropdown)
* [Step 10: Nested Form Submission](https://github.com/graphiti-api/employee-directory-vue/compare/step_9_dropdown...step_10_nested_create)
* [Step 11: Validation Errors](https://github.com/graphiti-api/employee-directory-vue/compare/step_10_nested_create...step_11_validations)
* [Step 12: Nested Destroy](https://github.com/graphiti-api/employee-directory-vue/compare/step_11_validations...step_12_nested_destroy)
* [Step 13: Vue-Specific Glue Code](https://github.com/graphiti-api/employee-directory-vue/compare/step_12_nested_destroy...step_13_vue)

</div>
</div>

<br />
<br />
<br />
<br />
<br />
