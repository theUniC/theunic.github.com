---
layout: post
title: "Why you shoud not be using annotations"
date: 2013-07-25 21:03
comments: true
categories: [development]
---

In fact. **Annotations** are evil. If you're using it, you're doing it wrong. But why? Why are they so evil? Next, I'm going to present a real-world use-case in which the use of annotations
lead to a serious coupling situation.

## The use-case ##

Supose you have an application in which you want to integarate an ORM with annotated support to define your domain objects, in order to keep under control the growing complex business logic. You
look at the resulting code, and ... Wow! Have you ever seen such a clean code?

```php
<?php

namespace Acme\Shop\Entity;

use Doctrine\ORM\Mapping as ORM;

/**
 * @ORM\Entity
 */
class Product
{
  /**
   * @ORM\Id
   */
  private $id;
  
  /**
   * @ORM\Column(type="string")
   */
  private $title;
  
  /**
   * @ORM\Column(type="decimal")
   */
  private $price;
}

```

## Coupling, coupling and more coupling ##

This piece of code flaws of a serious coupling problem. **YOU ARE ACTUALLY COUPLING YOUR DOMAIN OBJECT TO ITS OWN CONFIGURATION**. A statement of **domain-driven design** that I
really like is that "_Separating the domain layer from the infrastructure and user interface layers allows much cleaner design of each layer. Isolated layers are much less expensive
to mantain, because they tend to evolve at different rates and respond to different needs._"

But, how could it be? It's only a plain PHP class with no defined behaviour and a few comments, it can be changed whenever it's needed.

OK! Supose that there's a need to reuse this entity in another persistence engine. How could it be done with annotations? It cannot be done, because I cannot define the same annotation
twice without overriding the previous one! 

The simplest example could be different table names between engines that share common entities. How could it be mapped without breaking each other? Or the use of sequences in Oracle/Postgres 
... So, indeed, this entity is coupled with the persistence engine.

The solution would be to move all the persistence mapping (aka entity metadata) to another format like YAML, XML or PHP (no annotated code). Doing it this way several mapping formats
could be used and share common entities between each of them.

## And with couping comes the rest ... ##

In case we need to perform unit testing or to do some kind of runtime substitution, the entity is absolutely contextualized and the dependencies cannot be changed at runtime. It's actually
tied to an implementation. So there is no chance to substitute any of the dependencies.

## Conclusion ##

At a first glance annotations could seem a big chance to keep your code simple and clean. But It actually leads to a coupling problem and serious design flaws. If you don't mind about that
coupling level, then probably annotations are for you.