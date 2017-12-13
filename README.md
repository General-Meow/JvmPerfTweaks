# JvmPerfTweaks
Some notes on jvm performance tuning the VM, finding memory leaks and the garbage collector

Memory
- Memory is split into 2 main sections
  - The Stack
  - The Heap

Stack
- There can be many stacks in an app - 1 stack per thread - at least 1 stack (main) in an app
- data is pushed to the top of the stack, each time you define a variable in a method, its pushed on
- FILO structure
- the jvm know that when you exit a method, it can remove items from the top of the stack to a state before it called the method (popping out of scope vars)
- as each thread owns its own stack, it cannot see other variables defined in other threads

Heap
- can be thought of all the memory of the app minus the stack
- shared across all threads and methods
- ALL objects are stored on the heap (stack stores primitives and variable pointers/references to objects on the Heap)
- basically anything created with the new keyword will end up on the Heap

Passing variables by Value
- Java copies values when calling methods
- This is also true to variables of objects - the pointer/reference value is copied

References
- escaping references are when you attempt the encapsulate properties but inadvertently provide getters for them that provide
references to it. This essentially makes the property public as the client code using the class has a reference to the private property
and therefore can mutate it. i.e providing a getter of an internal List property which can then have its contents clear/modified
- to avoid this, you can always return a new copy of the property in getter methods
- a slight alternative but better way would be to send a copy of it in an immutable version. this is so that if you do attempt
to change the copy it will fail - instead of confusing the user
- the above works for collections but for basic class types, you create a COPY constructor that takes the same type of itself which essentially duplicates it

Garbage collection
- Strings are immutable and when creating identical strings, only one is created on the heap. all variables will therefore point to it (internalized strings)
- Strings are placed into a pool and the jvm will use it whenever it can but this will happen to simple strings. strings that are calculated somehow will not be placed in the pool
- You can force a calculated string to the pool by using the intern() method on it
- the JVM automatically calculates which objects are eligible for garbage collection. it does so by looking for heap objects that can no longer be reached by any variables on the stack
- the static gc method on System can be used to hint to the garbage collector to initiate garbage collection. but it is just a hint and doesn't force it.
- Runtime static method get runtime has a method freeMemory that returns the amount of free heap memory in bytes
- the finalize method on an object is called when the garbage collector removes it from the heap.
- its generally bad practice to use the finalize method as its never gaurenteed to run as the gc may not clean it up (program may exit)
- Soft leaks are leaks that happen because you do have a reference to data but dont use it
- Mark and sweep is the general algorithm used to identitfy objects that must be retained (ignored) so that the garbage collector will not free its memory
  - First - marking the objects to save by first doing a stop the world pause then the garbage collcector looks at all the variables on the stack and follows the reference to the object and each of the objects that it refers to and marks them.
  - Second - sweep then removes all other objects
  - Once the sweep has happened, objects that are saved get moved into a single continuous block of memory, this helps with fragmentation and future allocation of objects

Generational Garbage collection
- Most objects live for a short period of time
- Objects that survive GC then its more likely to live a longer time, perhaps forever
- GC is better if there are few object surviving as the stop the world scan will be quicker
- Generational Garbage collection is a way of organising the heap. Young generation and Old generation
- Young generation is typically smaller than old but this can be tweeked
  - Young generation is made up of 3 areas, eden, S0 and S1
  - new objects get created in eden, saved objects get moved into survivor space (S0,S1)
  - objects get gc'ed in survivor space and swapped between the 2 a number of times (8 times typically but depends on memory size)
  - then gets moved to old gen
- Objects are created in the Young generation
- When the young generation gets full or when the GC feels like it, the GC takes place only on the young generation
- As most are likely to be garbage, it will be fast, objects surviving the GC will then be placed in the Old generation
- At this point the young generation is empty and new objects will start to fill it up again
- This GC of young generation is known as 'MINOR' garbage collection (baby)
- 'MAJOR' collections are on the Old generation which only happen when required (full) and these take longer


Tips
- If you want to force an out of memory error, you could lower the amount of heap space when running your app with Xmx flag i.e. java -Xmx32m app.java
- use Jvisualvm to visualize memory usage of java apps
- install plugins for Jvisualvm (Visual GC) - this will display the different heap spaces
- The default memory usage graph (heap) is for the entire heap space - including oldgen, new gen, meta and pool space
- To analyse memory usage by objects, you can create a heap dump from JVisualVM, save it as a file and use eclipse Memory Analyser (MAT)
- In MAT load the saved file and use the 'Leaks suspect report' to get analysis of used memory


Java Runtime Params
- -Xmx set the max heap size e.g. -Xmx32m (default is usually 1/4 of the system memory)
- -Xms set the starting heap size e.g. -Xms32m
- starting and max heap sizes can be set to the same value - best to set the same for servers as resizing the heap can take time
- size of memory can be k/m/g - kilobyte/megabytes/gigabytes
- -XX:MaxPermSize=<size> - set the size of the PermGen e.g. -XX:MaxPermSize=200m
- -verbose:gc print to console when gc takes place
- -Xmn set the size of the young generation heap space - by default it is 1/3 of the entire heap space. Oracle recommends 1/4 to 1/2 of the heap size. e.g. -Xmn32m . this also sets the initial size so both max and intial need to be bigger than this
- -XX:HeapDumpOnOutOfMemory creates a heap dump file when the exception occurs
- -XX:HeapDumpPath=<PATH> place heap dump file in the defined PATH
- Types of collector
  - Serial - single threaded marker, good for Apps with small amount of data. or single core pcs
    - -XX:+UseSerialGC
  - Parallel - runs gc on young generation (minor) in parallel, better for larger data apps
    - -XX:+UseParallelGC
  - Mostly concurrent
    - best for shorter stop the world pause
    - 2 main ones
    - -XX:+UseConcMarkSweepGC
    - -XX:+UseG1GC
- For Java Mission Control, you need to enable some flags on the app being profiled
  - -XX:+UnlockCommercialFeatures -XX:+FlightRecorder
  - To be able to run JMC over JMX, you need jmxremote_optional.jar in the jre/lib/ext directory.
  - Download the jmxremote_optional.jar with "mvn -q dependency:copy -DoutputDirectory=. -Dartifact=org.jvnet.opendmk:jmxremote_optional:1.0_01-ea:jar" and copy it to the directory

Profiling tools
- Profiling tools can be run locally and remotely.
- When running remotely, the data can be streamed over RMI or MP (Messaging Protocal)
- When using MP, you must setup the jmxremote_optional jar and place it in the correct places




Java 6
- PermGen is another section of the heap that isn't young or old gen.
- Objects in PermGen will survive forever - never GC'd
- Internalized strings and class meta data live in PermGen
- PermGen exceptions normally mean that theres too many Internalized strings or classes, so you need to increase the space
- Apps depolyed on a server container will have new sets of class meta data in the PermGen space everytime you deploy (even if its the same app deployed)

Java 7
- Internalized strings are no longer stored in the PermGen space and can be GC'd

Java 8
- No more PermGen space
- Created MetaSpace which is where all the class meta data is stored, this lives in main (system) memory and not apart of the heap
- meta data that isn't used can be cleared
