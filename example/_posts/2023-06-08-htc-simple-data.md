---
layout: post
title: >
   How to Code

# date: 2023-06-08 15:17:00 +0100
categories: [OSSU, Core Programming]
tags: [OSSU, BSL, HtC, course review, programming, data-driven , test-driven development, TDD, recursion]

toc: false
comments: false
math: false
mermaid: false
---

The gist:
* An exploration of functional programming with 'How to Code: Simple Data'
* Gaining command over data structures, types and the test-driven approach
* The importance of data definition, templates, and readable codes

**"How to Code: Simple Data"** serves as a competent introduction to **functional programming** and **coding at large**. This course harnesses the novice-friendly language of **Dr.Racket** to guide learners into the realm of **data-driven programming**. Initially, it **highlights the role of data structures and types** in designing **well-structured functions**.

A salient aspect of the course is its emphasis on **code readability and maintainability**. It underscores the importance of robust practices like **clear commenting and implementing signatures and stubs**.

What sets this course apart is its **test-driven approach**. By using **structured tests to form reliable functions**, it offers a scalability pathway and clarity on what to anticipate from a function.

The knowledge gained is applied in the **final project** - a replication of the 'Space Invaders' game. This was a **fun little project with challenges scattered here and there**. I enjoyed it, although BSL doesn't make coding very elegant. It is full of **parenthesis**, which made me quickly appreciate all the simplicity of **Python**. 

There were also some quirks that I did not enjoy. During my work, I tend to work in the **command line**, which was not possible with this language. I could code in the terminal (btw I use **VIM**) but I couldn't simply run the tests and see the rendering. This made me miss my home and appreciate my personal development environment. 

In conclusion, 'How to Code: Simple Data' ingrains a solid grasp of **Test-Driven Development** and fosters a **data-centric thought process**. Even if you're already adept at coding, the course provides insights into **good practices**. Particularly, **recursion** stood out, as it was a technique I had seldom used, coming from an object-driven language background.

## Complex Data

Recently, I embarked on an enlightening 6-week Python programming course that deepened my understanding of data manipulation and object-oriented programming concepts. I would love to share my learning experience with you, illustrating each week's module with an example involving a `Person` class. 

### Week 1: Mutual Reference 

We kicked off the course with the concept of mutual reference, where two or more data structures refer to each other, forming a cyclic relationship. In our case, a `Person` object might reference their parent and their children, illustrating the concept clearly.

```python
class Person:
    def __init__(self, name, parent=None, children=None):
        self.name = name
        self.parent = parent
        self.children = children if children is not None else []
```

### Week 2: Two One-Of Types 

The second week introduced us to the idea of handling two different types, Adult and Child. We created interactions between these two types, demonstrating their differing behaviors.

```python
class Adult:
    def __init__(self, name, children=None):
        self.name = name
        self.children = children if children is not None else []

class Child:
    def __init__(self, name, parent=None):
        self.name = name
        self.parent = parent

def greet(parent, child):
    print(f"{parent.name} the parent greets {child.name} the child.")
```

### Week 3: Abstraction

Abstraction, the heart of week three, taught us how to design more general and versatile functions. We developed an 'interact' function that could carry out any specified action between a parent and child.

```python
def interact(parent, child, action):
    action(parent, child)

def greet(parent, child):
    print(f"{parent.name} greets {child.name}.")
```

### Week 4: Generative Recursion

Week four led us into the fascinating world of generative recursion. Using this concept, we generated the renowned Sierpinski triangle, a classic example of a fractal.

```python
import turtle

def sierpinski(length, depth):
    # Base case: just draw a triangle
    # Recursive case: 3 smaller Sierpinski triangles
    # Note: this should be run in an environment supporting turtle graphics

turtle.speed(0)
sierpinski(200, 4)
turtle.done()
```

### Week 5: Accumulators

In the fifth week, we explored accumulators, variables that aggregate values over iterations. We implemented a function to count the number of a person's descendants.

```python
def count_descendants(person):
    count = len(person.children)
    for child in person.children:
        count += count_descendants(child)
    return count
```

### Week 6: Graphs

The final week exposed us to the concept of graphs. We modeled our family relationships as a graph, with each person as a node and parent-child relationships as edges.

```python
def find_ancestor(person, name):
    if person.name == name:
        return person
    if person.parent:
        return find_ancestor(person.parent, name)
    return None
```

This six-week Python course brought to life a multitude of concepts, from mutual references to generative recursion. Each week unveiled a new dimension of Python programming, underscoring the language's flexibility in modeling real-world situations. Such an enlightening journey underlines the beauty of learning and the power of Python.
