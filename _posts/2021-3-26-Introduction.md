---
layout: post
title: Introduction
---

## Who Am I?
My name is Quinn Okabayashi, and I hope to one day build tools to seamlessly integrate hardware and deep learning models. This is what I've been up to the past 6 years.

## Middle School
<figure>
    <p align="center">
        <img src="{{site.baseurl}}/images/introduction/qubits.jpg" alt="Lego Mindstorms robot" width="400"/>
        <figcaption>Qubits: Team Quantum's competition robot</figcaption>
    </p>
</figure>

My engineering journey began in middle school with LEGO Mindstorms, where my FIRST Lego League team, Team Quantum, was the [first team in my schools history to compete on the international stage](https://www.oregonlive.com/education/2015/01/two_middle_school_teams_one_fr.html). As the lead hardware engineer, I spent hours every day obsessing to find the most crazy-optimized mechanisms to complete the objectives. Whatever I ended up doing in life, I knew it would be making optimized systems.

## High School
I took Introduction to Computer Science with Python in my freshman year of high school, and Data Structures & Algorithms with Java the next year. However, the subject really clicked for me my senior year where I took Advanced Topics in Computer Sciences with Java. Projects included:
* [Decision Trees](https://github.com/QnnOkabayashi/Machine-Learning-3.0)
* [Neural Networks](https://github.com/QnnOkabayashi/Neural-Network)
* RSA Encryption
* Group Roster Optimization [[Presentation](https://docs.google.com/presentation/d/1_xkL0bVNTWQ10566lGNqzT6ibpshc-zhFau8AJnCDgo/edit?usp=sharing)]

The support I provided my peers on their own projects after finishing my own earned me an A+ in the course.

## Swarthmore College
In fall 2019, I took CPSC 31: Introduction to Computer Systems, where picked up C and Vim. Coming from Java, I loved the great power and responsibility that C provided, and the low-level assembly and knowledge of the exact steps the hardware takes fascinated me.

In the spring, I took CPSC 73: Programming Languages, instructed by [Zachary Palmer](https://www.cs.swarthmore.edu/~zpalmer/), where I gained exposure to functional programming through OCaml. For the first time since I started programming, I felt truly humbled: thinking from a functional perspective requires an entirely different skillset than thinking from a OOP perspective.

<figure>
    <p align="center">
        <img src="{{site.baseurl}}/images/introduction/computer_setup.jpg" alt="computer desk setup" width="400"/>
        <figcaption>My quarantine coding station</figcaption>
    </p>
</figure>

Around this time, my close friend [Avi Gupta](https://www.linkedin.com/in/avi-gupta/) inspired me to look into deep learning models, so I taught myself C++ and some CUDA in a 3-month long attempt to create a [convolutional neural network from scratch](https://github.com/QnnOkabayashi/ConvNet). I eventually abandoned the project because of my lack of CUDA, convnet theory, and C++ compilation experience.

Over the summer quarantine, Avi and I attempted to [create a generative adversarial network](https://github.com/avigupta33/gans_python) entirely from scratch. Being the systems programmer of the team, I created our budget NumPy clone: [Quantum Python](https://github.com/QnnOkabayashi/Quantum), a Python matrix library written in C and named after the robotics team that started it all for both of us. Unfortunately, the complexity of the algorithms and other life responsibilities forced us to suspend the project, but I still learned a lot about how Python works through the experience.

<figure>
    <p align="center">
        <img src="{{site.baseurl}}/images/introduction/rust.jpg" alt="Rust logo" width="120"/>
        <figcaption>Rust: Love at first sight</figcaption>
    </p>
</figure>

Soon after, my freshman roommate William Ball introduced me to the [Rust Programming Language](https://www.rust-lang.org/), where I was immediately enamored by the combination of low level memory control, high-level functional concepts, the cargo package and build manager, and my hatred for valgrind and `Makefile`s. As much as I wanted to love Rust, the learning curve was very steep and summer was coming to an end, so I put a hold on it for the fall semester.

<figure>
    <p align="center">
        <img src="{{site.baseurl}}/images/introduction/shine.svg" alt="SHINE Technologies logo" width="400"/>
        <figcaption>SHINE Technologies: My fall job</figcaption>
    </p>
</figure>

In fall 2020, I joined my friend [Solomon Olshin](https://www.solomonolshin.com/) to begin development on [ShineNet](https://github.com/QnnOkabayashi/ShineNet), a web interface to display live sensor readings for [JuiceBox 3.0](https://shinewithus.org/juicebox-3). I taught myself Arduino C++ and basic circuitry to connect sensors and flash storage to my micro controller, and basic HTML/CSS/JS for the front end. Although I don't intend to pursue web development, I'm grateful to have broadened my experiences and learn new technologies.

<figure>
    <p align="center">
        <img src="{{site.baseurl}}/images/introduction/planets.jpg" alt="Earth and Mars blended into one planet" width="400"/>
        <figcaption>Earth + Mars: A product of my Computer Vision class</figcaption>
    </p>
</figure>

Now I'm in the spring semester of my sophomore year, and my courses have strengthened my interest in ML in a systems-level context. In CPSC 72: Computer Vision, instructed by [Matt Zucker](https://mzucker.github.io/swarthmore/), we're learning about the underlying algorithmic implementations behind popular computer vision applications using Python/NumPy/OpenCV. In CPSC 75: Compilers, instructed by Zachary Palmer and using OCaml again, we're learning about compiler theory and optimization strategies, which fascinate me. These courses have provided invaluable exposure in what I believe to be the two fundemental aspects of machine learning: the algorithmic theory and the programmatic representation / execution of said algorithms.

In pursuit of these interests and learning Rust, I'm also currently working on a continuation of Quantum Python: [arrs](https://github.com/QnnOkabayashi/arrs), a multidimensional array library written entirely in Rust. Content for another post, though.

<br>
<br>

The common theme behind all these projects is that I hate black boxes: I want to know exactly how everything works. From assembly generation in compilers to ML theory, I want to understand how we can bridge complex ideas and hardware in perfect harmony.
