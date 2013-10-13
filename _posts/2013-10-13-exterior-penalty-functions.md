---
layout: post
title: Exterior penalty methods (constrained nonlinear programming)
---

# {{ page.title }}

Recently, my PhD research has taken me on an exciting journey through the world of nonlinear programming. Nonlinear programming is a subfield of mathematical optimisation that deals with optimisation problems which are nonlinear in nature. For example, minimising a quadratic function of one real variable is considered as nonlinear since the quadratic function itself is nonlinear.

In this blog post, I thought I'm going to share my thoughts and recent discoveries when it comes to solving constrained nonlinear problems using unconstrained programming. In particular, I want to demonstrate a class of methods falling under the name *exterior penalty methods*.

## The problem

