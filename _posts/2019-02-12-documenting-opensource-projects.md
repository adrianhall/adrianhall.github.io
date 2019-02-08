---
title: "Documenting open-source projects"
categories:
  - Development
---

Over the last 25 years, I've contributed to and written a lot of technical documentation, whether it is in the form of official documentation, blogs, tutorials, or books. Tech writers, who have nothing but access to engineers to work with, turn conversations into a guide that is equally suitable for a complete beginner and an expert. If you are an engineer, you definitely need to institute a "Tech Writer Appreciation Day".

That being said, learning research is rooted in psychology. The way we learn how to ski vs. how to learn a new programming language isn't so different. How we learn as developers should be an input to how you document your technology.

Documentation is a part of the product. It should be treated as such. Every time you do a release, your documentation should update to reflect it. You can have bugs and enhancements to documentation in just the same way as your framework, language, or library.
So how do I do it?

## Not everyone is at the same place

I categorize students into five groups.

![](assets/2019-02-11-image2.png)

I didn't come by this list by accident. A long time ago, I read a [research article written by Stuart and Hubert Dreyfus](https://apps.dtic.mil/dtic/tr/fulltext/u2/a084551.pdf) about how airforce pilots can learn a new skill. (You can read the short PDF for yourself). The paper itself is from 1980 and a really good read, but I treat it as source material and have adjusted my view over the years. Everyone goes through similar phases when learning something new. Instruction and practice - a lot of practice - are required to move along the curve towards mastery.

A beginner is context free. Sure, they may know that they want to learn more about a specific topic, but they don't necessarily know where to start. As they learn more and more about the subject, they transition from one phase of learning to another. A master has transcended the need for documentation because they know the subject matter so well. You are at different levels of learning on different technologies. 

As a documentation writer, your job is to move the student through the various phases to get to advanced or mastery. You won't have very many masters to write for, but they will be your most demanding users. At the other end of the scale, you will have a constant flow of new users transitioning from beginner to novice.

## The needs of each group are different

Let's take each group in turn.

### The beginner

A beginner doesn't know anything about the technology, but has a desire to learn. Your job as a tech writer is to transition them to a novice. You need to give them enough information for them to understand the technology and give them a desire to integrate that technology into their own projects. You cannot assume anything about their knowledge at this level. You cannot even assume that they have knowledge of the prerequisites for the technology. Example: if you are showing someone how to develop with React, you can't assume they have ever written a web app. A beginner is not willing to invest a huge amount of time learning your tech.

A **video introduction** and a **step by step tutorial** that just works are good options here. 

### The novice developer

The novice has very little context. They will understand that the technology might work in their project and they are willing to spend more time learning. Now, they want to integrate your technology into their own project. You cannot assume very much knowledge. The student must still be guided.

A **set of HOWTO instructions** for common scenarios is useful here.

The student can use these to integrate the common features into the project and get something working relatively quickly, as long as they are not straying too far from the happy path. However, they still don't fundamentally understand the technology. This is cut and paste land when developing software.

### The intermediate developer

At some point, the student will not be happy with cut and paste. It's likely going to be brought on by something that doesn't work quite right, or a desire to do something that is a little bit of an edge case. The student has reached the intermediate level. They are invested in the technology at this point and they want to learn more.

**Conceptual documentation** with deep technical knowledge is required.

Armed with this knowledge, the student will forge ahead with new projects and stretch the limits of the software, wanting ever more complex scenarios to be supported. 

### The advanced developer

As soon as they have digested the concepts, really understood them, and understand the design decisions that have gone into the technology, they have moved into the advanced stage. These are the developers that submit the perplexing bugs and continually pushing the boundaries of the technology.

Advanced users don't need "documentation". They need **samples** and **safe places to get implementation advice** from engineers, like hackathons or Stack Overflow.

I contend that samples are still a part of the documentation.

### The master developer

How does one get to the master stage? Mastery is elusive; the pinnacle of knowledge. It's unlikely you will get there unless you work on the team. You submit pull requests, suggest ways to fix bugs, share your knowledge, and contribute to the community. You are the go-to person, sought after for conference talks, and influential both inside and outside the team developing the technology.

So, what does a master require? Nothing more than excellent and **comprehensive API documentation**.

## Best practices for documenting your open source project

Learning is not a destination. It is a journey. Documentation is a guide on that journey; detailed instructions to get your started, but enough landmarks to allow you to wander. Where you are in that journey provides insight into what sort of documentation you need.
When writing, make sure you know who you are targeting. Don't mix and match information for different levels of learner in the same document. It will frustrate some and confuse others. 

When working on the information architecture for your docs, make sure you keep the learning path in mind . The documentation should flow naturally in the same way that learning progresses.

Start by addressing the beginner. If developers can't start, they will never progress.

Make sure you test your documentation with someone who is at the same level of learning as your audience. If you are advanced, you cannot judge the suitability of beginner documentation.

## Want to contribute?

Be a hero. Nothing has more impact than improving the documentation. Most open source projects put their documentation right alongside the framework or library. Just started? Write the tutorial that you wish you had. More advanced? Write about the concepts that you wish you had been told. 

Thinking of writing a blog? Before you do, see if the project would rather have that information in the docs. Just about every open source project accepts pull requests for their documentation. Your contribution will have more impact, be seen by more people, and help more people than that YouTube video or blog post.

Wondering where to get started? Check out the issues for the project, file issues for things you want to know, and talk to the maintainers. Trust me - they will appreciate the help. So will everyone else who comes after you.
