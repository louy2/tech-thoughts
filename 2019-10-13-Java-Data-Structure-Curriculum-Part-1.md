# On A New Java Data Structure Curriculum (Part 1)

My school teaches introductory computer science courses in Java. Among them is Data Strucuture. 
But I have observed that my schoolmates are not learning much from Data Structure.

They don't know which data structure to use. 
The professor teaches them how to implement different data strucutures with array.
Because they practice using array the most, they end up just using array.
Because they almost always use array, they lose the benefit of learning data structures:
Understanding the structure of a program with abstractions.

Granted, my school is not MIT and the CS 101 at my school does not use Structure and Interpretation of Computer Programs.
My school picks Java as the first language to learn, which is much more noisy than Lisp.
But I still think we can do better. I hope to contribute to this with my thoughts.

The current curriculum starts with the primitive types, then array, then class, then linked list, then tree, then graph.
It is mostly about implementing the data structures and their corresponding accessors and iteration methods.
Each data structure is treated as a separate concept, with few relationships among them.
As a result, my schoolmates are lost in the maze of implementation details, and lose the bird view.
Especially problematic is the insistance of the professors on explaining the data structures using the computer memory.
The structures appear to be composed of parts written in source code.
How they reside in the memory and how references refer to them are irrelevant to the structures themselves.
Worse, they confuse students such that they cannot focus on understanding the source code.

Another problem I see often is how my schoolmates are simply afraid of writing code.
They regard code as a foreign language, and they think the professor is fluent in it.
Therefore, when they are not sure how to write, they wait to ask the professor instead of exploring with the compiler.
I want to demystify the language of code. It is not a foreign language.
Donald Knuth envisioned Literary Programming. I think that is the correct approach to programming.
We know how to write articles. Programming should be just another way of writing an article.
`public static void main()` is the introductory paragraph that refers to the body paragraphs, 
which are other `public static void f()`s.
`class`es are nouns, and methods are verbs. Intuitively, parameters are objects.
`=` assigns a name to whatever is to the right, just like how we assign names to buildings and hurricanes.

Despite being introductory, understanding data structure is difficult.
Not everyone is ready to understand all of the curriculum within the semester.
Therefore I believe in empowering students to do as much as possible at each step.

The curriculum I want should hopefully do the following:

1. Start from intuitive use cases in real life, such as file organization or game development.
2. Show clearly various relationships each topic has with the previous ones.
3. Organize the assignments such that they compose a big project.
4. Teach basic project organization to hand off to Software Engineering.
5. Practice using official documentation to enable self-learning.
6. Pay much more attention to reading source code than to understanding computer memory.

The order in which I want to teach the various data structures:

0. Version control and real life data structures
1. String
2. Stream & File
3. Primitive types & java.time
4. Classes corresponding to the primitive types
5. List as persisted Stream
6. List as recursion
7. Tree
8. Graph as map
9. Project

Version control should be basic project management knowledge. 
Of course, we are using `git` and GitHub.
Showing off `git log` and `git diff` also shows us how data structures are used to solve daily problems.
Moreover, seeing `git diff` hopefully dissuades plagiarism.

String is the first type I want to introduce because it is both the most intuitive and the most empowering.
We deal with String all the time. This article is a String. The source code file itself is a String.
After learning about methods of String we can immediately apply them to our own work.
To the contrast, learning about primitive types only gives us an inferior calculator.
I also want to teach how to look up methods in the official Java documentation right away.
I would introduce it as part of a workflow: 
1) think about what to do; 2) Google and StackOverflow; 3) understand the solution with documentation.
Use the documentation as a dictionary, I'd say.
After this step, we can do ASCII arts and animations, ASCII game maps, basic data clean ups, etc.

Stream is introduced immediately after because it represents time. 
I want to elaborate the different models of time a program may contain.
For example, the standard input stream and the standard output stream. No more `Scanner`.
For example, the order of the statements executed. I show how to use the debugger here.
For example, how the order of lines read from a text file is converted to time and processed in time.
Moreover, methods of Stream are very intuitive verbs: `filter`, `findFirst`, `skip`, `sorted`, etc.
After all the talk about time, an obvious extension is to measure the time it takes to run the program.
Yes, I want to teach benchmarking. 
To do so, I introduce two ways to go through a Stream: for-each loop and Stream.forEach().
Then I show how to measure and compare how long it takes for them to finish running in the debugger.
